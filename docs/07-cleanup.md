# Cleanup: tearing down two tracks in the order that works

**What this covers / what you need before starting.** This doc is the teardown for everything the prove-out built: the Track 2 spoke footprints ([doc 04](04-hcp-spoke-sp-path.md), [doc 05](05-hcp-spoke-workload-identity.md), [doc 06](06-the-preview-gate.md)), the hub-side credential objects, and the two classic ARO clusters from [doc 02](02-classic-aro-walkthrough.md) — down to a resource group that contains only what the sandbox started with. The order is not arbitrary and this doc explains why at each step. You need: `oc` logged into the hub as cluster-admin, `az` CLI logged into the subscription, and — this is the part people skip — the principal IDs you recorded during setup, because several of these steps need to reference identities *after* they've been deleted. Everything here ran for real at the end of our 2026-08-18 run; the whole teardown, from the first CR delete to the final gate, took about 12 minutes of wall clock (we had budgeted an hour), and the observed outputs below are quoted from [`../evidence/`](../evidence/) — chiefly `final-cleanup-gate.txt` and the run transcript.

## Caveats first

> **Deleting the ASO `ResourceGroup` CRs deletes the REAL Azure resource groups and everything in them.** That is not a side effect to guard against — it *is* the cleanup mechanism, and it's what makes this teardown fast: in our run ASO cascade-deleted each spoke resource group in about 90 seconds. The same power cuts the other way. **Never run this against anything shared. Never point an ASO CR at a resource group you want to keep. Never delete your sandbox's provided resource group, its DNS zone, or the login service principal your credentials file authenticates as.** In our sandbox those were the RG `openenv-ftcqh`, the zone `ftcqh.azure.redhatworkshops.io`, and the RHDP-issued SP — none of them is referenced by any ASO CR in this repo, which is a design choice, not luck.

Three more things before you type anything:

- **Order matters, and the reason is always "who is doing the deleting".** The hub's CAPZ/ASO controllers are the only things that know how to tear down the Azure resources they created. Kill the hub cluster first and every spoke resource group is orphaned — you'd be cleaning them up by hand with `az group delete` and hoping you found them all. The hub dies **last** among the things it manages.
- **Don't delete a cluster's own identities or role assignments before that cluster's delete completes.** The ARO resource provider uses those permissions *during* teardown (it has to detach networks and remove its own resources). `--delete-identities` exists precisely so the RP can sequence this for you. The two *hub* UAMIs (`capz-hub-mi`, `aso-hub-mi`) are different: they belong to the MCE controllers, not to any ARO cluster, so they can go as soon as the controllers are done working.
- **One thing this cleanup cannot fix:** a tenant-level Entra application. Our run left exactly one behind (details at the bottom). Subscription-level cleanup — yours or RHDP's — never touches the tenant.

## The order, and why

| Step | What | Why here and not elsewhere |
|---|---|---|
| 0 | Archive evidence off-host | The sandbox can expire mid-teardown; the deletes below destroy the systems your evidence describes |
| 1 | Delete CAPI cluster CRs and ASO infra CRs | Hub controllers are still alive and do the Azure-side deleting |
| 2 | Confirm every spoke RG is gone (`az group exists` = `false`) | Hard gate before you touch the hub — after this, orphaning is impossible |
| 3 | Hub-side extras: namespaces, the 2 hub UAMIs and their role assignments | Controllers are idle now; these identities belong to the hub, not to any cluster teardown |
| 4 | `az aro delete` both clusters | The RP needs its permissions until now; `--delete-identities` removes the MI cluster's 9 UAMIs |
| 5 | Delete the vnets | Clusters hold subnet attachments until their deletes complete |
| 6 | Scoped orphaned-role-assignment sweep | Only RG-scope orphans can survive; sweep by allowlist, never by blank `principalName` |
| 7 | Final gate: RG contains only what it started with | The definition of done |

## Before you type anything: map your variables

