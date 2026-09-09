## Governance & Security Hardening on Azure

**Author:** Fabrizio Mastrogiovanni
**Cost:** $0 — no billable resources are deployed
**Deployment method:** Resource Group and VNet provisioned with **Terraform**; governance controls applied via portal

---

## 1. Objective

Move from being the subscription Owner who can do anything, to being the administrator who defines what *everyone else* can do.

This lab proves three independent control layers on the same Resource Group:

| Layer | Question it answers | Mechanism |
|---|---|---|
| **Identity / RBAC** | *Is this caller allowed to perform this action?* | Reader role assignment on the RG |
| **Governance / Policy** | *Is this request compliant, regardless of who sent it?* | `Allowed virtual machine size SKUs` policy |
| **Cost** | *Are we spending more than we agreed to?* | Budget with an 80% actual-spend alert |

The deliverable is three explicit "denied" or "alerted" outcomes — not three resources created.

---

## 2. Architecture

Every request to Azure — portal click, CLI command, Terraform apply — goes through the same front door: **Azure Resource Manager (ARM)**. RBAC and Policy are both enforced there, which is why neither can be bypassed by switching tools.

```mermaid
flowchart TB
    subgraph clients["Callers"]
        TF["Terraform CLI<br/>(azurerm 5.0.0)"]
        ADMIN["Admin — Portal<br/>(Owner on subscription)"]
        JUNIOR["junior-dev-fabrizio — Portal<br/>(Reader on RG only)"]
    end

    subgraph arm["Azure Resource Manager — control plane"]
        RBAC{"RBAC<br/>Is the caller authorized?"}
        POLICY{"Azure Policy<br/>Is the request compliant?"}
    end

    subgraph scope["Scope: rg-lab05-gov-fabrizio"]
        RG["Resource Group"]
        VNET["vn-lab05<br/>10.0.0.0/16"]
        BUDGET["Budget: $50/month<br/>Alert at 80% actual"]
    end

    TF -->|"Create RG + VNet"| RBAC
    ADMIN -->|"Create VM Standard_D2s_v3"| RBAC
    JUNIOR -->|"Create Storage Account"| RBAC

    RBAC -->|"DENIED — Reader has no write permissions"| DENY1["403 AuthorizationFailed"]
    RBAC -->|"ALLOWED — Owner"| POLICY
    POLICY -->|"DENIED — SKU not in allow list"| DENY2["RequestDisallowedByPolicy"]
    POLICY -->|"ALLOWED"| RG

    RG --> VNET
    RG -.->|"consumption metered"| BUDGET

    style DENY1 stroke-width:2px
    style DENY2 stroke-width:2px
```

**The key structural point:** RBAC and Policy are evaluated in that order, and they are not redundant. RBAC answers *who*; Policy answers *what*. An Owner passes RBAC and can still be stopped by Policy. That's the entire reason both exist.

**Why Terraform is drawn as a peer of the portal, not a side door:** it authenticates with the same Entra ID identity and submits the same ARM requests. If the Policy assignment in Phase 4 were in place before a non-compliant `terraform apply`, Terraform would fail with the same `RequestDisallowedByPolicy` — surfaced as an apply error instead of a red banner. IaC does not sit outside governance.

<img width="1234" height="637" alt="395274D3-1FF2-4C03-90E7-49496B8BA961_1_105_c" src="https://github.com/user-attachments/assets/6a7a8c1e-bb39-498e-9ce4-24a48031e30e" />


---

## 3. Prerequisites

- Active Azure subscription with **Owner** or **User Access Administrator** rights (needed to create role assignments)
- Permission to create users in Microsoft Entra ID
- A browser that supports Incognito / Private mode — you need two identities signed in at once
- Azure CLI installed and authenticated (`az login`)
- Terraform installed (`terraform -version`) — this lab was built against the **azurerm provider v5.0.0**
- Your subscription ID: `az account show --query id -o tsv`

---

## 4. Naming Convention

| Item | Value |
|---|---|
| Resource Group | `rg-lab05-gov-fabrizio` |
| Virtual Network | `vn-lab05` (`10.0.0.0/16`) |
| Region | East US |
| Working directory | `Governance-Security-Hardening5/` |
| Test user | `junior-dev-fabrizio@<yourtenant>.onmicrosoft.com` |
| Policy assignment | `Restrict-VM-Sizes` |
| Budget | `Monthly-Lab-Budget` |
| Allowed VM SKUs | `Standard_B1s`, `Standard_B1ms` |

