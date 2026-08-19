# PATH W: the same spoke, with zero secrets on the hub

**What this covers / what you need before starting.** This doc reruns the ARO-HCP spoke infrastructure provisioning from [04-hcp-spoke-sp-path.md](04-hcp-spoke-sp-path.md) — same 13 managed identities, same 28 role assignments, same resource group in real Azure — but with **no client secret anywhere on the hub**. This is the doc this repo exists for: Marek Veber's `cluster-api-installer` has a workload-identity mode in `gen.sh`, but it assumes two pre-existing identities from his personal environment, and the repo contains no federated-credential creation anywhere (`grep -rn federated-credential` across the clone at `ed240ab` returns zero hits — we checked). The five setup steps below are the missing pages. You need: the MCE 2.11.4 hub from [03-hub-mce-setup.md](03-hub-mce-setup.md) with the CAPZ/ASO controllers running, `oc` logged in as cluster-admin, `az` CLI 2.84.0 logged into your subscription with rights to create identities and role assignments (see the role-assignment trap below), `jq`, and a clone of `marek-veber/cluster-api-installer` at branch `capi-test-rebase` (we ran `ed240ab`). If Azure identity words like "federated identity credential" are new to you, read [01-identity-models-explained.md](01-identity-models-explained.md) first. Everything here ran for real on 2026-08-18; observed outputs are quoted from [`../evidence/`](../evidence/), and the full gated runbook is in [the master plan-and-results doc](../reference/full-plan-and-results.md).

## Caveats first

- **Dev preview, unsupported.** Same status as everything riding on `cluster-api-provider-azure-preview`: no published Red Hat procedure exists. This doc is a prove-out record, not a supported runbook.
- **The headline result here was unverified before this run.** MCE ships a stolostron *fork* of Azure Service Operator (ASO). Whether that fork kept upstream's secretless per-namespace credential fallback was an open question with no documentation either way. It does — proven below. But you're reading evidence from one run of one build (`azure-service-operator-rhel9`, MCE 2.11.4), not a support statement.
- **Your hub needs a public OIDC issuer.** If `oc get authentication cluster` returns `https://kubernetes.default.svc`, stop at step 2 — this path is not available to you without standing up a public issuer first (out of scope here; see step 2 for the pointer).
- **You need `Microsoft.Authorization/roleAssignments/write`** to grant the roles in step 3. Plain Contributor on the subscription can't do it. Owner, User Access Administrator, or Role Based Access Control Administrator can.
- **This doc provisions spoke *infrastructure* only** — the same `is.yaml` boundary as PATH S. The final `HcpOpenShiftCluster` create is behind the approval-gated `HcpPrivatePreview` feature flag; in our subscription the PUT came back `404 InvalidResourceType`. See [the master doc](../reference/full-plan-and-results.md) for that boundary in full.
- **Never hand-patch the MCE-managed deployments or their Secrets.** The backplane operator force-reapplies them on every reconcile. The pattern in this doc never needs to touch them — that's the point. Details at the bottom.

## The idea in three sentences

The CAPZ and ASO controller pods on your hub **already** mount projected service-account tokens with audience `api://AzureADTokenExchange` — MCE shipped them that way, no patching needed (we verified this in [03-hub-mce-setup.md](03-hub-mce-setup.md)). We create two Azure user-assigned managed identities and tell Entra to trust the hub's OIDC issuer for exactly two service accounts — so Azure accepts those pods' tokens the same way your API server does, because we pointed Entra at the same issuer. Then the credential Secrets on the hub simply omit the client secret, and everything else — templates, apply, convergence, the resulting Azure resources — is identical to PATH S.

That's it. No secret is minted, stored, rotated, or leaked, because no secret exists.

## What we're building

Two hub-side identities, two trust records, three role assignments, and two Kubernetes objects:

