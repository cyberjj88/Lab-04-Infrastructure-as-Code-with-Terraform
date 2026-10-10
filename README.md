# Lab 04: Infrastructure as Code with Terraform
 
**Author:** Jair Smith
**Series:** Azure Cloud Security Series
**Lab Number:** 04 of ongoing curriculum
**Difficulty:** Intermediate
**Estimated Time:** 45 Minutes
**Prerequisites:** Active Azure subscription, Completed Week 4 Video Modules
 
---
 
## Overview
 
In this lab you stop clicking buttons in the Azure portal. Instead, you write a plain text file that describes the infrastructure you want and let Terraform build it for you automatically.
 
By the end you will have deployed a Resource Group, a Virtual Network, and a Subnet using four commands. You will then add a Network Security Group to a running environment and watch Terraform update only the new piece without touching anything that already exists. Then you tear it all down with one command.
 
This is Infrastructure as Code. It is how every production Azure environment at scale is managed and one of the most transferable skills in cloud engineering.
 
> **No installation required.** This entire lab runs inside Azure Cloud Shell, a browser-based terminal built into the Azure portal. Terraform is already installed there.
 
---
 
## Architecture Overview
 
The diagram below shows two states of the environment: the initial deployment after Phase 3 (three resources), and the updated state after Phase 5 (four resources). Terraform tracks the difference in its state file and only creates what is new.
 
```
Azure Cloud Shell (browser-based terminal)
+--------------------------------------------------+
|  ~/terraform-lab/                                |
|  main.tf          <-- your configuration file    |
|  terraform.tfstate <-- Terraform's memory        |
|  .terraform/       <-- provider plugin cache     |
+--------------------------------------------------+
          |
          | terraform apply
          v
 
Azure Region: East US
+----------------------------------------------------------+
|  rg-lab04-tf-[yourname]  (Resource Group)               |
|                                                          |
|  +----------------------------------------------------+  |
|  |  vnet-terraform  (Virtual Network: 10.0.0.0/16)   |  |
|  |                                                    |  |
|  |  +----------------------------------------------+ |  |
|  |  |  snet-backend  (Subnet: 10.0.1.0/24)        | |  |
|  |  +----------------------------------------------+ |  |
|  |                                                    |  |
|  +----------------------------------------------------+  |
|                                                          |
|  [After Phase 5]                                         |
|  +----------------------------+                          |
|  |  nsg-web                   |                          |
|  |  (Network Security Group)  |                          |
|  |  Added without touching    |                          |
|  |  VNet or Subnet            |                          |
|  +----------------------------+                          |
+----------------------------------------------------------+
 
Terraform Workflow (always in this order):
  terraform init --> terraform plan --> terraform apply --> terraform destroy
```
 
---
 
## How Terraform Works: Read This First
 
Before running a single command, understand these three concepts. They make every step below make sense.
 
### Terraform describes WHAT, not HOW
 
When you click through the Azure portal you are telling Azure exactly how to build something, one step at a time. Terraform works differently. You write a file that says "I want a resource group called X in East US" and Terraform figures out the steps to make that happen. This is called **declarative infrastructure**.
 
### The state file is Terraform's memory
 
After Terraform creates resources, it writes a file called `terraform.tfstate` to your project folder. This file is Terraform's record of everything it has created: the resource IDs, configuration values, and relationships. Every time you run `plan` or `apply`, Terraform reads this file to know what already exists so it only changes what is different. **Do not delete it.**
 
### The four commands, always in this order
 
| Command             | What It Does                                          | When to Run It                        |
|---------------------|-------------------------------------------------------|---------------------------------------|
| `terraform init`    | Downloads the Azure provider plugin                   | Once per project, the very first step |
| `terraform plan`    | Previews what will change without doing anything      | Before every apply, never skip this   |
| `terraform apply`   | Executes the plan and creates or updates resources    | After you review and approve the plan |
| `terraform destroy` | Removes every resource tracked in the state file      | When tearing down a lab or environment|
 
---
 
## Naming Convention Reference
 
Replace `[yourname]` with your actual first name, lowercase, no spaces.
 
