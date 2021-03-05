# Terraform Best Practices

Welcome! This repo contains a set of best practices to be followed when contributing to the Palo Alto Networks terraform modules used by Professional Services.

## Table of Contents

1. [Versioning](#1-versioning)
2. [Coding Practices](#2-coding-practices)
3. [Tips and Tricks](#3-tips-and-tricks)
4. [Terraform Module Structure](#4-Terraform-Module-Structure)
<!-- TODO 5. [Document Generation](#5-Document-Generation) -->
<!-- TODO 6. [Terraform Module Testing](#6-Terraform-Module-Testing) -->

## 1. Versioning

### 1.1 Terraform Version

All code should support the latest minor version of the latest Terraform GA release.

<!-- TODO All code should work with the latest three(?) major Terraform releases, requiring all the critical tests to pass as specified in the [Testing](#6-Terraform-Module-Testing) section. Workarounds, if needed, are allowed if they are clearly documented. -->

### 1.1 Module Version

* All terraform modules should be versioned using Git tags. Git tag name should adhere to the following naming convention

> Major.Minor.Patch => 0.1.0

* Version should be pinned in the root module where the modules are called from using the `ref` argument:

      module "bootstrap_bucket" {
        source          = "spring.paloaltonetworks.com/mekanayake/terraform-aws-panfw-bootstrap?ref=0.1.0"
        bucket_prefix   = "GlobalProtect-bootstrap-"
        local_directory = "vmseries-bootstrap-package"
      }

## 2. Coding Practices

### 2.1 Variables

* Static values shouldn't be hardcoded inside the Terraform configuration, static values should be defined as Terraform variables, this extends the flexibility of the Terraform nodules for future use.

  * BAD example, all the arguments provided to the `contains()` function are hardcoded

        contains(["169.254.0.0/28", "169.254.1.0/28", "169.254.2.0/28"], "169.254.1.0/28")

  * GOOD example, all the arguments provided to the `contains()` function are extracted and defined as variables

        variable "exceptionList" {
          default = ["169.254.0.0/28", "169.254.1.0/28", "169.254.2.0/28"]
        }

        variable "tunnelCidrRange" {
          default = "169.254.1.0/28"
        }

        contains(var.exceptionList, var.tunnelCidrRange)

* Optional variables for resources should still be included (where sensible) to ensure future extendability of the module

  * GOOD example, `ipv6_cidr_block` is optional, however the optional argument is included and the value is provided with the help of `lookup` function to ensure future extensibility

        resource "aws_subnet" "in_secondary_cidr" {
          vpc_id          = vpc_id
          cidr_block      = each.value.cidr_block
          ipv6_cidr_block = lookup(each.value, "ipv6_cidr_block", null)
        }

### 2.2 Looping

* `for_each` looping should be used instead of `count` when multiple resources need to be created. This results in a resource `map` instead of `list` when the created resources are added to the Terraform state tree which allows you to remove an object from Terraform state despite the position of the object where it is inserted. Removing an item from the middle of a list will result in Terraform deleting and re-creating the resources.

  * BAD example, `subnets` is a list of maps and `count` is used to create multiple subnets

        variable "subnets" {
          default = [
            {
              name = "Public"
              az   = "eu-est-2a"
              cidr = "192.168.0.0/24"
            },
            {
              name = "Private"
              az = "eu-est-2b"
              cidr = "192.168.1.0/24"
            }
          ]
        }

        resource "aws_subnet" "this" {
          count             = length(var.subnets)
          cidr_block        = var.subnets[count.index].cidr
          availability_zone = var.subnets[count.index].az
          vpc_id            = aws_vpc.this.id
        }

  * GOOD example, `subnets` is a map of maps and `for_each` is used to create multiple subnets

        variable "subnets" {
          default = {
            "Public" = {
              name = "Public"
              az   = "eu-est-2a"
              cidr = "192.168.0.0/24"
            },
            "Private" = {
              name = "Private"
              az = "eu-est-2b"
              cidr = "192.168.1.0/24"
            }
          }
        }

        resource "aws_subnet" "this" {
          for_each          = var.subnets
          cidr_block        = each.value.cidr
          availability_zone = lookup(each.value, "az", null)
          vpc_id            = aws_vpc.this.id
        }

* `count` should only be used when creating a single resource with a conditional statement. For an example you may decide to create a resource based on an existence of an input variable

  * GOOD example, `count` creates a single resource with a conditional statement

        resource "aws_iam_role_policy_attachment" "bs-attach" {
          count      = var.enable_bs ? 1 : 0
          role       = aws_iam_role.fw-role.name
          policy_arn = aws_iam_policy.bs-policy[0].arn
        }

### 2.3 Intuitive Variable Structure

* It is important to define your variable structure as intuitively as possible for the end user. The input variable can be easily transformed within the module code to meet your requirement later.

  * BAD example, expect disjointed input variables

        module "vpc" {
          source = ...
          subnet_names = ["Public", "Private"]
          subnet_cidr = ["192.168.0.0/24", "192.168.1.0/24"]
        }

  * GOOD example 1, accept a `list` of maps and transform the `list` to a `map` inside the module code

        module "vpc" {
          source = ...
          subnet_cidr = [
            {
              name = "Public"
              az   = "eu-est-2a"
              cidr = "192.168.0.0/24"
            },
            {
              name = "Private"
              az = "eu-est-2b"
              cidr = "192.168.1.0/24"
            }
          ]
        }

  * GOOD example 2, accept a `map` of `objects` that wouldn't require variable transformation within the module code. Note that the `name` parameter is defined as the key of the object.

        module "vpc" {
          source = ...
          subnet_cidr = {
            "Public" = {
              az   = "eu-est-2a"
              cidr = "192.168.0.0/24"
            },
            "Private" = {
              az = "eu-est-2b"
              cidr = "192.168.1.0/24"
            }
          }
        }

Input variable structure as in good example 2 is preferred over example 1 where possible. In this particular case the user is forced to use unique names for the subnets by using subnet name as the key of the object.

### 2.4 Segment resource creation between multiple resource

* In some cases it does make sense to create Public Cloud components (ie. virtual machines, Load balancers, Routing tables) using multiple Terraform resources to avoid deletion of the entire component upon minor modifications to the object. A few examples adding or removal of an interface to an EC2 instance destroys and recreates the entire instance. This could be avoided by splitting the network interface and virtual machine creation between two Terraform resources.

  * Example

        # Input variable
        variable "firewall" {
          default = {
            "MOJ-AW2-FW01A" = {
              interfaces = [
                {
                  name            = "mgmt"
                  subnet_id       = module.vpc.subnets["Management-A"].id
                  security_groups = [module.vpc.security_groups["SG_Management"].id]
                  public_ip       = false
                  index           = 0
                },
                {
                  name              = "public"
                  subnet_id         = module.vpc.subnets["Public-A"].id
                  security_groups   = [module.vpc.security_groups["SG_Public"].id]
                  index             = 1
                  public_ip         = true
                  source_dest_check = false
                },
                {
                  name              = "private"
                  subnet_id         = module.vpc.subnets["Private-A"].id
                  security_groups   = [module.vpc.security_groups["SG_Private"].id]
                  index             = 2
                  source_dest_check = false
                }
              ]
            }
          }
        }

        # Flatten variable
        locals {
          interfaces = flatten([
            for fw_name, firewall in var.firewalls : [
              for if_name, interfaecs in firewall.interfaces : [
                merge(interfaecs, { "fw_name" = fw_name })
              ]
            ]
          ])
        }

        # Create instance
        resource "aws_instance" "this" {
          for_each                             = var.firewalls
          disable_api_termination              = false
          instance_initiated_shutdown_behavior = "stop"
          ebs_optimized                        = true
          ami                                  = var.custom_ami != null ? var.custom_ami : data.aws_ami.this.id
          instance_type                        = var.instance_type
          key_name                             = var.key_name
          user_data                            = var.user_data
          monitoring                           = false
          iam_instance_profile                 = var.iam_instance_profile

          root_block_device {
            delete_on_termination = "true"
          }

          # Attach primary interface to the instance
          network_interface {
            device_index         = 0
            network_interface_id = aws_network_interface.primary[each.key].id
          }

          tags = { Name = each.key }
        }

        # Attach interfaces
        # Create the primary interface (index 0) for the instance
        resource "aws_network_interface" "primary" {
          for_each          = { for i in local.interfaces : i.fw_name => i if i.index == 0 }
          subnet_id         = each.value.subnet_id
          private_ips       = lookup(each.value, "private_ips", null)
          security_groups   = lookup(each.value, "security_groups", null)
          source_dest_check = lookup(each.value, "source_dest_check", false)
          description       = lookup(each.value, "description", null)

          tags = { "Name" = each.key }
        }

        resource "aws_network_interface" "this" {
          for_each          = { for i in local.interfaces : "${i.fw_name}-${i.name}" => i if i.index != 0 }
          subnet_id         = each.value.subnet_id
          private_ips       = lookup(each.value, "private_ips", null)
          security_groups   = lookup(each.value, "security_groups", null)
          source_dest_check = lookup(each.value, "source_dest_check", false)
          description       = lookup(each.value, "description", null)
          attachment {
            instance     = aws_instance.this[each.value.fw_name].id
            device_index = lookup(each.value, "index", null)
          }
          tags = { "Name" = each.key }
        }

### `2.5 Outputs`

* Terraform output values allow you to export structured data about your resources. You can use this data to configure other parts of your infrastructure with automation tools, or as a data source for another Terraform workspace. Outputs are also necessary to share data from a child module to your root module.

  * Example - output variable for a single resource

        # Output variable snippet:
        
        output "virtual_network_id" {
          description = "The ID of the created Virtual Network."
          value       = azurerm_virtual_network.this.id
        }
        
        # Output list:

        $ terraform output
        virtual_network_id = subscriptions/[redacted]/example-vnet

    This constuct is fine when you want to export data about a single resource, but if you are using the `for_each` loop to create multiple resources, the output variable should be updated accordingly to reflect this scenario.

    It's important to give the user an idication of what to expect from the output return value. In the example above, the output variable has been defined as `virtual_network_id`, where the `id` suffix suggest that a singular occurrence of a given resource will be presented as the output return value. When expecting multiple resources of the same kind as the output result, both the output variable name and value should be adjusted to reflect those changes, like in the example below. Notice that now we are creating a map which is using the resource name as the `key`, and its `id` as the value. We also changed the output variable name suffix to `ids` to suggest that multiple return values for the specified resource should be expected by the user.

  * Example - output variable for multiple resource occurrences

          # Output variable snippet:
          
          output "virtual_network_ids" {
            description = "The IDs of the created Virtual Network."
            value = {
              for k, v in azurerm_network_security_group.this : k => v.id
            }
          }

          # Output list:

          $ terraform output
          virtual_network_ids = {
            example_vnet1 = subscriptions/[redacted]/example-vnet1
            example_vnet2 = subscriptions/[redacted]/example-vnet2
          }

  Reference - [https://learn.hashicorp.com/tutorials/terraform/outputs](https://learn.hashicorp.com/tutorials/terraform/outputs)

## 3. Tips and Tricks

### 3.1 Transform lists to maps

* `for_each` only accepts type `set` or `map`, therefore you may have to transform lists to maps when creating multiple resources with `for_each`

  * Example - Transform subnets `list` in to a `map`

        variable "subnets" {
          default = [
            {
              name = "Public"
              az   = "eu-est-2a"
              cidr = "192.168.0.0/24"
            },
            {
              name = "Private"
              az   = "eu-est-2b"
              cidr = "192.168.1.0/24"
            }
          ]
        }

        locals {
          subnet_map = { for subnet in var.subnets: subnet.name => subnet }
        }

Reference - [https://www.hashicorp.com/blog/hashicorp-terraform-0-12-preview-for-and-for-each/](https://www.hashicorp.com/blog/hashicorp-terraform-0-12-preview-for-and-for-each/)

### 3.2 Flattening nested structures

* Sometimes your input data structure isn't naturally in a suitable shape for use in a `for_each` argument, and `flatten` can be a useful helper function when reducing a nested data structure into a flat one.

  * Example - You might have an input variable to define your routes in a routing table, however this variable structure may not be suitable to be used in `aws_route` resource

        variable "route_tables" {
          default = {
            "Public" = [
              { destination_cidr = "0.0.0.0/0", target = "igw" }
            ],
            "Priv" = [
              { destination_cidr = "10.0.0.0/8", target = "tgw" }
            ],
            "Mgmt" = [
              { destination_cidr = "0.0.0.0/0", target = "igw" },
              { destination_cidr = "192.168.100.0/24", target = "tgw" }
            ]
          }
        }

        # Create route tables 
        resource "aws_route_table" "this" {
          for_each = var.route_tables
          vpc_id   = aws_vpc.this.id
          tags     = { Name = each.key }
        }

        # Create individual routes
        resource "aws_route" "this" {
          for_each               = ??
          route_table_id         = ??
          destination_cidr_block = ??
          gateway_id             = ??
        }

  * Ideally, in order to declare all of the routes with a single resource block, we must first flatten the structure to produce a collection where each top-level element represents a single route

        # Flatten the variable structure
        locals {
          routes = flatten([
            for rtb, rts in var.route_tables : [
              for rt in rts : [
                merge(rt, { "rtb" = rtb })
              ]
            ]
          ])
        }

        resource "aws_route" "igw_rt" {
          for_each               = { for r in local.routes : "${r.rtb}-${r.destination_cidr}" => r }
          route_table_id         = aws_route_table.this[each.value.rtb].id
          destination_cidr_block = each.value.destination_cidr
        } 

* This technique will likely result in a minor problem on Terraform 0.12 (`Invalid for_each argument` in specific situations). The first recommendation is **to use Terraform 0.13**. There is [another less-preferred workaround](empty_state_and_tf12.md) available.

Reference - [https://www.terraform.io/docs/configuration/functions/flatten.html](https://www.terraform.io/docs/configuration/functions/flatten.html)

### 3.3 How to normalise data

* In certain cases our modules will have to support brownfield resources (that is, pre-existing resources), in which case we will have the `data` object instead of the `resource` object.

  * Example - Support brownfield deployments where Azure `resource_group` already exists. We can use [`try()` function](https://www.terraform.io/docs/configuration/functions/try.html) to determine the value for `local.rg` based on the existence of `resource "azurerm_resource_group"` or `data "azurerm_resource_group"`

        resource "azurerm_resource_group" "this" {
          count    = var.existing_rg == false ? 1 : 0
          name     = var.resource_group_name
          location = var.location
        }

        data "azurerm_resource_group" "this" {
          count = var.existing_rg == true ? 1 : 0
          name  = var.resource_group_name
        }

        locals {
          rg = try(azurerm_resource_group.this[0], data.azurerm_resource_group.this[0])
        }

* When including `data` objects in reusable modules, consider including this note in the documentation:

  > If the code needs to retain Terraform-0.12 and 0.13 compatibility, the input `var.varname` needs to be a static string
  > and not come from another Terraform object (in a typical case, it should not come from a `resource`). It is allowed for Terraform-0.14+ though.

  * You may omit the note for any input if you test that you can assign it `varname = some_resource.this.field` and then plan/apply succeeds.
  * Do not assume here that the test succeeding for 0.12 will succeed for 0.13, test both. In this case, incompatibility is likely to show up.
  * All the inputs that are used inside `data` blocks might be independently impacted:

        data "azurerm_resource" "my_data_object" {
            name                = var.input1 // possibly impacted
            resource_group_name = var.input2 // also possibly impacted
            required_tags       = var.input3 // also possibly impacted
        }

* As long as our code supports Terraform-0.12.x, avoid creating a `resource` object and later passing it through a `data` object.
This may cause [a known problem](data_depends_on_and_tf13.md) and thus requires careful testing. It's hard to determine before
the tests whether the particular code can be ever made compatible with both 0.12 *and* with 0.13 at the same time. The linked page
proposes workarounds. The 0.14 version is not impacted by this issue.

## 4. Terraform Module Structure

We will maintain one mono repo per cloud provider. Three repos, representing AWS, Azure and GCP, as we kick off our efforts.

      ├── README.md <- Documentation explaining the purpose of this mono repo
      ├── LICENSE.md <- Palo Alto Networks Script Software Agreement
      ├── modules/
      │   ├── vpc/
      │   │   ├── README.md <- Documentation on how to use submodules on its own
      │   │   ├── variables.tf
      │   │   ├── main.tf
      │   │   ├── outputs.tf
      │   │   ├── versions.tf <- Pin supported terraform provider versions
      │   ├── vmseries/
      │   │   ├── README.md
      │   │   ├── variables.tf
      │   │   ├── main.tf
      │   │   ├── outputs.tf
      │   │   ├── versions.tf
      ├── examples/ <- Place reference architecture use cases here, subdirectory per use case
      │   ├── tgw-inbound-inspection/
      │   │   ├── README.md <- Documentation for specific ref arch use case
      │   │   ├── variables.tf
      │   │   ├── main.tf
      │   │   ├── outputs.tf
      │   │   ├── versions.tf
      │   ├── tgw-outbound-inspection/

<!-- ## 5. Document Generation -->

<!-- ## 6. Terraform Module Testing -->