---

## 5. Phase 1 — Provision the Scope with Terraform

All three controls in this lab are scoped to a single Resource Group. Scoping matters: a policy applied at subscription level would affect every future lab.

The lab as written builds this in the portal. I built it with Terraform instead — carrying forward the IaC work from Lab 04 and giving the Reader identity in Phase 3 something real to look at rather than an empty group.

### 5.1 `provider.tf`

```hcl
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "=5.0.0"
    }
  }
}

# Configure the Microsoft Azure Provider
provider "azurerm" {
  subscription_id = "<your-subscription-id>"
  features {}
}
```

> **azurerm v4.0 onward requires `subscription_id` explicitly** — it no longer falls back silently to the CLI's active subscription. Rather than hardcoding it, export it instead and delete the line:
> ```bash
> export ARM_SUBSCRIPTION_ID=$(az account show --query id -o tsv)
> ```
> The provider reads `ARM_SUBSCRIPTION_ID` automatically. This keeps a tenant-identifying value out of the repo.

<img width="3158" height="2098" alt="9F238CF3-6F88-4E3E-8D48-0248A39E12B0" src="https://github.com/user-attachments/assets/80d34012-b376-4c77-b06a-646ff274ee52" />


### 5.2 `main.tf`

```hcl
resource "azurerm_resource_group" "example" {
  name     = "rg-lab05-gov-fabrizio"
  location = "East US"
}

# Create a virtual network within the resource group
resource "azurerm_virtual_network" "example" {
  name                = "vn-lab05"
  resource_group_name = azurerm_resource_group.example.name
  location            = azurerm_resource_group.example.location
  address_space       = ["10.0.0.0/16"]
}
```

Referencing `azurerm_resource_group.example.name` rather than retyping the string is what creates the implicit dependency graph — Terraform will create the RG first and destroy it last without being told to.

<img width="1392" height="564" alt="395EAD85-503D-4EE4-A721-A7EC5D113416_1_105_c" src="https://github.com/user-attachments/assets/d3ab94ec-f7f8-4da7-8015-7dbce11d13d7" />


### 5.3 Deploy

```bash
cd Governance-Security-Hardening5
pwd          # confirm you are in the lab directory before anything else

terraform init
terraform plan
terraform apply
```

Expected tail of the apply:

```
Plan: 2 to add, 0 to change, 2 to destroy.

Do you want to perform these actions?
  Enter a value: yes

azurerm_resource_group.example: Creation complete after 27s
azurerm_virtual_network.example: Creation complete after 4s

Apply complete! Resources: 2 added, 0 changed, 2 destroyed.
```

> **The `2 destroyed` in my output was not a mistake — it was a rename.** I initially deployed as `rg-lab05`, then changed the `name` attribute to `rg-lab05-gov-fabrizio`. A Resource Group name is a **ForceNew** attribute: it cannot be updated in place, so Terraform destroys and recreates. The VNet went with it, because it depends on the RG. Always read the plan header — `2 to add, 2 to destroy` on what looks like a cosmetic edit is the provider telling you it's replacing infrastructure, not editing it.

### 5.4 Confirm

```bash
terraform state list
az group show --name rg-lab05-gov-fabrizio --query "{name:name, location:location, state:properties.provisioningState}" -o table
```

### 5.5 Before your first commit

The state file records every resource attribute, including your subscription ID. Confirm it is ignored:

```bash
git check-ignore -v terraform.tfstate
```

If that returns nothing, the file is untracked but **not** ignored, and the next `git add .` will commit it. Add to `.gitignore`:

```gitignore
**/.terraform/*
*.tfstate
*.tfstate.*
*.tfvars
```

Commit `.terraform.lock.hcl` — it pins provider versions and belongs in the repo.

---

## 6. Phase 2 — RBAC: Assign Least Privilege

### 6.1 Create the test identity

**Portal:** search **Microsoft Entra ID** → **Users** → **New user** → **Create new user**

- User principal name: `junior-dev-fabrizio`
- Display name: `Junior Developer`
- Password: uncheck **Auto-generate**, set one you'll remember (you'll type it in Incognito shortly)
- **Review + create** → **Create**

> Note the full UPN including the `@<tenant>.onmicrosoft.com` suffix. You need it to sign in.

