# The preview gate: applying the full cluster manifest and hitting the wall, on purpose

**What this covers / what you need before starting.** This is the last mile of the ARO-HCP track: the full Cluster API manifest — `Cluster`, `AROCluster`, `AROControlPlane`, `AROMachinePool`, `MachinePool`, with the actual `HcpOpenShiftCluster` embedded inside — applied to the hub, for real, on 2026-08-18. Read the next sentence before anything else: **in a normal Azure subscription this step cannot succeed today, and we knew that going in.** ARO-HCP's ARM resource type sits behind an approval-gated private-preview flag (`HcpPrivatePreview`) that our sandbox — like almost every subscription — does not have. We ran the step anyway, deliberately, to pin down exactly where the flow stops and to prove everything before that point works. It does: all 48 embedded Azure infrastructure resources converged, and the one and only failure was the final cluster PUT to ARM, rejected with a clean, typed `404 InvalidResourceType`. This doc walks the manifest, the api-version trap we hit on the way (two stacked mistakes, one fix — you'll hit them too at this repo commit), what success-up-to-the-wall looks like, how to read the rejection like an ops person, and what a GO would take. You need: the MCE 2.11.4 hub from [doc 03](03-hub-mce-setup.md) and a working hub credential — meaning you've already watched ASO build a 13-identity spoke footprint at least once. The commands in this doc assume [doc 04](04-hcp-spoke-sp-path.md)'s service-principal credential, because that's the namespace our run drove this stage in; if you only ran [doc 05](05-hcp-spoke-workload-identity.md)'s secretless path and have no SP secret, that's fine — the render block below has a WI variant right after it that tells you exactly what to change. Background on why HCP spokes are managed-identity-only by construction is in [doc 01](01-identity-models-explained.md). Every observed output below is quoted from [`../evidence/`](../evidence/); the full gated runbook is in [the master plan-and-results doc](../reference/full-plan-and-results.md).

## Caveats first

- **This step is expected to fail in your subscription.** Not "might" — will, unless someone at Red Hat/Microsoft has enabled `HcpPrivatePreview` for it. There is no self-service signup. The value of running it anyway is evidence: it proves your hub, your credential, your manifests, and the entire ASO infrastructure layer end to end, leaving the preview flag as the single remaining variable.
- **Everything here is unsupported dev-preview**, same as docs 04 and 05. Disposable subscription only.
- **The repo has an api-version mismatch at the commit we tested** (`cluster-api-installer` @ `ed240ab`, branch `capi-test-rebase`). Render with its defaults against an MCE 2.11 hub and the `HcpOpenShiftCluster` never even gets created — CAPZ fails client-side before ARM sees anything. The fix is one environment variable (`ARO_HCP_VERSION=v1api20240610preview`). Full story below; don't skip it.
- **Never type bare `cluster` in an `oc` command on an ARO hub.** The hub is itself a classic ARO cluster and carries the aro-operator's cluster-scoped `clusters.aro.openshift.io` CRD, whose singleton instance is literally named `cluster`. Once MCE installs CAPI's `clusters.cluster.x-k8s.io`, bare `cluster` resolves to the *ARO* group (alphabetical group precedence) — so a careless `oc delete cluster $CLUSTER_NAME` at best hits NotFound in the wrong group and at worst aims at the hub's own live ARO configuration object. Every CAPI command in this doc writes the kind fully qualified.
- **If the gate ever opens for you, read the boxed warning near the bottom before your first real create.** There's a known race that fails the cluster *terminally*, and a terminally-failed `HcpOpenShiftCluster` cannot be repaired in place.

## The manifest: five CRs, one of which is the actual cluster

Docs 04 and 05 applied `credentials.yaml` and `is.yaml` and deliberately stopped at the infrastructure boundary. This doc applies gen.sh's third rendering: `aro.yaml`, the full Cluster API manifest. It contains five custom resources. If you know CAPI, skim this; if you don't, here's the map, anchored to things you already run:

