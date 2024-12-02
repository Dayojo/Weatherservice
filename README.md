# AWS EKS Cluster with Jenkins CI/CD and Weather Microservice

This project sets up a complete infrastructure on AWS including:
- EKS (Kubernetes) Cluster
- Jenkins CI/CD installation on Kubernetes
- Weather microservice with CI/CD pipeline

## Prerequisites

- AWS CLI installed and configured
- Terraform installed
- kubectl installed
- Git installed

## Project Structure

```
.
├── terraform/
│   ├── main.tf           # Main Terraform configuration
│   ├── variables.tf      # Variable definitions
│   ├── providers.tf      # Provider configurations
│   └── outputs.tf        # Output definitions
├── kubernetes/
│   ├── jenkins.yaml      # Jenkins Kubernetes deployment
│   └── weather-service.yaml  # Weather service deployment
├── microservice/
│   ├── app.py           # Weather service application
│   ├── requirements.txt  # Python dependencies
│   ├── Dockerfile       # Container image definition
│   └── Jenkinsfile      # CI/CD pipeline definition
└── deploy.sh            # Main deployment script
```

## Step 1: AWS EKS Cluster Setup

### VPC and Networking Setup

First, we'll create a VPC for our EKS cluster.

**Reference**: [AWS VPC Module](https://registry.terraform.io/modules/terraform-aws-modules/vpc/aws/latest)

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.0.0"

  name = "eks-vpc"
  cidr = "10.0.0.0/16"

  azs             = ["us-east-1a", "us-east-1b"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24"]

  enable_nat_gateway = true
  single_nat_gateway = true

  tags = {
    "kubernetes.io/cluster/weather-eks-cluster" = "shared"
  }

  public_subnet_tags = {
    "kubernetes.io/cluster/weather-eks-cluster" = "shared"
    "kubernetes.io/role/elb"                    = "1"
  }

  private_subnet_tags = {
    "kubernetes.io/cluster/weather-eks-cluster" = "shared"
    "kubernetes.io/role/internal-elb"           = "1"
  }
}
```

### EKS Cluster Setup

**Reference**: [AWS EKS Module](https://registry.terraform.io/modules/terraform-aws-modules/eks/aws/latest)

```hcl
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "19.15.3"

  cluster_name    = "weather-eks-cluster"
  cluster_version = "1.27"

  vpc_id                         = module.vpc.vpc_id
  subnet_ids                     = module.vpc.private_subnets
  cluster_endpoint_public_access = true

  eks_managed_node_group_defaults = {
    ami_type = "AL2_x86_64"
  }

  eks_managed_node_groups = {
    general = {
      desired_size = 2
      min_size     = 1
      max_size     = 3

      instance_types = ["t3.medium"]
      capacity_type  = "ON_DEMAND"
    }
  }

  tags = {
    Environment = "dev"
    Project     = "WeatherService"
  }
}
```

## Step 2: Jenkins Installation on Kubernetes

The Jenkins deployment will be configured using Kubernetes manifests to run in the EKS cluster.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: jenkins

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jenkins
  namespace: jenkins
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jenkins
  template:
    metadata:
      labels:
        app: jenkins
    spec:
      containers:
      - name: jenkins
        image: jenkins/jenkins:lts
        ports:
        - containerPort: 8080
        volumeMounts:
        - name: jenkins-home
          mountPath: /var/jenkins_home
      volumes:
      - name: jenkins-home
        emptyDir: {}

---
apiVersion: v1
kind: Service
metadata:
  name: jenkins
  namespace: jenkins
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: jenkins
```

## Step 3: Weather Microservice

### Python Application (app.py)

```python
from flask import Flask, request, jsonify
import requests

app = Flask(__name__)

@app.route('/weather', methods=['GET'])
def get_weather():
    lat = request.args.get('lat', '40.7128')  # Default to New York
    lon = request.args.get('lon', '-74.0060')
    
    url = f"https://api.open-meteo.com/v1/forecast?latitude={lat}&longitude={lon}&current_weather=true"
    response = requests.get(url)
    
    if response.status_code == 200:
        return jsonify(response.json())
    else:
        return jsonify({"error": "Failed to fetch weather data"}), 500

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

### Dockerfile

```dockerfile
FROM python:3.9-slim

WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY app.py .

EXPOSE 5000
CMD ["python", "app.py"]
```

### Jenkinsfile

```groovy
pipeline {
    agent any
    
    environment {
        DOCKER_IMAGE = 'weather-service'
        DOCKER_TAG = "${BUILD_NUMBER}"
    }
    
    stages {
        stage('Build') {
            steps {
                sh 'docker build -t $DOCKER_IMAGE:$DOCKER_TAG .'
            }
        }
        
        stage('Deploy') {
            steps {
                sh '''
                    kubectl apply -f kubernetes/weather-service.yaml
                    kubectl set image deployment/weather-service weather-service=$DOCKER_IMAGE:$DOCKER_TAG
                '''
            }
        }
    }
}
```

## Deployment Script (deploy.sh)

```bash
#!/bin/bash

# Initialize and apply Terraform
cd terraform
terraform init
terraform apply -auto-approve

# Configure kubectl
aws eks update-kubeconfig --name weather-eks-cluster --region us-east-1

# Install Jenkins
kubectl apply -f ../kubernetes/jenkins.yaml

# Wait for Jenkins to be ready
echo "Waiting for Jenkins to be ready..."
kubectl wait --namespace jenkins \
  --for=condition=ready pod \
  --selector=app=jenkins \
  --timeout=300s

# Get Jenkins admin password
echo "Jenkins initial admin password:"
kubectl exec -n jenkins $(kubectl get pods -n jenkins -l app=jenkins -o jsonpath='{.items[0].metadata.name}') -- cat /var/jenkins_home/secrets/initialAdminPassword

# Deploy weather service
kubectl apply -f ../kubernetes/weather-service.yaml

echo "Deployment complete!"
```

## Usage

1. Clone this repository
2. Run the deployment script:
```bash
chmod +x deploy.sh
./deploy.sh
```

3. Access Jenkins:
   - Get the Jenkins URL from:
   ```bash
   kubectl get svc -n jenkins jenkins
   ```
   - Use the initial admin password printed during deployment

4. Access the Weather Service:
   ```bash
   kubectl get svc weather-service
   ```
   Example API call:
   ```bash
   curl "http://<WEATHER_SERVICE_URL>/weather?lat=40.7128&lon=-74.0060"
   ```

## Important Notes

- All resources are created in the us-east-1 region
- The EKS cluster uses t3.medium instances for cost-effectiveness
- Jenkins is exposed via a LoadBalancer service
- The weather service uses the free Open-Meteo API
- Remember to destroy resources when done:
  ```bash
  cd terraform
  terraform destroy -auto-approve
