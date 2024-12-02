
# Weather Service Infrastructure Project

This project implements a complete weather service infrastructure using Terraform, Kops, Jenkins, and a Python microservice.

## Infrastructure Components and Resources

Before implementation, here are the components we'll be using:

### 1. AWS Resources
- [AWS Provider](https://registry.terraform.io/providers/hashicorp/aws/latest)
- [S3 Bucket](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/s3_bucket) - For Kops state store
- [VPC Module](https://registry.terraform.io/modules/terraform-aws-modules/vpc/aws/latest) - For networking
- [IAM Role](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_role) - For Kops permissions
- [Security Group](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/security_group) - For cluster access

### 2. Kubernetes Resources
- Jenkins Deployment
- Weather Service Deployment
- Load Balancer Services
- RBAC Configurations

## Prerequisites

1. AWS CLI installed and configured
```bash
aws configure
# Enter your AWS credentials when prompted
```

2. Required tools:
```bash
# Install Terraform
brew install terraform

# Install Kops
brew install kops

# Install kubectl
brew install kubectl

# Install jq (for JSON processing)
brew install jq
```

## Project Structure
```
weather-service-project/
├── terraform/
│   ├── main.tf          # Main infrastructure configuration
│   ├── variables.tf     # Variable definitions
│   ├── outputs.tf       # Output definitions
│   └── providers.tf     # Provider configurations
├── kubernetes/
│   ├── jenkins.yaml     # Jenkins deployment manifest
│   └── weather-service.yaml # Weather service manifest
├── microservice/
│   ├── app.py          # Weather service application
│   ├── requirements.txt # Python dependencies
│   ├── Dockerfile      # Container configuration
│   └── Jenkinsfile     # CI/CD pipeline
└── deploy.sh           # Main deployment script
```

## Step-by-Step Deployment Guide

### 1. Clone the Repository
```bash
git clone <repository-url>
cd weather-service-project
```

### 2. Initialize Terraform
```bash
cd terraform
terraform init
```
Expected output:
```
Initializing the backend...
Initializing provider plugins...
- Finding hashicorp/aws versions matching "~> 5.0"...
- Installing hashicorp/aws v5.x.x...
Terraform has been successfully initialized!
```

### 3. Create Base Infrastructure
```bash
terraform apply
```
Expected output:
```
Apply complete! Resources: 10 added, 0 changed, 0 destroyed.

Outputs:
kops_state_store = "s3://kops-state-store-xxxxx"
cluster_name = "weather.k8s.local"
vpc_id = "vpc-xxxxxxxxxxxxxxxxx"
```

### 4. Create Kubernetes Cluster
```bash
export KOPS_STATE_STORE=s3://kops-state-store-xxxxx
kops create cluster \
    --name=weather.k8s.local \
    --cloud=aws \
    --zones=us-east-1a,us-east-1b \
    --node-count=2 \
    --node-size=t3.medium \
    --kubernetes-version=1.27.0 \
    --yes
```
Expected output:
```
Creating cluster & instancegroup ...
Cluster is starting. It should be ready in a few minutes.
```

### 5. Validate Cluster
```bash
kops validate cluster --wait 10m
```
Expected output:
```
Your cluster weather.k8s.local is ready
```

### 6. Deploy Jenkins
```bash
kubectl apply -f kubernetes/jenkins.yaml
```
Expected output:
```
namespace/jenkins created
serviceaccount/jenkins created
deployment.apps/jenkins created
service/jenkins created
```

### 7. Get Jenkins Credentials
```bash
# Get Jenkins admin password
JENKINS_POD=$(kubectl get pods -n jenkins -l app=jenkins -o jsonpath="{.items[0].metadata.name}")
kubectl exec -n jenkins $JENKINS_POD -- cat /var/jenkins_home/secrets/initialAdminPassword
```
Expected output:
```
2b8e19e0abc84b84a8c2b8784a8c2b87
```

### 8. Deploy Weather Service
```bash
# Build and deploy
cd microservice
docker build -t weather-service:latest .
kubectl apply -f ../kubernetes/weather-service.yaml
```
Expected output:
```
namespace/weather-service created
deployment.apps/weather-service created
service/weather-service created
```

### 9. Access Services

Get service URLs:
```bash
# Jenkins URL
kubectl get svc -n jenkins jenkins -o jsonpath="{.status.loadBalancer.ingress[0].hostname}"

# Weather Service URL
kubectl get svc -n weather-service weather-service -o jsonpath="{.status.loadBalancer.ingress[0].hostname}"
```

Test weather service:
```bash
curl http://<weather-service-url>/weather/London
```
Expected output:
```json
{
  "city": "London",
  "coordinates": {
    "latitude": 51.5074,
    "longitude": -0.1278
  },
  "weather": {
    "temperature": 18.2,
    "windspeed": 15.5,
    "winddirection": 180,
    "weathercode": 1,
    "time": "2023-11-09T12:00"
  }
}
```

## Automated Deployment

To deploy everything automatically:
```bash
chmod +x deploy.sh
./deploy.sh
```

The script will:
1. Create infrastructure using Terraform
2. Set up Kubernetes cluster using Kops
3. Deploy Jenkins
4. Deploy the weather service
5. Display all necessary URLs and credentials

## Clean Up

To delete all resources:

```bash
# 1. Delete Kubernetes resources
kubectl delete -f kubernetes/weather-service.yaml
kubectl delete -f kubernetes/jenkins.yaml

# 2. Delete the cluster
kops delete cluster --name weather.k8s.local --yes

# 3. Destroy Terraform resources
cd terraform && terraform destroy -auto-approve
```

## Troubleshooting

1. If cluster validation fails:
```bash
kops validate cluster
```

2. Check pod status:
```bash
kubectl get pods --all-namespaces
```

3. View service logs:
```bash
kubectl logs -n weather-service deployment/weather-service
```

## Screenshots

[Screenshots of the deployed services will be added here after deployment]

## Notes
- The LoadBalancer URLs might take a few minutes to become available
- Jenkins initial setup requires the admin password shown in the output
- The weather service automatically scales between 2-5 replicas based on load
