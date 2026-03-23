
# Microservices Kubernetes Deployment (Helm + Helmfile)

## Overview

This project demonstrates deploying a **realistic microservices application on Kubernetes using Helm charts and Helmfile**.

- [Kubernetes cluster](https://github.com/ranjitha-su/devops-projects/tree/main/03-terraform-eks-infrastructure) is provisioned separately via Terraform (AWS EKS)
- Application deployment is handled entirely using **Helm**
- Multiple microservices are orchestrated together using **Helmfile**

---

## Goals of This Project

- Deploy a microservice application using Helm charts
    - Link to the application source code - https://github.com/GoogleCloudPlatform/microservices-demo
    - This project demonstrates the deployment process using helm
- Use Helmfile to coordinate multiple Helm releases
- Externalize configuration using environment-specific values files
- Demonstrate clean separation between infrastructure and application delivery

---

## Prequisities

  - [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
  - [Terraform](https://developer.hashicorp.com/terraform/install)
  - [kubectl](https://kubernetes.io/docs/tasks/tools/)
  - [Helm](https://helm.sh/docs/intro/install)
  - AWS account configured with admin permissions

---

## High-Level Architecture

All services:
- Run as Kubernetes Deployments
- Are exposed via Kubernetes Ingress
- Are deployed and versioned via Helm releases
Nginx Ingress controller is installed to allow access from outside the cluster

---

## Deployment Workflow

Application deployment is handled **exclusively via Helm and Helmfile**.

- Each microservice is deployed as a Helm release
- Service-specific configuration is provided through dedicated values files
- Helmfile acts as the single entry point for deploying the entire system
- There are no `kubectl apply` steps for application resources in this project.

This ensures:
- Consistent, repeatable deployments
- Centralized release management
- Easy upgrades and rollbacks 

---

## Deployment Commands (Helm & Helmfile)

1. Save kubeconfig to default location or within the project (--kubeconfig option required)  
   ```aws eks update-kubeconfig --name <cluster-name>```
3. Application deployment is performed using Helmfile as the primary interface.

### Helmfile Commands

Validate Helmfile configuration:
```bash
helmfile lint
```

Deploy or upgrade all microservices defined in helmfile.yaml:
```bash
helmfile apply
```

Synchronize the cluster state exactly with helmfile.yaml:
```bash
helmfile sync
```

List all helm releases:
```bash
helm list
```

## Install nginx ingress controller in linode kubernetes cluster

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
```

```bash
kubectl create namespace ingress-nginx
helm install ingress-nginx ingress-nginx/ingress-nginx -n ingress-nginx
```

For DNS resolution locally, add this in /etc/hosts file
```declarative
<ingress external IP>  ranjitha-onlineshopping-app.com
```
Access the application from a browser using 'demo-onlineshopping-app.com'

---

## Summary

This project demonstrates how a microservices application can be **deployed on Kubernetes** using helm + helmfile, with infrastructure provisioned separately using Terraform.