| Resource        | Name to Use              | Example                  |
|-----------------|--------------------------|--------------------------|
| Resource Group  | rg-lab04-tf-[yourname]   | rg-lab04-tf-jsmith       |
| Virtual Network | vnet-terraform           | vnet-terraform           |
| Subnet          | snet-backend             | snet-backend             |
| NSG (Phase 5)   | nsg-web                  | nsg-web                  |
| Location        | East US                  | East US                  |
 
---
 
## Prerequisites
 
- [ ] Active Azure subscription with access to https://portal.azure.com
- [ ] Completed Week 4 Video Modules
---
 
## Step-by-Step Instructions
 
### Phase 1: Open Cloud Shell
 
**Step 1: Log in to the Azure portal**
 
Open your browser and go to https://portal.azure.com. Sign in with your Azure account credentials.
 
**Step 2: Open Cloud Shell**
 
Look at the top-right toolbar in the portal. Find the icon that looks like `>_` to the right of the search bar and click it.
 
> **First time opening Cloud Shell?** A setup panel will appear asking you to configure storage. Select your subscription, leave all other settings as default, and click **Create**. Azure creates a small storage account to save your files between sessions. This takes about 60 seconds and only happens once.
 
**Step 3: Select Bash, not PowerShell**
 
When Cloud Shell opens, check the top-left dropdown inside the terminal. It should say **Bash**. If it says **PowerShell**, click the dropdown and switch to Bash now. The commands in this lab will not work in PowerShell mode.
 
**Step 4: Create your project folder**
 
Run these two commands exactly as shown. The first creates a new folder. The second moves you into it.
 
```bash
mkdir terraform-lab
cd terraform-lab
```
 
Your prompt should now end with: `~/terraform-lab$`
 
---
 
### Phase 2: Write the Configuration
 
You will now create the Terraform configuration file. This file is a plain text description of the three resources you want Azure to build.
 
**Step 5: Open the built-in code editor**
 
```bash
code main.tf
```
 
A blank white editor panel opens above the terminal. This is where you will paste your code. Do not close the terminal below it.
 
**Step 6: Paste the full configuration**
 
Copy everything below, from the first `#` comment all the way to the final closing `}`, and paste it into the editor. Do not paste it into the terminal.
 
> **Before you paste:** Find the line `name = "rg-lab04-tf-[yourname]"`. You must change `[yourname]` to your own name before saving. If you leave a name that already exists in Azure, your apply will fail.
 
```hcl
# ── 1. Tell Terraform which provider to use ──────────────────────────────
# The "provider" is the plugin that knows how to talk to Azure.
# version = "~> 3.0" means use version 3.x - anything from 3.0 upward.
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.0"
    }
  }
}
 
# features {} is required by azurerm even when empty.
# It enables default behaviors that would otherwise need to be opted into.
provider "azurerm" {
  features {}
}
 
# ── 2. Resource Group ─────────────────────────────────────────────────────
# Every Azure resource must live inside a resource group.
# CHANGE "[yourname]" to YOUR name below.
resource "azurerm_resource_group" "rg" {
  name     = "rg-lab04-tf-[yourname]"   # <-- CHANGE THIS TO YOUR NAME
  location = "East US"
}
 
# ── 3. Virtual Network ────────────────────────────────────────────────────
# Notice: location and resource_group_name reference the resource group above.
# This creates a dependency - Terraform will create the RG first automatically.
resource "azurerm_virtual_network" "vnet" {
  name                = "vnet-terraform"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  address_space       = ["10.0.0.0/16"]
}
 
# ── 4. Subnet ─────────────────────────────────────────────────────────────
# 10.0.1.0/24 is a slice of the 10.0.0.0/16 VNet address space.
# A VNet can contain many subnets - each is a separate network segment.
resource "azurerm_subnet" "subnet" {
  name                 = "snet-backend"
  resource_group_name  = azurerm_resource_group.rg.name
  virtual_network_name = azurerm_virtual_network.vnet.name
  address_prefixes     = ["10.0.1.0/24"]
}
```
 
**Step 7: Change the resource group name to YOUR name**
 