This doc names things the way the master runbook does, which is not always the way docs 04–06 did. If you followed the walkthrough docs in one shell, set these now — an unset variable in a destructive command is the worst kind of surprise (at best `oc delete -f /aro.yaml` fails on a missing file; at worst a delete lands somewhere you didn't aim):

```bash
export T2=~/aro-hcp-work                # doc 04's working directory; re-export it if this shell is new
export REPO="$T2/cluster-api-installer" # doc 04's clone location
export GEN_SP_DIR="$T2/gen-sp"          # doc 04's render output directory
export GEN_WI_DIR=<doc-05-output-dir>   # you chose this path in doc 05; re-export it if this shell is new
export GEN_FULL_DIR="$REPO/gen-full"    # doc 06 rendered into ./gen-full inside your cluster-api-installer clone
export NS_SP=aro-clusters               # doc 04 called this $NS
export NS_WI=aro-clusters-wi            # same name as doc 05
export CS1=spk1-<yourid>                # the CS_CLUSTER_NAME from doc 04 (ours: spk1-ftcqh)
export CS2=spk2-<yourid>                # from doc 05 (ours: spk2-ftcqh)
export CS3=spk3-<yourid>                # from doc 06 (ours: spk3-ftcqh)
export EVIDENCE=<your-evidence-dir>     # wherever you've been saving recorded outputs
```

The Azure-side steps (3, 4, 5) consume variables from docs 02 and 05 too. In a fresh shell every one of these is empty, and `az aro delete -g "" -n ""` is exactly the surprise we're avoiding — re-export them all:

```bash
export SUBSCRIPTION_ID=$(az account show --query id -o tsv)
export RESOURCEGROUP=<your-resource-group>   # doc 02's sandbox RG (ours: openenv-ftcqh)
export CLUSTER_A=aro-spoke-a                 # doc 02's SP cluster
export CLUSTER_B=aro-spoke-b                 # doc 02's MI cluster
export VNET_A=aro-spoke-a-vnet               # doc 02's vnets
export VNET_B=aro-spoke-b-vnet
export UAMI_CAPZ=capz-hub-mi                 # doc 05's two hub identities
export UAMI_ASO=aso-hub-mi
export CAPZ_PID=<capz-hub-mi-principalId>    # recorded in doc 05 step 1; if you lost it, recover it NOW,
export ASO_PID=<aso-hub-mi-principalId>      # while the identity still exists: az identity show -g $RESOURCEGROUP -n $UAMI_CAPZ --query principalId -o tsv
```

The master runbook guards each of these with a `:?empty` check before using it; do the cheap version here — eyeball them before the first delete:

```bash
for V in T2 GEN_SP_DIR GEN_WI_DIR GEN_FULL_DIR NS_SP NS_WI CS1 CS2 CS3 EVIDENCE \
         SUBSCRIPTION_ID RESOURCEGROUP CLUSTER_A CLUSTER_B VNET_A VNET_B \
         UAMI_CAPZ UAMI_ASO CAPZ_PID ASO_PID; do
  printf '%s=%s\n' "$V" "${!V:-<EMPTY - fix before continuing>}"
done
```

You should see a value on every line and no `<EMPTY>` markers. Fix any before touching a delete.

And build the sweep allowlist for step 6 while the principals still exist. It's one GUID per line: the 9 classic-MI operator principal IDs you recorded in doc 02 (`identity-principals.tsv`, second column), the 2 hub UAMI principal IDs from doc 05 step 1, and — if you created a dedicated cluster SP in doc 02 — its object ID:

```bash
cut -f2 "$EVIDENCE/identity-principals.tsv"          >  "$EVIDENCE/run-principals.txt"
printf '%s\n%s\n' "$CAPZ_PID" "$ASO_PID"             >> "$EVIDENCE/run-principals.txt"
# plus the dedicated SP's object id, if you made one:  az ad sp show --id $CLUSTER_A_CLIENT_ID --query id -o tsv
```

You should end up with 11 or 12 GUIDs in the file. If you never recorded the principal IDs, stop and collect them now (`az identity show ... --query principalId`) — after step 4 the identities are gone and nothing can tell you what they were.

## Step 0 — archive the evidence first

Everything below is destructive, and a disposable sandbox can expire while you work. Tar the evidence directory and copy it off the host before the first delete.

```bash
tar czf "evidence-$(date -u +%Y%m%dT%H%M%SZ).tar.gz" "$EVIDENCE"/
# scp it somewhere that isn't this sandbox before continuing
```

You should see a tarball; confirm the off-host copy landed before moving on. In our run we archived twice — once before teardown (`evidence-20260818T022212Z.tar.gz`) and once after step 7 so the bundle includes the cleanup gates themselves (`evidence-final-20260818T023456Z.tar.gz`).

## Step 1 — delete the cluster CRs and infra CRs while the hub is still alive

This is the step where the cascade warning is live. Deleting these CRs tells ASO to delete the real Azure resources they represent — resource groups, vnets, KeyVaults, the 13 UAMIs per spoke, all of it.

First the full cluster manifest from [doc 06](06-the-preview-gate.md). Note the fully qualified kind in the poll below: on an ARO hub, a bare `oc delete cluster` or `oc get cluster` resolves to the hub's own `clusters.aro.openshift.io` CRD (the aro-operator's cluster-scoped singleton, literally named `cluster`), not the CAPI kind — at best you get NotFound in the wrong API group, at worst you're aiming a command at the hub's live ARO configuration. Always write `clusters.cluster.x-k8s.io`.