- **`Cluster`** (`cluster.x-k8s.io`) — CAPI's top-level umbrella. Think of it as the spoke's `ClusterVersion` as seen from the hub: it does no work itself, it points at two children — an `infrastructureRef` (the AROCluster) and a `controlPlaneRef` (the AROControlPlane) — and rolls up their status.
- **`AROCluster`** (`infrastructure.cluster.x-k8s.io`) — the infrastructure half. Its `spec.resources[]` is a literal embedded list of ASO resources: the resource group, vnet, NSG, subnet(s), Key Vault, the 13 `UserAssignedIdentity` objects, and the role assignments — the same footprint you built in docs 04/05, carried inline instead of in a separate `is.yaml`. CAPZ walks the list and applies each entry as an ASO CR; ASO then makes the ARM calls.
- **`AROControlPlane`** (`controlplane.cluster.x-k8s.io`) — the control-plane half, and the one that matters here. Its `spec.resources[]` embeds the **`HcpOpenShiftCluster`** — the ASO resource whose reconcile is a `PUT https://management.azure.com/.../Microsoft.RedHatOpenShift/hcpOpenShiftClusters/$CLUSTER_NAME`. That PUT *is* the cluster create, and it's the gated step. The embedded spec also shows, in one screen, why HCP is managed-identity-only: `identity.userAssignedIdentities` lists 10 UAMI ARM IDs (9 control-plane operators + the service-managed identity), and `properties.platform.operatorsAuthentication.userAssignedIdentities` maps every control-plane operator, all 3 data-plane operators, and the service-managed identity to their ARM IDs. There is no field anywhere for a service principal.
- **`AROMachinePool`** (`infrastructure.cluster.x-k8s.io`) — the same embedding pattern one level down: its `spec.resources[]` carries an `HcpOpenShiftClustersNodePool`, the worker-pool child of the cluster. It cannot exist before its parent does — remember that when you read the pending states later.
- **`MachinePool`** (`cluster.x-k8s.io`) — CAPI's generic worker-pool umbrella pointing at the AROMachinePool, the way `Cluster` points at `AROCluster`. Closest OpenShift analogue: a MachineSet, managed from the hub instead of from inside the cluster.

So the layering is: CAPI kinds own lifecycle and status rollup, ASO kinds own ARM calls, and the two preview kinds (`HcpOpenShiftCluster`, `HcpOpenShiftClustersNodePool`) are ordinary ASO resources that happen to target a resource type your subscription doesn't have. Which brings us to the trap.

## The api-version trap we hit (read this before you render)

This cost us several reconcile cycles, and it's worth telling as the story it was, because the failure mode is confusing: **the `HcpOpenShiftCluster` never gets created on the hub at all.** There's nothing to `oc get`, no events on the object (there is no object), and the AROControlPlane just sits there. In our run its conditions read, verbatim:

```text
"message": "HcpOpenShiftCluster spk3-ftcqh does not exist yet",
"reason": "HcpOpenShiftClusterNotFound"
```

The only truth is in the CAPZ controller log. Two separate mistakes stack up to get you there.

**Mistake 1: authoring against the CRD's *storage* version name.** Before rendering, ask the hub which API versions the `hcpopenshiftclusters` CRD actually serves:

```bash
oc get crd hcpopenshiftclusters.redhatopenshift.azure.com -o json \
  | jq -r '.spec.versions[] | "\(.name) served=\(.served) storage=\(.storage)"'
```

In our run this returned ([`track2-h6-crd-versions.txt`](../evidence/track2-h6-crd-versions.txt)):

```text
v1api20240610preview served=true storage=false
v1api20240610previewstorage served=true storage=true
```

