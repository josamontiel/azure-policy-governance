# Results & Retrospective

The closing artifact for the project. Three sections: where compliance landed, how the controls map to recognised security frameworks, and an honest retrospective on what broke and what I'd do differently.

---

## Compliance metrics

The most useful framing isn't a single percentage — it's the per-policy breakdown showing what's compliant, what's exempted, and why.

After Phase 5's Audit-then-Deny rollout and the cleanup work that preceded it, the management group `mg-governance-lab-prod` evaluates against five policies in the `Governance Lab Security Baselines` initiative:

| Policy | Effect | Compliant resources | Non-compliant | Exempt | Status |
|---|---|---|---|---|---|
| Require a tag and its value on resources | Deny | All in scope | 0 | 0 | ✅ |
| Allowed locations | Deny | All in scope | 0 | 0 | ✅ |
| Deny Storage Accounts Without Shared Key Access Disabled (custom) | Audit | All applicable | 0 | 0 | ✅ |
| Storage accounts should disable public network access | Deny | All applicable | 0 | 1 | ✅ + waiver |
| Configure diagnostic settings for Storage Accounts to Log Analytics workspace | DeployIfNotExists | All applicable | 0 | 0 | ✅ |

Resources currently in scope and being evaluated:

- **State backend storage account** (`sttfstategovlab20901`) — fully governed, with one time-bound exemption for public network access
- **Log Analytics workspace** (`law-governance-lab`) — compliant on all applicable policies
- **Network Watcher** (`networkwatcher_eastus`, in `networkwatcherrg`) — auto-created by Azure when Network Watcher is enabled, picked up by the management group's policy scope, compliant

**Headline**: 100% compliant against the controls the initiative enforces, with one documented waiver for a legitimate technical exception.

The waiver is on the state backend's public network access. It's required because both local engineering machines and GitHub-hosted CI runners need to reach the storage account to read and write Terraform state. Closing public access would require private endpoints with self-hosted runners or a jumphost — out of scope for the lab. The waiver is time-bound (expires 2026-11-09), categorised as a Waiver (accepted risk, no compensating control), and scoped narrowly to the public network access policy reference within the initiative.

The journey to that state matters more than the endpoint:

- **Phase 1**: zero policies enforced. Anything could be deployed without tags, public storage was the default, no regional restrictions, no audit trail.
- **Phase 3**: three policies enforced (tags, allowed locations, custom blob access). Compliance dashboard reading around 75% during testing, primarily because of duplicate audit findings on overlapping assignments and a deliberately-misconfigured test resource.
- **Phase 4**: full migration to Terraform with proper parameterisation. During the migration, three drift findings discovered that the Portal had hidden — a misnamed custom policy, an orphaned duplicate definition, and three redundant assignments double-counting non-compliance.
- **Phase 5**: cleanup, hardening (Shared Key disabled on the state backend), exemption written and deployed, public network access flipped from Audit to Deny.
- **Now**: 100% compliance with one documented waiver, full enforcement, all changes deployed through a CI/CD pipeline with manual approval gates.

The compliance percentage isn't the achievement. The traceability is.

---

## Control mapping

The five policies in the initiative map to the CIS Microsoft Azure Foundations Benchmark v2.0.0 and NIST SP 800-53 Rev. 5 control families. Some mappings are direct; some are indirect because not every governance control corresponds neatly to a single security control.

| Policy | CIS Azure Foundations v2.0.0 | NIST 800-53 Rev. 5 | Notes |
|---|---|---|---|
| Require a tag and its value on resources | Indirect — not a CIS Storage control | **CM-8** Information System Component Inventory | Asset management / cost attribution control. Tagging discipline is a precondition for inventory, not itself a benchmark control. |
| Allowed locations | Indirect — covered under organisational data residency policy | **AC-4** Information Flow Enforcement; **SC-7** Boundary Protection | Geographic restriction. Limits where data can land, supporting data sovereignty and reducing the regions that need monitoring coverage. |
| Deny Shared Key Access (custom) | **3.8** related — Account access management. Maps closely to the spirit of "use Azure AD authentication over Shared Key" | **AC-2** Account Management; **IA-2** Identification & Authentication; **AC-6** Least Privilege | Forces identity-based access (Entra ID) over key-based access. Stronger authentication, clearer audit trails, no long-lived shared secrets. |
| Storage accounts should disable public network access | **3.10** Ensure Private Endpoints are used to access Storage Accounts (related); **3.1** default network access rule should be Deny (related) | **SC-7** Boundary Protection | Network-layer control. Prevents storage accounts from being reachable over the public internet without explicit allow-listing or private endpoint deployment. |
| Configure diagnostic settings for Storage Accounts to Log Analytics workspace | **3.15** Ensure Storage Logging is Enabled; **5.1.2** Ensure Diagnostic Setting captures appropriate categories | **AU-2** Audit Events; **AU-12** Audit Generation; **AU-6** Audit Review, Analysis, & Reporting | Logging/audit control. Ensures operations on the storage account are recorded and centralised, supporting investigation and detective capability. |