```bash
oc delete -f "$GEN_FULL_DIR/aro.yaml" --wait=false
```

Expect a partial failure here — this is the first trap. In our run the `Cluster` and `AROCluster` CRs deleted, and then:

```text
Error from server (Forbidden): error when deleting ".../gen-full/aro.yaml": admission webhook
"validation.aromachinepools.infrastructure.cluster.x-k8s.io" denied the request: if the delete is
triggered via owner MachinePool please refer to trouble shooting section in
https://capz.sigs.k8s.io/topics/managedcluster.html: ARO Cluster must have at least one system pool
```

The `AROMachinePool` has a validating admission webhook that refuses to let a cluster lose its last system pool. Reasonable guardrail on a live cluster; a nuisance on one that never existed (our PUT was rejected at the preview gate, so there was never anything Azure-side behind this pool). Leave it for now — we come back to it after the Azure side is confirmed gone.

Then the two infra manifests from paths S and W. Deleting by `-f` targets exactly the CRs each `is.yaml` applied — no kind enumeration to get wrong:

```bash
oc delete -f "$GEN_SP_DIR/is.yaml" --wait=false
oc delete -f "$GEN_WI_DIR/is.yaml" --wait=false
```

You should see a long stream of `... deleted` lines ending with the role assignments. Immediately afterwards, watch ASO working:

```bash
oc get resourcegroups.resources.azure.com -A
```

In our run this returned all three spoke RGs mid-flight:

```text
NAMESPACE         NAME                  READY   SEVERITY   REASON     MESSAGE
aro-clusters-wi   spk2-ftcqh-resgroup   False   Info       Deleting   The resource is being deleted
aro-clusters      spk1-ftcqh-resgroup   False   Info       Deleting   The resource is being deleted
aro-clusters      spk3-ftcqh-resgroup   False   Info       Deleting   The resource is being deleted
```

That `Deleting` status is ASO issuing real DELETE calls to ARM. This is why the hub must outlive these CRs.

## Step 2 — hard gate: confirm the Azure side is actually gone

Do not proceed past this point until `az group exists` returns `false` for **every** spoke resource group. This is the line between "clean teardown" and "orphaned resources you'll be hunting next week".

```bash
for RG in "$CS1-resgroup" "$CS2-resgroup" "$CS3-resgroup"; do
  printf '%s exists: ' "$RG"; az group exists -n "$RG" -o tsv
done
```

In our run the first poll (about 30 seconds after the deletes) still showed `true` for all three; one minute later, all three were `false`. Launch-to-gone was roughly 90 seconds per resource group — everything inside each RG (vnet, NSG, KeyVault, 13 UAMIs, role assignments) went with it in one cascade. That speed is the payoff for building everything inside dedicated RGs: teardown is one delete per spoke, not fifty.