Both rows say `served=true`, and the `storage=true` one looks authoritative. It isn't. The `...storage` flavor is ASO's internal hub version — the shape objects get converted into for etcd — not a version you author against. Our own version-detection gate grabbed the storage row (`STORED=v1api20240610previewstorage` in the run transcript) and threaded that string into the render. That one wrong string does two bad things at once: gen.sh stamps it verbatim into the manifest's `apiVersion:` lines, and — because gen.sh selects its template with an exact string compare against `v1api20240610preview` — it silently falls through to the default template. Which is mistake 2.

**Mistake 2: the repo's default template is written for a newer api-version than MCE 2.11 ships.** At `ed240ab`, `gen.sh` defaults `ARO_HCP_VERSION` to `v1api20251223preview` and uses `aro-template.yaml` for anything that isn't exactly `v1api20240610preview`. That default template authors two fields from the newer API: `platform.vnetIntegrationSubnetReference` and `etcd.dataEncryption.customerManaged.kms.vaultName`. MCE 2.11's `v1api20240610preview` CRD schema declares neither. The result is a **client-side** failure inside CAPZ — server-side apply refuses to even build the patch — and in our run the controller log repeated this on every reconcile ([`track2-h6-typed-patch-error.txt`](../evidence/track2-h6-typed-patch-error.txt)):

```text
E0818 02:15:51.011023       1 controller.go:353] "Reconciler error" err=<
	error creating AROControlPlane aro-clusters/spk3-ftcqh: failed to reconcile ASO resources: failed to apply resource: failed to create typed patch object (aro-clusters/spk3-ftcqh; redhatopenshift.azure.com/v1api20240610previewstorage, Kind=HcpOpenShiftCluster): errors:
	  .spec.properties.etcd.dataEncryption.customerManaged.kms.vaultName: field not declared in schema
	  .spec.properties.platform.vnetIntegrationSubnetReference: field not declared in schema
```

`field not declared in schema` is your signature. If you see it, no ARM call happened and no `HcpOpenShiftCluster` object exists — don't waste time in the Azure portal, and don't stare at the AROControlPlane conditions waiting for them to change. Fix the render.

**The fix is one variable at render time.** `ARO_HCP_VERSION=v1api20240610preview` — the exact served (non-storage) version string — makes gen.sh select its older template variant, `aro-template-v1api20240610preview.yaml`, which matches what MCE 2.11's CRDs declare (the KMS key nests under `kms.activeKey.vaultName`, and there's no integration-subnet reference). Run it from your clone of `cluster-api-installer`, with the same six CI-mode credential variables as doc 04 — the client secret comes from your sandbox credentials file and must never be echoed or committed:

```bash
export REGION=$LOCATION                  # ours: eastus
export DEPLOYMENT_ENV=sandbox            # any non-EA value; CI mode skips gen.sh's internal EA subscription map
export AZURE_SUBSCRIPTION_ID=$SUBSCRIPTION_ID
export AZURE_TENANT_ID="$(az account show --query tenantId -o tsv)"  # the tenant GUID, not the *.onmicrosoft.com name
export AZURE_CLIENT_ID="$CLIENT_ID"      # your hub SP's appId, as in doc 04
export AZURE_CLIENT_SECRET="$CLIENT_SECRET"      # from your sandbox credentials file; all six set => CI mode
export NAMESPACE="$NS"                   # aro-clusters — ONE namespace for credential + every CR, the doc 04 rule
export USER=hcp                          # gen.sh name prefix; keep it under 5 chars
export CLUSTER_NAME=spk3-mylab           # the spoke's name — ours was spk3-ftcqh; the oc commands below reuse this
export CS_CLUSTER_NAME="$CLUSTER_NAME"   # gen.sh's name for the same thing (doc 04 set it directly)
export OCP_VERSION=4.20
export USE_EA=false
export ARO_HCP_VERSION=v1api20240610preview      # THE fix — the served version, exact string
export OPERATORS_UAMIS_SUFFIX_FILE="$T2/uamis-suffix.txt"  # reuse doc 04's suffix file so UAMI names stay stable across renders
unset GEN_ASO                # unset selects the combined full-manifest render: aro.yaml instead of is.yaml + aro-aso.yaml
unset OICD_RESOURCE_GROUP    # doc 05 exported it; for THIS (PATH S) render it must be unset — WI readers, see the variant below
bash scripts/aro-hcp/gen.sh ./gen-full
shred -u ./gen-full/credentials.yaml     # duplicates doc 04's objects AND contains the secret — never re-apply, destroy it
```

