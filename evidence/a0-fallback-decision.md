Gate A0 outcome: FALLBACK branch taken (A_SP_DEDICATED=false), per the plan's classification rule.
Evidence: Graph POST /v1.0/applications SUCCEEDED (app created, then deleted as orphan);
Graph POST /v1.0/servicePrincipals DENIED with Authorization_RequestDenied
("the backing application of the service principal being created must in the local tenant"
— the Graph error signature of an SP lacking rights to create servicePrincipal objects).
=> The sandbox SP cannot mint a dedicated cluster SP. Cluster A reuses the sandbox SP
(RHDP's own quickstart pattern; SP-per-cluster uniqueness holds: no other ARO cluster uses it).
Consequences applied: Section 8 rotation demo SKIPPED (protects the RHDP credential);
cleanup step 4 (app delete) skipped; secret-expiry evidence via Graph READ still applies.
Also: azure-cli 2.84.0/Py3.14 client bug made create-for-rbac unusable (see deviation-a0-cli-bug.md);
the Graph-API route above was the substitute and produced the authoritative authorization answer.

ORPHAN (manual action item for review): app 'aro-ftcqh-spoke-a-sp'
(appId add45f0c-ce56-49ff-a91d-31466c088d58, objectId 4cb7e821-6a61-4bf0-b7da-db366c677bab)
remains in tenant redhat0.onmicrosoft.com. Our delete was DENIED (Insufficient privileges —
sandbox SP can create but not delete apps). The object is INERT: no servicePrincipal exists,
no credential was ever added, so nothing can authenticate as it. It must be removed by a
tenant admin (RHDP subscription teardown does not touch tenant-level Entra objects).