Find this line in the editor:
```
name = "rg-lab04-tf-[yourname]"
```
Replace `[yourname]` with your own first name, lowercase, no spaces, no special characters.
 
Example: `name = "rg-lab04-tf-jsmith"`
 
**Step 8: Save and close the editor**
 
| Action          | Keyboard Shortcut |
|-----------------|-------------------|
| Save the file   | Ctrl + S          |
| Close the editor| Ctrl + Q          |
 
After pressing Ctrl + Q the editor panel closes and you are back at the terminal prompt. If the editor is still showing, press Ctrl + Q again.
 
**Step 9: Confirm the file was saved**
 
```bash
cat main.tf
```
 
Confirm your name appears in the resource group name and that the file is not empty. If the terminal prints nothing or shows an error, go back to Step 5 and repeat the process.
 
---
 
### Phase 3: Deploy - The Terraform Workflow
 
These three commands are the core Terraform workflow. Every cloud engineer runs them in this exact order, every time, without exception.
 
**Step 10: Initialize - download the Azure provider**
 
```bash
terraform init
```
 
What this does: Terraform reads your `main.tf`, sees that `azurerm` is a required provider, and downloads the Azure provider plugin into a hidden `.terraform` folder in your project directory. It also creates a `.terraform.lock.hcl` file that locks the exact provider version.
 
Expected output at the end:
```
Terraform has been successfully initialized!
```
 
> If you see an error: The most common cause is a network issue inside Cloud Shell. Wait 30 seconds and run `terraform init` again. If it fails three times, close Cloud Shell and reopen it from the portal toolbar.
 
**Step 11: Plan - preview what will be created**
 
```bash
terraform plan
```
 
What this does: Terraform compares your `main.tf` against the current state of Azure. Since nothing exists yet, it shows you a preview of exactly what it plans to create. Nothing is built or changed. This is read-only.
 
Look for this line in the output:
```
Plan: 3 to add, 0 to change, 0 to destroy.
```
 
The three resources are your Resource Group, Virtual Network, and Subnet. If you see a different number, check `main.tf` as you may have accidentally deleted a resource block.
 
> **Never skip plan.** In this lab with three resources, the plan output is simple. In a real project with dozens of resources, plan is what prevents you from accidentally modifying or destroying something you did not intend to touch. Build the habit now.
 
**Step 12: Apply - build the infrastructure**
 
```bash
terraform apply
```
 
What this does: Terraform shows the plan one more time and asks for explicit confirmation before making any changes to Azure.
 
When prompted, type the full word:
```
yes
```
 
Then press Enter. Typing `y` alone will not work. Terraform requires the complete word `yes` as a deliberate safety mechanism so you cannot accidentally confirm a destructive change.
 
Watch the output as Terraform works. Each resource shows `Creating...` followed by `Creation complete`. The Resource Group is created first, then the VNet (which depends on the RG), then the Subnet (which depends on the VNet). Terraform resolves the dependency order automatically.
 
Expected output at the end:
```
Apply complete! Resources: 3 added, 0 changed, 0 destroyed.
```
 
> Terraform has now written a `terraform.tfstate` file to your project folder. This is its record of everything it just created. Do not delete, move, or edit this file.
 
---
 
### Phase 4: Verify in the Azure Portal
 
**Step 13: Find your Resource Group**
 
1. Minimize Cloud Shell by clicking the minimize icon in the Cloud Shell toolbar
2. In the Azure portal top search bar, type **Resource groups** and select it
3. Find **rg-lab04-tf-[yourname]** in the list and click it
4. Inside the resource group you should see two resources: `vnet-terraform` and the subnet
**Step 14: Verify the Virtual Network and Subnet**
 
1. Click **vnet-terraform** to open it
2. In the left menu, click **Subnets**
3. Confirm `snet-backend` appears in the list with address prefix `10.0.1.0/24`
You just deployed three Azure resources without clicking a single form. That configuration file you wrote is the complete source of truth for this environment. You can share it, version it in Git, and recreate it anywhere.
 
---
 
### Phase 5: The Power of IaC - Add a Resource to a Live Environment
 