In our run gen.sh printed `✓ USING CI mode` and then `creating: .../gen-full/credentials.yaml` and `creating: .../gen-full/aro.yaml` ([`track2-h6-gen2.log`](../evidence/track2-h6-gen2.log)). Before applying, open `aro.yaml` once and check two things: the embedded `HcpOpenShiftCluster` says `apiVersion: redhatopenshift.azure.com/v1api20240610preview`, and the etcd block reads `kms.activeKey.vaultName`. That's how you know you got the right template variant.

### The WI variant of this render (if you skipped doc 04)

The block above is CI mode, and CI mode hard-requires `AZURE_CLIENT_SECRET` — an SP password. If your hub credential is [doc 05](05-hcp-spoke-workload-identity.md)'s secretless one, don't mint a secret just to render a file: use gen.sh's non-CI workload-identity mode instead, the same way doc 05 step 5 did. Take that step's environment exactly as written — `OICD_RESOURCE_GROUP` (the misspelling is the trigger), `USER_ASSIGNED_IDENTITY_ASO`, `USER_ASSIGNED_IDENTITY_ARO`, `AZURE_SUBSCRIPTION_NAME`, and `unset AZURE_CLIENT_SECRET DEPLOYMENT_ENV` — then change three things for this stage:

```bash
export ARO_HCP_VERSION=v1api20240610preview   # the api-version fix above applies to every render, WI included
unset GEN_ASO                                 # combined full-manifest render: aro.yaml, not is.yaml + aro-aso.yaml
export NAMESPACE="$NS_WI"                     # this spoke's CRs and credential live in the WI namespace
export CLUSTER_NAME=spk3-<yourid>; export CS_CLUSTER_NAME="$CLUSTER_NAME"
bash "$REPO/scripts/aro-hcp/gen.sh" "$REPO/gen-full"
```