| Piece | Name in our run | What it's for |
|---|---|---|
| UAMI #1 | `capz-hub-mi` | The Azure identity CAPZ acts as |
| UAMI #2 | `aso-hub-mi` | The Azure identity ASO acts as |
| FIC on #1 | `capz-manager-fic` | Entra trusts the hub issuer for SA `capz-manager` |
| FIC on #2 | `aso-fic` | Entra trusts the hub issuer for SA `azureserviceoperator-default` |
| Roles | Contributor ×2, RBAC Administrator ×1 | What the identities may do, subscription scope |
| `AzureClusterIdentity` | `type: WorkloadIdentity` | Tells CAPZ which identity to be |
| `aso-credential` Secret | 3 keys, no secret key | Tells ASO which identity to be |

A **user-assigned managed identity** (UAMI) is an Azure identity object with no password at all — Azure manages its certificate internally and rolls it automatically. A **federated identity credential** (FIC) is a record on a UAMI that says "if a token arrives signed by *this* OIDC issuer, with *this* subject and *this* audience, treat the bearer as me." Together they replace the service principal and its expiring secret from PATH S.

Set up your environment for the steps below:

```bash
export SUBSCRIPTION_ID=<your-subscription-guid>
export RESOURCEGROUP=<rg-where-hub-identities-live>   # ours: the sandbox RG openenv-ftcqh
export LOCATION=eastus
export UAMI_CAPZ=capz-hub-mi
export UAMI_ASO=aso-hub-mi
export NS_WI=aro-clusters-wi          # hub namespace for this spoke's CRs — see the namespace rule below
export REPO=<path-to-cluster-api-installer-clone>
export GEN_WI_DIR=<output-dir-for-generated-manifests>
```

## Step 1 — create the two hub identities

Create one UAMI per controller. Two, not one: CAPZ and ASO get separate identities so they can carry different privilege (step 3 gives only ASO the role-assignment power).

```bash
az identity create -g "$RESOURCEGROUP" -n "$UAMI_CAPZ" -o none
az identity create -g "$RESOURCEGROUP" -n "$UAMI_ASO" -o none
CAPZ_PID=$(az identity show -g "$RESOURCEGROUP" -n "$UAMI_CAPZ" --query principalId -o tsv)
ASO_PID=$(az identity show -g "$RESOURCEGROUP" -n "$UAMI_ASO" --query principalId -o tsv)
```

You should see nothing from the creates and two GUIDs in the variables. In our run `capz-hub-mi`'s principal came back `4c9f0e88-951b-406f-8625-fff103104fba` and `aso-hub-mi`'s `4bf2c774-a39f-492b-a448-15f25830e337` ([`track2-h5-hub-uami-roles.json`](../evidence/track2-h5-hub-uami-roles.json)). Keep both principal IDs — steps 3 needs them, and a freshly created principal can take a minute to propagate through Entra (if a later command says `PrincipalNotFound`, retry it at ~30 s intervals, up to 10 times, before treating it as real).

## Step 2 — read the hub issuer and GATE on it

This is the step that decides whether PATH W is available to you at all. Read the issuer your hub's API server stamps into every service-account token:

```bash
ISSUER=$(oc get authentication cluster -o jsonpath='{.spec.serviceAccountIssuer}')
echo "ISSUER=$ISSUER"
```

You should see a public https URL. In our run ([`track2-h5-issuer.txt`](../evidence/track2-h5-issuer.txt)):

```
ISSUER=https://eastus.oic.aro.azure.com/64dc69e4-d083-49fc-9569-ebece1dd1408/3daaa256-9628-4bf6-a438-988f1b34ed3f
```

That's the `https://eastus.oic.aro.azure.com/<tenant>/<uuid>` format you get for free on a managed-identity classic ARO cluster — one of the reasons we used one as the hub.

Now the gate. Entra has to be able to fetch this issuer's discovery document and signing keys over the public internet, and FIC matching is **exact-string** — so we verify both reachability and the exact issuer value before writing it into any trust record:

```bash
curl -sf "${ISSUER%/}/.well-known/openid-configuration" | jq '{issuer, jwks_uri}'
```