This phase demonstrates one of the most important things about Terraform: it only changes what is different. You are about to add a Network Security Group to your already-running environment. Terraform will add it without touching the Resource Group, VNet, or Subnet that already exist.
 
**Step 15: Reopen the editor**
 
Click the Cloud Shell restore icon in the portal toolbar to bring Cloud Shell back up. Then run:
 
```bash
code main.tf
```
 
**Step 16: Add the NSG resource block**
 
Scroll to the very bottom of the file, past the closing `}` of the subnet block. Add the following new block on a new line at the end. Do not modify any of the existing code.
 
```hcl
# ── 5. Network Security Group ────────────────────────────────────────────
# An NSG is a set of firewall rules that controls inbound and outbound traffic.
# This creates an empty NSG - rules would be added in a real deployment.
resource "azurerm_network_security_group" "nsg" {
  name                = "nsg-web"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
}
```
 
Save with **Ctrl + S**. Close with **Ctrl + Q**.
 
**Step 17: Plan - watch Terraform isolate only the new change**
 
```bash
terraform plan
```
 
Expected output:
```
Plan: 1 to add, 0 to change, 0 to destroy.
```
 
This is the key insight. The plan shows only `1 to add`, the NSG. Terraform compares your updated `main.tf` against the state file, sees that the Resource Group, VNet, and Subnet already exist and are unchanged, and correctly determines that only the NSG needs to be created. It will not rebuild or touch anything that does not need to change.
 
**Step 18: Apply - add the NSG to the live environment**
 
```bash
terraform apply
```
 
Type `yes` and press Enter.
 
Expected output:
```
Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```
 
Go back to the Azure portal and refresh your resource group. You should now see four resources: the VNet, the subnet, and the new `nsg-web`.
 
---
 
### Phase 6: Destroy - Clean Up Everything
 
When a lab is done, remove every resource to avoid any charges. One command handles it all.
 
**Step 19: Run the destroy command**
 
```bash
terraform destroy
```
 
Terraform reads the state file, compiles the complete list of everything it has created, and shows you the destruction plan before doing anything.
 
Look for this line in the plan:
```
Plan: 0 to add, 0 to change, 4 to destroy.
```
 
The four resources are the Resource Group, VNet, Subnet, and NSG. When prompted, type the full word:
```
yes
```
 
Then press Enter.
 
Expected output:
```
Destroy complete! Resources: 4 destroyed.
```
 
> **Always use `terraform destroy` instead of deleting resources through the Azure portal.** If you delete the resource group manually through the portal, the `terraform.tfstate` file still records that those resources exist. The next time you run `terraform plan`, Terraform will throw an error because it cannot find resources it believes are there. `terraform destroy` keeps your state file and your Azure environment perfectly in sync.
 
**Step 20: Verify resources are gone**
 
In the Azure portal, navigate to **Resource Groups**. Confirm that `rg-lab04-tf-[yourname]` no longer appears in the list. If it still appears, wait 30 seconds and refresh the page.
 
---
 
## Troubleshooting
 
| Error or Problem                     | Cause                                                                                  | Fix                                                                                                                                                         |
|--------------------------------------|----------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------|
| "Resource Group already exists"      | A resource group with this exact name already exists in your subscription              | Option 1: Change the name in `main.tf` to something unique. Option 2: Delete the existing resource group in the portal first.                               |
| `terraform: command not found`       | Cloud Shell dropped to a session where Terraform is not loaded                         | Close Cloud Shell and reopen it from the portal toolbar. Terraform is pre-installed in every new Cloud Shell session.                                       |
| Syntax error on line X               | A missing closing `}`, a missing quote `"`, or an extra character was introduced       | Run `code main.tf`. Go to the line number shown in the error. Look for a missing `}` or `"`. Every opening `{` needs a matching `}`.                        |
| Plan shows `0 to add`                | Terraform already created these resources and the state file records them as existing  | If you want to start fresh, run `terraform destroy` first, then `apply` again.                                                                              |
| `apply` fails partway through        | A transient Azure API error or quota limit                                             | Run `terraform apply` again. Terraform will skip resources that already exist and only retry what failed.                                                   |
| Editor does not close with Ctrl + Q  | Some browser key combinations intercept Ctrl + Q                                       | Click the X button in the top-right corner of the editor panel instead.                                                                                     |
 
