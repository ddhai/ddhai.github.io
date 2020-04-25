---
layout: post
tags: terraform index 
title: How preserve index based order in terraform maps
---

How preserve index based order in terraform maps

## What's problem

I have been adding new VPC peerings with another account today and noticed that my new peering would delete old peerings and recreate them again on top of adding a new one in terraform plan.

Here is my peering code:

```json
resource "aws_vpc_peering_connection" "apples_account" {
  count = "${length(var.apples_account_vpc_ids)}"
 
  vpc_id = "${aws_vpc.vpc.id}"
 
  peer_owner_id = "${var.apples_account}"
  peer_vpc_id   = "${element(values(var.apples_account_vpc_ids),count.index)}"
 
  auto_accept = false
  peer_region = "eu-west-1"
 
  tags = "${merge(
    map(
      "Name",
      "peer-${var.environment_group}-${var.aws_account}-${element(keys(var.apples_account_vpc_ids),count.index)}-company1"),
    local.all_tags
    )}"
}
```

And vars:

```json
"apples_account_vpc_ids" : {
  "vpc-staging-l": "vpc-111d4253",
  "vpc-staging-i": "vpc-222d4253"
}
```

As you can see, I am adding new VPC vpc-staging-i and here is what I get:

```bash
)
terraform plan
An execution plan has been generated and is shown below.
Resource actions are indicated with the following symbols:
  + create
-/+ destroy and then create replacement
 
Terraform will perform the following actions:
 
-/+ aws_vpc_peering_connection.apples_account[0] (new resource required)
      id:              "pcx-00888486b31516daa" => <computed> (forces new resource)
      accept_status:   "active" => <computed>
      accepter.#:      "0" => <computed>
      auto_accept:     "false" => "false"
      peer_owner_id:   "111111111111" => "111111111111"
      peer_region:     "eu-west-1" => "eu-west-1"
      peer_vpc_id:     "vpc-111d4253" => "vpc-222d4253" (forces new resource)
      requester.#:     "1" => <computed>
      tags.%:          "9" => "9"
      tags.CostCentre: "OPS_TEAM" => "OPS_TEAM"
      tags.CreatedBy:  "kayanazimov" => "kayanazimov"
      tags.Name:       "peer-vpc-secure-np-vpc-staging-l-company1" => "peer-vpc-secure-np-vpc-staging-i-company1"
      tags.Owner:      "Terraform" => "Terraform"
      tags.Product:    "PROD1" => "PROD1"
      tags.Region:     "eu-west-2" => "eu-west-2"
      tags.Role:       "secure" => "secure"
      tags.Scope:      "internal" => "internal"
      tags.SourcePath: "terraform/vpc/business/" => "terraform/vpc/business/"
      vpc_id:          "vpc-222eddef5e86fa65a" => "vpc-222eddef5e86fa65a"
 
  + aws_vpc_peering_connection.apples_account[1]
      id:              <computed>
      accept_status:   <computed>
      accepter.#:      <computed>
      auto_accept:     "false"
      peer_owner_id:   "111111111111"
      peer_region:     "eu-west-1"
      peer_vpc_id:     "vpc-111d4253"
      requester.#:     <computed>
      tags.%:          "9"
      tags.CostCentre: "OPS_TEAM"
      tags.CreatedBy:  "kayanazimov"
      tags.Name:       "peer-vpc-secure-np-vpc-staging-l-company1"
      tags.Owner:      "Terraform"
      tags.Product:    "PROD1"
      tags.Region:     "eu-west-2"
      tags.Role:       "secure"
      tags.Scope:      "internal"
      tags.SourcePath: "terraform/vpc/business/"
      vpc_id:          "vpc-222eddef5e86fa65a"
```

As you can see, vpc-222d4253 replaces vpc-111d4253, and then vpc-111d4253 added later. But I don’t want to recreate my peerings!

Because my other VPC side is in a different account and I can’t use auto_accept either, meaning my other account will need to accept new peerings again, and in between this – a breaking change…