You should see a discovery document whose `issuer` field matches `$ISSUER` **byte-for-byte** — including any trailing slash. In our run this returned:

```json
{
  "issuer": "https://eastus.oic.aro.azure.com/64dc69e4-d083-49fc-9569-ebece1dd1408/3daaa256-9628-4bf6-a438-988f1b34ed3f",
  "jwks_uri": "https://eastus.oic.aro.azure.com/64dc69e4-d083-49fc-9569-ebece1dd1408/3daaa256-9628-4bf6-a438-988f1b34ed3f/openid/v1/jwks"
}
```

Use the `oc`-reported value verbatim in step 4. If the discovery `issuer` and the `oc` value differ by even one character, the token exchange will fail later with an error that doesn't mention this step (see the debug map).

**If your issuer is `https://kubernetes.default.svc`:** your hub uses the default in-cluster issuer, Entra can't reach it, and PATH W stops here for you. Standing up a public issuer (storage-account JWKS hosting, `serviceAccountIssuer` surgery, a 24-hour token-trust window during the switch) is documented in the gap list inside [the master plan-and-results doc](../reference/full-plan-and-results.md) but was deliberately out of scope for this run — our hub came with a public issuer built in.

## Step 3 — grant the Azure roles (subscription scope, and not Owner)

Both identities get **Contributor** — the standard "can create and manage resources" role. The ASO identity additionally gets **Role Based Access Control Administrator**. Here's why, because both halves of that choice matter:

