# Track 1: build one SP cluster and one managed-identity cluster, then diff them

**What this covers / what you need before starting.** This is the classic-ARO half of the prove-out: we build two ARO 4.19.24 clusters side by side in the same subscription — `aro-spoke-a` with a service principal, `aro-spoke-b` with user-assigned managed identities — and then diff every observable difference between them: what `az aro show` reports, what's inside the operator credential secrets, which pods run, what the OIDC issuer looks like, and who owns the rotation problem. Every command here ran for real on 2026-08-18 in a disposable RHDP Azure sandbox; observed outputs are quoted from that run. You need: an Azure subscription you can make a mess in, azure-cli **2.84.0 or newer** (hard floor for the managed-identity flags), the `oc` CLI and `jq` for the verification steps, and — this is the one that catches people — a role that includes `Microsoft.Authorization/roleAssignments/write`. Plain Contributor does not have it, and the managed-identity path creates 20 role assignments before the cluster create even starts. If Azure identity words like "service principal" or "federated credential" are new to you, read the [identity primer](01-identity-models-explained.md) first; the [README](../README.md) has the full doc map, and every command and gate in its original runbook form lives in [the master plan-and-results document](../reference/full-plan-and-results.md).

## Caveats first

- **This decision is create-time and irreversible.** Microsoft's docs are explicit: an existing SP cluster can't be migrated to managed identities, and an MI cluster can't go back. Every cluster you create as SP today is a future rebuild. That's the whole reason we ran this prove-out before standardizing.
- **The managed-identity feature is GA (2026-02-02), but CLI 2.84.0 still stamps the flags `[Preview]`.** Every `--enable-managed-identity` and `--assign-platform-workload-identity` invocation in our run printed a preview warning. It's cosmetic lag between the CLI metadata and the GA announcement — the create succeeded and behaved exactly as the GA docs describe. Don't let the warnings scare you off; do capture them in your logs so you can tell reviewers you saw them too.
- **You may not be able to mint a dedicated service principal.** Creating an SP is an Entra ID (Azure's identity directory — think of it as the tenant-wide LDAP) write, and it's governed by directory permissions, not by your Azure subscription role. Our sandbox could create Entra *applications* but was denied creating *servicePrincipals* (`Authorization_RequestDenied`), so the SP cluster reused the sandbox-provided SP — which is exactly what RHDP's own quickstart does. Details below. The managed-identity path never touches Entra application objects, which is itself a point in its favor.
- **Watch your VM family quota before you start.** `az aro create` defaults to `Standard_D8s_v5` masters and `Standard_D4s_v5` workers. Our sandbox capped the Dsv5 family at 10 vCPUs total — the default sizes would have failed mid-create. We passed `--master-vm-size Standard_D8s_v3 --worker-vm-size Standard_D4s_v3` instead (the DSv3 family had a 1024-vCPU limit). Check first; a quota failure 40 minutes into a create is a bad afternoon.
- **The MI cluster ships without the Azure File StorageClass.** `azurefile-csi` needs storage-account shared keys, which the workload-identity model doesn't do. This is a documented limitation, and we confirmed it. If you depend on RWX Azure File volumes, factor that in before choosing MI.

## The two models in one paragraph each

**Service principal:** a bot account with a password. At create time the ARO resource provider (the "RP" — Microsoft's managed service that actually builds and babysits your cluster, the way an operator babysits a CR) stores one SP's client ID and client secret into the cluster, and every cloud-touching operator — machine-api creating VMs, ingress creating load balancers, the CSI drivers creating disks — authenticates to Azure with that same secret. One identity, broad permissions, and a password that expires in about a year. The password is the problem: when it expires, cloud operations fail with 401s until someone rotates it, and nothing alerts you beforehand unless you built that alerting.

**User-assigned managed identities + workload identity:** nine passwordless identities, one per job. A managed identity is an Entra identity whose credential is a certificate Azure holds and rolls automatically — no human ever sees a secret. "User-assigned" means it's a standalone Azure resource you create and own, rather than one welded to a single VM. Workload identity is how pods use them: Azure accepts the pod's projected service-account token the same way your API server does, because the cluster's OIDC issuer got federated to Entra. Each operator gets its own identity with a narrowly scoped role, so the blast radius of any one operator is its subnets, not the whole resource group.

## Before you touch anything: prerequisites and the write canary

Set up your environment variables once and reuse them everywhere. Our values are shown as comments; substitute your own.

```bash
export LOCATION=eastus
export RESOURCEGROUP=<your-resource-group>          # ours: openenv-ftcqh
export SUBSCRIPTION_ID=$(az account show --query id -o tsv)
export VERSION=4.19.24
export MASTER_SIZE=Standard_D8s_v3                   # DSv5 was quota-capped at 10 vCPUs in our sandbox
export WORKER_SIZE=Standard_D4s_v3
export CLUSTER_A=aro-spoke-a                         # service-principal cluster
export VNET_A=aro-spoke-a-vnet                       # 10.0.0.0/22
export CLUSTER_B=aro-spoke-b                         # managed-identity cluster
export VNET_B=aro-spoke-b-vnet                       # 10.1.0.0/22
```

Nothing to see yet — these just have to be set in every shell you run the rest from.

### Gate 1: CLI version, ARO version, providers

This checks the four things that would silently doom you later: CLI floor, the OCP version being available in your region, and the resource providers being registered (a "provider" is the ARM plugin that serves a resource type — like a CRD being installed; if `Microsoft.RedHatOpenShift` isn't registered, ARO resources don't exist in your subscription).

```bash
az version --query '"azure-cli"' -o tsv                      # need >= 2.84.0 for the MI flags
az aro get-versions --location "$LOCATION" -o tsv | grep -x "$VERSION"
for p in Microsoft.RedHatOpenShift Microsoft.ManagedIdentity Microsoft.Network \
         Microsoft.Compute Microsoft.Storage Microsoft.Authorization; do
  printf '%s: ' "$p"; az provider show -n "$p" --query registrationState -o tsv
done
```

In our run: `2.84.0`, `4.19.24` present in eastus, and all six providers `Registered`. If a provider says `NotRegistered`, fix it now with `az provider register -n <name>` — it takes minutes and requires no approval (unlike preview feature flags, which is a Track 2 story).

### Gate 2: quota

This shows your vCPU headroom for the family you plan to use. Two concurrent clusters peak at roughly 88 vCPUs during install (each cluster: 3 masters + a transient bootstrap VM + 3 workers).

```bash
az vm list-usage --location "$LOCATION" \
  --query "[?contains(name.value,'standardDSv3Family')].{limit:limit,cur:currentValue}" -o table
```

In our run this returned `Limit 1024, Cur 0` for DSv3 — comfortable. The same query against `standardDSv5Family` showed the 10-vCPU cap that forced us off the defaults. Also worth a glance: public IP quota (`az network list-usages`) — each public cluster consumes several; ours showed `PublicIPAddresses 0/10`, enough for two clusters but with no slack.

### Gate 3: the write canary

Before you burn 40 minutes on a cluster create, prove in 30 seconds that you can actually create an identity and write a role assignment. A role assignment is Azure RBAC's binding object — like a RoleBinding, but subscription-side — and writing one requires `Microsoft.Authorization/roleAssignments/write`, which Contributor does not include. Your role *definition* can also lie to you: a management-group Azure Policy or deny assignment can block writes that the role nominally allows. The canary settles it empirically.

```bash
az identity create -g "$RESOURCEGROUP" -n smoke-canary -o none
CANARY_OID=$(az identity show -g "$RESOURCEGROUP" -n smoke-canary --query principalId -o tsv)
az role assignment create --assignee-object-id "$CANARY_OID" --assignee-principal-type ServicePrincipal \
  --role Reader --scope "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCEGROUP" -o none
az role assignment delete --assignee "$CANARY_OID" --role Reader \
  --scope "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCEGROUP"
az identity delete -g "$RESOURCEGROUP" -n smoke-canary
```

All four commands should succeed. In our run the whole sequence took 23 seconds and the gate log ends with `GATE 0.2 PASS: identity create + role-assignment write + cleanup all succeeded`. Any `AuthorizationFailed` here is a hard stop — get the permission fixed before proceeding, because the MI path does this twenty more times. One propagation note that applies to this whole doc: right after creating any Entra principal, the first command that references it can fail with `PrincipalNotFound`. That's Entra's directory replication lag, not a permissions problem — retry the failed command every ~30 seconds for up to about 5 minutes before treating it as real.

One more optional-but-recommended item: a Red Hat pull secret (from console.redhat.com) passed via `--pull-secret @$HOME/pull-secret.txt` at create time. Without it both clusters still build, but the `redhat-operators` catalog content is reduced — and if you ever want the cluster to become an MCE hub like ours did in Track 2, you'll want it wired in from the start rather than merged afterwards.

## Path A: the service-principal cluster

### The cluster SP — and the fallback that bit us

The docs want a dedicated SP per cluster ("service principals must be unique per Azure Red Hat OpenShift cluster") with Contributor on the resource group. Creating one is normally a single command.

```bash
SP_JSON=$(az ad sp create-for-rbac --name "aro-spoke-a-sp" --role Contributor \
  --scopes "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCEGROUP" -o json)
export CLUSTER_A_CLIENT_ID=$(jq -r .appId <<<"$SP_JSON")       # the appId GUID
export CLUSTER_A_CLIENT_SECRET=$(jq -r .password <<<"$SP_JSON") # the one-time password — never echo or log it
unset SP_JSON
```

You should end up with both variables set: `$CLUSTER_A_CLIENT_ID` holds the new SP's appId (a GUID — safe to print), and `$CLUSTER_A_CLIENT_SECRET` holds its password, which Azure shows you exactly once here and never again. The validate and create commands below consume both. **In our run this failed twice, in two different and instructive ways:**

1. **A client-side CLI crash before Azure was ever contacted.** azure-cli 2.84.0 on Python 3.14 dies inside argparse with `ValueError: unsupported format character 'Y'` — a literal `%Y` in a help string that Python 3.14 no longer tolerates. `az ad sp credential reset` crashes the same way. This is a local tooling bug, not an authorization answer, so we worked around it by calling Microsoft Graph directly with `az rest` (Graph is Entra's REST API — `create-for-rbac` is really just three Graph/ARM calls in a trench coat: create application, create servicePrincipal, create role assignment).
2. **A genuine Entra permission wall.** Via Graph, the application create *succeeded*, but the servicePrincipal create was denied with `Authorization_RequestDenied`. Translation: our sandbox SP can register apps in the `redhat0.onmicrosoft.com` tenant but cannot create SP objects. No subscription role fixes this — it's a directory permission.

So we took the documented fallback: reuse the sandbox-provided SP (from your sandbox credentials file) as the cluster SP. RHDP's own quickstart does exactly this, and the uniqueness rule holds as long as no other ARO cluster uses it. On this branch you still have to fill the same two variables, from the credentials file instead of a fresh SP — the file's client/application ID (a GUID, sometimes labeled `appId`) is the ID, and its password/secret field is the secret:

```bash
export CLUSTER_A_CLIENT_ID="<the sandbox SP's client id (appId) from your credentials file>"
export CLUSTER_A_CLIENT_SECRET="<the sandbox SP's password from your credentials file>"  # never echo or log this
```

After this, `echo $CLUSTER_A_CLIENT_ID` should print a GUID; don't echo the secret — the validate step below will tell you soon enough if the pair is wrong (a 401 `invalid_client` means the secret; a missing-argument error means one of the variables is empty). The sandbox SP is wildly over-privileged for the job — which, note for later, is itself a data point in the blast-radius column of the final diff. One cleanup wrinkle: the half-created application from step 2 is a *tenant-level* orphan that subscription teardown will never remove, and our sandbox SP couldn't delete it either. It's inert (no SP, no credential), but a tenant admin has to remove it. If you script SP creation, always check for and clean up partial `create-for-rbac` failures.

The takeaway for your real environment: the SP path has a hidden prerequisite — Entra rights to create app registrations — that the MI path simply doesn't have.

### Network

ARO wants a vnet (Azure's VPC) with two subnets: one for control-plane nodes, one for workers. This creates all three.

```bash
az network vnet create -g "$RESOURCEGROUP" -n "$VNET_A" --address-prefixes 10.0.0.0/22 -o none
az network vnet subnet create -g "$RESOURCEGROUP" --vnet-name "$VNET_A" -n master-subnet --address-prefixes 10.0.0.0/23 -o none
az network vnet subnet create -g "$RESOURCEGROUP" --vnet-name "$VNET_A" -n worker-subnet --address-prefixes 10.0.2.0/23 -o none
az network vnet subnet list -g "$RESOURCEGROUP" --vnet-name "$VNET_A" \
  --query '[].{n:name,p:addressPrefix}' -o table
```

The final list should show both subnets with their /23s. In our run: `master-subnet 10.0.0.0/23`, `worker-subnet 10.0.2.0/23`. (Depending on the network API version, the CIDR may land in the plural `addressPrefixes` array instead of the singular field — query both if the column comes back blank.) If you've seen older ARO guides demanding `--service-endpoints` on the subnets or disabling private-link service network policies: both requirements were dropped in 2023, and the RP handles the private-link policies itself now. We omitted both and everything worked.

### Validate, then create

`az aro validate` is a free pre-flight — same argument shape as the create, checks permissions and network layout, costs nothing. Run it before every create; it turns a 40-minute failure into a 30-second one.

```bash
az aro validate -g "$RESOURCEGROUP" -n "$CLUSTER_A" \
  --vnet "$VNET_A" --master-subnet master-subnet --worker-subnet worker-subnet \
  --version "$VERSION" \
  --client-id "$CLUSTER_A_CLIENT_ID" --client-secret "$CLUSTER_A_CLIENT_SECRET"
```

Expect no failed checks; warnings are informational. In our run it reported two warnings of the same shape: `Resource aro-spoke-a-vnet is missing role assignment 4d97b98b-1d4f-4787-a291-c67834d212e7 for service principal ... (These roles will be automatically added during cluster creation)`. That GUID is the Network Contributor role, and the two principals were our cluster SP and the ARO RP's first-party SP — more on both in a second. This warning is expected and self-healing.

Now the create. `--no-wait` returns immediately; you poll for completion. Never sit blocking on a 40-minute foreground command.

```bash
az aro create -g "$RESOURCEGROUP" -n "$CLUSTER_A" \
  --vnet "$VNET_A" --master-subnet master-subnet --worker-subnet worker-subnet \
  --version "$VERSION" \
  --master-vm-size "$MASTER_SIZE" --worker-vm-size "$WORKER_SIZE" --worker-count 3 \
  --client-id "$CLUSTER_A_CLIENT_ID" --client-secret "$CLUSTER_A_CLIENT_SECRET" \
  --apiserver-visibility Public --ingress-visibility Public \
  --no-wait
```

Before submitting, the CLI does something worth knowing about: it auto-grants **Network Contributor on your vnet** to two principals — your cluster SP, and the **ARO RP first-party service principal** (`Azure Red Hat OpenShift RP`, a Microsoft-owned SP that exists in every tenant; "first-party" means Microsoft operates it, and it's how the managed service reaches into your network to build load balancers and manage the private link). This is why the create needs `roleAssignments/write` even on the SP path. In our run the launch log shows exactly those two grants being promised, and after the create finished the vnet's role-assignment list confirmed both landed:

```json
[
  {"who": "980772f8-...", "role": "Network Contributor"},   // our cluster SP
  {"who": "f1dd0a37-...", "role": "Network Contributor"}    // Azure Red Hat OpenShift RP
]
```

Poll until done — each poll is a cheap one-liner, every few minutes.

```bash
az aro show -g "$RESOURCEGROUP" -n "$CLUSTER_A" --query provisioningState -o tsv
```

You want `Succeeded`. In our run, cluster A launched at 00:40:57Z and reached `Succeeded` in about 40 minutes. On `Failed`: capture `az aro show -o json`, delete the cluster (the name can't be reused until you do), fix the cause, retry once.

## Path B: the managed-identity cluster

We built B concurrently with A — the `--no-wait` launch of A means you can run this entire section while A provisions. B gets its own vnet (`10.1.0.0/22`) so both clusters coexist.

### Network

Same shape as Path A, different address space.

```bash
az network vnet create -g "$RESOURCEGROUP" -n "$VNET_B" --address-prefixes 10.1.0.0/22 -o none
az network vnet subnet create -g "$RESOURCEGROUP" --vnet-name "$VNET_B" -n master-subnet --address-prefixes 10.1.0.0/23 -o none
az network vnet subnet create -g "$RESOURCEGROUP" --vnet-name "$VNET_B" -n worker-subnet --address-prefixes 10.1.2.0/23 -o none
```

Gate it the same way as A1; in our run: `master-subnet 10.1.0.0/23`, `worker-subnet 10.1.2.0/23`.

### The nine identities

One cluster identity plus eight operator identities, named after the exact operator keys the CLI expects (these names match Microsoft's MI how-to verbatim, which makes diffing against the official docs trivial):

| Identity | Used by |
|---|---|
| `aro-cluster` | the cluster identity — the RP acts as this to wire up everything else |
| `cloud-controller-manager` | node/LB lifecycle against Azure |
| `ingress` | ingress load balancers |
| `machine-api` | creating and deleting worker VMs |
| `disk-csi-driver` | managed-disk volumes |
| `cloud-network-config` | secondary IP config for OVN |
| `image-registry` | the registry's storage account |
| `file-csi-driver` | Azure File volumes |
| `aro-operator` | ARO's own in-cluster operator |

Create them all — this is nine fast control-plane writes, no VMs involved.

```bash
OPERATOR_IDS=(cloud-controller-manager ingress machine-api disk-csi-driver \
              cloud-network-config image-registry file-csi-driver aro-operator)
for ID in aro-cluster "${OPERATOR_IDS[@]}"; do
  az identity create -g "$RESOURCEGROUP" -n "$ID" -o none
done
az identity list -g "$RESOURCEGROUP" --query 'length(@)' -o tsv
```

The count should be `9`. In our run it was, and the nine creates took about 90 seconds. Also record `az identity list --query "[].[name,principalId]" -o tsv` somewhere now — after a future `--delete-identities` teardown you can no longer resolve names to principal IDs, and orphan-assignment cleanup needs them.

### The 20 role assignments, and the logic behind them

This is the part that looks intimidating and isn't. The scoping logic in plain words:

- **Subnet-scoped operators (8 assignments).** Four operators — `cloud-controller-manager`, `ingress`, `machine-api`, `aro-operator` — each get their purpose-built ARO role on the master subnet and the worker subnet. That's it. The machine-api identity can join VMs to those two subnets and nothing else in your network.
- **Vnet-scoped operators (3 assignments).** `cloud-network-config`, `image-registry`, and `file-csi-driver` need vnet-wide visibility, so their roles land at the vnet.
- **`disk-csi-driver` gets nothing pre-create.** Its permissions live in the cluster's managed resource group, which the RP grants during install. It still needs its identity — just no assignment from you.
- **`aro-cluster` gets the "Federated Credential" role over each of the 8 operator identities (8 assignments).** This is the clever one. During install, the RP — acting as the cluster identity — must write a *federated identity credential* (FIC) onto every operator identity. A FIC is the Entra object that says "accept tokens from this OIDC issuer for this service-account subject" — it's the trust anchor of the whole workload-identity scheme, the reason Azure will honor a pod's projected token. Writing a FIC onto an identity is a write *on that identity resource*, so the cluster identity needs a role scoped to each operator identity. Least privilege, applied to the identities themselves.
- **The ARO RP first-party SP gets "First Party Network" on the vnet (1 assignment).** Same Microsoft-operated RP principal as in Path A, but here it gets a narrow ARO-specific network role instead of full Network Contributor.

8 + 3 + 8 + 1 = 20. Here's the whole thing, using the role-definition GUIDs from Microsoft's own doc examples (assign by GUID — role display names can drift, GUIDs can't):

```bash
SCOPE_VNET_B="/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCEGROUP/providers/Microsoft.Network/virtualNetworks/$VNET_B"
SCOPE_MASTER_B="$SCOPE_VNET_B/subnets/master-subnet"
SCOPE_WORKER_B="$SCOPE_VNET_B/subnets/worker-subnet"

ROLE_CCM=a1f96423-95ce-4224-ab27-4e3dc72facd4         # ARO Cloud Controller Manager
ROLE_INGRESS=0336e1d3-7a87-462b-b6db-342b63f7802c     # ARO Cluster Ingress Operator
ROLE_MACHINE_API=0358943c-7e01-48ba-8889-02cc51d78637 # ARO Machine API Operator
ROLE_NETWORK=be7a6435-15ae-4171-8f30-4a343eff9e8f     # ARO Network Operator
ROLE_IMAGE_REG=8b32b316-c2f5-4ddf-b05b-83dacd2d08b5   # ARO Image Registry Operator
ROLE_FILE_CSI=0d7aedc0-15fd-4a67-a412-efad370c947e    # ARO File Storage Operator
ROLE_ARO_OP=4436bae4-7702-4c84-919b-c4069ff25ee2      # ARO Service Operator
ROLE_FED_CRED=ef318e2a-8334-4a05-9e4a-295a196c6a6e    # ARO Federated Credential
ROLE_FP_NET=42f3c60f-e7b1-46d7-ba56-6de681664342      # ARO First Party Network

assign() {  # assign <identity-name> <role-guid> <scope>
  az role assignment create \
    --assignee-object-id "$(az identity show -g "$RESOURCEGROUP" -n "$1" --query principalId -o tsv)" \
    --assignee-principal-type ServicePrincipal \
    --role "/subscriptions/$SUBSCRIPTION_ID/providers/Microsoft.Authorization/roleDefinitions/$2" \
    --scope "$3" -o none
}

# subnet-scoped operators: master + worker each
for S in "$SCOPE_MASTER_B" "$SCOPE_WORKER_B"; do
  assign cloud-controller-manager "$ROLE_CCM"         "$S"
  assign ingress                  "$ROLE_INGRESS"     "$S"
  assign machine-api              "$ROLE_MACHINE_API" "$S"
  assign aro-operator             "$ROLE_ARO_OP"      "$S"
done
# vnet-scoped operators
assign cloud-network-config "$ROLE_NETWORK"   "$SCOPE_VNET_B"
assign image-registry       "$ROLE_IMAGE_REG" "$SCOPE_VNET_B"
assign file-csi-driver      "$ROLE_FILE_CSI"  "$SCOPE_VNET_B"
# cluster identity: Federated Credential role over each operator identity
for OP in "${OPERATOR_IDS[@]}"; do
  assign aro-cluster "$ROLE_FED_CRED" \
    "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCEGROUP/providers/Microsoft.ManagedIdentity/userAssignedIdentities/$OP"
done
# ARO RP first-party SP: First Party Network on the vnet
az role assignment create \
  --assignee-object-id "$(az ad sp list --display-name 'Azure Red Hat OpenShift RP' --query '[0].id' -o tsv)" \
  --assignee-principal-type ServicePrincipal \
  --role "/subscriptions/$SUBSCRIPTION_ID/providers/Microsoft.Authorization/roleDefinitions/$ROLE_FP_NET" \
  --scope "$SCOPE_VNET_B" -o none
```

Verify the counts per scope before moving on — this is your cheap gate against a typo'd scope:

```bash
az role assignment list --scope "$SCOPE_MASTER_B" --query 'length(@)'   # expect 4
az role assignment list --scope "$SCOPE_WORKER_B" --query 'length(@)'   # expect 4
az role assignment list --scope "$SCOPE_VNET_B"   --query 'length(@)'   # expect 4 (3 operators + the RP)
```

In our run: `4 / 4 / 4`, and each of the eight operator identities showed exactly `1` assignment at its own scope (aro-cluster's Federated Credential role). All 20 landed first-try in about 90 seconds. Two practical notes: if the *first* assignment fails with `PrincipalNotFound`, that's the Entra propagation lag from the canary section — retry it, don't panic. And if you have to re-run the whole block, "role assignment already exists" errors are skips, not failures.

Total identity-plus-RBAC setup time in our run, timestamped from first identity create to last role assignment: **2 minutes 29 seconds**. That's the entire extra cost of the managed-identity model over SP. Hold that number against "removes the secret-expiry outage class forever" when you make this call.

### Validate, then create

Same free pre-flight as Path A, now with the MI flags. If validate reports missing permissions that your gate just showed as present, that's RBAC propagation lag — re-run every couple of minutes; treat it as real only if it persists past ~15 minutes.

```bash
az aro validate -g "$RESOURCEGROUP" -n "$CLUSTER_B" \
  --vnet "$VNET_B" --master-subnet master-subnet --worker-subnet worker-subnet \
  --version "$VERSION" \
  --enable-managed-identity \
  --assign-cluster-identity aro-cluster \
  --assign-platform-workload-identity file-csi-driver file-csi-driver \
  --assign-platform-workload-identity cloud-controller-manager cloud-controller-manager \
  --assign-platform-workload-identity ingress ingress \
  --assign-platform-workload-identity image-registry image-registry \
  --assign-platform-workload-identity machine-api machine-api \
  --assign-platform-workload-identity cloud-network-config cloud-network-config \
  --assign-platform-workload-identity aro-operator aro-operator \
  --assign-platform-workload-identity disk-csi-driver disk-csi-driver
```

In our run this printed ten `WARNING: Argument '--...' is in preview and under development` lines — one per MI flag — and **no failed checks**. That's the GA-feature-with-preview-CLI-metadata lag from the caveats; the warnings are the entire preview footprint we observed. A persistent permissions failure here almost always means one of the 20 assignments is missing or mis-scoped — go back to the count gate.

Each `--assign-platform-workload-identity KEY IDENTITY` pair maps an operator key to the identity you created; using identical names for both (as the official doc does) keeps it readable. Now create:

```bash
az aro create -g "$RESOURCEGROUP" -n "$CLUSTER_B" \
  --vnet "$VNET_B" --master-subnet master-subnet --worker-subnet worker-subnet \
  --version "$VERSION" \
  --master-vm-size "$MASTER_SIZE" --worker-vm-size "$WORKER_SIZE" --worker-count 3 \
  --apiserver-visibility Public --ingress-visibility Public \
  --pod-cidr 10.132.0.0/14 --service-cidr 172.31.0.0/16 \
  --enable-managed-identity \
  --assign-cluster-identity aro-cluster \
  --assign-platform-workload-identity file-csi-driver file-csi-driver \
  --assign-platform-workload-identity cloud-controller-manager cloud-controller-manager \
  --assign-platform-workload-identity ingress ingress \
  --assign-platform-workload-identity image-registry image-registry \
  --assign-platform-workload-identity machine-api machine-api \
  --assign-platform-workload-identity cloud-network-config cloud-network-config \
  --assign-platform-workload-identity aro-operator aro-operator \
  --assign-platform-workload-identity disk-csi-driver disk-csi-driver \
  --pull-secret @$HOME/pull-secret.txt \
  --no-wait
```

Same ten preview warnings, then it returns. No `--client-id`, no `--client-secret` — no Azure credential is passed anywhere on this path, which is the point. The `--pull-secret` line is there because our run passed it: B was destined to become the Track 2 MCE hub, and [doc 03](03-hub-mce-setup.md) explains why a hub needs it at create time — a cluster born without it ships the `redhat-operators` catalog disabled, and the retrofit is slow. Drop the flag only if this cluster will never need Red Hat operator content. (The non-default pod/service CIDRs are optional and unrelated to identity: cluster A keeps the defaults, and giving B distinct CIDRs is free insurance in case these spokes ever join a Submariner mesh — CIDRs are immutable after create.) Poll exactly as before:

```bash
az aro show -g "$RESOURCEGROUP" -n "$CLUSTER_B" --query provisioningState -o tsv
```

In our run, B launched at 00:46:39Z and reached `Succeeded` in about 65 minutes — noticeably longer than A's ~40, since the RP has more identity wiring to do. Both creates ran concurrently, so the pair cost us ~70 minutes of wall clock, not 105.

## The payoff: diff the two clusters

Everything below is a real observed contrast from our run. Grab the Azure-side view first, then log into each cluster with `oc` (API URL from `az aro show --query apiserverProfile.url`, kubeadmin password from `az aro list-credentials` — treat that password like the secret it is; keep it out of logs).

### Azure's view: `az aro show`

Ask each cluster to describe its identity model.

```bash
az aro show -g "$RESOURCEGROUP" -n "$CLUSTER_A" \
  --query '{state:provisioningState,spProfile:servicePrincipalProfile,identity:identity,pwi:platformWorkloadIdentityProfile}' -o json
```

In our run, cluster A returned a populated `servicePrincipalProfile` (`clientId` set, `clientSecret` masked to null in output) with `identity` and `pwi` both `null`. Cluster B was the mirror image: `spProfile: null`, `identity.type: "UserAssigned"` with `aro-cluster` under `userAssignedIdentities`, and a `platformWorkloadIdentityProfile` mapping all eight operator keys — each to that identity's `resourceId`, `clientId`, and `objectId`. Two disjoint schemas; a cluster is one or the other, never both, never convertible.

### In-cluster: what's actually in the operator credential secrets

This is the ground truth the Azure-side profile promises. Check what keys each operator's cloud-credential secret carries.

```bash
oc get secret azure-cloud-credentials -n openshift-machine-api -o jsonpath='{.data}' | jq 'keys'
```

In our run, on cluster A:

```json
["azure_client_id", "azure_client_secret", "azure_region", "azure_resource_prefix",
 "azure_resourcegroup", "azure_subscription_id", "azure_tenant_id"]
```

On cluster B:

```json
["azure_client_id", "azure_federated_token_file", "azure_region",
 "azure_subscription_id", "azure_tenant_id"]
```

Read that pair of lists twice — it's the whole argument in eleven strings. A carries `azure_client_secret`: a password, sitting in a Secret, expiring on a clock. B carries `azure_federated_token_file` instead: a *path* to a projected service-account token that the kubelet re-issues continuously and Azure trusts via federation. There is no client secret anywhere in B; we checked the image-registry operator's secret too and got the same shape.

### The webhook that makes it work

Workload identity needs something to inject the token-file path and volume into operator pods. That's `pod-identity-webhook`, a mutating admission webhook — same mechanism as any mutating webhook you've dealt with.

```bash
oc get pods -n openshift-cloud-credential-operator -o name
```

In our run, cluster A showed only `pod/cloud-credential-operator-...`. Cluster B showed the CCO pod plus two `pod/pod-identity-webhook-...` replicas. Present only on MI — a quick, decisive fingerprint of the model.

### The issuer and the CCO mode

Two more one-liners that tell you which world you're in.

```bash
oc get authentication cluster -o jsonpath='{.spec.serviceAccountIssuer}{"\n"}'
oc get cloudcredential cluster -o jsonpath='{.spec.credentialsMode}{"\n"}'
```

Cluster A: both empty — in-cluster default issuer, default CCO mode. Cluster B in our run:

```text
https://eastus.oic.aro.azure.com/64dc69e4-d083-49fc-9569-ebece1dd1408/3daaa256-9628-4bf6-a438-988f1b34ed3f
Manual
```

That issuer is a *public* OIDC endpoint Azure hosts for the cluster (format: `https://eastus.oic.aro.azure.com/<tenant>/<uuid>`) — Entra fetches its JWKS from there to verify pod tokens. `Manual` mode means the cloud-credential operator isn't minting credentials itself; the identity wiring came from the platform. This same public issuer is what the console's "uses Microsoft Entra Workload ID" banner reflects — and it's also what made cluster B reusable as a workload-identity MCE hub in Track 2 with zero issuer surgery.

### The federated credentials the RP created for you

Remember the `aro-cluster` identity holding the Federated Credential role over each operator identity? Here's what it did with it.

```bash
for OP in "${OPERATOR_IDS[@]}"; do
  az identity federated-credential list --identity-name "$OP" -g "$RESOURCEGROUP" -o json
done
```

In our run this returned **12 RP-created federated credentials across the 8 operator identities** — several identities carry more than one, because one Azure identity can serve multiple in-cluster service accounts. `file-csi-driver` is the busiest, with FICs for its operator, controller, and node service accounts; `disk-csi-driver` and `image-registry` carry two each (operator plus controller/registry SA); here's one verbatim:

```json
{
  "issuer": "https://eastus.oic.aro.azure.com/64dc69e4-.../3daaa256-...",
  "subject": "system:serviceaccount:openshift-cluster-csi-drivers:azure-file-csi-driver-operator",
  "audiences": ["openshift"]
}
```

Issuer = the cluster's public OIDC endpoint from the previous check. Subject = a plain Kubernetes service account, written the way you'd write it in an RBAC binding. The audience on all 12 of these RP-created FICs is `openshift` — ARO's RP uses that string for in-cluster operators, while the hub-controller federation in Track 2 uses Azure's generic `api://AzureADTokenExchange`; same mechanism, different agreed-on string. That's the entire trust contract, and you didn't create any of it — the RP did, using exactly the role you scoped for it.

### The liability you keep on the SP path

For cluster A, look up when its secret dies.

```bash
az ad app credential list --id "$CLUSTER_A_CLIENT_ID" --query '[].endDateTime' -o tsv
```

In our run this returned `2027-08-17T18:36:18Z` — one year out. Put it in your calendar, because Azure won't: there is no built-in expiry alerting, and when that date passes, cluster A's cloud operators start failing with 401 `invalid_client` until someone runs `az aro update --refresh-credentials` (which itself can take up to two hours). This date is the thing you now own monitoring. Cluster B has no equivalent: Azure rolls MI certificates automatically, and the recovery command for broken FICs (a bare `az aro update`) exists but should never be needed in normal life.

### The one thing MI takes away

```bash
oc get storageclass -o name
```

Cluster A: `azurefile-csi` and `managed-csi`. Cluster B in our run: `managed-csi` only. The Azure File StorageClass is absent on MI — a documented limitation (the Files CSI driver needs storage-account shared keys, which don't fit the federated model), not a bug, and the one concrete capability trade-off we found. Disk volumes (`managed-csi`, the default) are unaffected. In both clusters, `oc get co` showed every cluster operator Available and none Degraded.

### The diff at a glance

| Check | `aro-spoke-a` (SP) | `aro-spoke-b` (MI) |
|---|---|---|
| Create time (observed) | ~40 min | ~65 min (+ <3 min identity/RBAC setup) |
| `az aro show` | `servicePrincipalProfile` set; `identity`/`pwi` null | `identity.type: UserAssigned`; all 8 operator keys in `platformWorkloadIdentityProfile`; `spProfile` null |
| Operator cred secrets | contain `azure_client_secret` | contain `azure_federated_token_file`; no client secret anywhere |
| `pod-identity-webhook` | absent | present, 2 replicas |
| `serviceAccountIssuer` / CCO mode | empty (in-cluster default) / default | public `https://eastus.oic.aro.azure.com/...` / `Manual` |
| Federated credentials | n/a | 12, created by the RP across the 8 operator identities |
| Secret expiry liability | `2027-08-17` — you own the monitoring and the rotation | none; Azure rolls MI certs |
| Azure objects you manage | 1 SP (+ its Entra app) | 9 identities + 20 role assignments, all scripted above |
| Entra directory rights needed | app-registration rights (bit us — see Path A) | none |
| Azure File StorageClass | present | absent (documented limitation) |

## Timings observed

| Step | Wall clock in our run |
|---|---|
| Write canary | ~25 s |
| Path B: 9 identities + 20 role assignments | 2 min 29 s |
| `az aro create` cluster A (SP) | ~40 min |
| `az aro create` cluster B (MI) | ~65 min |
| Both creates, run concurrently | ~70 min end to end |

For teardown, two traps we hit so you don't have to: don't delete the identities or role assignments before the cluster delete *completes* — the RP needs them during teardown; use `az aro delete --delete-identities` on the MI cluster (in our run the blocking delete finished with all 9 operator identities gone — `az identity list` on the resource group returned 0 at the final gate), but note it's incompatible with `--no-wait`, so run that delete blocking. The full cleanup sequence, including the orphan-assignment sweep, is in [the master document](../reference/full-plan-and-results.md).

## The bottom line

Both clusters built cleanly with the documented flows, and every observable difference matched the docs. The managed-identity cluster cost us under three extra minutes of setup — nine `az identity create` calls and twenty role assignments, all scripted — and in exchange there is no client secret in any operator secret, no Entra app registration to own, per-operator permissions scoped to two subnets and a vnet instead of Contributor-on-everything, and no expiry date on your calendar. The SP cluster works fine today and hands you a `2027-08-17` deadline as a housewarming gift.

And remember the caveat this doc opened with, because it's the entire decision: **this choice is made at create time and cannot be changed** — no migration exists in either direction, SP-to-MI or MI-to-SP. A spoke created as SP stays SP until you rebuild it. If you're creating new classic-ARO spokes on CLI 2.84.0 or later, create them with managed identities. (And if your "new ARO spoke" is ARO-HCP rather than classic, there's no choice to make at all — the spoke API only speaks managed identities, and the identity question moves to the hub. That's Track 2; see the [README](../README.md) doc map.)
