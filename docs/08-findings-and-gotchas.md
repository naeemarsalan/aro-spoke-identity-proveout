# Findings and gotchas

**What this covers / what you need before starting.** This is everything we learned during the 2026-08-18 prove-out that wasn't written down anywhere else, ranked roughly by how much time each item will save you. It assumes you've read [01-identity-models-explained.md](01-identity-models-explained.md) for the vocabulary (managed identity, service principal, federated identity credential) and at least skimmed the walkthroughs it references: [02-classic-aro-walkthrough.md](02-classic-aro-walkthrough.md), [03-hub-mce-setup.md](03-hub-mce-setup.md), [04-hcp-spoke-sp-path.md](04-hcp-spoke-sp-path.md), [05-hcp-spoke-workload-identity.md](05-hcp-spoke-workload-identity.md), [06-the-preview-gate.md](06-the-preview-gate.md), and [07-cleanup.md](07-cleanup.md). Every item here cites the evidence file or reference-doc section that backs it — nothing below is speculation. You don't need an Azure subscription to read this; you very much want to have read it before you touch one.

Caveat up front: several of these findings are against preview or unpublished components (MCE 2.11.4's `cluster-api-provider-azure-preview` toggle, the `cluster-api-installer` repo at `ed240ab`, azure-cli 2.84.0 preview flags). They may be fixed by the time you run this. Check before you assume the bug is still there — but also don't assume it's gone.

---

## 1. Findings worth reporting upstream

These five are genuine defects or surprises in shipped software, not misreadings on our part. Each one cost us debugging time or would have cost you some. The reference doc's "New findings" section ([../reference/full-plan-and-results.md](../reference/full-plan-and-results.md)) and the run verdict ([../evidence/track2-h8-verdict.md](../evidence/track2-h8-verdict.md)) carry overlapping-but-not-identical lists from the same run: the reference doc swaps this doc's subnet-delegation finding for an RHDP sandbox-permissions one (covered here in section 2.2), and the verdict condenses to four. This doc is the full write-up of each.

### 1.1 `aro-template.yaml` authors fields the MCE 2.11 CRD doesn't have — the HcpOpenShiftCluster is silently never created

**Symptom.** You apply the full CAPI manifest from `gen.sh`. The infrastructure converges — the `AROCluster` even reports `All 48 infrastructure resources are ready` — but `oc get hcpopenshiftclusters -n <ns>` says `No resources found` forever, and the `AROControlPlane` sits at `HcpOpenShiftClusterNotFound`. Nothing looks obviously broken until you read the CAPZ controller logs, where you find this repeating:

```text
error creating AROControlPlane aro-clusters/spk3-ftcqh: failed to reconcile ASO resources:
failed to apply resource: failed to create typed patch object
(aro-clusters/spk3-ftcqh; redhatopenshift.azure.com/v1api20240610previewstorage, Kind=HcpOpenShiftCluster): errors:
  .spec.properties.platform.vnetIntegrationSubnetReference: field not declared in schema
  .spec.properties.etcd.dataEncryption.customerManaged.kms.vaultName: field not declared in schema
```

That's from our run: [../evidence/track2-h6-typed-patch-error.txt](../evidence/track2-h6-typed-patch-error.txt). You know this failure mode from OpenShift land: it's the same class of problem as applying a CR written for a newer CRD version than the one your cluster serves — except here the rejection happens inside the CAPZ controller's server-side apply, so your `oc apply` succeeds and the failure only shows in controller logs.

**Cause.** Two mistakes compound. First, the repo's default `aro-template.yaml` (in `cluster-api-installer` @ `ed240ab`) authors fields from the newer `v1api20251223preview` API era — `platform.vnetIntegrationSubnetReference` and `etcd.dataEncryption.customerManaged.kms.vaultName` — but the CRDs MCE 2.11 ships only declare the `v1api20240610preview` schema, which predates both fields. Second, it's easy to make it worse yourself: the CRD serves two version names, and one of them is a trap. In our run `oc get crd` reported:

```text
v1api20240610preview served=true storage=false
v1api20240610previewstorage served=true storage=true
```

([../evidence/track2-h6-crd-versions.txt](../evidence/track2-h6-crd-versions.txt)). The `...storage` variant is ASO's internal storage version — a hub-format schema you must never author against. We initially exported `ARO_HCP_VERSION=v1api20240610previewstorage` because that's what the `storage: true` row looked like; `gen.sh` doesn't match that string against its known template variants, falls through to the newer template, and you get exactly the typed-patch error above — note the `previewstorage` group version baked into it.

**Fix.** Regenerate with the plain served version name, which selects the repo's older template variant (no `vnetIntegrationSubnetReference`, no lifted `kms.vaultName`).

This tells `gen.sh` to render the manifest against the API version your MCE actually serves, then re-applies it:

```bash
export ARO_HCP_VERSION=v1api20240610preview
./scripts/aro-hcp/gen.sh "$GEN_DIR"
oc apply -f "$GEN_DIR/aro.yaml"
```

You should see the `HcpOpenShiftCluster` object exist within a couple of reconcile loops. In our run we re-applied at 02:20:37Z and by 02:21:05Z the object existed and carried the *ARM-side* preview-gate rejection instead ([06-the-preview-gate.md](06-the-preview-gate.md)) — which is progress: the manifest finally left the hub. Two rules worth reporting upstream and remembering yourself: the template should not author fields absent from the schema it targets, and `ARO_HCP_VERSION` must be a **served, non-storage** CRD version on your actual MCE build — check with `oc get crd hcpopenshiftclusters.redhatopenshift.azure.com -o jsonpath='{.spec.versions}'` before generating anything.

### 1.2 azure-cli 2.84.0 on Python 3.14 crashes client-side in `az ad sp create-for-rbac` and `az ad sp credential reset`

**Symptom.** The command dies before any network call with a Python traceback ending in:

```text
  File "/usr/lib64/python3.14/argparse.py", line 1750, in _check_help
    raise ValueError('badly formed help string') from exc
ValueError: badly formed help string
```

That's the tail of [../evidence/a0-sp-create.err](../evidence/a0-sp-create.err). It looks like an authorization or Azure failure. It is neither — nothing ever reached Azure.

**Cause.** The command's help text contains a literal `%Y` (a date-format example). Python 3.14's argparse got stricter about `%` in help strings and rejects it while building the parser. So this is a pure client-side packaging bug: azure-cli 2.84.0 plus Python 3.14, triggered by any command whose help text includes the bad string. `az ad sp create-for-rbac` and `az ad sp credential reset` are the two we hit.

**Fix / workaround.** Two options. Either run the CLI under a different Python (a 3.12/3.13 venv, or the container image), or bypass the broken command entirely with raw Microsoft Graph calls through `az rest` — which is what we did. `create-for-rbac` is really three operations glued together, and each has a direct equivalent.

This performs the same three steps `create-for-rbac` would have: create the Entra application, create its service principal, grant the role:

```bash
APP_ID=$(az rest --method POST --url https://graph.microsoft.com/v1.0/applications \
  --body "{\"displayName\": \"$APP_NAME\"}" --query appId -o tsv)
az rest --method POST --url https://graph.microsoft.com/v1.0/servicePrincipals \
  --body "{\"appId\": \"$APP_ID\"}"
az role assignment create --assignee "$APP_ID" --role Contributor \
  --scope "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCEGROUP"
```

You should see each call return a JSON object (application, then servicePrincipal, then role assignment). The password equivalent is `POST /v1.0/applications/{id}/addPassword` — do that only inside the shell invocation that consumes the secret, and never write it to disk. Full rationale for the substitution is recorded in [../evidence/deviation-a0-cli-bug.md](../evidence/deviation-a0-cli-bug.md). In our sandbox the second call was denied for a different, legitimate reason — see section 2.2 — and the Graph route is precisely what let us tell the tooling bug apart from the real permission boundary.

### 1.3 `az aro delete --delete-identities` refuses `--no-wait`

**Symptom.** Cleanup scripts that launch every long-running delete with `--no-wait` (a sane pattern) fail on the managed-identity cluster. In our run this returned:

```text
WARNING: Argument '--delete-identities' is in preview and under development. ...
ERROR: Must not specify --no-wait when --delete-identities is used
```

(Observed in the run transcript at 02:25:34Z — the full command trace is kept out of this repo; the error text above is verbatim.)

**Cause.** `--delete-identities` needs the CLI to stay alive after the cluster delete completes so it can then delete the 9 operator UAMIs — it can't do that fire-and-forget. Arguably reasonable, but it's undocumented and it breaks the obvious "launch all deletes async, poll later" cleanup shape.

**Fix.** Run that one delete blocking and budget the time for it (an ARO delete is ~30–45 min). Everything else can still be `--no-wait`. When we re-ran it blocking, it worked: the final sweep confirmed the cluster gone and `identities remaining: 0` in the resource group about nine minutes later ([../evidence/final-cleanup-gate.txt](../evidence/final-cleanup-gate.txt)). (The blocking run's own console output wasn't captured in the evidence bundle — the 0-identities gate is the proof.) If you can't afford a blocking 40-minute command, skip the flag and delete the identities yourself afterward with `az identity delete` — [07-cleanup.md](07-cleanup.md) has the manual fallback.

### 1.4 The `AROMachinePool` deletion webhook wedges `oc delete -f aro.yaml`

**Symptom.** Tearing down the Track 2 manifest, most objects delete, but:

```text
Error from server (Forbidden): error when deleting ".../aro.yaml": admission webhook
"validation.aromachinepools.infrastructure.cluster.x-k8s.io" denied the request:
if the delete is triggered via owner MachinePool please refer to trouble shooting section
in https://capz.sigs.k8s.io/topics/managedcluster.html: ARO Cluster must have at least one system pool
```

(Observed in the run transcript at 02:22:36Z — the full command trace is kept out of this repo; the error text above is verbatim.)

**Cause.** The CAPZ validating webhook protects a *live* cluster from losing its last system node pool. But our cluster never existed — the `HcpOpenShiftCluster` PUT was rejected at the preview gate — and the webhook still enforces the invariant against a machine pool whose cluster is a ghost. The `AROMachinePool` delete is denied, the CAPI `Cluster` sits in phase `Deleting` waiting on it, and the namespace can never terminate cleanly.

**Fix.** First wait until the Azure-side deletion of everything real has completed — in our run, `az group exists` went `false` for all three spoke resource groups by 02:24:32Z while `AROMachinePool spk3-ftcqh-mp1` was still stuck. Then take the pool down through its owner.

This deletes the owning CAPI `MachinePool` first, clears any finalizers on the `AROMachinePool`, and deletes it directly:

```bash
oc delete machinepool.cluster.x-k8s.io "$POOL_NAME" -n "$NS" --wait=false
oc patch aromachinepool "$POOL_NAME" -n "$NS" --type=merge -p '{"metadata":{"finalizers":[]}}'
oc delete aromachinepool "$POOL_NAME" -n "$NS" --wait=false
```

You should see all three commands succeed and the namespace empty out; in our run the finalizer patch reported `patched (no change)` (the finalizer had already been released by then) and the direct delete went through immediately — five seconds later `oc get clusters.cluster.x-k8s.io,aromachinepool,machinepool -n aro-clusters` printed `No resources found`. Finalizer-stripping is a last resort, not a habit — do it only after `az group exists -n $SPOKE_RG` returns `false`, because a finalizer is the only thing standing between you and orphaned Azure resources. Worth reporting upstream: the webhook should not block deletion when the owning cluster resource never provisioned.

### 1.5 ARM accepts the `hcpOpenShiftClusters` subnet delegation in a subscription that can't create `hcpOpenShiftClusters`

**Symptom** (a pleasant one, but a finding). The Track 2 full manifest includes an integration subnet delegated to `Microsoft.RedHatOpenShift/hcpOpenShiftClusters`. A subnet delegation is Azure handing a resource provider standing permission to inject things into your subnet — you'd expect ARM to reject a delegation to a resource type the subscription isn't enrolled for. It didn't. In our run all 48 embedded infra resources converged, including that delegated subnet and its associated role assignment — the gate capture shows `spk3 RAs ready: 29`, the 29th being the integration-subnet assignment — while the `HcpOpenShiftCluster` PUT itself was the only rejection ([../evidence/track2-h6-preview-gate.txt](../evidence/track2-h6-preview-gate.txt)).

**Cause.** Delegation validation apparently only checks that the RP namespace/type string is registered as a *delegatable* type platform-wide, not that your subscription has the gated feature.

**Why you care.** Two edges. First, it means the entire infra layer of an HCP spoke — resource group, VNet, NSG, KeyVault, 13 UAMIs, 28–29 role assignments, delegated subnet — is provable in any ordinary subscription; the preview gate bites at exactly one resource ([06-the-preview-gate.md](06-the-preview-gate.md)). Second, it means a green delegated subnet tells you nothing about whether the actual cluster create will work. Don't use it as a preflight signal.

---

## 2. Permission gotchas

Azure permissions come in two unrelated systems, and most of the failures below come from conflating them. **Azure RBAC** governs resources (subscriptions, resource groups, role assignments) — think of it as Kubernetes RBAC for cloud objects. **Entra ID** (the directory, formerly Azure AD) governs identity objects — applications, service principals, users — and has its own completely separate permission model. Being a god in one says nothing about the other.

### 2.1 Plain Contributor cannot write role assignments — the single most likely failure for a reader

Contributor sounds like it can do everything. It can create and delete nearly any *resource*, but the built-in Contributor role explicitly excludes `Microsoft.Authorization/roleAssignments/write` — it cannot grant permissions to anything. Both tracks of this prove-out are role-assignment-heavy: the classic MI path writes 20 scoped assignments before cluster create, and the HCP templates embed 28–29 `RoleAssignment` resources that ASO must write using *your* controller credential. If your identity is plain Contributor, everything works until the first role assignment, which fails with `403 AuthorizationFailed` — usually ten minutes into a run, inside a controller whose logs you then have to go read.

Our sandbox got away with it because the RHDP login SP holds a custom role with `actions: ["*"]` at subscription scope (reference doc, Section 3 recon table). Don't assume yours does. Prove it in thirty seconds before you start, the way our Gate 0.2 canary did.

This creates a throwaway identity, attempts one real role-assignment write, and cleans up — the exact permission everything downstream needs:

```bash
az identity create -g "$RESOURCEGROUP" -n smoke-canary -o none
CANARY_OID=$(az identity show -g "$RESOURCEGROUP" -n smoke-canary --query principalId -o tsv)
az role assignment create --assignee-object-id "$CANARY_OID" --assignee-principal-type ServicePrincipal \
  --role Reader --scope "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCEGROUP" -o none
az role assignment delete --assignee "$CANARY_OID" --role Reader \
  --scope "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCEGROUP"
az identity delete -g "$RESOURCEGROUP" -n smoke-canary
```

In our run this printed `GATE 0.2 PASS: identity create + role-assignment write + cleanup all succeeded` ([../evidence/gate-0.2-write-canary.txt](../evidence/gate-0.2-write-canary.txt)). If the `role assignment create` step fails, stop and fix your role before doing anything else — see 2.3 for what to ask for.

### 2.2 RHDP-style sandbox SPs can create Entra *applications* but not *service principals* — and can't delete applications

Quick decoder: in Entra, an **application** is the definition of a bot account (name, credentials list), and a **service principal** is the instance of it that can actually log in and hold roles in your tenant. `az ad sp create-for-rbac` creates both plus a role assignment. In a shared tenant like RHDP's `redhat0.onmicrosoft.com`, the sandbox credential is allowed to do some of that and not the rest, and the failure lands mid-sequence.

In our run: the Graph `POST /v1.0/applications` **succeeded**, then `POST /v1.0/servicePrincipals` was **denied** with `Authorization_RequestDenied` ("the backing application of the service principal being created must be in the local tenant" — the Graph error signature for a caller lacking rights to create servicePrincipal objects). Then our attempt to delete the now-useless application was *also* denied — the sandbox SP can create apps but not delete them. Full record: [../evidence/a0-fallback-decision.md](../evidence/a0-fallback-decision.md).

Three consequences to plan around:

- **You cannot mint a dedicated per-cluster SP in an RHDP sandbox.** Expect `Authorization_RequestDenied` and plan to reuse the SP the sandbox hands you (that's RHDP's own quickstart pattern). Our SP-path cluster did exactly that.
- **A failed `create-for-rbac` can orphan a tenant-level Entra application** that subscription teardown will never touch — RHDP recycles subscriptions, not tenant directory objects. We left exactly one such orphan (`aro-ftcqh-spoke-a-sp`, appId `add45f0c-ce56-49ff-a91d-31466c088d58` — inert: no SP, no credential ever attached) that needs a tenant admin to remove.
- **Never rotate or delete the sandbox-provided SP.** It's the credential everything else in the sandbox (including your own login) depends on. This is also why we skipped the rotation demo on the SP cluster.

### 2.3 The narrow alternative to Owner: Role Based Access Control Administrator

When you hit 2.1, the reflex is to ask for Owner. There's a tighter answer. Azure ships a built-in role called **Role Based Access Control Administrator** whose whole job is `Microsoft.Authorization/roleAssignments/*` — it can grant and revoke roles but can't touch resources. Pair it with Contributor and you have exactly what this workflow needs, nothing more.

That's what we gave the ASO hub identity on the workload-identity path. In our run `aso-hub-mi` held precisely two roles at subscription scope — `Contributor` and `Role Based Access Control Administrator`, not Owner — and all 28 role assignments reconciled Ready ([../evidence/track2-h5-hub-uami-roles.json](../evidence/track2-h5-hub-uami-roles.json)). The CAPZ identity needed only Contributor. If your security team balks at Owner (they should), this is the ask. One sharper option exists — RBAC Administrator supports *conditions* restricting which roles it may assign, so you could scope it to just the ten ARO role GUIDs — but we didn't need that in the sandbox and didn't test it.

---

## 3. Operational gotchas

Smaller traps, each worth a few minutes to a few hours.

### 3.1 ASO looks up `aso-credential` per namespace — the default-namespace pitfall

ASO v2 resolves the credential for each resource it reconciles by looking for a Secret literally named `aso-credential` **in that resource's own namespace**. Not in a central config namespace — the resource's. The draft MCE doc's example creates the Secret in `default` while the provisioning examples use a different namespace; follow both verbatim and every ASO resource fails authentication (published as ACM-30244 known issue (c); our namespace rule is stated once in the reference doc's H3 and enforced everywhere).