So first of all, why is this happening?

## How to resolve it

This is because keys(map) in terraform returns list sorted in alphabetical order, let’s prove it, if I change vpc-staging-i to vpc-staging-m:

```json
"apples_account_vpc_ids" : {
  "vpc-staging-l": "vpc-111d4253",
  "vpc-staging-m": "vpc-222d4253"
}
```
as M comes after L, as oppose to I coming before L, now the order will be artificially preserved:

```json
An execution plan has been generated and is shown below.
Resource actions are indicated with the following symbols:
  + create
 
Terraform will perform the following actions:
 
  + aws_vpc_peering_connection.apples_account[1]
      id:              <computed>
      accept_status:   <computed>
      accepter.#:      <computed>
      auto_accept:     "false"
      peer_owner_id:   "111111111111"
      peer_region:     "eu-west-1"
      peer_vpc_id:     "vpc-222d4253"
      requester.#:     <computed>
      tags.%:          "9"
      tags.CostCentre: "OPS_TEAM"
      tags.CreatedBy:  "kayanazimov"
      tags.Name:       "peer-vpc-secure-np-vpc-staging-m-company1"
      tags.Owner:      "Terraform"
      tags.Product:    "PROD1"
      tags.Region:     "eu-west-2"
      tags.Role:       "secure"
      tags.Scope:      "internal"
      tags.SourcePath: "terraform/vpc/business/"
      vpc_id:          "vpc-222eddef5e86fa65a"

Plan: 1 to add, 0 to change, 0 to destroy.
```

Indeed, only adding a new VPC peering,

But I don’t want to juggle with letters, becides this letter actually stands for a name of vpc(l – low risk, m – middle, etc) not just some random letter, I need another solution, luckily there is one.

If I add some sort of prefix to my names, I can then delete them once terraform sorted them in the order I need. Thanks to substr method:

```bash
terraform console
 
substr("01-hello world", 3, -1)
hello world
```

So let’s add a prefix:

```json
"apples_account_vpc_ids" : {
  "01-vpc-staging-l": "vpc-111d4253",
  "02-vpc-staging-i": "vpc-222d4253"
}
```

Now when I set peering name, I just delete it:

```json
resource "aws_vpc_peering_connection" "apples_account" {
  count = "${length(var.apples_account_vpc_ids)}"
 
  vpc_id = "${aws_vpc.vpc.id}"
 
  peer_owner_id = "${var.apples_account}"
  peer_vpc_id   = "${element(values(var.apples_account_vpc_ids),count.index)}"
 
  auto_accept = false
  peer_region = "eu-west-1"
 
  tags = "${merge(
    map(
      "Name",
      "peer-${var.environment_group}-${var.aws_account}-${substr(element(keys(var.apples_account_vpc_ids),count.index), 3, -1)}-company1"),
 
    local.all_tags
    )}"
}
```

And we run:

```bash
+ aws_vpc_peering_connection.apples_account[1]
      id:              <computed>
      accept_status:   <computed>
      accepter.#:      <computed>
      auto_accept:     "false"
      peer_owner_id:   "111111111111"
      peer_region:     "eu-west-1"
      peer_vpc_id:     "vpc-222d4253"
      requester.#:     <computed>
      tags.%:          "9"
      tags.CostCentre: "OPS_TEAM"
      tags.CreatedBy:  "kayanazimov"
      tags.Name:       "peer-vpc-secure-np-vpc-staging-i-company1"
      tags.Owner:      "Terraform"
      tags.Product:    "PROD1"
      tags.Region:     "eu-west-2"
      tags.Role:       "secure"
      tags.Scope:      "internal"
      tags.SourcePath: "terraform/vpc/business/"
      vpc_id:          "vpc-222eddef5e86fa65a"
```

As you can see only 1 resource being created, and with correct name, even though we refactored our code, changed the keys in the variables,
when terraform applies those to actual AWS resources and their names in the state file, everything is still same, hence no recreating and breaking the infra.

Hope new version of terraform will sort this issue in a more elegant way.