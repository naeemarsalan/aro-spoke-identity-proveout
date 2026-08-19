# Turning an ARO cluster into an MCE hub that can provision ARO-HCP spokes

**What this covers / what you need before starting.** This doc takes a running classic ARO cluster and turns it into a multicluster-engine (MCE) hub with the Cluster API Azure provider enabled — the machine that will later stamp out ARO-HCP spokes declaratively. You need: a classic ARO 4.19+ cluster you can `oc login` to as cluster-admin (we used the managed-identity cluster from the Track 1 runbook), a Red Hat pull secret (see below — this one is non-negotiable), and the `oc` and `jq` binaries. No Azure CLI calls happen in this doc; everything here is Kubernetes-side. Concepts like managed identities and workload-identity federation are explained in [01-identity-models-explained.md](01-identity-models-explained.md); the credential paths that ride on top of this hub are the next doc's job. The full command-by-command record of our 2026-08-18 run lives in [the master plan-and-results doc](../reference/full-plan-and-results.md), and every observed output quoted below is a file in [`../evidence/`](../evidence/).

## Caveats first

- **Everything in this doc is dev preview and unsupported.** The `cluster-api-provider-azure-preview` component ships *dark* in MCE >= 2.11: it's in the payload, it's toggleable, but the official procedure doc was merged to rhacm-docs (PR #8616) and then **pulled before publication**. There is no published Red Hat procedure for what you're about to do. Our commands come from Marek Veber's `cluster-api-installer` repo docs (`doc/ARO-capz-mce.md`, which matches the unpublished PR) and were verified live on 2026-08-18.
- **The component name has "preview" in it for a reason.** Don't put this on a production hub. We ran it on a disposable sandbox cluster that was deleted two hours later.
- **Enabling CAPI disables HyperShift on the same hub.** An admission webhook enforces mutual exclusivity (details below). If your hub hosts HyperShift control planes, this procedure is not for that hub.
- **MCE 2.11 was the only version whose hub-OCP support range we could verify** (4.19–4.22, per the MCE support matrix). The catalog offered us newer channels; we deliberately did not take them.

## Why we used the Track 1 managed-identity cluster as the hub

Any OpenShift 4.19+ cluster with a **public `serviceAccountIssuer`** works as this hub. We used `aro-spoke-b`, the managed-identity classic ARO cluster from Track 1, for one very good reason: a managed-identity ARO cluster's service-account token issuer is already a public, Azure-hosted OIDC endpoint. That's the exact thing workload-identity federation needs later (the hub's controller pods will present projected service-account tokens to Entra, and Entra has to be able to fetch the issuer's signing keys over the public internet — same trust mechanism your own API server uses to verify SA tokens, just pointed outward).

Check what your cluster's issuer is. This reads the cluster Authentication config:

```bash
oc get authentication cluster -o jsonpath='{.spec.serviceAccountIssuer}{"\n"}'
```

In our run this returned:

```
https://eastus.oic.aro.azure.com/64dc69e4-d083-49fc-9569-ebece1dd1408/3daaa256-9628-4bf6-a438-988f1b34ed3f
```

That's the `https://eastus.oic.aro.azure.com/<tenant>/<uuid>` issuer format a managed-identity ARO cluster gets for free — evidence: [`track2-h5-issuer.txt`](../evidence/track2-h5-issuer.txt). Its `/.well-known/openid-configuration` and JWKS are publicly fetchable, which means zero extra work when we federate the hub controllers in the next doc. If your issuer is the default in-cluster `https://kubernetes.default.svc`, workload identity for the hub controllers is off the table until you host a public issuer — the SP credential path still works, but you lose the best part.

On a non-ARO OCP cluster you'd check the same field; a self-managed cluster set up with Azure workload identity (or any cluster whose issuer is a public URL with reachable JWKS) qualifies too.

## The pull secret is REQUIRED — get it before you create the cluster

MCE ships in the `redhat-operators` catalog, whose images come from `registry.redhat.io`. Here's the trap: **an ARO cluster created without a Red Hat pull secret has the default OperatorHub catalog sources (`redhat-operators`, `certified-operators`) shipped `Disabled: true`**. Not "slow", not "reduced" — the CatalogSource for the operator you need does not exist. Track 1 treats the pull secret as optional; the moment a cluster is destined to become this hub, it stops being optional.

The cheap path: download the pull secret from <https://console.redhat.com/openshift/install/pull-secret> (browser + Red Hat account required) *before* the cluster exists, and pass it at create time:

```bash
az aro create ... --pull-secret @$HOME/pull-secret.txt
```

That's what we did — our `aro-spoke-b` create carried `--pull-secret @/home/anaeem/pull-secret.txt`, so when Track 2 started, `redhat-operators` was already there and healthy.

The expensive path — retrofitting mid-run — is documented pain, and it's three separate steps, not one:

1. **Merge, never replace.** The ARO cluster's existing pull secret carries the `arosvc.azurecr.io` credential that platform image pulls depend on. You must export the current secret, merge your new auths into it with `jq`, and `oc set data` the result back — clobbering it breaks the cluster's own registry access. (`az aro update` has no `--pull-secret` flag; the `oc` merge route per Microsoft's `howto-add-update-pull-secret` doc is the only post-create mechanism. We verified that flag absence against CLI 2.84.0.)
2. **Re-enable the catalog source.** Merging the secret alone never creates the `redhat-operators` CatalogSource — on a cluster born without a pull secret it's disabled in `operatorhub/cluster`, and you have to patch `disabled: true` to `false` for that one source.
3. **Wait an undocumented amount of time.** The newly enabled catalog pod has to pull its index image after the global pull secret propagates. Our plan budgeted a bounded retry of up to ~20 minutes for the `packagemanifest` to appear.

Do yourself the favor: fetch the file first, pass it at create.

Before installing anything, sanity-check the hub. This confirms version and that the catalogs are alive:

```bash
oc get clusterversion version -o jsonpath='{.status.desired.version}{"\n"}'
oc get catalogsource -n openshift-marketplace
```

In our run ([`track2-gate-h0.1-hub-sanity.txt`](../evidence/track2-gate-h0.1-hub-sanity.txt)) this returned `4.19.24` and all four default catalogs (`redhat-operators`, `certified-operators`, `community-operators`, `redhat-marketplace`) present — because the pull secret went in at create time.

## Install MCE via OLM

MCE installs like any operator you've installed via OLM: Namespace, OperatorGroup, Subscription, then the operator's own CR. One twist we insist on: **gate on what the catalog actually offers instead of hardcoding a channel.**

First, ask the packagemanifest what channels exist and which is the default:

```bash
oc get packagemanifest multicluster-engine -n openshift-marketplace \
  -o jsonpath='default={.status.defaultChannel}{"\n"}all={range .status.channels[*]}{.name}{" "}{end}{"\n"}'
```

In our run the catalog offered `stable-2.8`, `stable-2.9`, `stable-2.10`, `stable-2.11`, and `stable-2.17` — and **defaulted to `stable-2.17`** ([`track2-gate-h1.1-mce-default-channel.txt`](../evidence/track2-gate-h1.1-mce-default-channel.txt)). If we'd created a Subscription without a channel, OLM would have taken the default and handed us MCE 2.17. We pinned `stable-2.11` deliberately: it's the first release that ships `cluster-api-provider-azure-preview`, and it's the only one whose hub-OCP support range (4.19–4.22) we could verify against our 4.19.24 hub. A newer channel carries a newer ASO fork (more HCP api-versions), but an unverified floor. Pick from what you *observed*, and write down why.

Now the OLM trio. This creates the namespace, scopes an OperatorGroup to it, and subscribes to the channel you just verified:

```bash
oc apply -f - <<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: multicluster-engine
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: multicluster-engine-og
  namespace: multicluster-engine
spec:
  targetNamespaces:
  - multicluster-engine
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: multicluster-engine
  namespace: multicluster-engine
spec:
  channel: stable-2.11
  installPlanApproval: Automatic
  name: multicluster-engine
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF
```

Poll the CSV until it succeeds (every minute or two, ~15 min budget):

```bash
oc get csv -n multicluster-engine \
  -o jsonpath='{range .items[*]}{.metadata.name}{" "}{.status.phase}{"\n"}{end}'
```

In our run this hit `multicluster-engine.v2.11.4 Succeeded` on the third poll, about 90 seconds after the Subscription ([`track2-gate-h1.2-csv.txt`](../evidence/track2-gate-h1.2-csv.txt)).

Then create the MulticlusterEngine CR itself — defaults are fine, and the target namespace defaults to `multicluster-engine`:

```bash
oc apply -f - <<EOF
apiVersion: multicluster.openshift.io/v1
kind: MultiClusterEngine
metadata:
  name: multiclusterengine
spec: {}
EOF
```

Poll its phase:

```bash
oc get mce multiclusterengine -o jsonpath='{.status.phase}{"\n"}'
```

In our run it went `Progressing` → `Available` in just over 3 minutes, reporting `currentVersion: 2.11.4` ([`track2-gate-h1.3-mce-available.txt`](../evidence/track2-gate-h1.3-mce-available.txt)). Sizing note: our hub's three `Standard_D4s_v3` workers held MCE plus the three CAPI controllers without scaling; if pods sit `Pending`, add a worker.

## Component toggles: HyperShift off first, then CAPI on. The order is enforced.

MCE ships with a component list in `spec.overrides.components`, each entry `{name, enabled}`. Out of the box, `hypershift` and `hypershift-local-hosting` are **on** and all `cluster-api*` components are **off** (our recorded prior state: [`track2-h2-prior-components.json`](../evidence/track2-h2-prior-components.json) — worth capturing on your hub too if you ever intend to revert).

You cannot just flip `cluster-api` on. Since MCE 2.10 an admission webhook (`validateComponentExclusivity`) rejects the patch if HyperShift is still enabled — the error reads, in essence, *"HyperShift components (hypershift, hypershift-local-hosting) and Cluster API components (...) cannot be enabled simultaneously."* The two provisioning stacks fight over `cluster.x-k8s.io` CRD versions, so MCE forces you to choose. Order is therefore mandatory: HyperShift components off, *then* CAPI components on.

The patch pattern below is the same every time: read the current components array, rewrite one entry's `enabled` with `jq`, and merge-patch the whole array back. It looks heavy, but it's idempotent and never clobbers the other entries.

Turn off both HyperShift components:

```bash
oc patch mce multiclusterengine --type=merge -p "{\"spec\":{\"overrides\":{\"components\":$(oc get mce multiclusterengine -o json | jq -c '.spec.overrides.components | map(if .name == "hypershift" then .enabled = false else . end)')}}}"
oc patch mce multiclusterengine --type=merge -p "{\"spec\":{\"overrides\":{\"components\":$(oc get mce multiclusterengine -o json | jq -c '.spec.overrides.components | map(if .name == "hypershift-local-hosting" then .enabled = false else . end)')}}}"
```

Verify HyperShift is actually gone before proceeding (bounded retry, ~30 s apart):

```bash
oc get deployment -n hypershift
```

You want exactly: `No resources found in hypershift namespace.` — which is what our run showed ([`track2-gate-h2-toggles.txt`](../evidence/track2-gate-h2-toggles.txt)).

Now enable `cluster-api`, then `cluster-api-provider-azure-preview` — same pattern, `enabled = true`:

```bash
oc patch mce multiclusterengine --type=merge -p "{\"spec\":{\"overrides\":{\"components\":$(oc get mce multiclusterengine -o json | jq -c '.spec.overrides.components | map(if .name == "cluster-api" then .enabled = true else . end)')}}}"
oc patch mce multiclusterengine --type=merge -p "{\"spec\":{\"overrides\":{\"components\":$(oc get mce multiclusterengine -o json | jq -c '.spec.overrides.components | map(if .name == "cluster-api-provider-azure-preview" then .enabled = true else . end)')}}}"
```

Watch for the three controllers (poll every minute or two; if one sticks at 0/1 there's a known force-reconcile workaround, ACM-30244: `oc annotate mce multiclusterengine force-reconcile=$(date +%s) --overwrite`):

```bash
oc get deployment -n multicluster-engine | grep -E "(capi|capz|azureserviceoperator)"
```

In our run all three were 1/1 about 90 seconds after the patches — from [`track2-gate-h2-toggles.txt`](../evidence/track2-gate-h2-toggles.txt):

```
azureserviceoperator-controller-manager   1/1     1            1           88s
capi-controller-manager                   1/1     1            1           92s
capz-controller-manager                   1/1     1            1           88s
```

(A fourth deployment, `mce-capi-webhook-config`, comes up alongside them — that's expected.) Quick who's-who: **capi-controller-manager** is upstream Cluster API, the generic cluster-lifecycle machinery. **capz-controller-manager** is the Azure provider (CAPZ) — it knows how to talk to ARM. **azureserviceoperator-controller-manager** (ASO) is Azure Service Operator v2 — it reconciles individual Azure resources (resource groups, identities, role assignments) as Kubernetes CRs. CAPZ delegates most of the actual Azure resource creation to ASO.

## Verify, don't patch: the token projection is already wired

This was the single most reassuring discovery of the whole run. Everything workload identity needs *inside the pods* — the projected service-account token volume, the mount path, the Entra-shaped audience — **already ships in the MCE build**. You verify it; you do not patch it. (You couldn't anyway: the backplane operator server-side-applies its charts and reverts hand edits.)

What you're looking for: a *projected service-account token* volume. That's the same mechanism as any bound SA token in OpenShift — kubelet mints a short-lived JWT for the pod — except the `audience` is set to `api://AzureADTokenExchange`, the audience Entra expects during a workload-identity token exchange. If Entra trusts your cluster's issuer (next doc), it accepts this token like your API server accepts any SA token.

Check the CAPZ deployment's projected volume and its mount:

```bash
oc get deploy capz-controller-manager -n multicluster-engine -o json \
  | jq '.spec.template.spec.volumes[] | select(.projected.sources[]?.serviceAccountToken)'
oc get deploy capz-controller-manager -n multicluster-engine -o json \
  | jq '.spec.template.spec.containers[].volumeMounts[]? | select(.name | test("token|identity"))'
```

In our run ([`track2-gate-h2.5-token-projection.txt`](../evidence/track2-gate-h2.5-token-projection.txt)) this returned, verbatim:

```json
{
  "name": "azure-identity-token",
  "projected": {
    "defaultMode": 420,
    "sources": [
      {
        "serviceAccountToken": {
          "audience": "api://AzureADTokenExchange",
          "expirationSeconds": 3600,
          "path": "azure-identity-token"
        }
      }
    ]
  }
}
{
  "mountPath": "/var/run/secrets/azure/tokens",
  "name": "azure-identity-token",
  "readOnly": true
}
```

Same checks against ASO:

```bash
oc get deploy azureserviceoperator-controller-manager -n multicluster-engine -o json \
  | jq '.spec.template.spec.volumes[] | select(.projected.sources[]?.serviceAccountToken)'
oc get deploy azureserviceoperator-controller-manager -n multicluster-engine -o json \
  | jq '.spec.template.spec.containers[].volumeMounts[]? | select(.name | test("token|identity"))'
```

Observed: volume `azure-identity`, same `api://AzureADTokenExchange` audience, mounted at `/var/run/secrets/tokens`. Note the two controllers use **different paths** — CAPZ reads `/var/run/secrets/azure/tokens/azure-identity-token`, ASO reads `/var/run/secrets/tokens/azure-identity`. You'll never type these paths (the code defaults to them), but if you're debugging a token-exchange failure, that asymmetry is a classic red herring. It's correct.

Record the two service-account names — the next doc federates exactly these to Azure identities, and you should gate on live values rather than assume:

```bash
oc get deploy capz-controller-manager -n multicluster-engine -o jsonpath='{.spec.template.spec.serviceAccountName}{"\n"}'
oc get deploy azureserviceoperator-controller-manager -n multicluster-engine -o jsonpath='{.spec.template.spec.serviceAccountName}{"\n"}'
```

In our run: `capz-manager` and `azureserviceoperator-default` ([`track2-h5-sa-names.txt`](../evidence/track2-h5-sa-names.txt)).

Two more build facts worth confirming while you're here. First, that CAPZ actually has the ARO feature turned on:

```bash
oc get deploy capz-controller-manager -n multicluster-engine -o jsonpath='{.spec.template.spec.containers[0].args}' | jq .
```

Our run showed `--feature-gates=MachinePool=true,AKSResourceHealth=false,EdgeZone=false,ASOAPI=true,APIServerILB=false,ARO=true` — that trailing `ARO=true` is the whole point of the "preview" build. (ASO's args are interesting too: `--crd-management=none`, meaning MCE ships the ASO CRDs statically rather than letting ASO install its own.)

Second, that the ARO/HCP CRDs are present and which API versions they serve:

```bash
oc get crd | grep -Ei 'arocluster|arocontrolplane|aromachinepool|hcpopenshift'
oc get crd hcpopenshiftclusters.redhatopenshift.azure.com \
  -o jsonpath='{range .spec.versions[*]}{.name} served={.served} storage={.storage}{"\n"}{end}'
```

Our run showed all six CRDs present (`aroclusters`, `arocontrolplanes`, `aromachinepools` on the CAPI side; `hcpopenshiftclusters` plus its `externalauths`/`nodepools` siblings on the ASO side — we didn't capture that listing as a gate file), and the HCP CRD serving **only `v1api20240610preview`** — the version rows are the captured evidence ([`track2-h6-crd-versions.txt`](../evidence/track2-h6-crd-versions.txt)):

```
v1api20240610preview served=true storage=false
v1api20240610previewstorage served=true storage=true
```

Write that version down. The MCE 2.11 CRDs serve nothing newer, and authoring manifests against a newer api-version's fields is a real failure mode we hit later (CAPZ rejects the object with `failed to create typed patch object` and the HcpOpenShiftCluster is simply never created — covered in the spoke doc).

> [!WARNING]
> **The bare-`cluster` trap on an ARO hub.** Your hub is itself a classic ARO cluster, so it already carries the aro-operator CRD `clusters.aro.openshift.io` — kind `Cluster`, cluster-scoped, with a singleton instance literally named `cluster` that holds the hub's own live ARO configuration. Once the CAPI toggle installs `clusters.cluster.x-k8s.io`, every bare `cluster` in an `oc` command is ambiguous, and the bare name resolves to the **aro.openshift.io** group (alphabetical group precedence). So `oc get cluster` shows you the hub's ARO config singleton, not your CAPI clusters — and the upstream doc's cleanup command `oc delete cluster $CLUSTER_NAME` (written for a non-ARO hub) at best hits NotFound in the wrong group and deletes nothing, and at worst you fat-finger your way to touching `cluster.aro.openshift.io/cluster`, the hub's own ARO configuration. Always type the CAPI kind fully qualified: `oc get clusters.cluster.x-k8s.io -n <ns>`, `oc delete clusters.cluster.x-k8s.io/<name> -n <ns>`. Never bare `cluster` on this hub.

## Where you are now

You have an MCE 2.11.4 hub with CAPI, CAPZ (with `ARO=true`), and ASO running 1/1 in `multicluster-engine`, the ARO/HCP CRDs installed, and — verified, not patched — both controllers already projecting Entra-audience service-account tokens. Total time in our run: under 10 minutes from Subscription to verified controllers (~3 min for MCE to go Available, ~2 min for the toggled controllers to come up).

What the hub does *not* have yet is any way to authenticate to Azure. That's the next doc: the two hub credential paths — a service-principal secret versus federating those two service accounts to Azure managed identities for a fully secretless hub — both of which we proved against real ARM. Continue with [04-hcp-spoke-sp-path.md](04-hcp-spoke-sp-path.md) for the service-principal path and [05-hcp-spoke-workload-identity.md](05-hcp-spoke-workload-identity.md) for the secretless one; [06-the-preview-gate.md](06-the-preview-gate.md) then drives what this hub builds up to the exact preview gate where our sandbox was stopped.