The rule we enforced: the `AzureClusterIdentity`, the `aso-credential` Secret, and *all* cluster/infra CRs for a given spoke live in the same namespace, and `NAMESPACE` is pinned explicitly on every `gen.sh` run so nothing lands in `default`. You can verify attribution directly — ASO emits an event per resource naming the credential it picked. In our run: `Using credential from "aro-clusters/aso-credential"` on every resource ([../evidence/track2-h4-credential-events.txt](../evidence/track2-h4-credential-events.txt)). If you run both credential paths side by side like we did, this is also your isolation proof: each namespace's resources cite each namespace's Secret.

### 3.2 The env var is spelled `OICD_RESOURCE_GROUP`, not `OIDC_`

`gen.sh` switches into workload-identity mode when `OICD_RESOURCE_GROUP` is set. That's not a typo in this document — it's a typo *in the script*, consistently, and the misspelling is required verbatim. Export `OIDC_RESOURCE_GROUP` (the correct spelling) and gen.sh silently generates the SP-path credentials template instead, and you'll waste time wondering why your workload-identity run has a `clientSecret` in it. Reference: [../reference/full-plan-and-results.md](../reference/full-plan-and-results.md) H5.5, and the reconciled research notes. Worth an upstream PR; until then, copy-paste, don't retype.