---
 
## Key Concepts
 
| Term                        | Definition                                                                                                          |
|-----------------------------|---------------------------------------------------------------------------------------------------------------------|
| Infrastructure as Code (IaC)| Managing infrastructure through machine-readable configuration files rather than manual portal clicks.              |
| Terraform                   | An open-source IaC tool by HashiCorp that uses declarative configuration to provision cloud resources.              |
| Declarative Infrastructure  | You describe the desired end state. The tool figures out how to get there.                                          |
| Provider                    | A plugin that gives Terraform the ability to talk to a specific platform such as Azure, AWS, or GCP.               |
| azurerm                     | The official HashiCorp Terraform provider for Microsoft Azure.                                                      |
| terraform.tfstate           | Terraform's state file. Records every resource Terraform has created. Never delete or edit this manually.           |
| terraform init              | Downloads the required provider plugins. Run once per project.                                                      |
| terraform plan              | A read-only preview of what Terraform will create, change, or destroy. Always run before apply.                     |
| terraform apply             | Executes the configuration and creates or updates resources in Azure.                                               |
| terraform destroy           | Removes all resources tracked in the state file. Keeps the state and Azure environment in sync.                     |
| Resource Group              | A logical container for Azure resources. Every resource must live inside one.                                       |
| Virtual Network (VNet)      | An isolated network in Azure. Equivalent to a private data center network.                                          |
| Subnet                      | A range of IP addresses within a VNet used to segment resources into separate network zones.                        |
| NSG                         | Network Security Group. A set of firewall rules controlling inbound and outbound traffic to Azure resources.        |
| address_space               | The total IP range available to a VNet (e.g., `10.0.0.0/16` gives 65,536 addresses).                               |
| address_prefixes            | The IP range assigned to a specific subnet carved out of the VNet address space.                                    |
| Dependency resolution       | Terraform automatically determines the correct creation order based on resource references in the configuration.    |
 
---
 
## Knowledge Check
 
Before moving on, make sure you can answer these questions without looking at the lab:
 
1. What does `terraform init` download and why does it need to?
2. What is the difference between `terraform plan` and `terraform apply`?
3. What is the `terraform.tfstate` file and why should you never delete it manually?
4. Why did `terraform plan` show `1 to add` (not 4) when you added the NSG in Phase 5?
5. Why should you use `terraform destroy` instead of deleting resources through the Azure portal?
---
 
## What You Accomplished
 
- Wrote a Terraform configuration file describing a Resource Group, Virtual Network, and Subnet
- Ran the full `init` then `plan` then `apply` workflow and deployed all three resources to Azure
- Added a Network Security Group to a live environment and watched Terraform change only the new resource
- Destroyed the entire environment with a single `terraform destroy` command
- Understood why IaC is the professional standard for managing cloud infrastructure at scale
---
 
## Lab Completion Checklist
 
- [ ] Cloud Shell opened and Bash mode confirmed
- [ ] `terraform-lab` project folder created
- [ ] `main.tf` written with your name in the resource group name
- [ ] `terraform init` completed successfully with "Terraform has been successfully initialized!"
- [ ] `terraform plan` showed `Plan: 3 to add, 0 to change, 0 to destroy`
- [ ] `terraform apply` showed `Apply complete! Resources: 3 added, 0 changed, 0 destroyed`
- [ ] Resource Group, VNet, and Subnet verified in the Azure portal
- [ ] NSG block added to `main.tf`
- [ ] Second `terraform plan` showed `Plan: 1 to add, 0 to change, 0 to destroy`
- [ ] Second `terraform apply` showed `Apply complete! Resources: 1 added, 0 changed, 0 destroyed`
- [ ] `terraform destroy` showed `Destroy complete! Resources: 4 destroyed`
- [ ] Resource group confirmed gone in the Azure portal
---
 
*Lab authored by **Jair Smith** · Azure Cloud Security Series · Lab 04 of ongoing curriculum*
 
