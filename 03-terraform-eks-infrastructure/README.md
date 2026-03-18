# Terraform AWS EKS Cluster with EBS CSI Driver & IRSA

This repository provisions an **AWS EKS cluster** using Terraform and configures **persistent storage for stateful workloads** (MySQL) using the **AWS EBS CSI driver** with **IAM Roles for Service Accounts (IRSA)**.

It is designed to avoid common PVC binding and permission issues that occur in Kubernetes v1.22+ (with CSI migration enabled by default) when using legacy EBS provisioners.

<details>
<summary>EKS Pod Identiy Agent (modern preferred alternative to IRSA)</summary>
> A new EKS cluster addon eks-pod-identity-agent now replaces IRSA, but it is outside the purview of this project.
> eks-pod-identity-agent plugin 
> Pods in EKS often need to call AWS APIs, but we don't have to configure longlived AWS credentials for them as it's insecure.
> AWS installs pod-identity-agent on nodes
> Pods asks pod-identity-agent for AWS credentials.
> This is an AWS simplified way for pods to access AWS APIs. 

</details>
---

## What This Repository Creates

### AWS Infrastructure
- EKS cluster (Kubernetes v1.22+)
- 2 Availability Zones
    - Each AZ contains:
        - 1 public subnet (Internet Gateway attached)
        - 1 private subnet (routes through a shared NAT Gateway)
- Managed node groups running in private subnets
- VPC, route tables, IGW, and NAT Gateway
- All infrastructure fully provisioned using Terraform

### Kubernetes & Storage
- AWS EBS CSI Driver installed as an EKS add-on
- Custom StorageClass (`gp2`)
    - Provisioner: `ebs.csi.aws.com`
- IAM Role for Service Accounts (IRSA) for EBS CSI controller
- Dynamic EBS volume provisioning
- Java Maven application deployment
- MySQL deployment with persistent EBS-backed volume

---

## Why the AWS EBS CSI Driver Is Required

### The Problem with `aws-ebs`

Before v1.22, Kubernetes used the `kubernetes.io/aws-ebs` EBS provisioner

Starting with Kubernetes **v1.22+**:
- CSI migration is enabled by default
- The in-tree `aws-ebs` plugin is deprecated
- Volume operations are redirected to CSI drivers (cluster addon)

Because of this:
- StorageClasses using `aws-ebs` may cause PVCs to remain in `Pending`
- Volume provisioning can fail silently or with permission errors
- Cluster version upgrades can break existing storage behavior

---

### The Solution

Use the **AWS EBS CSI Driver** with the CSI provisioner:

```yaml
provisioner: ebs.csi.aws.com
```

The AWS EBS CSI driver:
- works correctly with Kubernetes v1.22 and above
- Successfully provisions EBS volumes for stateful workloads
---

## What Is IRSA and Why It’s Needed

### What Is IRSA?

**IRSA (IAM Roles for Service Accounts)** allows a Kubernetes **ServiceAccount** to assume an **IAM role** using the EKS cluster’s **OIDC provider**, without relying on the EC2 node IAM role.
Node IAM roles result in permissions sharing across many pods which is insecure and fails the least privilege principle.

> With IRSA, Pods get AWS permissions directly, instead of inheriting them from worker nodes.

---

### Why IRSA Is Required for EBS CSI

The **EBS CSI controller** runs as a Kubernetes pod and must call AWS EC2 APIs such as:

- `ec2:CreateVolume`
- `ec2:AttachVolume`
- `ec2:DescribeVolumes`
- `ec2:DeleteVolume`

Without IRSA:
- The controller pod has no AWS credentials
- Calls to AWS APIs fail
- PVCs remain in `Pending`
- Volume provisioning does not complete
- Stateful workloads remain in `Pending`

---

## How IRSA Solves the Problem
Though this can be done manually after the cluster creation, this project automates it using Terraform for efficiency and repeatability.
This project uses the terrafrom `ebs_csi_driver_irsa` module.

### High-Level Flow

1. Terraform creates an IAM role with the required EBS permissions
2. The IAM role trusts the EKS cluster’s OIDC provider
3. The role is associated with the Kubernetes ServiceAccount:
   ```
   kube-system:ebs-csi-controller-sa
   ```
4. The EBS CSI controller pod:
    - Receives a projected web identity token
    - Assumes the IAM role using AWS STS
5. The controller successfully calls AWS EC2 APIs
6. EBS volumes are dynamically created and attached to nodes

**Result:** PersistentVolumeClaims bind successfully.

```yaml
module "ebs_csi_driver_irsa" {
  source = "terraform-aws-modules/iam/aws//modules/iam-role-for-service-accounts"

  name = "ebs-csi"

  attach_ebs_csi_policy = true

  oidc_providers = {
    this = {
      provider_arn               = module.my_eks.oidc_provider_arn
      namespace_service_accounts = ["kube-system:ebs-csi-controller-sa"]
    }
  }

}
```

---

## Why Node IAM Roles Are Not Used

Relying on node IAM roles would mean:
- Every pod on the node inherits worker node's permissions
- Violates the principle of least privilege
- Increases the security blast radius

Using IRSA ensures:
- Only the EBS CSI controller can provision EBS volumes
- Other pods (eg. application pods) have no AWS permissions to provision volumes
- A clean separation between infrastructure and workloads

---

##  StorageClass Configuration

A custom StorageClass is defined to use the CSI driver as the default gp2 StorageClass use aws/ebs provisioner.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp2-ebs-csi
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
```

## MySQL + Persistent Volume Provisioning Flow
> [!NOTE]
> This project sets up the infrastructure needed to deploy a stateful workload like
> MySQL database with persistent volume, however MySQL itself is not deployed as part of this project.

1. A MySQL PersistentVolumeClaim requests for persistent storage
2. Kubernetes detects the StorageClass and it's provisioner `ebs.csi.aws.com`
3. The EBS CSI controller:
    - Calls `CreateVolume` using the AWS EC2 API
4. The EBS volume is created
5. The volume is attached to the node
6. MySQL starts with persistent storage backed by EBS

---

## Terraform Components

This repository provisions and configures:

- VPC and networking resources
- EKS cluster and managed node groups
- AWS EBS CSI Driver EKS add-on
- IAM Role for Service Accounts (IRSA) for ebs-csi-controller pod

---


## Provisioning Infrastructure

Initialize Terraform:

```
terraform init
```

Review the plan:

```
terraform plan -var-file="terraform.tfvars"
```

Apply the infrastructure:

```
terraform apply -var-file="terraform.tfvars"
```

---