### 3.3 Name-length limits: `$USER` under 5 characters, KeyVault names 24 characters and globally unique

Two naming constraints collide in `gen.sh`'s generated names. The repo doc requires `USER` < 5 chars (it's a prefix component), and Azure KeyVault names are capped at 24 characters **and globally unique across all of Azure** — not per-subscription, global, because the name becomes a DNS label (`<name>.vault.azure.net`). Same idea as an S3 bucket name. A too-long or colliding vault name doesn't fail at generation; it surfaces later as a Vault CR error during reconcile. We used `USER=hcp` and `spk1/spk2/spk3-ftcqh` cluster prefixes and never hit it; the failure mode and the fix (rename the prefix, re-run gen.sh) are documented at risk item 17 in [../reference/full-plan-and-results.md](../reference/full-plan-and-results.md).

### 3.4 Tenant GUID vs tenant domain — templates need the GUID

Azure accepts two spellings of "which tenant": the domain form (`yourcorp.onmicrosoft.com`) and the GUID. `az login` happily takes either. The ASO/CAPZ templates do **not** — `Vault.spec.properties.tenantId` carries a GUID-pattern validation in the shipped CRD, so a domain string renders into the manifest as an admission-rejected value. Sandbox credentials files typically hand you the domain form. Our rule: the domain form is used *only* for `az login`; everything that renders into a manifest gets the GUID, fetched once.

