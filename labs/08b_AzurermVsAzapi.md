# Lab overview

In this lab, you will explore the limits of the azurerm provider and use as fallback the azapi provider.  

We will first deploy a Virtual Network with an ARM template. We use the **azurerm** provider and more specifically the `azurerm_resource_group_template_deployment` resource, and wrap the Azure RM template for the VNet in this resource block.  
Next we switch to a new provider named **azapi** which allows us to make API calls to the ARM REST API.  

Short notes on _azurerm_ versus _azapi_:

- **azurerm** exposes each Azure resource as a typed Terraform resource, easily maintainable and documented. Each resource is built on a specific version of the Azure REST API. A property or a resource that Azure supports does not exist in Terraform until someone adds it to the provider and a new version is released.
- **azapi** is a thin layer over the Azure Resource Manager (ARM) REST API. You give it a resource type, an API version and a JSON-like `body`. Any resource type and API version that Azure supports can be used the day it is published.

To make the difference visible, this lab **pins azurerm to a dedicated version (`~> 4.0.0`)** and next tries to use new features that were added to azurerm later on. These new features, out of reach for azurerm, will be deployed using the **azapi** provider.  

> Note: Drift detection is compared in depth in lab _08c_.

- [Lab overview](#lab-overview)
  - [Objectives](#objectives)
  - [Instructions](#instructions)
    - [Before you start](#before-you-start)
    - [Exercise 1: Setup your environment (_tfstate_ and project template)](#exercise-1-setup-your-environment-tfstate-and-project-template)
      - [Backend](#backend)
      - [Variables](#variables)
      - [Providers](#providers)
      - [Locals](#locals)
      - [Terraform Init](#terraform-init)
    - [Exercise 2: Create a Virtual Network using an ARM template](#exercise-2-create-a-virtual-network-using-an-arm-template)
    - [Exercise 3: Hit the limit of azurerm (a missing property)](#exercise-3-hit-the-limit-of-azurerm-a-missing-property)
    - [Exercise 4: Solve the limitation (two options)](#exercise-4-solve-the-limitation-two-options)
      - [Option A: the ARM template](#option-a-the-arm-template)
      - [Option B: azapi provider](#option-b-azapi-provider)
    - [Exercise 5: Hit the limit of azurerm (a missing resource)](#exercise-5-hit-the-limit-of-azurerm-a-missing-resource)
    - [Exercise 6: The limits of azapi](#exercise-6-the-limits-of-azapi)
    - [Exercise 7: Remove Resources](#exercise-7-remove-resources)

## Objectives

After you complete this lab, you will be able to:

- Identify a feature that Azure supports but that a given azurerm version does not,
- Deploy that feature with an ARM template wrapped in Terraform, and next with azapi,
- Compare state content, plan readability and destroy behaviour between the approaches,
- Explain the limits and risks of azapi.

## Instructions

### Before you start

- Ensure Terraform (version ~> 1.13.0) is installed and available from system PATH.
- Ensure Azure CLI is installed.
- Check your access to the Azure Subscriptions and Resource Groups provided for this training.

### Exercise 1: Setup your environment (_tfstate_ and project template)

1/ Create the container for _tfstate_

In your _main_ Resource Group (the one tagged with `layer` = `main`), create a Storage Account, with a Blob container named `tfstate` to store the _tfstate_ file.

2/ Get project template

Clone the repository <https://github.com/Anne-Gaelle-Cellenza/training-terraform-intermediate-labs-setup>

```bash
git clone https://github.com/Anne-Gaelle-Cellenza/training-terraform-intermediate-labs-setup.git
cd training-terraform-intermediate-labs-setup
```

> This template contains a basic Terraform project configuration that:
>
> - uses a `data` resource group,
> - defines two variables `resource_group_name` and `location`,
> - contains a `configuration\dev` folder with _backend_ and _tfvars_ for dev.

3/ Configure the project template to use your environment

#### Backend

The project template uses a partial backend configuration: we don't define the backend configuration in the `terraform` block but in an external file, read at `terraform init` time.

In the _configuration/dev_ folder, update the `backend.hcl` file as:

```hcl
  resource_group_name  = "the_name_of_your_main_resource_group"
  storage_account_name = "the_name_of_the_storage_account_you_just_created"
  container_name      = "tfstate"
  key                 = "dynamicappservice.tfstate"
```

> Define your backend using the Storage Account you created few minutes ago.

#### Variables

In the _configuration/dev_ folder, update the `dev.tfvars` file:

```hcl
resource_group_name = "the_name_of_your_main_resource_group"
```

#### Providers

We will be using in this lab two providers, azurerm and azapi, declared in the terraform block.  

Replace the content of `src/version.tf` with:

```hcl
terraform {
  required_version = ">= 1.13.0"

  backend "azurerm" {}

  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 4.0.0"
    }
    azapi = {
      source  = "Azure/azapi"
      version = "~> 2.0"
    }
  }
}
```

> azurerm provider version
> `~> 4.0.0` means: any 4.0.x version, and nothing from 4.1 onwards. This is the deliberate limitation of the lab.

Add a new provider block in `provider.tf` and udpate the `features` block of the azurerm provider:

```hcl
provider "azurerm" {
  features {
    template_deployment {
      delete_nested_items_during_deletion = false
    }
  }
}

provider "azapi" {
}
```

> The azapi provider block is **not required** if using default settings.  
> If no provider block is specified azapi will pick the subscription and tenant from the azurerm provider configuration (e.g. the `ARM_SUBSCRIPTION_ID` environment variable and Azure CLI login).  
>
> The `delete_nested_items_during_deletion` field controls whether deleting any `azurerm_resource_group_template_deployment` resource will automatically attempt to delete resources deployed by the ARM Template. By setting it to `false` we do not force ARM resources deletion when destroying the `azurerm_resource_group_template_deployment` resource (See observation in Exercise 7 below).

#### Locals

Create a `src/locals.tf` file (or update existing file if any) to set VNet names:

```hcl
locals {
  vnet_name       = "vnet-lab08b-arm"
  vnet_name_limit = "vnet-lab08b-limit"
  vnet_name_azapi = "vnet-lab08b-azapi"
}
```

#### Terraform Init

Run the `terraform init` and check the installed versions:

PowerShell

```powershell
az login
az account set --subscription "the_main_subscription_id"
$env:ARM_SUBSCRIPTION_ID="the_main_subscription_id"
# use the '-reconfigure' option in case your local folder was previously configured with a different backend (e.g. from previous labs)
terraform init -backend-config="..\configuration\dev\backend.hcl" -reconfigure
terraform version
```

Bash

```bash
az login
az account set --subscription "the_main_subscription_id"
export ARM_SUBSCRIPTION_ID="the_main_subscription_id"
# use the '-reconfigure' option in case your local folder was previously configured with a different backend (e.g. from previous labs)
terraform init -backend-config="..\configuration\dev\backend.hcl" -reconfigure
terraform version
```

> **Question:**
>
> - Which azurerm version did Terraform install?  
> - Is it the latest one published in the registry?  
> - Why not?

### Exercise 2: Create a Virtual Network using an ARM template

Create `src/arm_deployment.tf` file to deploy a VNet using an ARM Template and the name defined above:

```hcl
resource "azurerm_resource_group_template_deployment" "example" {
  name                = "example-deploy"
  resource_group_name = data.azurerm_resource_group.self.name
  deployment_mode     = "Incremental"
  parameters_content = jsonencode({
    "vnetName" = {
      value = local.vnet_name
    }
  })
  template_content = <<TEMPLATE
{
    "$schema": "https://schema.management.azure.com/schemas/2015-01-01/deploymentTemplate.json#",
    "contentVersion": "1.0.0.0",
    "parameters": {
        "vnetName": {
            "type": "String",
            "metadata": {
                "description": "Name of the VNET"
            }
        }
    },
    "variables": {},
    "resources": [
        {
            "type": "Microsoft.Network/virtualNetworks",
            "apiVersion": "2020-05-01",
            "name": "[parameters('vnetName')]",
            "location": "[resourceGroup().location]",
            "properties": {
                "addressSpace": {
                    "addressPrefixes": [
                        "10.0.0.0/16"
                    ]
                }
            }
        }
    ],
    "outputs": {
      "exampleOutput": {
        "type": "String",
        "value": "someoutput"
      }
    }
}
TEMPLATE
}

output "arm_example_output" {
  value = jsondecode(azurerm_resource_group_template_deployment.example.output_content).exampleOutput.value
}
```

Run the following command to deploy the VNET (you can verify the resoruce creation using the Azure portal):

```bash
terraform apply -var-file="..\configuration\dev\dev.tfvars" -auto-approve
```

**Observe** and record (below output will be used for comparison with output from Exercise 4/option B):

```bash
terraform state list
terraform state show azurerm_resource_group_template_deployment.example
az network vnet show --resource-group "the_name_of_your_lab_resource_group" --name vnet-lab08b-arm --query "{name:name, location:location, addressSpace:addressSpace.addressPrefixes}"
```

### Exercise 3: Hit the limit of azurerm (a missing property)

Azure now supports a `privateEndpointVNetPolicies` property for Virtual Networks. It enables network policies for private endpoints at the VNet level.  
This new property was added to azurerm provider **starting version 4.15.0** and is named `private_endpoint_vnet_policies` (pull request [#27830](https://github.com/hashicorp/terraform-provider-azurerm/issues/27830), merged 8 January 2025).

Let's try to configure our Virtual Network with this new property.  

Create `src/limit_property.tf` file with:

```hcl
resource "azurerm_virtual_network" "limit" {
  name                = var.vnet_name_limit
  location            = var.location
  resource_group_name = data.azurerm_resource_group.self.name
  address_space       = ["10.2.0.0/16"]

  private_endpoint_vnet_policies = "Basic"
}
```

Validate this configuration with:

```bash
terraform validate
```

You should get an error as (with azurerm 4.0.1):

```text
Error: Unsupported argument

  on limit_property.tf line 11, in resource "azurerm_virtual_network" "limit":
  11:   private_endpoint_vnet_policies = "Basic"

An argument named "private_endpoint_vnet_policies" is not expected here.
```

> **Questions:**
>
> - The feature exists in Azure. Why can Terraform not use it?
> - Compare the azurerm documentation of the 4.0.x version with the latest one for `azurerm_virtual_network`.
> - Terraform and azurerm are updated frequently. Is a frequent release cycle the same thing as being up to date with the Azure API?

**Delete `limit_property.tf`** before continuing, otherwise `validate`, `plan` and `apply` keep failing.

### Exercise 4: Solve the limitation (two options)

#### Option A: the ARM template

The ARM template is not limited by the azurerm version, because Azure itself reads the template.  
Update the template in `arm_deployment.tf`: set the `apiVersion` of the VNet to `2024-03-01` and add the missing property in the `properties` block.

```json
"apiVersion": "2024-03-01",
[...]
            "properties": {
                "addressSpace": {
                    "addressPrefixes": [
                        "10.0.0.0/16"
                    ]
                },
                "privateEndpointVNetPolicies": "Basic"
            }
[...]
```

Run the following command to update the resource:

```bash
terraform apply -var-file="..\configuration\dev\dev.tfvars" -auto-approve
```

**Observe** the terraform plan logs, and the effective resource from Azure for the added property on private endpoint policies:

```bash
terraform plan -var-file="..\configuration\dev\dev.tfvars" 
az network vnet show --resource-group "the_name_of_your_main_resource_group" --name vnet-lab08b-arm --query "privateEndpointVNetPolicies"
```

> **Question:**
>
> - What does the plan show?
> - Do you see the property change, or a change inside a JSON string?

#### Option B: azapi provider

The AzAPI provider is a very thin layer on top of the Azure ARM REST APIs. Using AzAPI provider results into an API call.  

Create `src/azapi_vnet.tf` file with below contents:

```hcl
resource "azapi_resource" "vnet" {
  type      = "Microsoft.Network/virtualNetworks@2024-03-01"
  name      = local.vnet_name_azapi
  parent_id = data.azurerm_resource_group.self.id
  location  = var.location

  body = {
    properties = {
      addressSpace = {
        addressPrefixes = ["10.1.0.0/16"]
      }
      privateEndpointVNetPolicies = "Basic"
    }
  }
}

output "azapi_vnet_id" {
  value = azapi_resource.vnet.id
}
```

> Notice:
>
> - the `type` value which is the ARM resource type, adding the API version after `@` sign,
> - the **2024-03-01** API version which is the oldest one from which the new property is available,
> - the `parent_id` value that refers to the Resource ID of the scope for the API call. For a resource in a Resource Group, it is the ID of the Resource Group,
> - the `body` contents which is the same JSON that you would send to the REST API, written as an HCL object.

```powershell
terraform plan -var-file="..\configuration\dev\dev.tfvars"
terraform apply -var-file="..\configuration\dev\dev.tfvars"
az network vnet show --resource-group "the_name_of_your_lab_resource_group" --name vnet-lab08b-azapi --query "privateEndpointVNetPolicies"
terraform state list
terraform state show azapi_resource.vnet
```

> **Questions:**
>
> - Compare `terraform state list` and `terraform state show` with Exercise 2. What does Terraform now know?
> - Compare how the plan behaves in Option A and Option B.

### Exercise 5: Hit the limit of azurerm (a missing resource)

Network Security Perimeters (NSP) is a (quite) new Azure feature. In azurerm, `azurerm_network_security_perimeter` and its related resources were added in **provider version 4.58.0**.

Create `src/limit_resource.tf` file with:

```hcl
resource "azurerm_network_security_perimeter" "limit" {
  name                = "nsp-lab08b"
  location            = var.location
  resource_group_name = data.azurerm_resource_group.self.name
}
```

Validate this configuration with:

```bash
terraform validate
```

You should get an error as (with azurerm 4.0.1):

```text
Error: Invalid resource type

  on limit_resource.tf line 1, in resource "azurerm_network_security_perimeter" "limit":
   1: resource "azurerm_network_security_perimeter" "limit" {

The provider hashicorp/azurerm does not support resource type
"azurerm_network_security_perimeter".
```

This time the _whole resource type_ is missing, not only a property.  
**Delete `limit_resource.tf`** to avoid endless failures, and create `src/azapi_nsp.tf` file to use the AzAPI provider with:

```hcl
resource "azapi_resource" "nsp" {
  type      = "Microsoft.Network/networkSecurityPerimeters@2024-07-01"
  name      = "nsp-lab08b"
  parent_id = data.azurerm_resource_group.self.id
  location  = var.location
  body      = {}
}
```

> Two attention points here:
>
> - The GA API versions of `Microsoft.Network/networkSecurityPerimeters` start at **2024-07-01**. Earlier versions (**2024-06-01-preview** and older) are preview.  
> - Not all regions necessarily support this new resource.  
>
> You can list the API versions and the regions where Azure offers this resource type with:
>
> ```bash
> az provider show --namespace Microsoft.Network --query "resourceTypes[?resourceType=='networkSecurityPerimeters'].{versions: apiVersions, locations: locations}" -o json
> ```
>
> The locations list is not filtered by the targeted subscription. To see the regions this subscription can use, run:
>
> ```bash
> az account list-locations --query "[].displayName" -o tsv
> ```
>
> Choose a region that appears in both lists for the `location` variable.

Run the following command to deploy the resource:

```bash
terraform apply -var-file="..\configuration\dev\dev.tfvars" -auto-approve
```

### Exercise 6: The limits of azapi

azapi removes the limits of the provider, but it has costs of its own. Explore them.

**1. API version ownership.**  
In `azapi_vnet.tf`, change the version to `2024-01-01` and run `terraform plan`.

```text
Error: Invalid configuration
embedded schema validation failed: the argument "body" is invalid:
`properties.privateEndpointVNetPolicies` is not expected here. ...
You can try to update `azapi` provider to the latest version or disable the validation using the feature flag
`schema_validation_enabled = false` within the resource block
```

> API version is really important when using the AzAPI provider!

**2. Schema validation.**  
Restore `2024-03-01`, then introduce a typo in a property name (for example `privateEndpointVNetPolicy` instead of `privateEndpointVNetPolicies`) and run `terraform plan`.  
Then add `schema_validation_enabled = false` to the resource block and run the plan again.

> **Questions:**
>
> - What does the schema validation catch? What happens when it is disabled?
> - Who is responsible for picking the API version, and for upgrading it later?
> - Does changing the API version change the behaviour of the resource, even when the `body` does not change?

**3. Other pros and cons** using the azapi provider:

- The `body` is not typed: no autocompletion, no per-argument documentation in the provider,
- The state contains what the API returns. Check for secrets with `terraform state show`,
- azurerm and azapi can be mixed in the same configuration (this lab does it: the Resource Group comes from an azurerm data source),
- Other azapi resources: `azapi_update_resource` (to change a few properties of a resource created elsewhere), `azapi_resource_action` (to call an action, for example a `POST` operation), `azapi_resource_list` (to list resources of a specific type under a given scope), ...

### Exercise 7: Remove Resources

First, check what destroying the ARM template deployment really removes. Destroy only that resource:

```bash
terraform destroy -target=azurerm_resource_group_template_deployment.example -var-file="..\configuration\dev\dev.tfvars"
az network vnet list --resource-group "the_name_of_your_lab_resource_group" --query "[].name" -o tsv
```

> **Question:**
>
> - Is `vnet-lab08b-arm` still there?
> - Look up the `template_deployment` setting in the `features` block of the azurerm provider and explain what you observe...
> The `azurerm_resource_group_template_deployment` resource will automatically attempt to delete resources deployed by the ARM Template when it is deleted. This behaviour is disabled in the provider `features` block by setting the `delete_nested_items_during_deletion` field to `false` within the `template_deployment` block.
>
> ```hcl
> provider "azurerm" {
>   features {
>     template_deployment {
>       delete_nested_items_during_deletion = false
>     }
>   }
> }
> ```

Then remove everything else:

```bash
terraform destroy -var-file="..\configuration\dev\dev.tfvars" -auto-approve
az resource list --resource-group "the_name_of_your_lab_resource_group" -o table
```