<img width="1244" height="632" alt="0F97C6A0-996C-4EC8-B326-931FEB8BC83F_1_105_c" src="https://github.com/user-attachments/assets/1b78f9c3-ef0b-4c5e-8d98-991fa6a1f6ea" />


### 6.2 Grant Reader on the Resource Group

**Portal:** Resource groups → `rg-lab05-gov-fabrizio` → **Access control (IAM)** → **+ Add** → **Add role assignment**

- Role: **Reader** → **Next**
- Members: **+ Select members** → search `Junior Developer` → select → **Select**
- **Review + assign** → **Review + assign**

<img width="1190" height="660" alt="F68B808C-65CF-42DB-A9A7-1FBF9885BD00_1_105_c" src="https://github.com/user-attachments/assets/e215e6d4-cc6f-4926-a862-3c31454136df" />
<img width="3344" height="1856" alt="C612A6F4-A7F2-4CEA-A0AA-19D9F26F229E" src="https://github.com/user-attachments/assets/cbc69308-37a6-4a3a-a882-f12177f12444" />



**Why Reader:** it grants read on all resource types in scope and write on none. It is the smallest role that still lets someone see the environment — the correct default for anyone who doesn't need to change things.

---

## 7. Phase 3 — Validate RBAC (Assertion 1)

The assignment only means something if you can demonstrate the denial.

1. Open an **Incognito / Private window** → `portal.azure.com`
2. Sign in as `junior-dev-fabrizio@<yourtenant>.onmicrosoft.com`
3. Navigate to **Resource groups** → `rg-lab05-gov-fabrizio`
   - ✅ The group is **visible**, and so is `vn-lab05` including its `10.0.0.0/16` address space — Reader grants full read on resource configuration, not just the ability to see that something exists
4. Click **+ Create** → search **Storage Account** → attempt to create one

**Expected result:**

```
The client 'junior-dev-fabrizio@<tenant>.onmicrosoft.com' with object id '<guid>'
does not have authorization to perform action
'Microsoft.Storage/storageAccounts/write' over scope
'/subscriptions/<sub-id>/resourceGroups/rg-lab05-gov-fabrizio'
```

The Create button may also render disabled, depending on where in the flow validation fires.

<img width="1222" height="643" alt="EDCA4CFB-1486-41BE-919D-C3A1AAF3B676_1_105_c" src="https://github.com/user-attachments/assets/76aabb76-c0a4-42c6-bdfb-4860f2f860bd" />
<img width="1210" height="620" alt="A688F3C9-1C70-444B-93A9-9D00F779EB28" src="https://github.com/user-attachments/assets/ea11e7c9-402e-431c-b8fd-65b4aff95e24" />
<img width="1000" height="787" alt="A0D49728-1D2E-433F-B2EE-05D647CA9720_1_105_c" src="https://github.com/user-attachments/assets/3825597a-ac21-4ecc-8aa5-e1c2f43727c9" />
<img width="1424" height="553" alt="5B134E9D-FAEF-4418-9041-B41AE5414A9F_1_105_c" src="https://github.com/user-attachments/assets/62031edb-95a2-43b3-9f4c-85e485c83248" />

**Assertion 1 satisfied:** the identity can read the scope and cannot write to it.

Close the Incognito window and return to your admin session.

---

## 8. Phase 4 — Azure Policy: Constrain What Can Be Built

RBAC restricts *people*. Policy restricts *requests*, including your own.

**Portal:** search **Policy** → **Authoring** → **Assignments** → **Assign policy**

**Basics**
- Scope: click **…** → select your subscription → select **`rg-lab05-gov-fabrizio`** as the Resource Group → **Select**
- Policy definition: click **…** → search `Allowed virtual machine size SKUs` → select
- Assignment name: `Restrict-VM-Sizes`

**Parameters**
- Uncheck **Only show parameters that need input**
- Allowed Size SKUs: select **only** `Standard_B1s` and `Standard_B1ms`

**Review + create** → **Create**

<img width="1264" height="621" alt="A9AD0861-39F5-4E43-9331-9F5123F7D101_1_105_c" src="https://github.com/user-attachments/assets/0a7ee2a9-2196-4c15-b41c-e5b22760349e" />


> ⏱️ **Propagation delay:** assignments take roughly 10–30 minutes to become enforceable. If your test in Phase 5 succeeds when it should fail, the policy hasn't replicated yet — wait, don't rebuild it.

---