This resolves the GUID form of your tenant regardless of how you logged in:

```bash
export AZURE_TENANT_GUID=$(az account show --query tenantId -o tsv)
```

You should get a bare GUID back (in our run, `64dc69e4-...`). Then `export AZURE_TENANT_ID="$AZURE_TENANT_GUID"` before every `gen.sh` invocation (reference doc, Track 2 session setup).

### 3.5 `[Preview]` warnings on GA features are cosmetic — capture them, don't fear them

Managed-identity classic ARO went GA on 2026-02-02, with CLI 2.84.0 as the documented floor. That same CLI still stamps every MI flag as preview. In our run, the create emitted:

```text
WARNING: Argument '--enable-managed-identity' is in preview and under development. ...
WARNING: Argument '--assign-cluster-identity' is in preview and under development. ...
WARNING: Argument '--assign-platform-workload-identity' is in preview and under development. ...
```

([../evidence/b-create-launch.log](../evidence/b-create-launch.log)) — and the cluster created successfully as a fully supported GA configuration. Treat the warnings as CLI metadata lagging the support statement. Same story for `az aro delete --delete-identities` (works, modulo finding 1.3). Do tee the launch output into your evidence anyway; if support ever questions the configuration, the log showing exactly which flags you passed is the artifact you'll want.

