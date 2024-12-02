

# Weather Service with Kops on AWS

This project deploys a weather microservice on a Kubernetes cluster created with kops in AWS us-east-1 region.

## Terraform Resources Used

1. [AWS VPC](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/vpc)
2. [AWS S3 Bucket](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/s3_bucket)
3. [AWS IAM Role](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_role)
4. [AWS Subnet](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/subnet)
5. [AWS Route Table](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/route_table)
6. [AWS Internet Gateway](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/internet_gateway)

## Prerequisites

- AWS CLI configured
- Terraform installed
- kops installed
- Docker installed
- jq installed

## Deployment Steps and Expected Outputs

### 1. Initialize Terraform
```bash
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

### 2. Apply Terraform Configuration
```bash
terraform apply
```
Expected output:
```
Apply complete! Resources: 10 added, 0 changed, 0 destroyed.

Outputs:
kops_state_store = "s3://kops-state-store-weather-service"
cluster_name = "weather.k8s.local"
vpc_id = "vpc-xxxxxxxxxxxxxxxxx"
```

### 3. Create Kubernetes Cluster with Kops
```bash
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
Using cluster from kubectl context: weather.k8s.local
I0123 12:34:56.789012    1234 apply_cluster.go:1000] Created cluster/weather.k8s.local
...
Cluster is starting. It should be ready in a few minutes
```

### 4. Validate Cluster
```bash
kops validate cluster --wait 10m
```
Expected output:
```
Using cluster from kubectl context: weather.k8s.local
Validating cluster weather.k8s.local
...
Your cluster weather.k8s.local is ready
```

### 5. Get Cluster Information
```bash
kops get cluster
```
Expected output:
```
NAME                CLOUD   ZONES
weather.k8s.local   aws     us-east-1a,us-east-1b
```

### 6. Deploy Jenkins
```bash
kubectl apply -f kubernetes/jenkins-deployment.yaml
```
Expected output:
```
namespace/jenkins created
serviceaccount/jenkins created
deployment.apps/jenkins created
service/jenkins created
```

### 7. Get Jenkins Password
```bash
kubectl exec -n jenkins $(kubectl get pods -n jenkins -l app=jenkins -o jsonpath="{.items[0].metadata.name}") -- cat /var/jenkins_home/secrets/initialAdminPassword
```
Expected output:
```
2b8e19e0abc84b84a8c2b8784a8c2b87
```

### 8. Deploy Weather Service
```bash
kubectl apply -f kubernetes/weather-service.yaml
```
Expected output:
```
namespace/weather-service created
deployment.apps/weather-service created
service/weather-service created
```

### 9. Access Services

Jenkins URL format:
```
http://<jenkins-elb-url>:8080
```

Weather Service URL format:
```
http://<weather-service-elb-url>/weather/<city>
```

Example weather service request:
```bash
curl http://<weather-service-url>/weather/London
```
Expected output:
```json
{
  "city": "London",
  "coordinates": {
    "lat": 51.5074,
    "lon": -0.1278
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

## Clean Up

1. Delete the cluster:
```bash
kops delete cluster --name weather.k8s.local --yes
```
Expected output:
```
Type "yes" to delete cluster: yes
...
Deleted cluster: "weather.k8s.local"
```

2. Destroy Terraform resources:
```bash
terraform destroy -auto-approve
```
Expected output:
```
Destroy complete! Resources: 10 destroyed.
```

## Troubleshooting

1. If cluster validation fails:
```bash
kops validate cluster
```
This will show detailed cluster status and any issues.

2. Check node status:
```bash
kops get instancegroups
```
Shows the status of worker nodes.

3. View cluster configuration:
```bash
kops get cluster -o yaml
```
Displays full cluster configuration.

## Note
- The ELB URLs might take a few minutes to become available after service creation
- Jenkins initial setup requires the admin password shown in the output
- The weather service automatically scales with 2 replicas for high availability