## 9. Phase 5 — Validate Policy (Assertion 2)

Run this test as **your admin account**. That's the point of the test.

**Portal:** `rg-lab05-gov-fabrizio` → **Create** → **Virtual machine**

- Name: `vm-policy-test`
- Image: Ubuntu Server
- Size: change to **`Standard_D2s_v3`** (anything outside the allow list)
- **Review + create**

**Expected result:** `Validation failed`. Expand the error:

```
Resource 'vm-policy-test' was disallowed by policy.
Policy assignment: Restrict-VM-Sizes
Policy definition: Allowed virtual machine size SKUs
Code: RequestDisallowedByPolicy
```
<img width="1228" height="640" alt="E170D8AA-A573-4430-9FC1-49DD1409719F_1_105_c" src="https://github.com/user-attachments/assets/63f3f1ed-109d-49ff-a34c-7bbf1686397b" />
<img width="1378" height="569" alt="3FC806B0-3995-4D4A-BB96-454B96C57770_1_105_c" src="https://github.com/user-attachments/assets/adf6ed40-8fc5-43d2-8624-f31727d1a4b8" />


**Assertion 2 satisfied:** a subscription Owner was blocked by a governance control. This is the difference between a permission and a guardrail.

Do **not** complete the VM creation. Discard the form.

**Optional CLI verification:**

```bash
az policy assignment list \
  --scope "/subscriptions/$(az account show --query id -o tsv)/resourceGroups/rg-lab05-gov-fabrizio" \
  --output table
```

---

## 10. Phase 6 — Budget & Cost Alerting (Assertion 3)

RBAC and Policy are preventive. Budgets are **detective** — they don't stop spend, they tell you it happened.

**Portal:** `rg-lab05-gov-fabrizio` → **Budgets** (under Cost Management) → **+ Add**

**Budget details**
- Name: `Monthly-Lab-Budget`
- Reset period: **Billing month**
- Creation date: today
- Expiration date: one year out
- Amount: **50** (USD)
- **Next**

**Set alerts**
- Alert condition type: **Actual**
- % of budget: **80**
- Alert recipients: `growtonicconsulting@gmail.com`
- **Create**

<img width="3390" height="1534" alt="F266DB74-AD81-4D0E-AE36-79860A4CDE16" src="https://github.com/user-attachments/assets/e3d2f4cf-e01f-4b11-a9ef-0b15ddd2d8f9" />
<img width="2332" height="1850" alt="4C55F60C-7FB8-4C7F-8FFF-713962CD5B45" src="https://github.com/user-attachments/assets/50cba68c-801c-4f0b-b370-80bc71d05db1" />
<img width="1158" height="679" alt="DBC6A8F7-B9B6-4BC6-98AE-C1017E2613B7_1_105_c" src="https://github.com/user-attachments/assets/0cb6e362-f3f5-45f2-971b-0bf99582ea95" />

**Assertion 3 satisfied:** an alert fires at $40 of a $50 monthly ceiling.

> **Actual vs. Forecasted:** *Actual* alerts on money already spent. *Forecasted* alerts when Azure projects you'll breach the threshold before the period ends. Forecasted gives you earlier warning; actual gives you fewer false positives. For a lab budget, actual is correct.

---

## 11. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Junior Dev can still create resources | Reader was assigned at subscription scope, or an inherited role grants write | Check **IAM → Role assignments**, filter by the user, and confirm the **Scope** column reads the Resource Group. Inherited assignments show as "Inherited" |
| Junior Dev sees nothing at all | Role assignment hasn't propagated (usually under 5 min) | Sign out and back in to refresh the token — RBAC changes aren't reflected in an already-issued token |
| Policy didn't block the VM | Assignment hasn't replicated | Wait 15–30 min. Confirm the assignment scope matches the RG you're deploying into |
| Policy blocks everything, including B1s | Parameter saved with an empty or wrong allow list | Open the assignment → **Edit assignment** → **Parameters** → confirm both SKUs are present |
| Can't create a role assignment | You're Contributor, not Owner / User Access Administrator | Contributor can manage resources but cannot grant access — that separation is deliberate |
| Can't find Budgets on the RG | Cost Management is subscription-scoped in some tenants | Create at subscription scope and filter by resource group |

---

## 12. Cleanup

Tear down in reverse order of creation. The policy assignment and budget were created outside Terraform, so they are not in state — remove them first, or `terraform destroy` will leave orphaned assignments pointing at a scope that no longer exists.