### 3.6 The bare-`cluster` trap on ARO hubs

If your MCE hub is itself an ARO cluster (ours was), `oc get cluster` and `oc delete cluster` do not mean what the CAPI docs think they mean. ARO ships its own `clusters.aro.openshift.io` CRD with a cluster-scoped singleton named `cluster`, and bare-name resolution picks *that* group — not CAPI's `clusters.cluster.x-k8s.io`. Two failure shapes: a bare `oc delete cluster <name>` returns NotFound and your spoke's cascade teardown never starts; worse, a bare `oc get cluster -n <ns>` in a poll loop always prints the hub's singleton row (the `-n` flag is ignored for cluster-scoped kinds), so a "wait until gone" gate can never pass. Always fully qualify.

This targets the CAPI cluster object unambiguously — the upstream doc's bare form assumed a non-ARO hub:

```bash
oc delete clusters.cluster.x-k8s.io "$SPOKE_NAME" -n "$NS" --wait=false
```

You should see the spoke's CAPI `Cluster` enter phase `Deleting` and its cascade start (in our run the resource groups showed `Deleting` within seconds). Reference: risk item 21 and the H9 cleanup commentary in [../reference/full-plan-and-results.md](../reference/full-plan-and-results.md).

### 3.7 If the cluster will become a hub, pass the pull secret at create time