You should see the `⚠ NON CI mode` warning (expected — doc 05 explains why WI rendering needs non-CI mode) and the same two `creating:` lines. The rendered `credentials.yaml` is the secretless 3-key variant, so there's nothing to shred, but it duplicates doc 05's already-applied objects — don't re-apply it. Then pick up at the apply below with `$NS_WI` in place of `$NAMESPACE`'s PATH S value. One honesty note: our run drove this stage in the PATH S namespace, by design (doc 05's scope note explains why), so every observed output below came from the SP credential. The WI render is the same templates under the credential doc 05 already proved against ARM — but we didn't watch it cross this particular stage, and you'd be the first.

One honest wrinkle so our numbers below make sense: we hit this trap live, which means our *first* apply used the newer template. Its embedded infrastructure — including a second "integration" subnet and a 29th role assignment scoped to it — went in fine, because the typed-patch check fails per-resource and only the `HcpOpenShiftCluster` tripped it. All of that had converged before we re-rendered with the fix. So the footprint quoted below carries the integration subnet and 29 role assignments. A clean render at `v1api20240610preview` declares one subnet and 28 role assignments; both counts are correct for their template variant. Don't chase a "missing" 29th assignment you never rendered.

## Applying it, and what success-up-to-the-wall looks like

Apply the manifest — this creates the five CAPI-layer CRs, and CAPZ fans the embedded ASO resources out from there:

```bash
oc apply -f ./gen-full/aro.yaml
oc get clusters.cluster.x-k8s.io,arocluster,arocontrolplane,aromachinepool,machinepool -n $NAMESPACE
```

Seconds after apply, our run showed everything in early `Provisioning` — this is the healthy starting state ([`track2-h6-crs.txt`](../evidence/track2-h6-crs.txt)):

```text
NAME                                  ...  PHASE          AGE
cluster.cluster.x-k8s.io/spk3-ftcqh        Provisioning   2s

NAME                                                    CLUSTER      READY   PROVISIONED
arocluster.infrastructure.cluster.x-k8s.io/spk3-ftcqh   spk3-ftcqh

NAME                                                       CLUSTER      READY   CONSOLE URL
arocontrolplane.controlplane.cluster.x-k8s.io/spk3-ftcqh   spk3-ftcqh   false

machinepool.cluster.x-k8s.io/spk3-ftcqh-mp1                spk3-ftcqh   ...     Provisioning   3s
```

Now watch the infrastructure half converge — same rhythm as doc 04, one `Using credential from "aro-clusters/aso-credential"` event per resource, role assignments trailing the rest:

```bash
oc get arocluster $CLUSTER_NAME -n $NAMESPACE -o json | jq '{ready: .status.ready, conditions: .status.conditions}'
oc get roleassignments.authorization.azure.com -n $NAMESPACE -o json \
  | jq '[.items[] | select(.metadata.name | test("'$CLUSTER_NAME'"))
         | select(.status.conditions[]? | select(.type=="Ready" and .status=="True"))] | length'
```

Within a few minutes our run reached the state this whole exercise exists to demonstrate — the AROCluster itself stating that every standard-RP resource is Ready ([`track2-h6-preview-gate.txt`](../evidence/track2-h6-preview-gate.txt)):

```text
{
  "lastTransitionTime": "2026-08-18T02:12:55Z",
  "message": "All 48 infrastructure resources are ready",
  "reason": "InfrastructureReady",
  "status": "True",
  "type": "ResourcesReady"
}
```

The arithmetic behind 48, so you can audit your own run: 1 resource group + 1 vnet + 1 NSG + 2 subnets + 1 Key Vault + 13 UAMIs + 29 role assignments. The leaf checks matched: the role-assignment count returned `29`, the UAMI count `13`, and the flagship resources all reported `True  Succeeded`:

```text
resourcegroup.resources.azure.com/spk3-ftcqh-resgroup   True   Succeeded
virtualnetwork.network.azure.com/spk3-ftcqh-vnet        True   Succeeded
networksecuritygroup.network.azure.com/spk3-ftcqh-nsg   True   Succeeded
vault.keyvault.azure.com/spk3-ftcqh-kv                  True   Succeeded
```

**A small surprise worth recording:** among those 48 was the integration subnet carrying the delegation `Microsoft.RedHatOpenShift/hcpOpenShiftClusters` — and **ARM accepted it**, along with the role assignment scoped to it, in a subscription where the HCP preview is *not* enabled. ARM validates subnet delegations against the services available to the subscription, so we'd flagged this as a place the gate might surface early. It didn't. The delegation went `Succeeded`. File that away: the preview gate is enforced at the resource-type PUT, not at every resource that merely references the `Microsoft.RedHatOpenShift` namespace.

## The wall: 404 InvalidResourceType, exactly where predicted

With infrastructure Ready, the AROControlPlane reconciler moved on — the CAPZ log printed `"AROCluster infrastructure is ready, proceeding with HcpOpenShiftCluster creation"` — created the `HcpOpenShiftCluster` object on the hub, and ASO sent the PUT to ARM. Check what came back:

```bash
oc get hcpopenshiftclusters -n $NAMESPACE
oc get hcpopenshiftclusters $CLUSTER_NAME -n $NAMESPACE -o json | jq '.status.conditions'
```

In our run the answer arrived fast — the fixed manifest was applied at 02:20:37Z and the object carried ARM's rejection by 02:21:05Z, under 30 seconds later. The summary line:

```text
NAME         READY   SEVERITY   REASON                MESSAGE
spk3-ftcqh   False   Error      InvalidResourceType   The resource type could not be found in the namespace 'Microsoft.RedHatOpenShift' for api version '2024-06-10-preview'.: PUT https://management.azure.com/...
```

and the full condition, verbatim from [`track2-h6-preview-gate.txt`](../evidence/track2-h6-preview-gate.txt) — this error is the evidence the whole track was built to capture:

```text
The resource type could not be found in the namespace 'Microsoft.RedHatOpenShift' for api version '2024-06-10-preview'.:
PUT https://management.azure.com/subscriptions/8e15b613-d1f9-41a6-a23d-e8b3ce94d6fe/resourceGroups/spk3-ftcqh-resgroup/providers/Microsoft.RedHatOpenShift/hcpOpenShiftClusters/spk3-ftcqh
--------------------------------------------------------------------------------
RESPONSE 404: 404 Not Found
ERROR CODE: InvalidResourceType
--------------------------------------------------------------------------------
{
  "error": {
    "code": "InvalidResourceType",
    "message": "The resource type could not be found in the namespace 'Microsoft.RedHatOpenShift' for api version '2024-06-10-preview'."
  }
}
--------------------------------------------------------------------------------
```

### How to read that error like an ops person

- **It's a typed provider error, not a 401.** ARM authenticated the hub's bearer token, resolved the subscription, resolved the resource group, and only *then* failed to find the resource type in the provider manifest this subscription sees. If the hub credential were broken you'd see `401` / `AADSTS` / `invalid_client` long before a provider-level error code. So this single error simultaneously proves the credential path end to end **and** isolates the failure to provider enablement. That's why we ran the step at all.
- **`InvalidResourceType` here means "not exposed to *you*", not "doesn't exist".** `Microsoft.RedHatOpenShift` is registered in the subscription — classic ARO works fine (doc 02 built two clusters with it). But the resource-type list a subscription sees is feature-flag-dependent, and without `HcpPrivatePreview`, `hcpOpenShiftClusters` simply isn't in it. Same namespace, different visible type list.
- **Everything downstream sits in Provisioning forever, and that's expected, not a failure.** The `HcpOpenShiftClustersNodePool` can't be created — its owner is the rejected cluster. The `MachinePool` blocks on the `$CLUSTER_NAME-kubeconfig` bootstrap secret that only a provisioned control plane writes. The `Cluster` and `AROControlPlane` report not-ready with reasons like `HcpOpenShiftClusterNotFound` and `Waiting for the Control Plane to get ready`. Record that steady state; don't "fix" it. There is nothing to fix on your side.

The precise boundary, stated once: **every resource ARM serves to normal subscriptions converged; the single resource type behind the private preview was rejected; its dependents wait as a consequence.** That's the gate, pinned to the millimeter.

## What GO would take

Three things, in order.

**1. The `HcpPrivatePreview` feature flag reaches `Registered`.** Azure feature flags (AFEC — Azure Feature Exposure Control) are per-subscription toggles under a provider namespace; think of them as a FeatureGate whose approval side you don't control. This one is **approval-gated**: registering it puts you in a queue, and there is **no self-service signup anywhere** — approval comes from the ARO-HCP program itself. Ask your Red Hat or Microsoft contact; nothing you can click will do it. The mechanics, for a subscription where it's appropriate to try (we did *not* run this in our sandbox — flipping AFEC flags on a shared-pool RHDP subscription is a courtesy question for RHDP first):

```bash
az feature register --namespace Microsoft.RedHatOpenShift --name HcpPrivatePreview
az feature show --namespace Microsoft.RedHatOpenShift --name HcpPrivatePreview --query properties.state -o tsv
```

Expect the second command to print `Pending`, likely indefinitely, until a human approves it. Our pre-run recon showed the flag as `NotRegistered` on the sandbox — that observation is what predicted this whole gate. Registration is reversible with `az feature unregister`.

**2. Re-propagate the provider registration, then confirm the type and its regions.** A newly granted flag doesn't rewrite your provider manifest until the provider re-registers:

```bash
az provider register --namespace Microsoft.RedHatOpenShift
az provider show --namespace Microsoft.RedHatOpenShift -o json \
  | jq '.resourceTypes[] | select(.resourceType | ascii_downcase == "hcpopenshiftclusters") | .locations'
```

Before the flag, that `jq` returns nothing — our sandbox's provider manifest listed only the classic types (`openShiftClusters` and friends). After GO it should return a region list; confirm your target region is in it before any create attempt. Don't assume the classic-ARO region list carries over.

**3. Re-run this doc's apply — after reading the warning below.** The manifest, the credential, and the 48-resource infrastructure layer are already proven; the PUT is the only thing that changes.

> [!WARNING]
> **For whenever the gate opens for you — the inflight-validation race. Do not lose this.**
>
> There is a known race in the HCP resource provider (documented in the repo as `doc/aro-hcp-missing-role-assignments-bug.md`, reproduced 2026-04-02): when the cluster PUT lands, the RP inflight-validates that the service-managed identity holds its required actions over the data-plane identities (`Microsoft.ManagedIdentity/userAssignedIdentities/read`, `.../federatedIdentityCredentials/read` and `write`) — and Azure role assignments propagate lazily. If those assignments exist but haven't propagated when the PUT arrives, the check fails and the cluster goes `provisioningState: Failed` **terminally** (`READY False / InvalidRequestContent`). A terminally-failed `HcpOpenShiftCluster` **cannot be updated in place** — no retry or patch revives it.
>
> Standing rules for any real create:
>
> 1. **Wait for every role assignment to be Ready before the cluster resource is allowed to reach ARM.** `oc get roleassignments.authorization.azure.com -n $NAMESPACE` — the full set (28 or 29 depending on your template variant) all `READY True / Succeeded`, no exceptions. Only then let the `HcpOpenShiftCluster` PUT happen.
> 2. **If a cluster still lands terminally Failed: delete it, don't poke it.** `oc delete -n $NAMESPACE hcpopenshiftclusters/$CLUSTER_NAME` and let the AROControlPlane reconciler recreate it — a fresh PUT is the only path out. Then confirm `provisioningState: Accepted` (for example via `az rest` against the cluster's ARM ID).
>
> An ARM-side fix for the race is reportedly expected; we could not confirm its rollout. Treat the ordering rule as mandatory until you've disproven it in your own environment.

