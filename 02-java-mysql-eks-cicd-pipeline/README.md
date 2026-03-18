# Java MySQL Application Deployment on Amazon EKS from CI/CD Pipeline

This project demonstrates an end-to-end workflow for deploying a **containerized Java application** backed by **MySQL** onto **Amazon EKS**, using a fully automated **CI/CD pipeline with Jenkins**.

It showcases how modern cloud-native applications are built, packaged, and deployed using industry-standard tools and practices.

---

##  Prerequisites

<details>
<summary>kubectl</summary>
CLI tool to interact with Kubernetes clusters\

https://kubernetes.io/docs/tasks/tools/
</details>

<details>
<summary>AWS CLI</summary>
CLI tool to interact with AWS services (required for EKS)\

https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html

After installing AWS CLI, configure it:

```bash
aws configure
```
</details>

<details>
<summary>Amazon EKS Setup</summary>
The Kubernetes cluster was created using **eksctl**.

CLI tool to create and manage Amazon EKS clusters\

https://eksctl.io/introduction/#installation

### Create Cluster

```bash
# Config file location
eksctl/eks-config.yaml

# Create cluster
eksctl create cluster -f eksctl/eks-config.yaml
```

Verify cluster:

```bash
kubectl get nodes
```
</details>

<details>
<summary>Jenkins</summary>

The following repository provisions an **AWS EC2** instance and sets up and configures Jenkins on that instance:\
https://github.com/ranjitha-su/devops-projects/tree/main/04-aws-terraform-ansible-jenkins

</details>

---

## ✅ Architecture Overview

This project follows a cloud-native deployment flow:

1. **Application Code (Java + Gradle)**
2. **Containerization (Docker)**
3. **CI/CD Pipeline (Jenkins)**
4. **Container Registry (Docker Hub)**
5. **Kubernetes Deployment (AWS EKS)**

```
Push to Git → Jenkins Pipeline → Docker Build → Push Image → Deploy to EKS
```

---

## ✅ Containerization

The application was packaged into a Docker container by building the app and image inside a multi-stage Dockerfile.

- Docker image is stored in a **private DockerHub repository**
- Kubernetes pulls image using Docker registry credentials
- Multi-stage builds can be implemented for optimization (optional improvement)

---

## ✅ Kubernetes Deployment

Kubernetes manifests are located in the `kubernetes/` directory for java app and mysql and applied by the Jenkins pipeline.

These typically include:

- **Deployment** → Runs the application pods
- **Service** → Exposes the application
- **ConfigMaps/Secrets** → Configuration and credentials

### Credentials

Create kubernetes secret for the kubernetes manifests to download the docker image
```bash
kubectl create secret docker-registry dockerhub-creds -n <namespace> \
--docker-server=https://index.docker.io/v1/ \
--docker-username=<dockerhub-username> \
--docker-password=<dockerhub-password> \
--docker-email=<email-address>
```

---

## ✅ CI/CD Pipeline Flow (Jenkins)

The `Jenkinsfile` automates the entire deployment lifecycle:

1. **Build Docker Image**
2. **Push Image to Registry**
3. **Deploy to EKS using kubectl**



A webhook between Jenkins and GitLab can be set up to automatically trigger builds and deployments whenever code changes are made.

---

## ✅ Deployment Steps

1. Push code to your Git repository
2. Jenkins pipeline triggers
3. Application is built and containerized
4. Image is pushed to registry
5. Kubernetes manifests are applied to EKS
6. Application becomes accessible via Service/LoadBalancer. Alternatively I can set up an Ingress controller for host matching and routing.

---

## ✅ Summary

This project is a practical example of a **real-world DevOps pipeline**, combining:

- Application development
- Containerization
- Infrastructure provisioning (AWS EC2 instance) using Terraform (links provided)
- Jenkins setup and configuration using ansible (links provided)
- Continuous integration & deployment using Jenkins CI/CD pipeline
- Kubernetes orchestration on AWS EKS

It reflects how production-grade systems are built and deployed in modern cloud environments.

---