A classic ARO cluster creates fine without a Red Hat pull secret — but it ships with the OperatorHub default catalog sources (`redhat-operators`, `certified-operators`) **disabled**. MCE installs from `redhat-operators`, whose images come from `registry.redhat.io`. So a cluster destined to be a hub that was created without the pull secret forces you into the slow path: fetch the secret mid-run from console.redhat.com (a manual browser step), merge — never replace — it into the cluster's existing pull secret, re-enable `redhat-operators` on `operatorhub/cluster`, and then wait out an undocumented catalog-propagation delay before the MCE subscription can even resolve. We front-loaded it instead: download the pull secret *before* anything else and pass `--pull-secret @$HOME/pull-secret.txt` on the create. Details and the exact fallback procedure: [../reference/full-plan-and-results.md](../reference/full-plan-and-results.md) section 0.4 and H0.2, and [03-hub-mce-setup.md](03-hub-mce-setup.md).

### 3.8 Bonus: ASO cascade-delete is the cleanup mechanism, and it's fast

Not a gotcha so much as a "trust the machine" note with a sharp edge. Deleting an ASO `ResourceGroup` CR deletes the real Azure resource group and everything in it. That's alarming the first time and then it's just how you clean up: in our run, `oc delete -f is.yaml` on each spoke sent all three spoke resource groups to `Deleting` within seconds, and each was gone about 90 seconds later — all three confirmed deleted by 02:24:32Z against a 02:22:50Z launch (timestamps from the run transcript; the `az group exists` gate is in [../evidence/final-cleanup-gate.txt](../evidence/final-cleanup-gate.txt)). The sharp edge: never point an ASO CR at a resource group you want to keep, and never delete a namespace that still has live ASO CRs in it — see [07-cleanup.md](07-cleanup.md) for the ordering rules.