Now finish the webhook holdout from step 1. With the Azure side confirmed empty, delete the owner `MachinePool`, strip the `AROMachinePool`'s finalizer, and delete it directly:

```bash
oc delete machinepool.cluster.x-k8s.io "$CS3-mp1" -n "$NS_SP" --wait=false
oc patch aromachinepool "$CS3-mp1" -n "$NS_SP" --type=merge -p '{"metadata":{"finalizers":[]}}'
oc delete aromachinepool "$CS3-mp1" -n "$NS_SP" --wait=false
```

In our run the finalizer patch returned `patched (no change)` — it had none left to strip — and the direct delete then went through: `aromachinepool.infrastructure.cluster.x-k8s.io "spk3-ftcqh-mp1" deleted`. If your webhook still refuses the direct delete, the finalizer strip is the escape hatch — but only after `az group exists` said `false`. Stripping finalizers is how you tell Kubernetes "stop waiting for external cleanup"; do it before the external cleanup happened and the external resources live forever. Verify nothing is left:

```bash
oc get clusters.cluster.x-k8s.io,aromachinepool,machinepool -n "$NS_SP"
```

In our run: `No resources found in aro-clusters namespace.`

## Step 3 — hub-side extras: namespaces, then the two hub UAMIs

The namespaces first — and only now, because deleting a namespace that still contains live ASO CRs triggers uncontrolled parallel Azure deletes and stuck finalizers. Steps 1–2 proved it's empty of anything real.

```bash
oc delete ns "$NS_SP" "$NS_WI" --wait=false
```

You should see both `deleted` lines and nothing hang.

Now the two hub credential identities from [doc 05](05-hcp-spoke-workload-identity.md) — the UAMIs the CAPZ and ASO controllers federated to. Two things to know:

- **Deleting a UAMI deletes its federated identity credentials with it.** FICs are child objects of the identity; there is nothing separate to clean.
- **Role assignments do NOT die with the identity.** They become orphaned rows pointing at a principal that no longer resolves. That's why you delete the assignments *first*, and why you recorded the principal IDs at setup time — after the identity is gone, `az identity show` can't tell you what its principal ID was.

```bash
az role assignment delete --assignee "$CAPZ_PID" --scope "/subscriptions/$SUBSCRIPTION_ID"
az role assignment delete --assignee "$ASO_PID"  --scope "/subscriptions/$SUBSCRIPTION_ID"
az identity delete -g "$RESOURCEGROUP" -n "$UAMI_CAPZ"
az identity delete -g "$RESOURCEGROUP" -n "$UAMI_ASO"
```

All four commands return silently on success; in our run this whole block took about 20 seconds.

One decision point here if your hub is going to live on: the MCE preview toggles (`cluster-api`, `cluster-api-provider-azure-preview`) should be flipped back to their recorded prior state, and there's a published ACM known issue about stale `cluster.x-k8s.io` CRDs blocking `hypershift-local-hosting` re-enablement afterwards. We skipped all of it — our hub was `aro-spoke-b`, which dies in the next step anyway. Our state file records the decision: `MCE-toggle-restore=skipped-hub-deleted`. If your hub outlives the teardown, see H9 step 8 in [the master plan](../reference/full-plan-and-results.md) for the restore procedure.

## Step 4 — delete the ARO clusters themselves

Now, and only now, the clusters. The SP cluster is a plain delete; launch it non-blocking:

```bash
az aro delete -g "$RESOURCEGROUP" -n "$CLUSTER_A" --yes --no-wait
```

Returns immediately; the cluster shows `Deleting` in `az aro list` from here on.

The MI cluster is where the flag gotcha lives. `--delete-identities` tells the RP to also delete the cluster's 9 operator UAMIs after the cluster teardown — exactly what you want. But it refuses to combine with `--no-wait`. We tried:

