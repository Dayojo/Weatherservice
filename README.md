# Weatherservice
Weather Service on EKS


# Weather Service on EKS

This project sets up a weather microservice on AWS EKS with Jenkins CI/CD.

## Prerequisites

- AWS CLI configured with admin permissions
- Terraform installed
- kubectl installed
- Docker installed

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
- Finding hashicorp/kubernetes versions matching "~> 2.23"...
- Installing hashicorp/aws v5.x.x...
- Installing hashicorp/kubernetes v2.23.x...

Terraform has been successfully initialized!
```

### 2. Apply Terraform Configuration
```bash
terraform apply
```
Expected output:
```
Apply complete! Resources: 53 added, 0 changed, 0 destroyed.

Outputs:
cluster_endpoint = "https://XXXXXXXXXXXX.gr7.us-east-2.eks.amazonaws.com"
cluster_name = "my-eks-cluster"
configure_kubectl = "aws eks update-kubeconfig --region us-east-2 --name my-eks-cluster"
```

### 3. Configure kubectl
```bash
aws eks update-kubeconfig --region us-east-2 --name my-eks-cluster
```
Expected output:
```
Added new context arn:aws:eks:us-east-2:XXXXXXXXXXXX:cluster/my-eks-cluster to /Users/username/.kube/config
```

### 4. Deploy Jenkins
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

To get Jenkins admin password:
```bash
kubectl exec -n jenkins $(kubectl get pods -n jenkins -l app=jenkins -o jsonpath="{.items[0].metadata.name}") -- cat /var/jenkins_home/secrets/initialAdminPassword
```
Expected output:
```
2b8e19e0abc84b84a8c2b8784a8c2b87
```

To get Jenkins URL:
```bash
kubectl get svc -n jenkins jenkins
```
Expected output:
```
NAME      TYPE           CLUSTER-IP      EXTERNAL-IP                                    PORT(S)
jenkins   LoadBalancer   10.100.50.175   a1b2c3d4e5f6g.us-east-2.elb.amazonaws.com    8080:31234/TCP
```

### 5. Deploy Weather Service
```bash
kubectl apply -f kubernetes/weather-service.yaml
```
Expected output:
```
namespace/weather-service created
deployment.apps/weather-service created
service/weather-service created
```

To get Weather Service URL:
```bash
kubectl get svc -n weather-service weather-service
```
Expected output:
```
NAME             TYPE           CLUSTER-IP       EXTERNAL-IP                                    PORT(S)
weather-service  LoadBalancer   10.100.100.123   h1i2j3k4l5m6n.us-east-2.elb.amazonaws.com    80:32456/TCP
```

### Testing the Weather Service
```bash
curl http://<weather-service-external-ip>/weather/London
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

### Clean Up
```bash
terraform destroy
```
Expected output:
```
Destroy complete! Resources: 53 destroyed.
```

## Note
- The External-IP URLs might take a few minutes to become available after service creation
- Jenkins initial setup requires the admin password shown in the output
- The weather service automatically scales with 2 replicas for high availability