---

## 4. Day-2 notes

What owning each identity model actually feels like after create day.

**SP cluster: you own an expiry clock.** The client secret created at cluster build expires — ours on **2027-08-17**, one year out, captured in [../evidence/a-sp-secret-expiry.txt](../evidence/a-sp-secret-expiry.txt) (`2027-08-17T18:36:18Z`). Azure gives you no built-in alerting for it; the first notification most teams get is the outage. Rotation is `az aro update --refresh-credentials -g $RESOURCEGROUP -n $CLUSTER` — it needs `roleAssignments/write` and can take **up to 2 hours** ([rotation doc](https://learn.microsoft.com/en-us/azure/openshift/howto-service-principal-credential-rotation)). Put the expiry date in your monitoring the day you create the cluster, and rotate well before it. (In the sandbox we deliberately skipped the rotation demo — the cluster SP *was* the shared sandbox credential, and rotating it would have broken every subsequent login. See 2.2.)

**MI cluster: zero rotation, by design.** The managed identities authenticate with certificates Azure itself rolls; there is no secret and no clock you own. If federated identity credentials (FICs — the Entra objects that tell Azure to trust the cluster's OIDC issuer for a given service account) ever get broken or deleted, a bare `az aro update -g $RESOURCEGROUP -n $CLUSTER` reconciles them back — safe to run on a healthy cluster, up to 2 hours ([reconcile doc](https://learn.microsoft.com/en-us/azure/openshift/howto-reconcile-federated-identity-credentials)). That's the entire day-2 credential story.

**Upgrades can add operator identities.** New OCP versions may introduce new platform operators that need their own identity. The flow is day-2 and order-sensitive: create the new UAMI, create its documented role assignments **first**, then `az aro update --assign-platform-workload-identity <key> <identity>` ([replace-identity doc](https://learn.microsoft.com/en-us/azure/openshift/howto-replace-cluster-identity)). Run `az aro update --upgradeable-to <x.y.z>` before an upgrade to find out whether you're affected. A deleted identity is recoverable the same way — replacement, not cluster rebuild.

**ACM import doesn't care which model you chose.** ARO is import-only in ACM (Create/Destroy = No in the support matrix), and import works on a kubeconfig/token `auto-import-secret` plus a klusterlet install — no Azure credential of either kind is involved anywhere in the flow. The MI-vs-SP decision is completely invisible to the hub. The only ACM consideration for a private spoke is klusterlet-to-hub API reachability. Reference: [../reference/full-plan-and-results.md](../reference/full-plan-and-results.md), Section 8 ACM spoke note.

---

## 5. Open questions we could not answer from a sandbox

Honest list. Each of these needs either a privileged subscription, a Microsoft/Red Hat program contact, or a newer build to resolve.

**Does `HcpPrivatePreview` registration ever approve for a normal subscription?** The gate we hit ([06-the-preview-gate.md](06-the-preview-gate.md)) is an AFEC flag — Azure's feature-flag system for subscriptions — that sits `NotRegistered` and is approval-gated: registering it puts you in `Pending` until Microsoft/Red Hat approve, and there is no self-service signup anywhere. We deliberately did **not** run `az feature register` from the sandbox (reference doc H6.4 — left as a user decision; a disposable sandbox is the wrong place to enroll). So we can't say whether registration from an ordinary subscription approves, sits Pending forever, or errors — nor what region list `hcpOpenShiftClusters` exposes once granted. If you have a subscription with a real support relationship, this is the experiment to run.

**Has the inflight-validation race fix rolled out?** The known `managed-service-identity-inflight` bug: if the service-managed identity's role assignments over the data-plane identities haven't *propagated* by the time the `HcpOpenShiftCluster` provisions, ARM's inflight check fails the cluster **terminally** — `provisioningState: Failed`, not retriable in place; you must delete the CR and let the controller recreate it. An ARM-side fix was "expected" as of the bug's repro date, rollout unconfirmed. Since our PUT never got past the preview gate, we couldn't observe whether it's fixed. Until someone confirms it, treat the ordering rule as mandatory: apply the role assignments first and wait for all of them `READY True/Succeeded` before the cluster resource reaches ARM. The standing rule is in [../reference/full-plan-and-results.md](../reference/full-plan-and-results.md) (H8/H10, item 2).

**What do newer MCE channels (2.17+) ship?** Everything in Track 2 was proven on MCE 2.11.4 from OLM channel `stable-2.11`, whose CRDs serve only `v1api20240610preview` — which is what made finding 1.1 bite. Newer channels presumably ship newer CRD api-versions (the repo's default template targets `v1api20251223preview`), which would change which template variant is correct, possibly resolve the mismatch, and possibly introduce new ones. Unverified on a 4.19 hub: whether the newer channels' hub-OCP floor even admits it, which api-versions their CRDs serve, and whether the stolostron ASO fork in those builds keeps the secretless per-namespace credential fallback that the workload-identity path depends on (proven for 2.11 — [../evidence/track2-h8-verdict.md](../evidence/track2-h8-verdict.md) — but re-verify per build; it's one `oc get crd` and one Secret-keys check, per [05-hcp-spoke-workload-identity.md](05-hcp-spoke-workload-identity.md)).

---

*Previous: [07-cleanup.md](07-cleanup.md) · Master runbook and executed results: [../reference/full-plan-and-results.md](../reference/full-plan-and-results.md)*