- **Why more than Contributor at all:** the spoke templates embed 28 `Microsoft.Authorization/roleAssignments` resources (the per-operator grants for the spoke's 13 UAMIs), and Contributor explicitly lacks `roleAssignments/write`. ASO is the controller that reconciles those 28, so ASO's identity needs the power. Miss this and every RoleAssignment CR sits NotReady on `403 AuthorizationFailed` while everything else converges.
- **Why not Owner:** Role Based Access Control Administrator grants *only* role-assignment read/write/delete — nothing else. Owner grants everything. Same outcome for our 28 assignments, materially smaller blast radius. (Production hardening we didn't exercise: an ABAC condition restricting which role definitions it may assign.)
- **Why subscription scope, not resource-group scope:** ASO creates **new resource groups** for each spoke (`spk2-<guid>-resgroup` in our run). You can't scope a grant to a resource group that doesn't exist yet, so the grant has to sit one level up.

Assign Contributor to both identities and RBAC Administrator to the ASO identity, by role GUID (GUIDs are stable across tenants; display names can be shadowed):

```bash
ROLE_CONTRIB=b24988ac-6180-42a0-ab88-20f7382dd24c      # Contributor (built-in)
ROLE_RBAC_ADMIN=f58310d9-a9f6-439a-9e8d-f62e7b41a168   # Role Based Access Control Administrator (built-in)
SCOPE_SUB="/subscriptions/$SUBSCRIPTION_ID"
for PID in "$CAPZ_PID" "$ASO_PID"; do
  az role assignment create --assignee-object-id "$PID" --assignee-principal-type ServicePrincipal \
    --role "$SCOPE_SUB/providers/Microsoft.Authorization/roleDefinitions/$ROLE_CONTRIB" --scope "$SCOPE_SUB" -o none
done
az role assignment create --assignee-object-id "$ASO_PID" --assignee-principal-type ServicePrincipal \
  --role "$SCOPE_SUB/providers/Microsoft.Authorization/roleDefinitions/$ROLE_RBAC_ADMIN" --scope "$SCOPE_SUB" -o none
```

You should see silent success three times (the propagation-retry rule from step 1 applies to the first assignment). Verify with a list:

```bash
az role assignment list --all \
  --query "[?principalId=='$CAPZ_PID' || principalId=='$ASO_PID'].{p:principalId,role:roleDefinitionName,scope:scope}" -o json
```

You should see exactly three rows, all at subscription scope. In our run ([`track2-h5-hub-uami-roles.json`](../evidence/track2-h5-hub-uami-roles.json)): Contributor on `4c9f0e88-…` (capz-hub-mi), Contributor on `4bf2c774-…` (aso-hub-mi), and `Role Based Access Control Administrator` on `4bf2c774-…` — all with scope `/subscriptions/8e15b613-d1f9-41a6-a23d-e8b3ce94d6fe`.

## Step 4 — create the two federated identity credentials

First, a gate: **don't assume the service-account names.** The FIC subject must match `system:serviceaccount:<namespace>:<name>` exactly, so read the live names off the deployments rather than trusting this doc:

```bash
oc get deploy capz-controller-manager -n multicluster-engine \
  -o jsonpath='{.spec.template.spec.serviceAccountName}{"\n"}'
oc get deploy azureserviceoperator-controller-manager -n multicluster-engine \
  -o jsonpath='{.spec.template.spec.serviceAccountName}{"\n"}'
```

You should see one name each. In our run ([`track2-h5-sa-names.txt`](../evidence/track2-h5-sa-names.txt)): `capz-manager` and `azureserviceoperator-default`. If yours differ, substitute the recorded values in the subjects below.

Now create the two FICs — **sequentially**, not in parallel (concurrent FIC writes against one identity return 409; the limit is 20 FICs per identity and we use one). Each FIC is the trust record: issuer, subject, audience — the three fields Entra checks against the pod's token.

```bash
az identity federated-credential create --name capz-manager-fic \
  --identity-name "$UAMI_CAPZ" --resource-group "$RESOURCEGROUP" \
  --issuer "$ISSUER" --subject "system:serviceaccount:multicluster-engine:capz-manager" \
  --audiences api://AzureADTokenExchange -o none
az identity federated-credential create --name aso-fic \
  --identity-name "$UAMI_ASO" --resource-group "$RESOURCEGROUP" \
  --issuer "$ISSUER" --subject "system:serviceaccount:multicluster-engine:azureserviceoperator-default" \
  --audiences api://AzureADTokenExchange -o none
```

You should see silent success twice. Verify both:

```bash
az identity federated-credential list --identity-name "$UAMI_CAPZ" -g "$RESOURCEGROUP" -o json
az identity federated-credential list --identity-name "$UAMI_ASO"  -g "$RESOURCEGROUP" -o json
```

You should see one FIC per identity, issuer equal to `$ISSUER` exactly, audiences `["api://AzureADTokenExchange"]`. In our run ([`track2-h5-fic-capz.json`](../evidence/track2-h5-fic-capz.json)) the CAPZ one read:

```json
{
  "audiences": ["api://AzureADTokenExchange"],
  "issuer": "https://eastus.oic.aro.azure.com/64dc69e4-d083-49fc-9569-ebece1dd1408/3daaa256-9628-4bf6-a438-988f1b34ed3f",
  "name": "capz-manager-fic",
  "subject": "system:serviceaccount:multicluster-engine:capz-manager"
}
```

and [`track2-h5-fic-aso.json`](../evidence/track2-h5-fic-aso.json) the same shape with subject `system:serviceaccount:multicluster-engine:azureserviceoperator-default`.

The audience deserves one sentence: `api://AzureADTokenExchange` is the fixed audience Entra expects on tokens presented for federation, and it's the audience MCE already projects into both controller pods (CAPZ at `/var/run/secrets/azure/tokens`, ASO at `/var/run/secrets/tokens` — verified in [03-hub-mce-setup.md](03-hub-mce-setup.md)). Three things now agree — the projected token, the FIC, the issuer — and that agreement *is* the authentication.

## Step 5 — render the secretless credentials with gen.sh's own WI mode

Marek's `gen.sh` has a workload-identity branch, and we use it deliberately rather than hand-writing the two Kubernetes objects: it reads both UAMIs' client and tenant IDs via `az identity show` itself and renders `credentials-wi-template.yaml`, so the artifacts come out of the same verified tool as PATH S's.

Two traps before the command block:

- **The trigger variable is misspelled in the repo: `OICD_RESOURCE_GROUP`.** That's `OICD`, not `OIDC`. It's required verbatim — export the correctly-spelled name and the script silently takes the service-principal branch instead.
- **WI mode needs non-CI mode, or the rendered credential comes out empty.** The template choice itself hangs only on `OICD_RESOURCE_GROUP` — set it and you get the WI template regardless of anything else. But the `az identity show` calls that resolve the two UAMIs' client IDs into that template only run in non-CI mode; CI mode (all six of `REGION DEPLOYMENT_ENV AZURE_SUBSCRIPTION_ID AZURE_TENANT_ID AZURE_CLIENT_ID AZURE_CLIENT_SECRET` set) skips them and renders the WI template with empty identity fields. So `AZURE_CLIENT_SECRET` and `DEPLOYMENT_ENV` must be **unset** to break CI mode — which also keeps the run secretless, fittingly.

The namespace rule from PATH S applies unchanged: ASO resolves credentials by looking for a Secret literally named `aso-credential` **in the namespace of each resource it reconciles**, so the credentials and all the spoke's CRs must land in one namespace. `gen.sh` bakes `$NAMESPACE` into every manifest; we pin it to `$NS_WI` so nothing falls into `default`.

```bash
mkdir -p "$GEN_WI_DIR"
export ENV=stage                                # inert here: AZURE_SUBSCRIPTION_NAME below overrides its default
export AZURE_SUBSCRIPTION_NAME="$SUBSCRIPTION_ID"   # az accepts the id; gen.sh resolves it via az account show
export OICD_RESOURCE_GROUP="$RESOURCEGROUP"     # the misspelling is the trigger — where the two hub UAMIs live
export USER_ASSIGNED_IDENTITY_ASO="$UAMI_ASO"
export USER_ASSIGNED_IDENTITY_ARO="$UAMI_CAPZ"  # gen.sh's "ARO" identity feeds the CAPZ AzureClusterIdentity
export REGION="$LOCATION" NAMESPACE="$NS_WI" USER=hcp CS_CLUSTER_NAME=spk2-<yourid> OCP_VERSION=4.20
export GEN_ASO=true USE_EA=false                # infra-only route; skip the ExternalAuth manifest
unset AZURE_CLIENT_SECRET DEPLOYMENT_ENV        # keep non-CI mode AND keep the run secretless
bash "$REPO/scripts/aro-hcp/gen.sh" "$GEN_WI_DIR"
```

You should see a "NON CI mode" warning — that's expected, it's just the script noting which of the six CI variables are absent — followed by the resolved identities and three generated files. In our run ([`track2-h5-gen.log`](../evidence/track2-h5-gen.log)):

```
⚠  NON CI mode - required variables:  DEPLOYMENT_ENV AZURE_SUBSCRIPTION_ID AZURE_TENANT_ID AZURE_CLIENT_ID AZURE_CLIENT_SECRET
AZURE_SUBSCRIPTION_NAME=pool-01-391 <8e15b613-d1f9-41a6-a23d-e8b3ce94d6fe>
ENV=stage - AZURE_SUBSCRIPTION_NAME=pool-01-391 AZURE_SUBSCRIPTION_ID=8e15b613-d1f9-41a6-a23d-e8b3ce94d6fe, REGION=eastus
AZURE_TENANT_ID=64dc69e4-d083-49fc-9569-ebece1dd1408
AZURE_CLIENT_ID=0fb23eb4-e762-4801-b762-3975122ad073 AZURE_ASO_CLIENT_ID=0e9a1f08-c290-42cb-9507-e72a8fd59813
creating: .../gen-wi/credentials.yaml
creating: .../gen-wi/is.yaml
creating: .../gen-wi/aro-aso.yaml
```

Those two client IDs are `capz-hub-mi`'s and `aso-hub-mi`'s — gen.sh read them itself. (Client IDs are public identifiers, not secrets.)

Gate on the rendered output before applying — the right template was chosen and nothing secret got in:

```bash
grep -q 'type: WorkloadIdentity' "$GEN_WI_DIR/credentials.yaml" && echo "WI template: PASS"
grep -c AZURE_CLIENT_SECRET "$GEN_WI_DIR/credentials.yaml"
```

You should see `WI template: PASS` and `0`. In our run: exactly that. Compare with PATH S, where the rendered `credentials.yaml` held the client secret in plaintext and had to be shredded in the same shell that applied it. This file contains nothing sensitive at all.

Apply the credentials and the infrastructure manifest — and **not** `aro-aso.yaml` (that's the `HcpOpenShiftCluster` manifest, which belongs to the preview-gate stage):

```bash
oc get namespace "$NS_WI" >/dev/null 2>&1 || oc create namespace "$NS_WI"
oc apply -f "$GEN_WI_DIR/credentials.yaml"
oc apply -f "$GEN_WI_DIR/is.yaml"
```

You should see the `AzureClusterIdentity`, the `aso-credential` Secret, and then the full infra set created: 1 ResourceGroup, 1 VNet, 1 NSG, 1 subnet, 1 KeyVault, 13 UserAssignedIdentities, 28 RoleAssignments — the identical resource list PATH S applied, in a different namespace, under a different credential.

## The proof

Three checks on the hub, then the Azure ground truth.

First, the secretless proof itself — dump the credential Secret's keys, look for the CAPZ secret object, and read the identity type:

```bash
oc get secret aso-credential -n "$NS_WI" -o jsonpath='{.data}' | jq 'keys'
oc get secret cluster-identity-secret -n "$NS_WI"
oc get azureclusteridentity cluster-identity -n "$NS_WI" -o jsonpath='{.spec.type}{"\n"}'
```

In our run ([`track2-h5-secretless-proof.txt`](../evidence/track2-h5-secretless-proof.txt)), verbatim:

```
[
  "AZURE_CLIENT_ID",
  "AZURE_SUBSCRIPTION_ID",
  "AZURE_TENANT_ID"
]
Error from server (NotFound): secrets "cluster-identity-secret" not found
WorkloadIdentity
```

Read that top to bottom: the ASO credential has exactly three keys and **no `AZURE_CLIENT_SECRET`** (PATH S had four, including it); the Secret that held the SP password for CAPZ on PATH S **does not exist at all**; and the `AzureClusterIdentity` is `type: WorkloadIdentity`. There is no secret material on this hub for this path. That absence isn't a gap ASO tolerates — it's the signal: a namespaced `aso-credential` with no secret-type key is what makes ASO authenticate via its projected token and the FIC you created in step 4.

Second, ASO's own attribution — which credential did it actually use? Events answer that (capture early; they expire after about an hour):

```bash
oc get events -n "$NS_WI" -o json \
  | jq -r '.items[] | select(.message | test("aso-credential")) | .message' | sort -u
```

In our run this returned exactly one line:

```
Using credential from "aro-clusters-wi/aso-credential"
```

That line is the fork-behavior verdict in eleven words: the MCE-shipped ASO found the per-namespace 3-key credential and used it.

Third, convergence — the same counts PATH S had to hit:

```bash
oc get userassignedidentities.managedidentity.azure.com -n "$NS_WI" --no-headers | wc -l
oc get roleassignments.authorization.azure.com -n "$NS_WI" -o json \
  | jq '[.items[] | select(.status.conditions[]? | select(.type=="Ready" and .status=="True"))] | length'
```

You should see `13` and `28`. In our run ([`track2-h5-k8s-ready.txt`](../evidence/track2-h5-k8s-ready.txt)): 13 and 28, all Ready. RoleAssignments go Ready last — ASO retries internally while the fresh UAMI principals propagate through Entra; you don't retry anything by hand.

And the Azure-side ground truth, straight from ARM:

```bash
az group show -n "spk2-<yourid>-resgroup" --query properties.provisioningState -o tsv
az identity list -g "spk2-<yourid>-resgroup" --query 'length(@)' -o tsv
az role assignment list --all --query "length([?contains(scope, 'spk2-<yourid>-resgroup')])" -o tsv
```

In our run ([`track2-h4-h5-azure-state.txt`](../evidence/track2-h4-h5-azure-state.txt)): `Succeeded`, `13`, `28` — and the 13 identity names are the full HCP operator set, `hcp-spk2-ftcqh-cp-control-plane-5254c8` through `hcp-spk2-ftcqh-service-managed-identity-5254c8`: 9 control-plane, 3 data-plane, 1 service-managed identity. Byte-for-byte the same outcome class as PATH S's `spk1` resource group.

**The headline finding, stated plainly:** MCE 2.11's shipped ASO fork honors the secretless per-namespace workload-identity credential. Before this run that was undocumented and unverified — the fork could have predated upstream's fallback, and nothing in any Red Hat or stolostron doc said either way. It works, and it needed zero patching of anything MCE manages.

One honesty note on scope. This stage's `is.yaml` is pure ASO — no CAPI/CAPZ resources — so the end-to-end ARM evidence here covers **ASO** under workload identity. The **CAPZ** half of PATH W is proven to the precondition level: token projection verified, FIC created, `AzureClusterIdentity` typed `WorkloadIdentity`. The one stage that drives CAPZ against ARM (the full cluster manifest at the preview boundary) ran in the PATH S namespace in our run, by design — see [the master doc](../reference/full-plan-and-results.md) for why. If you need the CAPZ-under-WI datapoint end to end, run that stage in `$NS_WI` instead.

Cleanup, for completeness: `oc delete -f is.yaml` and ASO cascade-deletes the whole spoke resource group — ours went from Succeeded to gone in about 90 seconds.

## Debug map

The three failures you can hit on this path, and what each one actually means:

- **`AADSTS70021: No matching federated identity record found`** — Entra received the pod's token and found no FIC matching its issuer + subject + audience. Two causes. Either a real mismatch (issuer differs from the token's `iss` by so much as a trailing slash, subject namespace or SA name is wrong — recheck step 2's byte-for-byte rule and step 4's gate) or plain FIC propagation delay right after creation. Rule out propagation first: retry at ~30 s intervals, up to 10 times, before you start editing anything.
- **`403 AuthorizationFailed`** — authentication *worked* (Entra accepted the token, the identity is who it says it is) and Azure RBAC then denied the operation. That's a missing role assignment from step 3. If it's the RoleAssignment CRs failing while everything else converges, it's specifically the missing Role Based Access Control Administrator grant on the ASO identity.
- **ASO log: `No global credential configured, continuing without default global credential.`** — normal and expected on this path. There is deliberately no global credential; ASO is telling you it will resolve credentials per namespace, which is exactly the mechanism you're relying on. Don't "fix" this.

The tell between the first two: AADSTS errors are Entra rejecting *who you are*; 403 is Azure rejecting *what you tried to do*. Different layer, different fix.

## The warning: leave the MCE-managed objects alone

Everything CAPZ- and ASO-related in `multicluster-engine` — the deployments, their args, their volumes, the chart-managed Secrets like `aso-controller-settings` — is owned by the backplane operator, and it force-reapplies the chart state on every reconcile. Hand-patch a deployment to add an env var, or edit `aso-controller-settings` to inject a global credential, and your change is silently reverted minutes later. The global-credential route to workload identity is therefore not viable on MCE at all.

The pattern in this doc is built to never need it: every piece we created lives either in Azure (identities, FICs, role assignments) or in *our* spoke namespace (`aso-credential`, `AzureClusterIdentity`, the spoke CRs). Nothing in `multicluster-engine` was touched, so nothing fights the operator, and the whole setup survives every backplane reconcile and MCE upgrade untouched. If you find yourself about to `oc edit` anything in `multicluster-engine` to make credentials work — stop; the per-namespace mechanism above is the supported-shaped way through.
