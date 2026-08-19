# Managed identities vs service principals for new ARO spoke clusters — a prove-out

This repo is the complete writeup of a prove-out we ran for real on 2026-08-18 in a disposable Red Hat demo-platform (RHDP) Azure sandbox. The question: when you stand up a new Azure Red Hat OpenShift (ARO) cluster as a spoke, should its cloud identity be a service principal or user-assigned managed identities? It turns out "a new ARO spoke" means two different things right now, so we proved both. Track 1 built two classic ARO 4.19.24 clusters side by side — one service-principal, one managed-identity — and diffed every observable difference. Track 2 turned the managed-identity cluster into an MCE hub and drove the declarative ARO-HCP spoke flow (CAPI/CAPZ/ASO) with both hub credential models, all the way to the exact ARM error where the private preview gate stops you. Every command, every gate output, and every trap we hit is written down. If you're comfortable with `oc`, operators, and OLM but haven't lived in Azure identity land, this repo was written for you — each Azure concept gets explained the first time it shows up.

## Read this first — caveats

Honesty up front. Some of what's in here you can use in production today, and some of it you cannot use at all yet. Know which is which before you copy a single command.

- **ARO-HCP is a gated private preview. The final create cannot succeed in a normal subscription.** The `HcpOpenShiftCluster` resource type doesn't exist in your subscription until Microsoft approves the `HcpPrivatePreview` feature flag for it, and there is no self-service signup. We drove the flow to that exact wall on purpose. In our run, ARM rejected the PUT with `404 InvalidResourceType — "The resource type could not be found in the namespace 'Microsoft.RedHatOpenShift' for api version '2024-06-10-preview'"`. Everything before that PUT — all 48 infrastructure resources, a set that includes the 13 managed identities and 29 role assignments — worked against real public Azure. The last step will fail for you too, in exactly this way, and [docs/06](docs/06-the-preview-gate.md) shows you what that looks like so you can tell "expected gate" from "you broke something".
- **The MCE path is unpublished dev-preview and NOT supported by Red Hat.** Track 2 rides the `cluster-api-provider-azure-preview` component that ships dark in MCE 2.11. The official procedure was written for rhacm-docs (PR #8616) and pulled before publication. We drive it from Marek Veber's `cluster-api-installer` repo instead. Do not open a support case about anything in Track 2.
- **Classic ARO with managed identities IS GA and supported.** GA since 2026-02-02, CLI floor 2.84.0. Track 1's managed-identity half (Path B in [docs/02](docs/02-classic-aro-walkthrough.md)) is the part of this repo you can take to production as-is.
- **This is a lab writeup, not an official Red Hat or Microsoft document.** It records what happened in one sandbox on one day. Where we cite official docs, we link them; everything else is our own observation.
- **Versions are pinned and will drift.** MCE 2.11.4 (channel `stable-2.11`), OCP 4.19.24, azure-cli 2.84.0, `cluster-api-installer` @ `ed240ab` (branch `capi-test-rebase`). Two of our findings are literally version-skew bugs (a CLI crash on Python 3.14, a template-vs-CRD api-version mismatch), so expect newer versions to behave differently — better in some places, broken in new ones.
- **You need rights most subscriptions don't hand out. This is the #1 trap.** Nearly everything here creates Azure *role assignments* — 20 for the classic MI cluster, 28–29 per HCP spoke. That needs `Microsoft.Authorization/roleAssignments/write`, which plain **Contributor does not have**. Contributor can create every resource in this repo and will still fail at the first role assignment. You need Owner, User Access Administrator, or Role Based Access Control Administrator on the scope. Check this before you start ([docs/02](docs/02-classic-aro-walkthrough.md) has the pre-flight write canary we used).
- **The cleanup steps delete real Azure resource groups.** ASO cascade-deleted each spoke resource group in about 90 seconds in our run — everything inside, gone. The classic-cluster teardown deletes clusters, vnets, and all 9 operator identities. Run this in a disposable subscription only. Never point it at a shared one.

## What we proved

Everything below happened in one ~2h05m window and is backed by a captured gate file cited in the docs.

| Claim | Evidence from the run |
|---|---|
| Classic SP cluster (`aro-spoke-a`) builds with the documented flow | `provisioningState: Succeeded` in ~40 min; `servicePrincipalProfile` set; operator secrets contain `azure_client_secret`; that secret expires 2027-08-17 and rotating it is your problem |
| Classic MI cluster (`aro-spoke-b`) builds with 9 user-assigned managed identities + 20 scoped role assignments | Succeeded in ~65 min; identity+RBAC setup took under 3 minutes; `identity.type: UserAssigned`; operator secrets contain `azure_federated_token_file` and **no client secret**; 12 RP-created federated credentials; `pod-identity-webhook` running |
| The two models are observably different in exactly the documented ways | Full diff in [docs/02](docs/02-classic-aro-walkthrough.md): CCO mode, issuer URL, credential secret keys, storage classes (Azure File SC absent on MI — known limitation, confirmed) |
| An MCE 2.11.4 hub on the MI cluster runs the CAPI/CAPZ/ASO controllers with workload-identity plumbing already wired | `capi`/`capz`/`azureserviceoperator` controllers 1/1 in ~2 min; token projection pre-configured, audience `api://AzureADTokenExchange`; CAPZ feature gate `ARO=true` |
| Hub credential path S (SP secret in hub Secrets) provisions a full HCP spoke identity set | RG Succeeded; **13 UAMIs, 28/28 role assignments Ready**; ASO event `Using credential from "aro-clusters/aso-credential"` |
| Hub credential path W (workload identity, **zero secrets on the hub**) does the identical job | Identical Azure outcome, 13 UAMIs + 28/28 Ready; 3-key `aso-credential` with no `AZURE_CLIENT_SECRET`; `cluster-identity-secret` NotFound; MCE 2.11's ASO fork honors the secretless per-namespace credential — this was our biggest open question and it's answered yes |
| The private-preview gate is exactly where the docs imply, and nowhere earlier | All 48 embedded infra resources converged (including the delegated integration subnet, which ARM accepted — a finding); only the `HcpOpenShiftCluster` PUT to `management.azure.com` failed, with the typed `404 InvalidResourceType` above. A typed provider error, not a 401 — proof the hub credential authenticated fine |

## How to read this repo

The docs are a sequence, but each one opens with "what this covers / what you need before starting" so you can jump in anywhere.

| Doc | One line |
|---|---|
| [docs/01-identity-models-explained.md](docs/01-identity-models-explained.md) | Azure identity for OpenShift people: service principals, managed identities, OIDC federation, and workload identity — anchored to concepts you already know |
| [docs/02-classic-aro-walkthrough.md](docs/02-classic-aro-walkthrough.md) | Track 1 end to end: prereqs and the write canary, the SP cluster (with the Entra-permission fallback that bit us), the MI cluster's 9 identities + 20 role assignments, and the full side-by-side diff |
| [docs/03-hub-mce-setup.md](docs/03-hub-mce-setup.md) | Track 2 begins: MCE 2.11.4 via OLM, the preview toggles, and verifying the CAPZ/ASO controllers and their token projection |
| [docs/04-hcp-spoke-sp-path.md](docs/04-hcp-spoke-sp-path.md) | Hub credential PATH S: a service-principal secret on the hub provisions the full 13-UAMI, 28-role-assignment HCP spoke footprint |
| [docs/05-hcp-spoke-workload-identity.md](docs/05-hcp-spoke-workload-identity.md) | Hub credential PATH W: the same spoke with zero secrets on the hub — including the federation steps the upstream repo doesn't document |
| [docs/06-the-preview-gate.md](docs/06-the-preview-gate.md) | The full CAPI manifest, the api-version trap, and the exact ARM 404 at the preview boundary |
| [docs/07-cleanup.md](docs/07-cleanup.md) | Tearing both tracks down in the order that works, and the teardown traps we hit |
| [docs/08-findings-and-gotchas.md](docs/08-findings-and-gotchas.md) | The five findings we'd report upstream, the permission gotchas, day-2 notes, and the open questions a sandbox can't answer |
| [reference/full-plan-and-results.md](reference/full-plan-and-results.md) | The master document: the full pre-execution plan with every command and gate, plus the executed-results summary |

Gate outputs quoted in the docs (files like `track2-h6-preview-gate.txt`, `identity-summary-aro-spoke-b.json`) come from the run's evidence bundle, archived before teardown and vendored in this repo at [`evidence/`](evidence/) — every evidence link in the docs resolves there.

## The 30-second version

A service principal is a bot account with a password. The password is the problem: it expires in about a year, someone has to rotate it, and until they do your cluster's cloud operators fail with 401s. A managed identity is an Azure-issued identity with no password anyone holds — Azure rolls its certificates itself, and workloads prove who they are by presenting projected service-account tokens that Entra trusts because you federated it to the cluster's OIDC issuer.

For **classic ARO**: pick managed identities at create time. There is no migration in either direction — a cluster born SP stays SP until you rebuild it. The MI setup cost us under 3 minutes of extra work and removed the secret-expiry outage class permanently.

For **ARO-HCP**: the spoke has no SP option at all — the API only speaks user-assigned managed identities (13 per spoke, by construction). The real choice moves up to the hub: what credential do the CAPZ and ASO controllers use to call Azure? We proved workload identity does everything the SP secret does with zero secret material on the hub, so workload identity wins. Keep an SP only where the hub has no public OIDC issuer.

## Time and cost

Our full run — plan to clean sandbox — took about 2 hours 5 minutes of wall clock. The two classic cluster creates dominate: ~40 minutes for SP, ~65 for MI, and they can run concurrently. Everything Track 2 creates before the gate is identity and network objects, not VMs, so it's fast and cheap; teardown of each spoke resource group took ASO about 90 seconds. Budget the cluster-create windows, and check your sandbox's lifetime before you start.

What we'd push upstream — the template/CRD api-version mismatch, the azure-cli Python 3.14 crash, the delete-flag incompatibility, and the rest — is written up in [docs/08-findings-and-gotchas.md](docs/08-findings-and-gotchas.md).