```bash
# 1. Remove the policy assignment (created in the portal, not tracked by Terraform)
az policy assignment delete \
  --name Restrict-VM-Sizes \
  --scope "/subscriptions/$(az account show --query id -o tsv)/resourceGroups/rg-lab05-gov-fabrizio"

# 2. Destroy the Terraform-managed infrastructure
cd Governance-Security-Hardening5
terraform destroy

# 3. Confirm state is empty
terraform state list          # returns nothing
az group exists --name rg-lab05-gov-fabrizio   # false
```

Expected: `Destroy complete! Resources: 2 destroyed.`

Then delete the test identity — it does not live in the Resource Group and will not be removed with it:

**Portal:** Microsoft Entra ID → **Users** → `Junior Developer` → **Delete**

> Deleted Entra ID users sit in a recoverable state for 30 days under **Deleted users**. Purge from there if you want the directory fully clean.

---

## 13. What I Learned

**RBAC and Policy answer different questions, and the order matters.**
A request is authorized first, then evaluated for compliance. Being Owner gets you past step one and tells you nothing about step two. This is the same structural pattern as NSG evaluation from Lab 02 — a control is only meaningful if you can name the specific request it rejects. "I have a policy" is not a boundary. "An Owner attempting a D-series VM in this RG receives `RequestDisallowedByPolicy`" is.

**Scope is the most consequential field on the form.**
The identical policy definition assigned at subscription scope instead of RG scope would have blocked VM creation across every lab in this repo. Nothing in the UI flags that difference; it's a dropdown that looks like every other dropdown.

**Preventive and detective controls are not substitutes.**
Policy prevents a class of resource from existing. A budget detects spend after the fact. Neither covers the other's failure mode: a policy allowing `Standard_B1s` still permits fifty of them, and a budget alert arrives after the money is gone.

**Propagation delay is a real operational property, not a bug.**
Both RBAC and Policy are eventually consistent. Testing too early produces a false negative that looks exactly like a misconfiguration — and the instinct to rebuild the assignment makes it worse. Read the timestamp before rebuilding anything.

**Contributor cannot grant access.**
Managing resources and managing who can manage resources are separate privileges by design. That separation is what stops a compromised Contributor from escalating to Owner.

**Renaming in Terraform is replacement, not editing.**
Changing `name` on a Resource Group produced `2 to add, 0 to change, 2 to destroy` — a ForceNew attribute triggers destroy-and-recreate, and everything depending on it goes too. On an empty lab RG that costs 30 seconds. On a group holding a database, it's an outage. The plan output said so before I typed `yes`; the discipline is reading the header, not trusting that a small edit implies a small change.

**IaC is subject to the same governance as the portal.**
The instinct is to think of Terraform as a privileged back channel. It isn't — it authenticates as an Entra ID identity and submits ARM requests like any other client. Had I applied the SKU policy before running `terraform apply` on a D-series VM, the apply would have failed with the same `RequestDisallowedByPolicy`. This is the practical consequence of the architecture diagram: enforcement lives at ARM, so there is no tool you can switch to in order to escape it.

**State files are a disclosure risk, not just a merge-conflict nuisance.**
`terraform.tfstate` contains the subscription ID and full resource metadata in plaintext. In my working tree the state files showed as untracked rather than ignored — one `git add .` away from being published. `git check-ignore -v terraform.tfstate` is a five-second check that belongs before the first commit of every lab, not after.

---

## 14. Repository Structure

```
Governance-Security-Hardening5/
├── README.md
├── provider.tf
├── main.tf
├── .terraform.lock.hcl          # committed — pins provider versions
├── images/
│   ├── 00-resource-group-overview.png
│   ├── 01a-provider-tf.png
│   ├── 01b-main-tf.png
│   ├── 01c-terraform-apply.png
│   ├── 01d-rg-portal.png
│   ├── 02-create-user.png
│   ├── 03-rbac-assignment.png
│   ├── 04-rbac-denied.png
│   ├── 05-policy-assignment.png
│   ├── 06-policy-denied.png
│   └── 07-budget.png
│
├── .terraform/                  # ignored
├── terraform.tfstate            # ignored
└── terraform.tfstate.backup     # ignored
```

---

**Repo:** [github.com/fabrizio-mastrogiovanni](https://github.com/fabrizio-mastrogiovanni)
