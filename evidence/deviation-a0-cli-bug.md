# Deviation: Gate A0 executed via Graph API (az rest) instead of az ad sp create-for-rbac
Reason: azure-cli 2.84.0 on Python 3.14 crashes client-side in `az ad sp create-for-rbac`
and `az ad sp credential reset` (argparse rejects help string containing literal %Y —
"badly formed help string"; full traceback in a0-sp-create.err). This is a local tooling
defect, NOT an Azure/Graph authorization failure, so the plan's sandbox-SP fallback does
not apply. Equivalent operations performed instead:
  1. POST /v1.0/applications            (create app)         == create-for-rbac part 1
  2. POST /v1.0/servicePrincipals       (create SP for app)  == create-for-rbac part 2
  3. az role assignment create Contributor on RG             == create-for-rbac part 3
  4. (A2+A3) POST /applications/{id}/addPassword             == credential reset
Plan intent preserved: dedicated SP, Contributor on cluster RG, secret minted only inside
the invocation that consumes it, initial secret never captured (none is ever created here).
