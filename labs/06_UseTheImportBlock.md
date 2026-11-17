# Lab overview

In this lab, you will continue dealing with resources import, now using the *import* block.  
We create a VNet with Terraform, and add subnets outside Terraform (az cli).  
We will also have a look at code refactoring with the moved block.

- [Lab overview](#lab-overview)
  - [Objectives](#objectives)
  - [Instructions](#instructions)
    - [Before you start](#before-you-start)
    - [Exercise 1: Setup your environment (*tfstate* and project template)](#exercise-1-setup-your-environment-tfstate-and-project-template)
      - [Backend](#backend)
      - [Variables](#variables)
    - [Exercise 2: Create a Virtual Network with terraform and add subnets with az cli](#exercise-2-create-a-virtual-network-with-terraform-and-add-subnets-with-az-cli)
    - [Exercise 3: Import 1 subnet with an import block](#exercise-3-import-1-subnet-with-an-import-block)
    - [Exercise 4: Import the other subnets with import blocks and generate automatically the terraform configuration](#exercise-4-import-the-other-subnets-with-import-blocks-and-generate-automatically-the-terraform-configuration)
    - [Exercise 5: Refactor the subnets terraform configuration with a for\_each, and move the resources with a moved block](#exercise-5-refactor-the-subnets-terraform-configuration-with-a-for_each-and-move-the-resources-with-a-moved-block)

## Objectives

After you complete this lab, you will be able to:

- Import resources using the *import* block,
- Use terraform feature for generating automatically resources configuration,
- Refactor code using the *moved* block.

## Instructions

### Before you start

- Ensure Terraform (version ~> 1.13) is installed and available from system's PATH.
- Ensure Azure CLI is installed.
- Check your access to the Azure Subscriptions and Resource Groups provided for this training.

### Exercise 1: Setup your environment (*tfstate* and project template)

1/ Create the container for *tfstate*

In your *main* Resource Group (the one tagged with `layer` = `main`), create a Storage Account, with a Blob container named `tfstate` to store the *tfstate* file.

2/ Get project template

Clone the repository https://github.com/Anne-Gaelle-Cellenza/training-terraform-intermediate-labs-setup

```bash
git clone https://github.com/Anne-Gaelle-Cellenza/training-terraform-intermediate-labs-setup.git
cd training-terraform-intermediate-labs-setup
```

> This template contains a basic Terraform project configuration that:
>
> - uses a `data` resource group,
> - defines two variables `resource_group_name` and `location`,
> - contains a `configuration\dev` folder with *backend* and *tfvars* for dev.

3/ Configure the project template to use your environment

#### Backend

The project template uses a partial backend configuration: we don't define the backend configuration in the `terraform` block but in an external file, read at `terraform init` time.

In the *configuration/dev* folder, update the `backend.hcl` file as:

```hcl
  resource_group_name  = "the_name_of_your_main_resource_group"
  storage_account_name = "the_name_of_the_storage_account_you_just_created"
  container_name      = "tfstate"
  key                 = "importblock.tfstate"
```

> Define your backend using the Storage Account you created few minutes ago.

#### Variables

In the *configuration/dev* folder, update the `dev.tfvars` file:

```hcl
resource_group_name = "the_name_of_your_main_resource_group"
```

### Exercise 2: Create a Virtual Network with terraform and add subnets with az cli

Add the following object to your `variable.tf` file:

```hcl
variable "vnet_nameconfig" {
  type = object({
    address_space = optional(list(string), ["10.100.0.0/24"])
    location      = optional(string, "westeurope")
    vnet_namename = optional(string,"vnet_name")
  })
}

```

Specify the value in the `dev.tfvars` file:

```hcl
vnet_nameconfig = {
    address_space = ["10.0.0.0/24"]
}
```

Add the following to your `main.tf` file:

```hcl
resource "azurerm_virtual_network" "main_vnet" {
  name                = var.vnet_nameconfig.vnet_namename
  address_space       = var.vnet_nameconfig.address_space
  location            = var.vnet_nameconfig.location
  resource_group_name = data.azurerm_resource_group.self.name
}
```

Create the vnet with a `terraform apply` command:

PowerShell

```powershell
az login
az account set --subscription "the_main_subscription_id"
$env:ARM_SUBSCRIPTION_ID="the_main_subscription_id"
# use the '-reconfigure' option in case your local folder was previously configured with a different backend (e.g. from previous labs)
terraform init -backend-config="..\configuration\dev\backend.hcl" -reconfigure
terraform apply -var-file="..\configuration\dev\dev.tfvars" -auto-approve
```

Bash

```bash
az login
az account set --subscription "the_main_subscription_id"
export ARM_SUBSCRIPTION_ID="the_main_subscription_id"
# use the '-reconfigure' option in case your local folder was previously configured with a different backend (e.g. from previous labs)
terraform init -backend-config="..\configuration\dev\backend.hcl" -reconfigure
terraform apply -var-file="..\configuration\dev\dev.tfvars" -auto-approve
```

You can check the existence of your virtual network from Azure Portal or using the command `az network`:

```powershell
az network vnet show -n vnet_name -g "the_name_of_your_main_resource_group"
```

Create 4 subnets on your virtual network using az cli (use /26 on the VNet CIDR range)

```powershell
az network vnet subnet create -n "subnet1" -g "the_name_of_your_main_resource_group" --vnet-name vnet_name --address-prefixes 10.0.0.0/26
az network vnet subnet create -n "subnet2" -g "the_name_of_your_main_resource_group" --vnet-name vnet_name --address-prefixes 10.0.0.64/26
az network vnet subnet create -n "subnet3" -g "the_name_of_your_main_resource_group" --vnet-name vnet_name --address-prefixes 10.0.0.128/26
az network vnet subnet create -n "subnet4" -g "the_name_of_your_main_resource_group" --vnet-name vnet_name --address-prefixes 10.0.0.192/26
```

Check again the virtual network with the command `az network vnet show -n vnet_name -g "the_name_of_your_main_resource_group"`. You should now see the created subnets in the `subnets` section.

We now have

- a VNet managed by Terraform
- with 4 subnets created outside Terraform.

If we have a look to the VNet resource in tfstate with `terraform state show azurerm_virtual_network.main_vnet`, we get

```bash
$ terraform state show azurerm_virtual_network.main_vnet
# azurerm_virtual_network.main_vnet:
resource "azurerm_virtual_network" "main_vnet" {
    address_space                  = [
        "10.0.0.0/24",
    ]
    bgp_community                  = null
    dns_servers                    = []
    edge_zone                      = null
    flow_timeout_in_minutes        = 0
    guid                           = "51024ff9-a1e8-410d-b819-39b07baf440a"
    id                             = "/subscriptions/0d164587-df91-4ac8-993b-9c531ff15e7c/resourceGroups/rg-annegaelle-262306/providers/Microsoft.Network/virtualNetworks/vnet_name"
    location                       = "westeurope"
    name                           = "vnet_name"
    private_endpoint_vnet_policies = "Disabled"
    resource_group_name            = "rg-annegaelle-262306"
    subnet                         = []
}
```

> The result displays an empty list of subnets, because those are not (yet) managed by Terraform.

### Exercise 3: Import 1 subnet with an import block

To add the subnets in the Terraform coonfiguration, we will use the Terraform *import* block on the first subnet.

> Import block must respect the following syntax
>
> ```hcl
> 
> import {
>     to = <resource_address_in_tf_config>
>     id = <resource_id_in_the_cloud_provider>
> }
> ```

Create a new file `imports.tf` and add the following block to import your first subnet:

```hcl
import {
    to = azurerm_subnet.subnet1
    id = "subnet_resource_id" # get the Azure resource id from the portal or from the `az network vnet show` cmd run before
}
```

Create a new file `subnets.tf` and add the following configuration to match the imported resource:

```hcl
resource "azurerm_subnet" "subnet1" {
  name = "subnet1"
  resource_group_name = data.azurerm_resource_group.self.name
  virtual_network_name = azurerm_virtual_network.main_vnet.name
  address_prefixes = ["10.0.0.0/26"]
}
```

Run `terraform plan`

```powershell
terraform plan -var-file="..\configuration\dev\dev.tfvars"
```

The plan indicates no resource to create or modify, but there is a `1 to import` mention.  
![import](../Img/lab6-001.png)

Run a terraform apply to validate the import:

```powershell
terraform apply -var-file="..\configuration\dev\dev.tfvars" -auto-approve
```

### Exercise 4: Import the other subnets with import blocks and generate automatically the terraform configuration

To import th other subnets, add the following *import* blocks in the `imports.tf` file (replace with each subnet id):

```hcl
import {
    to = azurerm_subnet.subnet2
    id = "subnet2_resource_id"
}


import {
    to = azurerm_subnet.subnet3
    id = "subnet3_resource_id"
}

import {
    to = azurerm_subnet.AzureBastionSubnet
    id = "AzureBastionSubnet_resource_id"
}
```

Run a `terraform plan` command (with `-var-file` option).

> You should get an error for each subnet to be imported such as `The configuration for the given import target azurerm_subnet.subnet2 does not exist.`.  
> This is an expected behaviour, as we didn't add the Terraform resource block for each of these subnets in Terraform template files.

```bash
Planning failed. Terraform encountered an error while generating this plan.

╷
│ Error: Configuration for import target does not exist
│
│   on import.tf line 7, in import:
│    7:   to = azurerm_subnet.subnet2
│
│ The configuration for the given import target azurerm_subnet.subnet2 does not exist. If you wish to automatically generate
│ config for this resource, use the -generate-config-out option within terraform plan. Otherwise, make sure the target resource
│ exists within your configuration. For example:
│
│   terraform plan -generate-config-out=generated.tf
╵
```

> Note the suggestion to use `terraform plan -generate-config-out=generated.tf` !
> We will go that way and let Terraform generate the templates for the subnets.

Run a new `terraform plan` command adding the `-generate-config-out="generated.tf"` option (and don't forget the usual `-var-file` option).

```powershell
terraform plan -var-file="..\configuration\dev\dev.tfvars" -generate-config-out="generated.tf"
```

You may get the folllowing error:

```bash
Planning failed. Terraform encountered an error while generating this plan.

╷
│ Error: Not enough list items
│ 
│   with azurerm_subnet.subnet2,
│   on generated.tf line 6:
│   (source code not available)
│ 
│ Attribute service_endpoint_policy_ids requires 1 item minimum, but config has only 0 declared.
```

> **Ignore** this error for now, an check in your folder the existence of a `generated.tf` file.

It should contains something similar to:

```go
# __generated__ by Terraform
# Please review these resources and move them into your main configuration files.

# __generated__ by Terraform
resource "azurerm_subnet" "AzureBastionSubnet" {
  address_prefixes                              = ["10.0.0.192/26"]
  name                                          = "AzureBastionSubnet"
  private_endpoint_network_policies_enabled     = false
  private_link_service_network_policies_enabled = true
  resource_group_name                           = "rg_name"
  service_endpoint_policy_ids                   = []
  service_endpoints                             = []
  virtual_network_name                          = "vnet_name"
}

# __generated__ by Terraform
resource "azurerm_subnet" "subnet3" {
  address_prefixes                              = ["10.0.0.128/26"]
  name                                          = "subnet3"
  private_endpoint_network_policies_enabled     = false
  private_link_service_network_policies_enabled = true
  resource_group_name                           = "rg_name"
  service_endpoint_policy_ids                   = []
  service_endpoints                             = []
  virtual_network_name                          = "vnet_name"
}

# __generated__ by Terraform
resource "azurerm_subnet" "subnet2" {
  address_prefixes                              = ["10.0.0.64/26"]
  name                                          = "subnet2"
  private_endpoint_network_policies_enabled     = false
  private_link_service_network_policies_enabled = true
  resource_group_name                           = "rg_name"
  service_endpoint_policy_ids                   = []
  service_endpoints                             = []
  virtual_network_name                          = "vnet_name"
}

```

Comment the line for `service_endpoints` and `service_endpoint_policy_ids` as those were the parameters involved in the error, and re-run a terraform plan. You sould get an output similar to:

```bash
Terraform will perform the following actions:

  # azurerm_subnet.AzureBastionSubnet will be imported
    resource "azurerm_subnet" "AzureBastionSubnet" {
        address_prefixes                               = [
            "10.0.0.192/26",
        ]
        enforce_private_link_endpoint_network_policies = true
        enforce_private_link_service_network_policies  = false
        id                                             = "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsg_name/providers/Microsoft.Network/virtualNetworks/vnet_name/subnets/AzureBastionSubnet"
        name                                           = "AzureBastionSubnet"
        private_endpoint_network_policies_enabled      = false
        private_link_service_network_policies_enabled  = true
        resource_group_name                            = "rsg_name"
        service_endpoint_policy_ids                    = []
        service_endpoints                              = []
        virtual_network_name                           = "vnet_name"
    }

  # azurerm_subnet.subnet2 will be imported
    resource "azurerm_subnet" "subnet2" {
        address_prefixes                               = [
            "10.0.0.64/26",
        ]
        enforce_private_link_endpoint_network_policies = true
        enforce_private_link_service_network_policies  = false
        id                                             = "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsg_name/providers/Microsoft.Network/virtualNetworks/vnet_name/subnets/subnet2"
        name                                           = "subnet2"
        private_endpoint_network_policies_enabled      = false
        private_link_service_network_policies_enabled  = true
        resource_group_name                            = "rsg_name"
        service_endpoint_policy_ids                    = []
        service_endpoints                              = []
        virtual_network_name                           = "vnet_name"
    }

  # azurerm_subnet.subnet3 will be imported
    resource "azurerm_subnet" "subnet3" {
        address_prefixes                               = [
            "10.0.0.128/26",
        ]
        enforce_private_link_endpoint_network_policies = true
        enforce_private_link_service_network_policies  = false
        id                                             = "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsg_name/providers/Microsoft.Network/virtualNetworks/vnet_name/subnets/subnet3"
        name                                           = "subnet3"
        private_endpoint_network_policies_enabled      = false
        private_link_service_network_policies_enabled  = true
        resource_group_name                            = "rsg_name"
        service_endpoint_policy_ids                    = []
        service_endpoints                              = []
        virtual_network_name                           = "vnet_name"
    }

Plan: 3 to import, 0 to add, 0 to change, 0 to destroy.

─────────────────────────────────────────────────────────────────────────────────────────────────────

Note: You didn't use the -out option to save this plan, so Terraform can't guarantee to take exactly these actions if
you run "terraform apply" now.
```

Run the apply command to validate the import.  
After the apply is complete, check the state with a `terraform state list`.You should now see the subnets in the state:

```bash
df@df2204lts:~/Documents/myrepo/tf-intermediaire-lab2/src$ terraform state list
data.azurerm_resource_group.feature_rg
data.azurerm_resource_group.main
azurerm_subnet.AzureBastionSubnet
azurerm_subnet.subnet1
azurerm_subnet.subnet2
azurerm_subnet.subnet3
azurerm_virtual_network.main_vnet
```

### Exercise 5: Refactor the subnets terraform configuration with a for_each, and move the resources with a moved block

After the previous exercise, we have now, available in our terraform configuration, the 4 subnets defined as independant resources.
However, this is not optimized, and we would like to DRY this with a for_each.

To get started, we will comment all the import blocks in the `imports.tf` file.
To get a better view, we will also move all of our subnets configuration in a new file called `subnets.tf`

In this file, we will add the following configuration for the subnets:

```go

resource "azurerm_subnet" "vnet_name_subnets" {

  for_each = var.vnet_name_subnets_list

  name = each.key
  resource_group_name = data.azurerm_resource_group.self.name
  virtual_network_name = azurerm_virtual_network.main_vnet.name
  address_prefixes = [each.value.address_space]
}

```

We will also add the following variable in the `variable.tf` file:

```go

variable "vnet_name_subnets_list" {
  type = map(object({
    address_space = string
  }))
  description = "List of subnets create"

  default = {
    subnet1 = {
        address_space = "10.0.0.0/26"
    },
        subnet2 = {
        address_space = "10.0.0.64/26"
    },
        subnet3 = {
        address_space = "10.0.0.128/26"
    },
        AzureBastionSubnet = {
        address_space = "10.0.0.192/26"
    }
  }
}

```

Launching a terraform plan, we get something similar to this:

```bash

Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the
following symbols:
  + create

Terraform will perform the following actions:

  # azurerm_subnet.vnet_name_subnets["AzureBastionSubnet"] will be created
  + resource "azurerm_subnet" "vnet_name_subnets" {
      + address_prefixes                               = [
          + "10.0.0.192/26",
        ]
      + enforce_private_link_endpoint_network_policies = (known after apply)
      + enforce_private_link_service_network_policies  = (known after apply)
      + id                                             = (known after apply)
      + name                                           = "AzureBastionSubnet"
      + private_endpoint_network_policies_enabled      = (known after apply)
      + private_link_service_network_policies_enabled  = (known after apply)
      + resource_group_name                            = "rsg_name"
      + virtual_network_name                           = "vnet_name"
    }

  # azurerm_subnet.vnet_name_subnets["subnet1"] will be created
  + resource "azurerm_subnet" "vnet_name_subnets" {
      + address_prefixes                               = [
          + "10.0.0.0/26",
        ]
      + enforce_private_link_endpoint_network_policies = (known after apply)
      + enforce_private_link_service_network_policies  = (known after apply)
      + id                                             = (known after apply)
      + name                                           = "subnet1"
      + private_endpoint_network_policies_enabled      = (known after apply)
      + private_link_service_network_policies_enabled  = (known after apply)
      + resource_group_name                            = "rsg_name"
      + virtual_network_name                           = "vnet_name"
    }

  # azurerm_subnet.vnet_name_subnets["subnet2"] will be created
  + resource "azurerm_subnet" "vnet_name_subnets" {
      + address_prefixes                               = [
          + "10.0.0.64/26",
        ]
      + enforce_private_link_endpoint_network_policies = (known after apply)
      + enforce_private_link_service_network_policies  = (known after apply)
      + id                                             = (known after apply)
      + name                                           = "subnet2"
      + private_endpoint_network_policies_enabled      = (known after apply)
      + private_link_service_network_policies_enabled  = (known after apply)
      + resource_group_name                            = "rsg_name"
      + virtual_network_name                           = "vnet_name"
    }

  # azurerm_subnet.vnet_name_subnets["subnet3"] will be created
  + resource "azurerm_subnet" "vnet_name_subnets" {
      + address_prefixes                               = [
          + "10.0.0.128/26",
        ]
      + enforce_private_link_endpoint_network_policies = (known after apply)
      + enforce_private_link_service_network_policies  = (known after apply)
      + id                                             = (known after apply)
      + name                                           = "subnet3"
      + private_endpoint_network_policies_enabled      = (known after apply)
      + private_link_service_network_policies_enabled  = (known after apply)
      + resource_group_name                            = "rsg_name"
      + virtual_network_name                           = "vnet_name"
    }

Plan: 4 to add, 0 to change, 0 to destroy

```

where the subnets to be created already exist in our `subnets.tf` file, and in Azure.
The idea here is to somehow change the current configuration with the new one, without recreating the resources.
To achieve this, terraform state mv command can be used in an imperative way, or the moved block can be used in a declarative way.

The moved block is defined as below:

```go

moved {
  from = azurerm_<resource_type>.a
  to   = azurerm_<resource_type>.b
}

```

In the present case, we are considering moving the subnets in a terraform declaration with a `for_each` meta argument. We will add the following moved block:

```go

moved {
  from = azurerm_subnet.subnet1
  to = azurerm_subnet.hub_subnets["subnet1"]
}

moved {
  from = azurerm_subnet.subnet2
  to = azurerm_subnet.hub_subnets["subnet2"]
}

moved {
  from = azurerm_subnet.subnet3
  to = azurerm_subnet.hub_subnets["subnet3"]
}

moved {
  from = azurerm_subnet.AzureBastionSubnet
  to = azurerm_subnet.hub_subnets["AzureBastionSubnet"]
}

```

We also need to comment the first definition of the subnets and keep only the new one. The `subnets.tf` file should look like this:

```go

resource "azurerm_subnet" "hub_subnets" {

  for_each = var.hub_subnets_list

  name = each.key
  resource_group_name = data.azurerm_resource_group.self.name
  virtual_network_name = azurerm_virtual_network.main_vnet.name
  address_prefixes = [each.value.address_space]
}

moved {

  
  from = azurerm_subnet.subnet1
  to = azurerm_subnet.hub_subnets["subnet1"]
}

moved {
  from = azurerm_subnet.subnet2
  to = azurerm_subnet.hub_subnets["subnet2"]
}

moved {
  from = azurerm_subnet.subnet3
  to = azurerm_subnet.hub_subnets["subnet3"]
}

moved {
  from = azurerm_subnet.AzureBastionSubnet
  to = azurerm_subnet.hub_subnets["AzureBastionSubnet"]
}

```

Run a terraform plan. The output should be similar to this:

```bash

Terraform will perform the following actions:

  # azurerm_subnet.AzureBastionSubnet has moved to azurerm_subnet.hub_subnets["AzureBastionSubnet"]
    resource "azurerm_subnet" "hub_subnets" {
        id                                             = "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsg_name/providers/Microsoft.Network/virtualNetworks/vnet_name/subnets/AzureBastionSubnet"
        name                                           = "AzureBastionSubnet"
        # (9 unchanged attributes hidden)
    }

  # azurerm_subnet.subnet1 has moved to azurerm_subnet.hub_subnets["subnet1"]
    resource "azurerm_subnet" "hub_subnets" {
        id                                             = "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsg_name/providers/Microsoft.Network/virtualNetworks/vnet_name/subnets/subnet1"
        name                                           = "subnet1"
        # (9 unchanged attributes hidden)
    }

  # azurerm_subnet.subnet2 has moved to azurerm_subnet.hub_subnets["subnet2"]
    resource "azurerm_subnet" "hub_subnets" {
        id                                             = "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsg_name/providers/Microsoft.Network/virtualNetworks/vnet_name/subnets/subnet2"
        name                                           = "subnet2"
        # (9 unchanged attributes hidden)
    }

  # azurerm_subnet.subnet3 has moved to azurerm_subnet.hub_subnets["subnet3"]
    resource "azurerm_subnet" "hub_subnets" {
        id                                             = "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsg_name/providers/Microsoft.Network/virtualNetworks/vnet_name/subnets/subnet3"
        name                                           = "subnet3"
        # (9 unchanged attributes hidden)
    }

Plan: 0 to add, 0 to change, 0 to destroy.

─────────────────────────────────────────────────────────────────────────────────────────────────────

Note: You didn't use the -out option to save this plan, so Terraform can't guarantee to take exactly these actions if
you run "terraform apply" now.

```

Notice that no change is planned. Notice also the `azurerm_subnet.subnet_name has moved to azurerm_subnet.hub_subnets["subnet_name"]` which indicates the move to the new place in the state, without any rela change on the infrastructure.
Run a terraform apply to validate the change.

```bash

Plan: 0 to add, 0 to change, 0 to destroy.

Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value: yes


Apply complete! Resources: 0 added, 0 changed, 0 destroyed.

```

check the state to validate the move:

```bash

yumemaru [ ~ ]$ terraform state list
data.azurerm_resource_group.feature_rg
data.azurerm_resource_group.main
azurerm_subnet.hub_subnets["AzureBastionSubnet"]
azurerm_subnet.hub_subnets["subnet1"]
azurerm_subnet.hub_subnets["subnet2"]
azurerm_subnet.hub_subnets["subnet3"]
azurerm_virtual_network.main_vnet


```

Remove the resource with a terraform destroy command.