```bash
az aro delete -g "$RESOURCEGROUP" -n "$CLUSTER_B" --yes --delete-identities --no-wait
```

In our run this returned:

```text
WARNING: Argument '--delete-identities' is in preview and under development. Reference and support levels: https://aka.ms/CLI_refstatus
ERROR: Must not specify --no-wait when --delete-identities is used
```

So cluster B's delete runs blocking. Drop `--no-wait` and let it hold the terminal:

```bash
az aro delete -g "$RESOURCEGROUP" -n "$CLUSTER_B" --yes --delete-identities
```

In our run the blocking delete completed: about nine minutes after launch, `az aro list` came back empty and the identity count below read 0 — all 9 operator UAMIs gone with the cluster, which is the confirmation you want, because it means the manual fallback isn't needed. (The blocking run's console output wasn't captured in our evidence bundle; the `identities remaining: 0` gate in `final-cleanup-gate.txt` is the record.) If `--delete-identities` misbehaves for you (it's `[Preview]` in CLI 2.84.0), the fallback is: delete the 20 role assignments by the principal IDs you recorded at setup (in our run, `identity-principals.tsv` — name to principalId, written the moment the identities were created), then `az identity delete` each of the 9 by name. This is the second place the record-at-setup habit pays off.

Poll until both clusters are gone:

```bash
az aro list -g "$RESOURCEGROUP" -o table
```