A few honest caveats worth surfacing:

**The mapping is illustrative, not certification-grade.** A real CIS or NIST audit involves many controls beyond these five, and the mappings would need to be validated by an assessor. The point of this section is to demonstrate that the controls implemented here aren't arbitrary — they correspond to recognised best practices.

**Tagging maps weakly to security frameworks.** Asset management is a NIST control family (CM-8), but neither CIS nor NIST treat resource tagging as a security control per se. It's enforced here because cost attribution and ownership tracking are operational prerequisites, not because they're CIS-mandated.

**The Shared Key custom policy is broader than CIS 3.8.** CIS 3.8 specifically addresses periodic key regeneration. The custom policy goes further — it denies the use of Shared Key auth entirely. The intent matches; the specific control text differs.

**Some CIS Azure Storage Services controls aren't covered.** Notably, this initiative doesn't cover encryption-at-rest with customer-managed keys (CIS 3.2), HTTPS enforcement, soft delete, or minimum TLS version. Those are sensible next-quarter additions.

---

## Retrospective

What broke during the build, what got handled cleanly, what would happen differently in v2.

### What broke

**A custom policy lied about itself for months.** The `Block Public Blob Storage` policy I built in Phase 1 had a display name that described one control while its rule logic implemented another (Shared Key auth, not blob public access). I built it, wrote about it, watched it catch test resources, and never noticed. The Portal showed display name on one tab and rule logic on another — they never sat side-by-side. The mismatch surfaced in seconds the moment I imported the policy into Terraform and `terraform plan` produced a single diff containing both fields.

That moment crystallised the IaC value proposition more than any tutorial could: *the Portal lets you build governance that doesn't match its own description and have nobody notice. You can run that for months.*

**Three policy assignments were redundantly duplicating evaluations.** During the migration I found the management group had four assignments where it should have had one — the initiative, plus three standalone assignments of policies the initiative already covered. Compliance percentages we'd been quoting were averages across overlapping evaluations, double- and triple-counting non-compliance findings on the same resources. None of this was visible without listing assignments via CLI; the Portal's grouped views had hidden it.

**A GitHub secret got corrupted, breaking the DINE engine silently.** Mid-Phase-5, a routine `terraform plan` flagged two unexpected changes. Investigation revealed the workspace ID parameter in Azure was a bare GUID instead of the full resource ID — meaning the diagnostic settings remediation engine had been pointing at nothing and silently failing. The fix was straightforward (replace the secret, apply the correct value). The discovery is the point: *without the routine plan comparison, this could have run for weeks before manual inspection caught it.*

**A provider bug nearly destroyed three resources.** While importing the initiative assignment, the AzureRM provider's `azurerm_management_group_policy_assignment` resource flagged a `forces replacement` change on `policy_definition_id` because Azure had stored the ID with mixed casing while Terraform produced canonical casing. Same identifier, different cased strings, treated as different identities. Applying that plan would have destroyed the assignment, recreated it with a fresh managed identity, and cascaded into both role assignments being rebuilt. The destroy markers in the plan output were the only protection between me and a self-inflicted minor outage.

**A federated credential was missing for the apply workflow's environment context.** When the apply workflow first ran, it failed authentication with a confusing `AADSTS700213` error. Cause: GitHub's OIDC subject claim format changes when a workflow runs inside an Environment block (which the apply workflow uses for the manual approval gate). Two federated credentials existed; a third was needed. Each one binds to a specific subject claim format; missing one means workflows in that context can't authenticate.

