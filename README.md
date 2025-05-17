complete **EKS Project** for deploying a **containerized Employee Management System (ReactJS + Golang)** using:

* **Terraform** for infrastructure provisioning
* **EKS** for Kubernetes cluster
* **GitHub Actions** for CI/CD
* **ECR** for container images

---

## 🔧 Tech Stack

* **Frontend:** ReactJS
* **Backend:** Golang
* **Database (optional extension):** PostgreSQL or MongoDB
* **Containerization:** Docker
* **Kubernetes:** AWS EKS
* **CI/CD:** GitHub Actions
* **Infra as Code:** Terraform

---

## 📁 Project Structure

```
eks-employee-app/
├── terraform/
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   └── eks.tf
├── backend/
│   ├── main.go
│   ├── Dockerfile
│   ├── k8s.yaml
├── frontend/
│   ├── App.js
│   ├── index.js
│   ├── Dockerfile
│   ├── k8s.yaml
├── .github/
│   └── workflows/
│       └── ci-cd.yaml
└── README.md
```

---

## 🧱 Step-by-Step Setup

### 1. ✅ Provision EKS using Terraform

#### `terraform/main.tf`

```hcl
provider "aws" {
  region = "us-west-2"
}

module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  name    = "eks-vpc"
  cidr    = "10.0.0.0/16"
  azs     = ["us-west-2a", "us-west-2b", "us-west-2c"]
  public_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  enable_dns_hostnames = true
}

module "eks" {
  source          = "terraform-aws-modules/eks/aws"
  cluster_name    = "employee-cluster"
  cluster_version = "1.27"
  subnets         = module.vpc.public_subnets
  vpc_id          = module.vpc.vpc_id
  node_groups = {
    default = {
      desired_capacity = 2
      max_capacity     = 3
      min_capacity     = 1
      instance_type    = "t3.medium"
    }
  }
}
```

#### Apply Terraform

```bash
cd terraform
terraform init
terraform apply -auto-approve
```

---

### 2. 🚀 Configure `kubectl` for EKS

```bash
aws eks --region us-west-2 update-kubeconfig --name employee-cluster
```

---

### 3. 🐳 Build and Push Docker Images to ECR

1. Create ECR repos:

```bash
aws ecr create-repository --repository-name employee-frontend
aws ecr create-repository --repository-name employee-backend
```

2. Authenticate:

```bash
aws ecr get-login-password | docker login --username AWS --password-stdin <your_aws_account_id>.dkr.ecr.us-west-2.amazonaws.com
```

3. Build & Push:

```bash
# Backend
cd backend
docker build -t employee-backend .
docker tag employee-backend <AWS_ID>.dkr.ecr.us-west-2.amazonaws.com/employee-backend
docker push <AWS_ID>.dkr.ecr.us-west-2.amazonaws.com/employee-backend

# Frontend
cd frontend
docker build -t employee-frontend .
docker tag employee-frontend <AWS_ID>.dkr.ecr.us-west-2.amazonaws.com/employee-frontend
docker push <AWS_ID>.dkr.ecr.us-west-2.amazonaws.com/employee-frontend
```

---

### 4. ☸️ Deploy to EKS (YAMLs)

#### Example: `backend/k8s.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: backend
        image: <AWS_ECR_BACKEND_IMAGE>
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  selector:
    app: backend
  ports:
    - port: 80
      targetPort: 8080
  type: ClusterIP
```

#### Example: `frontend/k8s.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: frontend
        image: <AWS_ECR_FRONTEND_IMAGE>
        ports:
        - containerPort: 3000
---
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
spec:
  selector:
    app: frontend
  ports:
    - port: 80
      targetPort: 3000
  type: LoadBalancer
```

> Apply the deployments:

```bash
kubectl apply -f backend/k8s.yaml
kubectl apply -f frontend/k8s.yaml
```

---

### 5. 🔁 GitHub Actions CI/CD (`.github/workflows/ci-cd.yaml`)

```yaml
name: CI-CD Pipeline

on:
  push:
    branches: [main]

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    env:
      AWS_REGION: us-west-2
    steps:
    - name: Checkout Code
      uses: actions/checkout@v3

    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v2
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: ${{ env.AWS_REGION }}

    - name: Build and Push Backend
      run: |
        docker build -t ${{ secrets.BACKEND_IMAGE }} ./backend
        docker push ${{ secrets.BACKEND_IMAGE }}

    - name: Build and Push Frontend
      run: |
        docker build -t ${{ secrets.FRONTEND_IMAGE }} ./frontend
        docker push ${{ secrets.FRONTEND_IMAGE }}

    - name: Deploy to EKS
      run: |
        aws eks update-kubeconfig --region $AWS_REGION --name employee-cluster
        kubectl apply -f backend/k8s.yaml
        kubectl apply -f frontend/k8s.yaml
```

---

## 🧪 Test the App

* Get the external URL:

```bash
kubectl get svc frontend-service
```

* Access in browser via `EXTERNAL-IP`

---
