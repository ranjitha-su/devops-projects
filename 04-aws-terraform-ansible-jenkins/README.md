# Jenkins Server on AWS EC2 (Terraform + Ansible)

## Overview

This project provisions an **AWS EC2 instance using Terraform** for Infrastructure as Code (IaC) and then **configures it as a Jenkins server using Ansible**.  
Terraform provisions the infrastructure, and Ansible handles server configuration

The workflow is:

1. **Terraform**
    - Provisions AWS infrastructure
    - Creates the VPC and EC2 server
    - Configures security groups and SSH access

2. **Ansible**
    - Connects to the EC2 instance
    - Runs Jenkins as a container
    - Installs required tools inside the Jenkins container
      - docker
      - awscli
      - kubectl
      - docker cli inside jenkins container (socket sharing)
    - Installs the necessary Jenkins plugins

---

## Prerequisites

- **Terraform** >= 1.0.0
- **AWS CLI** installed and configured (`~/.aws/credentials`)
- An **AWS account** with sufficient IAM permissions

---

## Terraform Modules

This project uses **modular Terraform structure**.

### VPC Module
Creates the networking layer:

- VPC
- Subnets
- Routing

```
terraform/modules/vpc
```

### Server Module
Creates the Jenkins server:

- EC2 instance
- Security group
- SSH access configuration

```
terraform/modules/server
```


## Provisioning Infrastructure

Navigate to the Terraform directory:

```
cd terraform
```

Initialize Terraform:

```
terraform init
```

Review the plan:

```
terraform plan
```

Apply the infrastructure:

```
terraform apply
```

Terraform will create:

- VPC
- Subnets
- Security groups
- EC2 instance
- SSH key pair

---

## Dynamic EC2 Discovery with Ansible (AWS EC2 Inventory Plugin)

This project uses the **Ansible `aws_ec2` dynamic inventory plugin** to automatically discover the Jenkins EC2 instance.

Instead of hardcoding server IPs in an inventory file, Ansible queries AWS and retrieves instances dynamically.

The configuration is defined in:

```
ansible/inventory_aws_ec2.yaml
```

The plugin filters EC2 instances using the tag:

```
role = jenkins
```

Terraform assigns this tag when creating the EC2 instance.  
Ansible then queries AWS and automatically discovers the instance using keyed_groups.

Example configuration:

```yaml
---
plugin: aws_ec2
region: us-west-2

keyed_groups:
  - prefix: role
    key: tags.Role
```

## How it Works

Terraform creates the EC2 instance and assigns the tag:

```
Role = jenkins
```

The Ansible `aws_ec2` plugin queries AWS and builds inventory groups dynamically from instance tags.


<details>
<summary>Inventory Filters</summary>

The dynamic inventory uses filters to return only the relevant EC2 instances.

```yaml
filters:
  instance-state-name: running
  tag:Role: "ec2_*"
```

This configuration:

- Includes only **running EC2 instances**
- Selects instances whose `Role` tag matches the pattern `ec2_*`

Examples:

```
Role = ec2_jenkins
Role = ec2_runner
Role = ec2_builder
```

All matching instances are automatically included in the Ansible inventory.

</details>

<details>
<summary>Inventory keyed_groups</summary>

Using `keyed_groups` groups hosts by tag, making it easy to target multiple hosts with the same tag.


An instance tagged:

```
Role = jenkins
```

will appear in the inventory group:

```
role_jenkins
```

Playbooks can then target the instance using this group.

Example:

```yaml
hosts: role_jenkins
```
</details>


### View the dynamic inventory

```bash
cd ansible
ansible-inventory -i inventory_aws_ec2.yaml --graph
```

---

## Configuring the Jenkins Server

Once the EC2 instance is created, configure it using **Ansible**.

Run the Ansible playbook:

```bash
cd ansible
ansible-playbook -i inventory_aws_ec2.yaml jenkins-playbook.yaml
```

This playbook installs and configures:

- Docker
- Jenkins (running inside a container)
- docker cli inside the jenkins docker container
- AWS CLI
- kubectl

---