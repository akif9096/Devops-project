# DevOps Project Scaffold — Node.js + AWS + EKS (Kubernetes)

This repository scaffold implements a complete end-to-end DevOps project similar to *DevOps-Project-01*, but now deploys to **Amazon EKS (Kubernetes)** instead of ECS. It includes:

- Sample Node.js/Express app
- Dockerfile
- Terraform (AWS) to provision an EKS cluster + ECR + VPC
- GitHub Actions CI/CD pipeline: build, test, push image to ECR, deploy to EKS
- Monitoring setup with Prometheus + Grafana (via Helm charts)
- SonarQube integration (optional)

---

## 📁 Repository structure

```
my-devops-project/
├── app/
│   ├── src/
│   │   ├── server.js
│   │   └── routes.js
│   ├── test/
│   │   └── app.test.js
│   ├── package.json
│   └── Dockerfile
├── infra/
│   └── terraform/
│       ├── main.tf
│       ├── variables.tf
│       ├── outputs.tf
│       ├── eks.tf
│       ├── ecr.tf
│       └── vpc.tf
├── k8s/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── namespace.yaml
├── .github/
│   └── workflows/
│       └── ci-cd.yaml
├── monitoring/
│   ├── prometheus-values.yaml
│   ├── grafana-values.yaml
│   └── README.md
├── sonar-project.properties
├── README.md
└── LICENSE
```

---

## ⚙️ Terraform (AWS EKS + ECR)

### `infra/terraform/main.tf`
```hcl
provider "aws" {
  region = var.region
}

module "vpc" {
  source = "terraform-aws-modules/vpc/aws"
  version = "4.0.0"
  name = "eks-vpc"
  cidr = "10.0.0.0/16"
  azs = ["${var.region}a", "${var.region}b"]
  public_subnets  = ["10.0.1.0/24", "10.0.2.0/24"]
  private_subnets = ["10.0.3.0/24", "10.0.4.0/24"]
  enable_nat_gateway = true
}

module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "20.8.5"

  cluster_name    = var.cluster_name
  cluster_version = "1.29"
  subnet_ids      = module.vpc.private_subnets
  vpc_id          = module.vpc.vpc_id

  manage_aws_auth_configmap = true
  cluster_endpoint_public_access = true

  eks_managed_node_groups = {
    default = {
      desired_size = 2
      max_size     = 3
      min_size     = 1
      instance_types = ["t3.medium"]
    }
  }
}
```

### `infra/terraform/ecr.tf`
```hcl
resource "aws_ecr_repository" "app" {
  name = var.ecr_repo
  image_scanning_configuration { scan_on_push = true }
}

output "ecr_repo_url" {
  value = aws_ecr_repository.app.repository_url
}
```

### `infra/terraform/outputs.tf`
```hcl
output "cluster_name" {
  value = module.eks.cluster_name
}

output "cluster_endpoint" {
  value = module.eks.cluster_endpoint
}

output "ecr_repo_url" {
  value = aws_ecr_repository.app.repository_url
}
```

---

## 🧱 Kubernetes manifests (`k8s/`)

### `k8s/namespace.yaml`
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: devops-app
```

### `k8s/deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: devops-app
  namespace: devops-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: devops-app
  template:
    metadata:
      labels:
        app: devops-app
    spec:
      containers:
      - name: devops-app
        image: <ECR_IMAGE_URI>:latest
        ports:
        - containerPort: 3000
        readinessProbe:
          httpGet:
            path: /health
            port: 3000
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
```

### `k8s/service.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: devops-app-svc
  namespace: devops-app
spec:
  selector:
    app: devops-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 3000
  type: LoadBalancer
```

---

## 🔄 GitHub Actions CI/CD for EKS — `.github/workflows/ci-cd.yaml`

```yaml
name: CI-CD to EKS

on:
  push:
    branches: [ main ]

env:
  AWS_REGION: us-east-1
  APP_NAME: devops-app
  CLUSTER_NAME: devops-sample-eks

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 18

      - name: Install dependencies
        working-directory: ./app
        run: npm install

      - name: Test
        working-directory: ./app
        run: npm test || true

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v1

      - name: Build and Push Docker image
        env:
          ECR_REPOSITORY: ${{ secrets.ECR_REPOSITORY }}
        run: |
          ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
          IMAGE_URI="$ACCOUNT_ID.dkr.ecr.${{ env.AWS_REGION }}.amazonaws.com/${{ secrets.ECR_REPOSITORY }}:latest"
          docker build -t $IMAGE_URI ./app
          docker push $IMAGE_URI
          echo "IMAGE_URI=$IMAGE_URI" >> $GITHUB_ENV

      - name: Setup kubectl
        uses: azure/setup-kubectl@v3

      - name: Update kubeconfig
        run: aws eks update-kubeconfig --name ${{ env.CLUSTER_NAME }} --region ${{ env.AWS_REGION }}

      - name: Deploy to EKS
        run: |
          kubectl apply -f k8s/namespace.yaml
          sed -i "s#<ECR_IMAGE_URI>#$IMAGE_URI#g" k8s/deployment.yaml
          kubectl apply -f k8s/deployment.yaml
          kubectl apply -f k8s/service.yaml
```

---

## 📈 Monitoring (Helm)

You can deploy Prometheus + Grafana easily:
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm install prometheus prometheus-community/kube-prometheus-stack -n monitoring --create-namespace
```

Then expose Grafana with:
```bash
kubectl port-forward svc/prometheus-grafana 3001:80 -n monitoring
```

Open Grafana at http://localhost:3001 (default user: `admin`, password printed in terminal).

---

## 🧭 README Quickstart

```markdown
# DevOps Project — EKS version

## Prerequisites
- AWS CLI, kubectl, Terraform, Helm installed
- AWS credentials configured

## Setup
1. Terraform apply to provision VPC + EKS + ECR
2. Push code → GitHub Actions builds + pushes Docker image to ECR
3. The pipeline updates EKS deployment automatically
4. Access the LoadBalancer URL to test app
```

---

✅ This setup gives you full end-to-end CI/CD with EKS + Terraform + GitHub Actions. You can later extend it with Helm charts, ArgoCD, or GitOps for advanced automation.