## Tearing it down

Delete the CAPI umbrella — fully qualified, per the bare-`cluster` rule — and let CAPZ/ASO unwind the embedded infrastructure. No real cluster exists (the PUT was rejected), so this is pure infrastructure teardown:

```bash
oc delete clusters.cluster.x-k8s.io/$CLUSTER_NAME -n $NAMESPACE
```

Two observations from our teardown. The Azure side is the easy part: ASO cascade-deleted each spoke resource group in about 90 seconds. The Kubernetes side had one snag: the `AROMachinePool` has a deletion webhook ("must have at least one system pool") that blocked a blanket `oc delete -f aro.yaml` in our run — the pool needed its finalizer stripped after the Azure-side deletion had already completed. If your delete hangs on the machine pool, that's why.

## Where this leaves you

The verdict this doc contributes to the prove-out: the CAPZ/MCE declarative path is real and works end to end **up to and excluding** the one PUT that requires `HcpPrivatePreview` — and when that PUT is eventually accepted, the identity model is already settled, because the manifest you just applied encodes it. Thirteen user-assigned managed identities, twenty-eight or twenty-nine scoped role assignments, and no service-principal field anywhere. The MI-vs-SP question this repo set out to answer doesn't exist at the HCP spoke layer: it was decided at create-time for classic ARO in [doc 02](02-classic-aro-walkthrough.md), and for the hub's own credential in [doc 04](04-hcp-spoke-sp-path.md) versus [doc 05](05-hcp-spoke-workload-identity.md). The full command-by-command runbook with every gate output lives in [the master plan-and-results document](../reference/full-plan-and-results.md).
