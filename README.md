# Building an Azure Governance MVP 🛡️
 
A hands-on project simulating what a security engineer actually does when locking down a "Wild West" Azure environment — using **Azure Policy** as the enforcement layer.
 
The goal isn't to learn a tool. It's to build a working control framework: prevention that scales, visibility that lasts, and operations you can hand to someone else.
 
---
 
## The Problem
 
Imagine inheriting an Azure subscription with no tagging discipline, storage accounts open to the public internet, and resources deployed in whatever region the last engineer felt like clicking. No baseline, no inheritance, no audit trail.
 
This project builds the response to that scenario, one phase at a time — starting with a single resource group and ending with version-controlled policy deployed through a pipeline.
 
---
 
## Phase 1: Foundations ✅
 
A clean resource group (`rg-governance-lab`) as the initial sandbox, plus the first three guardrails:
 
* **Tag enforcement.** The built-in *Require a tag and its value on resources* policy, assigned with `Deny`, requiring `costCenter=IT` on every resource. First test deployment without the tag was blocked at validation — governance before deployment, not after.
* **Custom blob policy.** A custom definition targeting `Microsoft.Storage/storageAccounts`, blocking creation when `allowBlobPublicAccess` isn't explicitly false. Effect parameterized (`Audit` / `Deny` / `Disabled`) so the same definition can roll out gradually.
* **Foundational initiative.** Both policies bundled with the built-in *Allowed locations* policy into a single custom initiative — `governance-lab-security-baselines` — so the whole baseline assigns as one unit.

<img width="1448" height="577" alt="Screenshot 2026-04-25 at 13 23 19" src="https://github.com/user-attachments/assets/beff4a8c-885d-40c0-85c3-a5f628a4fe39" />


---

## Phase 2: Scaling Enforcement ✅
 
A baseline pinned to one RG isn't governance — it's a personal sandbox. This phase lifted the initiative up the hierarchy:
 
* Created a Management Group structure: `mg-governance-lab` (root) → `mg-governance-lab-prod` (child).
* Moved the subscription under `mg-governance-lab-prod`.
* Re-assigned the initiative at the MG scope, with a system-assigned managed identity attached for future remediation work.
* Removed the original RG-scope assignment only after confirming MG-level evaluation was live.

  <img width="974" height="219" alt="Screenshot 2026-04-25 at 13 01 15" src="https://github.com/user-attachments/assets/9003a719-8d63-4987-a16e-174d9a7417fe" />


The mechanics are trivial; the mindset shift is the whole point. Any future subscription that lands in this MG inherits the baseline automatically — no ticket, no checklist.
 
In parallel, the blob public-access policy was flipped from `Deny` to `Audit` to collect real non-compliance data ahead of the rollout work in Phase 5.

<img width="696" height="339" alt="Screenshot 2026-04-25 at 13 19 47" src="https://github.com/user-attachments/assets/6965cc43-2d00-4acd-bc6f-e9023a896067" />


---

## Phase 3: Proactive Remediation 🚧
 
Moving from blocking bad resources to fixing them. This phase wires up `DeployIfNotExists` for self-healing infrastructure.
 
**Completed:**
 
* **Log Analytics workspace.** Created `law-governance-lab` in `rg-governance-lab` (Pay-as-you-go, 30-day retention) as the destination for diagnostic logs. Deployment was briefly blocked by the Phase 1 tag policy until `costCenter=IT` was added — the system catching its own author.
* **Managed identity permissions.** Granted the initiative assignment's system-assigned identity two roles at `mg-governance-lab-prod` scope: *Log Analytics Contributor* (to write to the workspace) and *Monitoring Contributor* (to create diagnostic settings on target resources).
* **Initiative expansion.** Added two policies to `governance-lab-security-baselines`:
  * *Configure diagnostic settings for Storage Accounts to Log Analytics workspace* — built-in `DeployIfNotExists`. The remediation engine.
  * *Storage accounts should disable public network access* — built-in, `Audit` mode. A network-flavoured control alongside the existing public-blob policy.
* **Versioned to 1.1.0** with three new initiative parameters exposed (`diagnosticSettingsEffect`, `logAnalyticsWorkspaceId`, `publicNetworkAccessEffect`) so the same definition can be reused across dev/prod scopes without forking.
* **Assignment updated** with the workspace resource ID and effect values. Five policies now governed under one assignment.
**Pending:**
 
* Deploy a deliberately non-compliant storage account.
* Force a policy evaluation scan and confirm non-compliance is reported.
* Run a remediation task and verify the diagnostic setting actually lands on the target resource.
This is where prevention turns into self-healing.
 
---
 
## Phase 4: Policy as Code 📋
 
Everything built in the Portal goes into a Git repo and a pipeline:
 
* Policy and initiative definitions exported and rewritten as Bicep.
* GitHub Actions workflow with OIDC federated auth — `what-if` previews on PRs, deploy on merge to `main`.
* Version-controlled, reviewable, rollback-able governance.
The deliverable is a public repo, not a screenshot.
 
---
 
## Phase 5: Operating the Baseline 📋
 
The discipline that separates a lab from a production system:
 
* **Audit-then-Deny rollout** using the data already being collected from the blob and public-network policies in Audit mode.
* **Exemptions** with time-bound waivers, named owners, and ticket references — no open-ended exceptions.
* **Review cadence** for compliance drift, expiring exemptions, and new resource types entering scope.
---
 
## Phase 6: Results & Control Mapping 📋
 
Final writeup tying the build back to recognised frameworks:
 
* Compliance score before vs. after, captured at the MG level.
* Each policy in the initiative mapped to CIS Azure Benchmark and NIST 800-53 controls.
* Defender for Cloud regulatory compliance dashboard screenshot showing the initiative's effect on the CIS/NIST score.
* A short retrospective on what broke, what got exempted, and what I'd do differently.
---
 
> Prevention scales; detection doesn't. This lab is the working proof.
