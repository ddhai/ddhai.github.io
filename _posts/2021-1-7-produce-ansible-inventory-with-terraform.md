---
 layout: post
 tags: terraform, ansible
 title: PRODUCE AN ANSIBLE INVENTORY WITH TERRAFORM
---
Terraform is a great tool to create virtual machines. According to its documentation, it is not a tool to configure and manage them.

Ansible is a great tool to configure and manage virtual machines. To do this, Ansible will need an inventory file.

Since Terraform was used to create the instances, it has all the information needed to produce an Ansible inventory file. And Terraform can output information. Getting it to write an Ansible inventory should be simple.

The files mentioned in this document are at the end.

## Step 1, Create instances

In the file instances.tf, there is an example of a Terraform description to create instances on AWS.

It describes a bastion, in a public subnet, and six instances, in a private subnet. Terraform will output simple string variables to describe the bastion.

This file also describes a private instance, using the “count” directive to create six identical copies of it. Terraform will output a tuple variable describing these instances.


## Step 2, Output the Information

In the file outputs.tf, there is a resource of type “local_file”, with a “filename” attribute set to “inventory”. That will create a local file named “inventory”. It’s attribute “content” is the return of the “templatefile” function.

This function returns a file built from a template, replacing variables by their content. These variables have to be passed in arguments to the function, otherwise the function won’t see them.

## Step 3, the Template

In the template file inventory.tmpl, there is two Ansible groups. One for the bastion instance, one for all the other instances.

The “bastion” group is a very simply line where three variables will be replaced by their values. The variables are strings.

The “servers” group is a bit trickier. Since its variables are tuples and are in a list, we have to use a for loop. But Terraform for loops are meant to output one string variable at a time, like the variable “dns” in the example.

How to get a for loop to go through a list and output a few values out of a tuple? And, of course, we want to go through the tuples one at a time, so all variables on one line of the inventory, relate to the same instance.

The secret is a variable called “index”.

A Terraform for loop requires a starting and a stopping point which can’t be a number. In this example, the for loop use the “index” variable to iterates through the list of tuples and the “dns” variable to iterate through the elements of each tuple.

 

Let’s look at the template file code more closely:

```shell
%{ for index, dns in private-dns ~}
${dns} ansible_host=${private-ip[index]} # ${private-id[index]}
%{ endfor ~}
```

The `%{}` contain directives, as opposed to literals or variables.

The two `~` are to remove excessive newlines and white spaces from the output.  

Note there is one a the beginning and one at the end.

The `${}` indicate a substitution.

The `“dns”` variable is one value from private-dns. `“private-dns”` was known as `“aws_instance.i-private.*.private_dns”` in the Terraform configuration. Here, of the `“index”` instance. The loop iterates `“index”`. `“private-ip”` was known as `“aws_instance.i-private.*.private_ip” and “private-id”` was known as `“aws_instance.i-private.*.private_id”`.

## The Result
 

Here is the Ansible inventory file, called `“inventory”`:

```shell
[bastion]
ec2-99-79-28-17.ca-central-1.compute.amazonaws.com ansible_host=99.79.28.17 # i-0d8c4197a22797

[servers]
ip-10-1-2-114.ca-central-1.compute.internal ansible_host=10.1.2.114 # i-09752396073622ba6
ip-10-1-2-251.ca-central-1.compute.internal ansible_host=10.1.2.251 # i-0af7dddbe098fb1c9
ip-10-1-2-35.ca-central-1.compute.internal ansible_host=10.1.2.35 # i-0de0126836e547eba
ip-10-1-2-109.ca-central-1.compute.internal ansible_host=10.1.2.109 # i-060a362c0165ab3f2
ip-10-1-2-185.ca-central-1.compute.internal ansible_host=10.1.2.185 # i-07bd962925faafa3b
ip-10-1-2-93.ca-central-1.compute.internal ansible_host=10.1.2.93 # i-0fc357d17b62b1497
```

Once you know how to get Terraform to output an Ansible inventory file, it is rather easy. It was the figuring out that took a lot longer than one might expect.

## The File instances.tf

```shell
### The bastion
resource "aws_instance" "bastion" {
 ami = var.ami
 availability_zone = var.az
 instance_type = "t2.micro"
 key_name = var.ssh-key
 vpc_security_group_ids = [aws_default_security_group.sg-public.id]
 subnet_id = aws_subnet.public.id

 tags = {
  Name = "Bastion"
  Projet = var.projet
  Application = var.Application
 }

 volume_tags = {
  Name = "Bastion"
  Projet = var.projet
  Application = var.Application
 }

 root_block_device {
   volume_size = 8
  }
}

### The Elastic IP for the Bastion
resource "aws_eip" "eip-bastion" {
 vpc = true
 instance = aws_instance.bastion.id
 associate_with_private_ip = aws_instance.bastion.private_ip

 tags = {
  Name = "bastion"
  Projet = var.projet
  Application = var.Application
 }
}

### The private instances
resource "aws_instance" "i-private" {
 ami = var.ami
 availability_zone = var.az
 instance_type = var.ec2_type
 key_name = var.ssh-key
 vpc_security_group_ids = [aws_security_group.sg-private.id]
 subnet_id = aws_subnet.private.id

 tags = {
  Name = "i-private"
  Projet = var.projet
  Application = var.Application
 }

 volume_tags = {
  Name = "i-private"
  Projet = var.projet
  Application = var.Application
 }

 root_block_device {
   volume_size = 8
  }

 ebs_block_device {
  device_name = "/dev/sdb"
  volume_type = var.ebs_type
  volume_size = 8
 }

 count = var.ec2_nb
}
```

## The File outputs.tf

```shell
### The Ansible inventory file
resource "local_file" "AnsibleInventory" {
 content = templatefile("inventory.tmpl",
 {
  bastion-dns = aws_eip.eip-bastion.public_dns,
  bastion-ip = aws_eip.eip-bastion.public_ip,
  bastion-id = aws_instance.bastion.id,
  private-dns = aws_instance.i-private.*.private_dns,
  private-ip = aws_instance.i-private.*.private_ip,
  private-id = aws_instance.i-private.*.id
 }
 )
 filename = "inventory"
}
```

## The File inventory.tmpl

```shell
[bastion]
${bastion-dns} ansible_host=${bastion-ip} # ${bastion-id}

[servers]
%{ for index, dns in private-dns ~}
${dns} ansible_host=${private-ip[index]} # ${private-id[index]}
%{ endfor ~}
```