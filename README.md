# Tech Challenge 3 – Amazon EKS Deployment, Scaling & Monitoring

## Project Overview

This project demonstrates how to deploy, manage, scale, and monitor a containerized application using Amazon Elastic Kubernetes Service (EKS).

The project uses AWS EKS, EC2 worker nodes, Kubernetes Deployments and Services, Metrics Server, and Horizontal Pod Autoscaler (HPA).

## Technologies Used

- Amazon Web Services (AWS)
- Amazon EKS
- Amazon EC2
- Kubernetes and kubectl
- eksctl
- Docker container images
- Nginx
- Metrics Server
- YAML configuration files
- Linux / WSL Ubuntu

## Project Architecture

The Amazon EKS cluster contains three managed EC2 worker nodes distributed across two Availability Zones in the us-east-1 region.

The Nginx application runs as a Kubernetes Deployment and is accessible internally through a ClusterIP Service.

Metrics Server provides CPU and memory utilization measurements, while HPA automatically adjusts application replicas based on CPU demand.

## Implementation

### 1. EKS Cluster Provisioning

Created an Amazon EKS cluster named `enterprise-cluster` using eksctl.

Configured a managed node group named `standard-workers` with three EC2 worker nodes.

### 2. Application Deployment

Deployed the Nginx container image using Kubernetes.

Configured a Deployment to maintain the required number of application Pods.

### 3. Kubernetes Networking

Created `nginx-service` as a ClusterIP Service on port 80.

Verified application access using Kubernetes port forwarding.

### 4. Manual Scaling and Self-Healing

Scaled the Nginx Deployment from two to four replicas.

Deleted an application Pod and verified that Kubernetes automatically created a replacement.

### 5. Resource Monitoring

Installed Kubernetes Metrics Server.

Monitored CPU and memory usage using:

`kubectl top nodes`

`kubectl top pods`

### 6. Horizontal Pod Autoscaling

Configured Nginx HPA with a minimum of two replicas, maximum of six replicas, and a CPU utilization target of 50%.

Observed automatic scale-down from four Pods to two Pods when CPU demand was low.

Created a temporary CPU-intensive Deployment to demonstrate automatic scale-up from one Pod to three Pods.

Removed the temporary test resources after completing the demonstration.

### 7. Declarative Kubernetes Configuration

Created reusable YAML manifests for:

- Nginx Deployment
- Nginx ClusterIP Service
- Nginx Horizontal Pod Autoscaler

Validated the manifests, reviewed differences, and successfully applied them to the EKS cluster.

### 8. Troubleshooting and Verification

Used Kubernetes commands to inspect Pods, Deployments, Services, ReplicaSets, events, and application logs.

Verified three Ready worker nodes, two healthy Nginx Pods, and functioning CPU-based autoscaling metrics.

## Challenges Encountered

- An initial EKS managed node group creation failed because the selected EC2 instance type did not satisfy the account's instance eligibility restrictions.
- Resolved the provisioning issue by adjusting the worker node instance type and recreating the node group.
- Metrics Server was initially unavailable and needed to be installed.
- HPA initially displayed unknown CPU utilization until resource requests and metrics became available.
- The first HTTP load test generated insufficient Nginx CPU utilization to trigger automatic scale-up.
- A separate CPU-intensive test workload successfully demonstrated HPA scale-up.

## Lessons Learned

This project improved my understanding of Kubernetes architecture, managed EKS infrastructure, application deployment, networking, self-healing, resource management, monitoring, automatic scaling, troubleshooting, and declarative configuration.

## Project Files

- `cluster-config.yaml` – eksctl cluster configuration
- `kubernetes/nginx-deployment.yaml` – Nginx Deployment manifest
- `kubernetes/nginx-service.yaml` – Kubernetes Service manifest
- `kubernetes/nginx-hpa.yaml` – Horizontal Pod Autoscaler manifest


## Final Monitoring Results

### Amazon CloudWatch Observability
- Installed the Amazon CloudWatch Observability add-on.
- Verified the add-on status was ACTIVE with no health issues.
- Successfully deployed 3 CloudWatch Agent Pods.
- Successfully deployed 3 Fluent Bit Pods.
- Configured EKS Pod Identity with the CloudWatchAgentServerPolicy.
- Verified Container Insights performance logs for all 3 worker nodes.

### Application Verification
- Confirmed Nginx deployment had 2/2 healthy replicas.
- Verified the Nginx Service was accessible through port forwarding.
- Successfully opened the Nginx welcome page at http://127.0.0.1:8080.
- Verified HPA was configured with a 50% CPU target.
- Confirmed minimum replicas of 2 and maximum replicas of 6.

### Final Project Outcome
Successfully deployed, scaled, accessed, and monitored
a containerized application on Amazon EKS using Kubernetes,
Horizontal Pod Autoscaler, CloudWatch, and EKS Pod Identity.
## Conclusion

This project demonstrates practical DevOps skills in provisioning and managing a Kubernetes environment on AWS, maintaining application availability, monitoring resource utilization, and automating application scaling.