Mid-teardown in our run this showed `aro-spoke-a ... Deleting` while `aro-spoke-b` was still `Succeeded` (its blocking delete hadn't started yet); at the final gate it returned empty. Both clusters were gone about nine minutes after launch — far faster than the 30–45 minutes per cluster we'd budgeted. Nice when it happens; don't plan around it. Also confirm the RP's managed resource groups went with them, and that the MI cluster's identities really are gone:

```bash
az group list --query "[?starts_with(name,'aro-')].name" -o tsv | wc -l
az identity list -g "$RESOURCEGROUP" --query 'length(@)' -o tsv
```

In our run: `0` and `0`.

## Step 5 — vnets

The clusters held subnet attachments until their deletes completed; that's why the vnets wait until now. Both deletes are quick:

```bash
az network vnet delete -g "$RESOURCEGROUP" -n "$VNET_A"
az network vnet delete -g "$RESOURCEGROUP" -n "$VNET_B"
```

Silent success; in our run both were gone in about 20 seconds total. Deleting a vnet also deletes the role assignments scoped *to* it — which is why the sweep in the next step only has to think about RG-scope assignments.

## Step 6 — the scoped orphan sweep (read the trap before running anything)

When a principal is deleted, its role assignments linger as orphans. Vnet-, subnet-, and identity-scoped assignments died with their resources in steps 1–5; the only place an orphan from this run can survive is at resource-group scope. So the sweep is: list RG-scope assignments, keep only rows whose `principalId` is on **this run's allowlist** — the principal IDs you recorded at setup — and delete those by assignment ID.

Here's the trap, and it can end your sandbox. The tempting version of this sweep is `az role assignment list --all` filtered on `principalName == ''` — "find every assignment whose principal no longer resolves". **Never do that.** `principalName` is populated by a Microsoft Graph lookup at list time, and a Graph throttle or hiccup blanks it on **every** row — including your own login SP's subscription-scope assignment. Filter on empty `principalName` during a Graph blip and you will delete the role assignment that is your only way into the subscription. `--all` also drags in orphans that belong to the platform's own plumbing, which are not yours to delete. Allowlist by recorded `principalId`, scope to your RG, or don't sweep at all.

Build the allowlist from what you recorded (in our run: the 9 classic-MI principal IDs from `identity-principals.tsv`, the 2 hub UAMI principal IDs, and the dedicated SP's object ID if you created one), then sweep:

```bash
az role assignment list --scope "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCEGROUP" \
  --query "[?contains(scope, 'resourceGroups/$RESOURCEGROUP')].[principalId,id]" -o tsv \
  | grep -F -f "$EVIDENCE/run-principals.txt" | cut -f2 \
  | xargs -r -n1 az role assignment delete --ids
```

The `contains(scope, ...)` filter matters: listing at RG scope also returns *inherited* subscription-scope rows — your login SP's among them — and the filter keeps only assignments made directly on the RG. In our run the sweep found nothing to delete; `final-cleanup-gate.txt` records:

```text
no orphaned RG-scope assignments
```

Empty is the expected result when `--delete-identities` worked and the identity-scoped assignments died with their resources. The sweep is a verification, not a workhorse.

## Step 7 — final gate: the resource group contains only what it started with

One command defines done:

```bash
az resource list -g "$RESOURCEGROUP" -o table
```

In our run this returned exactly the sandbox's pre-existing DNS zone and nothing else:

```text
Name                            ResourceGroup    Location    Type                        Status
------------------------------  ---------------  ----------  --------------------------  ---------
ftcqh.azure.redhatworkshops.io  openenv-ftcqh    global      Microsoft.Network/dnszones  Succeeded
```

Gate PASS, 02:35Z. Re-run the step 0 archive now so the bundle includes these gate outputs — that's our `evidence-final-20260818T023456Z.tar.gz`.

Two loose ends worth knowing about:

- **KeyVault tombstones.** The spoke KeyVaults were soft-delete enabled, so cascade-deleting their RGs leaves subscription-level "deleted vault" tombstones that can block a future vault reusing the name. The plan carries an optional purge (`az keyvault list-deleted` piped to `az keyvault purge`); we didn't run it — our sandbox gets recycled wholesale. If your subscription lives on, spend the 30 seconds.
- **The evidence tarballs and kubeconfigs on the host** are outside the resource group and outside this gate. The kubeconfigs point at clusters that no longer exist; the tarballs are the point of the exercise.

## What a run can leave behind that Azure cleanup can't fix

Entra applications live at the **tenant** level, not the subscription level. Nothing that operates on your subscription — this teardown, your platform's automated sandbox recycling, even `az group delete` on everything — will ever touch them. If a run creates an app registration and can't delete it, that app outlives the sandbox.

Our run left exactly one. The dedicated-SP attempt in [doc 02](02-classic-aro-walkthrough.md) got far enough to create the application object (`aro-ftcqh-spoke-a-sp`, appId `add45f0c-ce56-49ff-a91d-31466c088d58`) before Graph denied the service-principal creation; our attempt to delete the app was itself denied — the sandbox SP can create applications but not delete them (`Insufficient privileges`). The result is an **inert** orphan: no service principal exists for it, no credential was ever added, so nothing can authenticate as it. Harmless, but real, and only a tenant admin can remove it. We recorded it (`a0-orphan-app-id.txt`, `a0-fallback-decision.md` in the evidence bundle) and reported it as a manual action item.

The general lesson: `az ad sp create-for-rbac` is really three operations (app create, SP create, role assignment), and a partial failure strands tenant-level objects your subscription cleanup will never see. If your run creates Entra apps, record their IDs the moment they exist, and put "delete the app" on a checklist that a human with tenant rights actually reads.

## The full picture

Teardown of two classic ARO clusters, an MCE hub, three HCP spoke footprints (39 UAMIs and 85 role assignments across them — 28 + 28 + 29), two hub identities, and two vnets took about 12 minutes of operator-attended time, and the only residue was one inert tenant-level app object. The reasons it went that fast are all decisions made at *build* time: every spoke in its own resource group (one cascade delete each), every principal ID recorded at creation (sweeps and fallbacks work after the identities are gone), and managed identities instead of secrets everywhere possible (nothing to revoke, nothing to rotate, nothing forgotten in a vault). Cleanup is where the identity model choice from [doc 01](01-identity-models-explained.md) quietly pays out one last time: the MI cluster's entire credential footprint was 9 Azure resource objects that `--delete-identities` removed in one pass, while the SP path's footprint included a tenant-level object that outlived the subscription. The verdict doc pulls this thread together with everything else — see the findings in [the master plan-and-results doc](../reference/full-plan-and-results.md).
