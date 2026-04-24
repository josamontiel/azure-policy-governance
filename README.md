# Building an Azure Governance MVP

A hands-on project to simulate what a security engineer actually does when locking down a "Wild West" Azure environment — using only **Azure Policy**.

---

## Phase 1: Lab Setup
Everything starts with a clean resource group. I created `rg-governance-lab` in my Azure subscription — no resources inside yet, just a sandbox where I could safely test guardrails without affecting anything else.  

<img width="273" height="282" alt="Screenshot 2026-04-24 at 20 41 51" src="https://github.com/user-attachments/assets/7884627f-c4a1-4443-8579-91bf2490c5f8" />

---

## Phase 2: Tagging 
I assigned the built-in policy **Require a tag and its value on resources** to the RG with effect = `Deny`.  
The rule? All resources must carry a `CostCenter=IT` tag. 

<img width="709" height="487" alt="Screenshot 2026-04-24 at 20 42 54" src="https://github.com/user-attachments/assets/473a157b-31d3-4d81-a665-2b3b45100631" />

<img width="1607" height="709" alt="Screenshot 2026-04-24 at 20 47 57" src="https://github.com/user-attachments/assets/4bf64ebf-abaf-4fbc-8901-f41fe415fedf" />


When I tried to spin up a Virtual Network without the tag, Azure blocked me instantly.  
> First win: governance before deployment, not after.

<img width="1610" height="282" alt="Screenshot 2026-04-24 at 20 50 08" src="https://github.com/user-attachments/assets/26f99f6a-cdfa-4753-a713-deefc7e3b637" />



<img width="462" height="149" alt="Screenshot 2026-04-24 at 20 53 31" src="https://github.com/user-attachments/assets/bb54a0a1-6a3d-4eb8-96b8-fb5ca1c38270" />



---

## Phase 3: Custom Policy – Block Public Blob Storage
Built my first custom policy definition. The JSON rule targets `Microsoft.Storage/storageAccounts` and checks `allowBlobPublicAccess` — if it’s true, creation is denied.  

This mimics a real-world, secure-by-default storage policy. A quick test deployment fails exactly as designed.

<img width="691" height="282" alt="Screenshot 2026-04-24 at 21 03 35" src="https://github.com/user-attachments/assets/079a0f99-e4cf-4b70-9d3d-4203e5e6e154" />

<img width="568" height="206" alt="Screenshot 2026-04-24 at 21 06 38" src="https://github.com/user-attachments/assets/6ab1ace0-62b2-44a9-98b2-3b81ed6d3939" />


---

## Phase 4: The Initiative 
Rather than assigning individual policies one by one, I bundled everything into a single custom **Initiative**:  
*"Governance Lab Security Baseline"*  

Now I can assign tagging, public blob blocking, and an ```Allowed locations``` policy all at once.  
This is the scalable way to enforce multiple controls across subscriptions — the jump from “someone who knows policy” to “someone who governs Azure.”

<img width="1201" height="562" alt="Screenshot 2026-04-24 at 21 31 35" src="https://github.com/user-attachments/assets/ae9e3621-10e7-4081-ae6b-0daaf0a34264" />


---

## In the pipeline

### Phase 5: Proactive Compliance & Remediation  
Moving beyond simple Deny actions — using `DeployIfNotExists` policies to automatically fix non-compliant resources (e.g., deploying diagnostic settings). Testing remediation tasks that correct drift without manual intervention.

### Phase 6: Scale & Visibility  
- **Management Groups:** Assigning policies higher up the hierarchy so all subscriptions inherit them automatically.  
- **Policy as Code:** Exporting policy definitions to JSON/Bicep and managing them via CLI/PowerShell — version-controlled, programmable governance.  
- **Compliance Dashboard:** Building a single-pane-of-glass view with the Policy Compliance widget, linked to the subscription or management group.
---

> The philosophy of security by design reduces so many downstream headaches. This lab brings that to life without spending a dime.

> NOTE: This lab will be constantly growing and evolving.
