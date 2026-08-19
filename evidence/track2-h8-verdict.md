# H8 verdict — Track 2 (executed 2026-08-18 00:30Z–02:21Z)
1. PATH S (SP secret): PASS. gen.sh CI mode; ASO event `Using credential from "aro-clusters/aso-credential"`;
   RG spk1-ftcqh-resgroup Succeeded; 13 UAMIs; 28/28 RoleAssignments Ready. Secret present on hub (4-key aso-credential + cluster-identity-secret).
2. PATH W (workload identity): PASS — IDENTICAL Azure outcome, ZERO secrets on hub. 3-key aso-credential,
   no cluster-identity-secret, AzureClusterIdentity type WorkloadIdentity; FICs on capz-hub-mi/aso-hub-mi
   against hub issuer https://eastus.oic.aro.azure.com/...; RG spk2-ftcqh-resgroup Succeeded; 13 UAMIs; 28/28 RAs Ready.
   => Open question 3 ANSWERED: MCE 2.11's stolostron ASO fork DOES honor per-namespace secretless credential fallback.
   No pod patching was needed: token projection pre-wired (CAPZ /var/run/secrets/azure/tokens, ASO /var/run/secrets/tokens, audience api://AzureADTokenExchange).
3. Gated boundary (H6): embedded infra (48 resources incl. delegated integration subnet + 29 RAs) converged;
   HcpOpenShiftCluster PUT rejected by ARM: 404 InvalidResourceType for api-version 2024-06-10-preview
   (hcpOpenShiftClusters absent from RP manifest; HcpPrivatePreview NotRegistered). Typed provider error = credential authenticated.
4. NEW FINDINGS for upstream (report to Marek):
   a. azure-cli 2.84.0 + Python 3.14: `az ad sp create-for-rbac`/`credential reset` crash client-side (argparse %Y bug).
   b. repo aro-template.yaml authors v1api20251223preview-era fields (platform.vnetIntegrationSubnetReference,
      etcd...kms.vaultName) not declared in MCE 2.11's v1api20240610preview CRD schema -> CAPZ 'failed to create
      typed patch object', HcpOpenShiftCluster never created. Fix: ARO_HCP_VERSION=v1api20240610preview selects
      the older template variant; also NEVER use the CRD *storage* version name for authoring.
   c. Subnet delegation Microsoft.RedHatOpenShift/hcpOpenShiftClusters IS accepted by ARM in a subscription
      without the HCP preview (integration subnet + its 29th RoleAssignment went Ready).
   d. Sandbox SP can create Entra apps but NOT servicePrincipals (Authorization_RequestDenied) and cannot delete
      apps -> dedicated-SP path unavailable in RHDP openenv; orphan app add45f0c-... needs tenant-admin deletion.
5. Track 1 verdict criteria all met: MI cluster built with only the 20 documented least-privilege assignments in
   <3 min setup; UserAssigned identity + 8-key pwi profile + RP-created FICs; no client secret in-cluster;
   pod-identity-webhook present; SP cluster shows finite secret expiry 2027-08-17 (liability on record).
RECOMMENDATION (evidence-backed): managed identities for new classic ARO spokes; workload identity for MCE hub
controllers (SP secret only where no public OIDC issuer exists); ARO-HCP spokes are UAMI-only by construction.
