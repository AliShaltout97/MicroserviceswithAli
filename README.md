# PWC Microservices Deployment - Complete Guide

## Table of Contents
- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Project Structure](#project-structure)
- [Step 1: Clone the Repository](#step-1-clone-the-repository)
- [Step 2: Dockerize the Application](#step-2-dockerize-the-application)
- [Step 3: Provision Kubernetes Cluster on AWS](#step-3-provision-kubernetes-cluster-on-aws)
- [Step 4: Deploy the Microservice](#step-4-deploy-the-microservice)
- [Step 5: Expose the Service](#step-5-expose-the-service)
- [Step 6: CI/CD Pipeline](#step-6-cicd-pipeline)
- [Step 7: Monitoring Stack](#step-7-monitoring-stack)
- [Accessing the Application](#accessing-the-application)

---

## Overview

This project demonstrates a complete DevOps workflow for deploying a Python Flask microservice to Kubernetes on AWS EKS with:
- **Infrastructure as Code**: Terraform for EKS cluster provisioning
- **Containerization**: Docker for application packaging
- **Orchestration**: Kubernetes for deployment and scaling
- **CI/CD**: GitHub Actions for automated deployment
- **Monitoring**: Prometheus + Grafana stack

---

## Prerequisites

- AWS CLI configured with credentials
- Terraform >= 1.0
- Docker Desktop
- kubectl
- Git
- GitHub account (for CI/CD)
- Docker Hub account

---

## Project Structure

```
Microservices/
├── app/                          # Flask application code
│   ├── __init__.py
│   └── routes/
│       ├── user_routes.py
│       └── product_routes.py
├── terraform/                    # Infrastructure as Code
│   ├── main.tf
│   ├── variables.tf
│   └── outputs.tf
├── k8s/                         # Kubernetes manifests
│   ├── namespace.yml
│   ├── deployment.yml
│   ├── service.yml
│   └── monitoring/
│       ├── namespace.yml
│       ├── prometheus.yml
│       ├── grafana.yml
├── .github/workflows/           # CI/CD pipeline
│   └── deploy.yml
├── Dockerfile                   # Container definition
├── requirements.txt             # Python dependencies
└── run.py                       # Application entry point
```

---

## Step 1: Clone the Repository

```bash
git clone https://github.com/sameh-Tawfiq/Microservices.git
cd Microservices
```

---

## Step 2: Dockerize the Application

### Application Structure
The Flask application exposes two REST endpoints:
- **GET /users** - Returns list of users
- **GET /products** - Returns list of products

### Container Configuration
The application is containerized using the `Dockerfile` which:
- Uses Python 3.11 slim base image
- Installs dependencies from `requirements.txt` (Needed Werkzeug==2.2.2 because by default Flask 2.2.2 uses 3 which is not compatible)
- Exposes port 5000
- Runs the Flask application

### Build and Push Docker Image

```bash
# Build the Docker image
docker build -t <your-dockerhub-username>/pwc-microservices:latest .

# Push to Docker Hub
docker push <your-dockerhub-username>/pwc-microservices:latest
```

---

## Step 3: Provision Kubernetes Cluster on AWS

### Terraform Configuration
The infrastructure is defined in `terraform/` directory with:
- **EKS Cluster**: Managed Kubernetes cluster
- **Node Group**: t3.micro instances with auto-scaling (2-4 nodes)
- **IAM Roles**: Proper permissions for cluster and nodes

### Provision Infrastructure

```bash
cd terraform

# Initialize Terraform
terraform init

# Review the infrastructure plan
terraform plan

# Create the EKS cluster (takes ~15 minutes)
terraform apply -auto-approve

# Configure kubectl to use the new cluster
aws eks update-kubeconfig --region us-east-1 --name pwc-eks-cluster
```

## Step 4: Deploy the Microservice

### Kubernetes Resources
The deployment consists of:

1. **Namespace** (`k8s/namespace.yml`): Isolates application resources
2. **Deployment** (`k8s/deployment.yml`): 
   - Liveness probe on `/users` endpoint
   - Readiness probe on `/products` endpoint
3. **Service** (`k8s/service.yml`): LoadBalancer type for external access

### Deploy Application

```bash
# Deploy all Kubernetes resources
kubectl apply -f k8s/namespace.yml
kubectl apply -f k8s/deployment.yml
kubectl apply -f k8s/service.yml

```

---

## Step 5: Expose the Service

### LoadBalancer Configuration
The service is exposed via AWS LoadBalancer (`k8s/service.yml`):
- **Type**: LoadBalancer
- **Port**: 80 (external) → 5000 (container)
- **Health Checks**: Kubernetes liveness/readiness probes

### Get Service URL

```bash
# Get the LoadBalancer URL
kubectl get svc microservices-app -n pwc-app -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
```

### Access Endpoints

```bash
# Get the LoadBalancer hostname
LOAD_BALANCER=$(kubectl get svc microservices-app -n pwc-app -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

# Test endpoints
curl http://$LOAD_BALANCER/users
curl http://$LOAD_BALANCER/products
```

---

## Step 6: CI/CD Pipeline

### GitHub Actions Workflow
The CI/CD pipeline (`.github/workflows/ci-cd.yml`) implements an **intelligent deployment strategy** with change detection to optimize build and deployment times.

### Pipeline Architecture

The workflow consists of three main jobs:

1. **Detect Changes Job**:
   - Analyzes git diff to identify which files changed
   - Sets output flags for `app-changed` (application code) and `k8s-changed` (Kubernetes manifests)
   - Enables conditional execution of subsequent jobs

2. **Build and Push Job** (runs only if app files changed):
   - Checks out code
   - Authenticates with Docker Hub
   - Builds Docker image from Dockerfile
   - Pushes image to Docker Hub registry with `latest` tag

3. **Deploy Job** (runs only if app or k8s files changed):
   - Configures AWS credentials
   - Updates kubeconfig to connect to EKS cluster
   - Applies all Kubernetes manifests (namespace, deployment, service, monitoring)
   - Performs rolling restart of deployment
   - Waits for rollout completion with 5-minute timeout

### Key Features
- **Conditional Execution**: Only builds/deploys what changed
- **Rolling Updates**: Zero-downtime deployments
- **Automated Monitoring**: Deploys Prometheus and Grafana alongside application
- **Status Verification**: Ensures deployment succeeds before completing

### Pipeline Triggers
- **Push to main branch**: Triggers full workflow with change detection

### Required GitHub Secrets
Configure these in GitHub repository settings → Secrets and variables → Actions:
- `DOCKER_HUB_TOKEN`: Docker Hub access token
- `AWS_ACCESS_KEY_ID`: AWS IAM access key
- `AWS_SECRET_ACCESS_KEY`: AWS IAM secret access key


## Step 7: Monitoring Stack

### Components
The monitoring stack includes:

1. **Prometheus** (`k8s/monitoring/prometheus.yml`):
   - Scrapes Kubernetes metrics
   - Collects pod CPU/Memory metrics from cAdvisor

2. **Grafana** (`k8s/monitoring/grafana.yml`):
   - Pre-configured dashboard for PWC microservices
   - Displays Memory usage, CPU usage graphs
   - Auto-refreshes every 10 seconds



---

## Accessing the Application

### Application Endpoints

```bash
# Get application URL
APP_URL=$(kubectl get svc microservices-app -n pwc-app -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

echo "Users endpoint: http://$APP_URL/users"
echo "Products endpoint: http://$APP_URL/products"
```

### Monitoring Dashboards

```bash
# Get Prometheus URL
PROM_URL=$(kubectl get svc prometheus -n monitoring -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
echo "Prometheus: http://$PROM_URL:9090"

# Get Grafana URL
GRAFANA_URL=$(kubectl get svc grafana -n monitoring -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
echo "Grafana: http://$GRAFANA_URL:3000"
```
**Grafana Credentials**: admin / admin



## Key Features Implemented

✅ **Containerization**: Dockerized Python Flask application  
✅ **Infrastructure as Code**: Terraform for AWS EKS provisioning  
✅ **Health Checks**: Liveness and readiness probes  
✅ **External Access**: LoadBalancer service for internet exposure  
✅ **CI/CD**: GitHub Actions pipeline for automated deployment  
✅ **Monitoring**: Prometheus + Grafana stack with pre-built dashboards  
✅ **Observability**: Pod health, resource usage, and performance metrics  

---