**An `az policy exemption list` quirk hid the active waiver.** When verifying Phase 5's exemption, the standard list command returned empty even though the compliance dashboard correctly reported `Exempt`. Cause: `az policy exemption list --scope <management-group>` only returns exemptions defined *at* that scope, not exemptions defined at resource scopes underneath. Resource-scoped exemptions need to be queried at the resource scope directly. Documented but easy to miss.

**One incident I caused: leaked subscription identifiers.** Mid-Phase-4, exported policy JSON files got committed to the public IaC repo. Subscription ID and email visible in the `systemData` metadata. Fixed by `git filter-repo` rewriting history, force-pushing the cleaned remote, and adding `*.json` to `.gitignore`. Subscription IDs aren't secrets in the cryptographic sense, but they don't belong in public repos either.

### What got handled cleanly

**Recovery from the workspace ID corruption.** ~30 minutes from "what's this drift?" to "everything verified clean." Diagnosis went via direct Azure CLI queries rather than Terraform (which hid the values behind sensitivity flags). Fix happened at the source — the GitHub secret — rather than just at the symptom.

**The state backend exemption.** Properly time-bound (six months), narrowly scoped to one policy reference (not the whole assignment), documented with technical justification, deployed through the pipeline with full review and approval. This is the model for how exemptions should be written. Not "we'll get to it later" — "here's why this is the correct configuration for this resource, here's when we revisit, here's who owns it."

**The Audit-then-Deny rollout.** The public network access policy spent weeks in Audit mode, accumulating compliance data. By the time it flipped to Deny, the only resource that would have been blocked was the state backend (covered by the now-existing exemption) and the Phase 3 test fixture (deleted as cleanup). The flip itself was a single-line change to a Terraform variable's default, deployed through the pipeline with plan preview and manual approval. Verified post-apply with both an Azure CLI parameter check and a deliberate violating deployment that got blocked with the expected policy violation error.

**The two-repo split.** Documentation (this repo) and IaC (the companion) ended up serving different audiences and different purposes. Mixing them would have made both worse.

### What I'd do differently in v2

**Set up the GitHub Actions pipeline at the start of Phase 4, not the end.** The plan-on-PR workflow would have caught several issues during the migration that I instead hit by running Terraform locally. Specifically: the federated credential gotcha, the `use_azuread_auth` issue, and the casing trap would all have surfaced in earlier PRs rather than blocking the apply at the very end.

**Configure all three federated credentials up front.** The pull-request, main-branch-push, and environment subject claim formats are all distinct. Two credentials covers the basic CI cycle; the third is needed the moment you add a deployment environment with required reviewers. There's no reason not to create all three from the start.

**Consider using `azurerm_management_group_policy_definition` from the start once it ships.** The deprecation warnings on `management_group_id` are noise now but signal future work. v5 of the AzureRM provider will require migration. Builds initiated after that ships should use the scope-specific resources from day one.

**Adopt strict-mode defaults in the policy module.** The custom blob policy used `notEquals: "false"` to catch unset-default cases, which is correct but bit me when a test resource was deployed via the Portal with the property unset rather than explicitly false. A cleaner pattern would have been to require the property be explicitly `false` and document that explicitly. Same outcome, less surprise.

**Write OPERATIONS.md in parallel with the engineering, not after.** Some of the runbook content (the workspace ID incident, the exemption listing quirk) was easier to write *because* it had just happened, but other parts (the change flow, the review cadence) would have been clearer if drafted while the system was being designed rather than after.

**Treat exemptions as first-class artifacts from day one.** The exemption module came late in the project. In retrospect, it should have been part of the initial IaC design — the moment you deploy any policy in Deny mode, the question of "what about legitimate exceptions?" exists. Better to have the answer ready than to retrofit it.

### What this proved about Policy as Code

The single strongest argument for IaC governance came from a side observation, not a designed experiment.

When I imported the custom policy into Terraform and saw `terraform plan` flag the display-name-vs-rule mismatch, I learned the policy didn't do what I thought it did. The Portal had let me build it, deploy it, watch it work, and tell the wrong story about what it was — because the metadata and the rule lived on different tabs and never appeared together. The IaC version put them in a single file. The mismatch surfaced in the first plan. Couldn't be missed.

That's the value proposition reduced to its smallest reproducible form: *Portal-built governance can lie about itself indefinitely. Code-defined governance forces the truth into the same file, where it can't.* Every other benefit of IaC — version control, peer review, automated deployment, rollback — is downstream of that core property.

The Portal lets you build governance. Git lets you maintain it.
