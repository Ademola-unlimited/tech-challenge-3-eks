Overview
This project demonstrates the deployment, management, scaling, and monitoring of containerized web applications on Amazon Elastic Kubernetes Service (Amazon EKS) in AWS Region us-east-1.

I provisioned an EKS cluster named enterprise-cluster, connected to it with kubectl, deployed an Nginx-based application, configured Kubernetes Horizontal Pod Autoscalers (HPA), enabled CloudWatch Container Insights, built and published my own Docker image, and deployed that custom application using a Helm chart.

The project includes both a basic Kubernetes-managed Nginx deployment and a separate Helm-managed custom application named eks-project. These are distinct workloads, which is important when interpreting autoscaling results.

GitHub repository: https://github.com/Ademola-unlimited/tech-challenge-3-eks
Custom Docker image: ademolajoshua/eks-app:1.0
Cluster: enterprise-cluster
Region: us-east-1
Helm release: eks-project (namespace: default)
Project status: The custom application was successfully deployed and accessed through an AWS Load Balancer. Its Helm-managed HPA was configured and successfully read CPU metrics, but the static application did not trigger a scale-up during HTTP load testing. A separate cpu-demo workload demonstrated CPU-driven scaling. Infrastructure and application availability may change after the recorded tests or cleanup.

2. Project Objectives
- Provision and access a managed Amazon EKS Kubernetes cluster.
- Understand control-plane and worker-node responsibilities.
- Package a custom web application in a Docker image and publish it to Docker Hub.
- Deploy applications using Kubernetes Deployments and Services.
- Expose an application through an AWS-managed Load Balancer.
- Configure resource requests, limits, and Horizontal Pod Autoscaling.
- Generate load and observe CPU metrics, replica counts, and workload behavior.
- Enable CloudWatch Container Insights for cluster-level observability.
- Package Kubernetes resources into a reusable Helm chart.
- Maintain project configuration and technical documentation in GitHub.
3. Tools and Technologies
Tool / Service	Purpose
AWS EKS	Managed Kubernetes control plane
Amazon EC2	Worker-node compute capacity
AWS CLI	AWS authentication and EKS operations
eksctl	EKS cluster provisioning and management
kubectl	Kubernetes cluster administration
Docker / Docker Desktop	Building and testing container images
Docker Hub	Hosting the custom application image
Nginx	Serving the static HTML application
Helm	Templating, installing, and upgrading Kubernetes resources
Kubernetes HPA	Adjusting replica counts using CPU metrics
Kubernetes Metrics Server	Providing resource metrics used by HPA
Amazon CloudWatch Container Insights	Cluster and container monitoring
Git / GitHub	Source control and project publication
VS Code / Ubuntu on WSL2	Local development and terminal environment


4. Architecture
```mermaid
flowchart TD
    U[User / Browser] --> LB[AWS Load Balancer]
    LB --> S[Kubernetes Service: eks-project]
    S --> P[Helm-managed Nginx Pod(s)]
    P --> I[Custom Docker image: ademolajoshua/eks-app:1.0]
    H[Horizontal Pod Autoscaler] --> D[Kubernetes Deployment: eks-project]
    D --> P
    M[Metrics Server] --> H
    P --> C[CloudWatch Container Insights]
    N[EKS Worker Nodes] --> P
    E[Amazon EKS Control Plane] --> D
```
Request flow: A user accesses the external AWS Load Balancer. The Load Balancer forwards requests to the Kubernetes Service, which routes traffic to available application Pods. Each Pod runs the Docker image containing the custom Nginx webpage.

Scaling flow: Metrics Server supplies CPU measurements to the HPA. The HPA compares observed CPU utilization with the target and adjusts the desired replica count of the Deployment, subject to minimum and maximum limits.

Monitoring flow: CloudWatch agents and Fluent Bit components collect cluster/container telemetry and send supported metrics and logs to AWS CloudWatch.
The diagram illustrates the logical architecture; it is not a complete AWS VPC, subnet, or security-group diagram.

5. Repository Structure
tech-challenge-3-eks/
├── README.md
├── .gitignore
├── cluster-config.yaml
├── eks-app/
│   ├── Dockerfile
│   └── index.html
├── kubernetes/
│   ├── nginx-deployment.yaml
│   ├── nginx-service.yaml
│   └── nginx-hpa.yaml
└── eks-project/
    ├── Chart.yaml
    ├── values.yaml
    ├── .helmignore
    └── templates/
        ├── deployment.yaml
        ├── service.yaml
        ├── serviceaccount.yaml
        ├── hpa.yaml
        ├── ingress.yaml
        ├── httproute.yaml
        ├── _helpers.tpl
        ├── NOTES.txt
        └── tests/
            └── test-connection.yaml
This tree documents the files observed during the project. CloudWatch-related IAM or local temporary files may exist outside the committed tree; they should be reviewed before publication.
6. Phase 1 — Environment Preparation
The development environment used VS Code with an Ubuntu/WSL2 terminal and Docker Desktop integration. AWS CLI, eksctl, kubectl, Helm, Docker, and Git were used for the deployment workflow.

Representative verification commands:
aws --version
eksctl version
kubectl version --client
helm version
docker version
git --version
Why this matters: Kubernetes development requires separate tools for cloud provisioning, cluster administration, container packaging, and application releases. Verifying them early prevents avoidable deployment errors.
7. Phase 2 — Provisioning and Connecting to Amazon EKS
The EKS environment was provisioned with cluster name enterprise-cluster in us-east-1. The running cluster was verified with three Linux worker nodes, as shown by kubectl get nodes output during the project.

To configure local access:
aws eks --region us-east-1 update-kubeconfig --name enterprise-cluster
kubectl config current-context
kubectl get nodes
The context identified the enterprise-cluster EKS cluster. kubectl communicates with the Kubernetes API server using the context and credentials configured in kubeconfig.

Challenge encountered: An initial managed node group provisioning attempt failed because of the selected EC2 instance type. The worker-node configuration was adjusted and the node group was recreated successfully.

Lesson: A successfully created control plane is not enough; workloads require healthy, schedulable worker nodes.
8. Phase 3 — Kubernetes Deployment and Services
An initial Nginx application was deployed using Kubernetes YAML manifests. The repository includes Deployment, Service, and HPA manifests in kubernetes/.

Useful commands:
kubectl get deployments
kubectl get pods
kubectl get services
kubectl get hpa
The initial nginx-app deployment ran two healthy Pods during verification. Its Service was configured separately from the later Helm-managed eks-project Service.

Key concepts:
- A Deployment declares the desired application replicas and Pod template.
- A Pod is the smallest deployable unit in Kubernetes.
- A Service provides a stable network entry point for Pods.
- A LoadBalancer Service asks the cloud provider to expose the application externally.
- Kubernetes can replace failed Pods to maintain the Deployment's desired state.
9. Phase 4 — Autoscaling and Load Testing
Horizontal Pod Autoscaling was configured and tested. The earlier cpu-demo workload demonstrated an increase in replica count under CPU load. During one observation, its HPA showed approximately 201% CPU utilization against a 50% target, with three replicas running.

A separate nginx-app HPA was also present. These observations are not evidence that the later Helm-managed custom application scaled.

Representative monitoring commands:
kubectl get hpa
kubectl get hpa -w
kubectl get pods -w
kubectl describe hpa cpu-demo
Why scaling may be delayed: The HPA must receive usable CPU metrics, calculate a desired replica count, and wait for Kubernetes to create and schedule Pods. Scaling behavior can also be affected by stabilization windows and workload resource requests.

Challenge encountered: Metrics Server was initially unavailable, and HPA CPU readings appeared as <unknown>. After metrics became available and CPU resource requests were configured, autoscaling could be evaluated.
10. Phase 5 — Custom Docker Application
A simple static webpage was created in eks-app/index.html, with the title Tech Pathway EKS Project and text confirming that it runs inside Kubernetes and was deployed using Docker and Amazon EKS.

The application uses an Nginx container image. A representative Dockerfile is:
FROM nginx:latest
COPY . /usr/share/nginx/html
The application was built and published to Docker Hub as:
ademolajoshua/eks-app:1.0
Representative commands:
cd eks-app
docker build -t ademolajoshua/eks-app:1.0 .
docker login
docker push ademolajoshua/eks-app:1.0
A local container was also tested using a port mapping to Nginx port 80. Docker Desktop/WSL2 integration initially caused a permission error accessing the Docker socket; Docker access was subsequently verified and the image built successfully.

Why containerization matters: The application and its web server are packaged into a reproducible image that Kubernetes can pull and run across worker nodes.
11. Phase 6 — Helm Chart Creation
A Helm chart named eks-project was created to manage the custom application declaratively. The chart contains templates for a Deployment, Service, HPA, ServiceAccount, and optional networking resources.

Important settings in eks-project/values.yaml:
replicaCount: 1

image:
  repository: ademolajoshua/eks-app
  pullPolicy: IfNotPresent
  tag: "1.0"

service:
  type: LoadBalancer
  port: 80

resources:
  requests:
    cpu: 300m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi

autoscaling:
  enabled: true
  minReplicas: 1
  maxReplicas: 5
  targetCPUUtilizationPercentage: 50
These values were validated with:
helm lint ./eks-project
helm template eks-project ./eks-project
The successful lint output was:
1 chart(s) linted, 0 chart(s) failed
Challenge encountered: A duplicate YAML key line, resources: resources:, caused Helm to reject values.yaml. Correcting the YAML structure and indentation resolved the error.

Why Helm matters: Helm provides reusable templates, centralized configuration, and versioned application upgrades without manually applying every manifest.
12. Phase 7 — Helm Deployment to EKS
After confirming the active Kubernetes context pointed to enterprise-cluster, the Helm release was installed:
helm install eks-project ./eks-project \
  --namespace default \
  --wait \
  --timeout 10m
Verification:
helm status eks-project
kubectl get pods
kubectl get svc
The release reported STATUS: deployed. The custom application Pod became 1/1 Running, and the eks-project Service received an AWS Load Balancer DNS name.

The webpage was opened successfully in a browser, showing:
Tech Pathway EKS Project
This application is running inside Kubernetes.
Deployed using Docker and Amazon EKS.

Result: The custom application was accessible through the AWS Load Balancer at the time of testing. Do not assume an old Load Balancer address remains active after infrastructure changes.
13. Phase 8 — Helm-Managed Horizontal Pod Autoscaler
The chart's autoscaling configuration was enabled with a 50% CPU target, minimum 1 replica, and maximum 5 replicas. The existing Helm release was upgraded rather than reinstalled:
helm upgrade eks-project ./eks-project \
  --namespace default \
  --wait \
  --timeout 10m
Helm reported STATUS: deployed and revision 2.

Verification commands:
kubectl get hpa eks-project
kubectl describe hpa eks-project
kubectl get deployment eks-project
Observed results:
HPA setting / condition	Observed result
CPU target	50%
Minimum replicas	1
Maximum replicas	5
Current replicas	1
CPU utilization at rest	0%
AbleToScale	True
ScalingActive	True
ScalingLimited	True — TooFewReplicas


ScalingLimited=True with reason TooFewReplicas was expected at low load: the desired replica count was below the configured minimum of one, so the HPA retained one Pod.
HTTP load-test findings
Temporary BusyBox Pods sent HTTP requests to the application Service. The initial test completed quickly. A corrected test ran for a sustained interval and briefly showed CPU usage around 3%, but the application remained below the 50% HPA threshold and stayed at one replica.

Accurate conclusion: The Helm-managed HPA was active and successfully evaluated CPU metrics, but scale-up of eks-project was not observed during the HTTP tests. The Nginx static webpage was too lightweight to create the required CPU pressure. Actual scale-up was observed on the separate cpu-demo workload.

This distinction is important for honest technical reporting.
14. Phase 9 — CloudWatch Monitoring and Container Insights
CloudWatch Container Insights was enabled for the EKS cluster. The eks-pod-identity-agent and amazon-cloudwatch-observability add-ons were verified as ACTIVE during setup.

Observed monitoring components included:
- cloudwatch-agent DaemonSet: 3 desired, 3 ready on Linux nodes.
- fluent-bit DaemonSet: 3 desired, 3 ready on Linux nodes.
- CloudWatch log groups under /aws/containerinsights/enterprise-cluster/.
Verified log groups:
/aws/containerinsights/enterprise-cluster/application
/aws/containerinsights/enterprise-cluster/dataplane
/aws/containerinsights/enterprise-cluster/host
/aws/containerinsights/enterprise-cluster/performance
Log streams were observed for the three worker nodes.

Representative verification commands:
kubectl get pods -n amazon-cloudwatch
kubectl get daemonsets -n amazon-cloudwatch
aws eks describe-addon \
  --cluster-name enterprise-cluster \
  --addon-name amazon-cloudwatch-observability \
  --region us-east-1
aws logs describe-log-groups \
  --log-group-name-prefix /aws/containerinsights/enterprise-cluster/ \
  --region us-east-1
IAM and identity troubleshooting: The CloudWatch agent initially encountered an ECR image-pull authorization error. The setup was reviewed and adjusted, including the EKS Pod Identity agent, IAM policy attachment, and Pod Identity association for the amazon-cloudwatch/cloudwatch-agent ServiceAccount. Monitoring components subsequently ran and log groups/streams appeared.

Why monitoring matters: Container Insights helps operators inspect resource usage, cluster behavior, and telemetry during load tests and troubleshooting.
Log-group creation and running agents were verified. Specific CloudWatch dashboard screenshots of CPU, memory, and pod behavior should be included as evidence if required by the instructor; do not infer that every dashboard view was captured.

15. Technical Challenges and Resolutions
Challenge	Resolution / learning
Managed node group creation failed	Adjusted worker-node instance type and recreated the node group.
Metrics Server unavailable / HPA showed unknown CPU	Verified metrics availability and resource requests before evaluating scaling.
CloudWatch agent image-pull error (ECR 403)	Investigated IAM, add-on, and Pod Identity setup; monitoring components subsequently became healthy.
Docker socket permission denied in WSL2	Corrected Docker Desktop integration/access and verified Docker commands.
Helm YAML parsing failure	Removed duplicate resources: entry and validated indentation with helm lint.
Load-test Pod name already existed	Deleted the completed temporary Pod before reusing its name.
Static Nginx workload did not scale under HTTP requests	Verified HPA metrics and documented the low CPU demand; distinguished it from successful scaling on cpu-demo.
GitHub repository needed updating	Added and pushed the custom application and Helm chart in separate commits.


16. Verification Checklist
- [x] EKS cluster connection established.
- [x] Three Linux worker nodes observed.
- [x] Kubernetes Nginx workload deployed.
- [x] Custom Docker image built and pushed to Docker Hub.
- [x] Custom webpage accessible through AWS Load Balancer.
- [x] Helm chart created and linted successfully.
- [x] Helm release installed and upgraded successfully.
- [x] CPU/memory requests and limits configured.
- [x] Helm-managed HPA active and reading CPU metrics.
- [x] HTTP load test executed against custom application.
- [x] Separate CPU-intensive workload demonstrated HPA scale-up.
- [x] CloudWatch observability add-on active.
- [x] CloudWatch agent and Fluent Bit running across three nodes.
- [x] CloudWatch Container Insights log groups and streams observed.
- [x] Application and Helm chart pushed to GitHub.
- [ ] Helm-managed eks-project scale-up demonstrated under sufficient CPU load (not observed).
- [ ] Final screenshot evidence linked or embedded in this README.
- [ ] AWS cleanup confirmed after submission (if no longer needed).
17. Suggested Screenshot Evidence
Store screenshots in a dedicated screenshots/ folder and link them in the README after verifying the filenames. Recommended evidence:
1. EKS cluster and three Ready worker nodes.
2. Initial Kubernetes Deployment and healthy Pods.
3. Docker image build and Docker Hub push.
4. Helm lint success (0 chart(s) failed).
5. Helm release status (deployed, revision 2).
6. Custom webpage displayed in the browser.
7. LoadBalancer Service and external address.
8. HPA configuration and conditions.
9. cpu-demo HPA scale-up evidence.
10. CloudWatch agent and Fluent Bit DaemonSets.
11. CloudWatch Container Insights log groups and relevant metrics dashboards.
12. GitHub repository structure and commits.
Evidence note: Do not include access keys, secret values, tokens, kubeconfig credentials, or private key files in screenshots or GitHub commits.
18. Useful Operational Commands
# Verify AWS/EKS connection
aws eks --region us-east-1 update-kubeconfig --name enterprise-cluster
kubectl get nodes

# Inspect workloads
kubectl get deployments,pods,svc,hpa
kubectl describe hpa eks-project

# Inspect Helm release
helm list
helm status eks-project
helm lint ./eks-project

# Inspect monitoring agents
kubectl get daemonsets -n amazon-cloudwatch
kubectl get pods -n amazon-cloudwatch

# Follow application logs (adjust labels if chart changes)
kubectl logs -l app.kubernetes.io/instance=eks-project --tail=100
19. Cleanup and Cost Awareness
Amazon EKS, EC2 worker nodes, AWS Load Balancers, networking resources, and CloudWatch ingestion/storage can incur charges. Once all evidence is collected and the project is no longer needed, remove the Helm release and cluster using the provisioning method that originally created them.

Do not execute cleanup until screenshots and final submission are complete.
# Optional: remove the Helm-managed application first
helm uninstall eks-project --namespace default

# Only when you are ready to delete the entire EKS environment:
eksctl delete cluster --name enterprise-cluster --region us-east-1
Verify deletion in the AWS Console and check for leftover billable resources. Do not run kubectl delete all --all indiscriminately; it may affect workloads beyond this project.
20. Key Lessons Learned
This project strengthened my practical understanding of managed Kubernetes infrastructure, container packaging, declarative application configuration, Helm releases, CPU-based autoscaling, and cloud observability.

The most important lesson was that a correctly configured autoscaler does not necessarily increase replicas. It scales only when measured demand and policy settings justify it. I also learned how to troubleshoot node-group provisioning, missing metrics, Docker permissions, Helm YAML syntax, and IAM permissions for monitoring components.
21. Conclusion
Tech Challenge 3 demonstrates an end-to-end workflow for building, deploying, operating, and observing a containerized application on Amazon EKS. The final custom application was packaged with Docker, deployed with Helm, exposed through an AWS Load Balancer, monitored through Kubernetes and CloudWatch, and documented in GitHub.

The project also includes a successful separate CPU-driven scaling demonstration and a transparent explanation of why the lightweight Helm-managed application did not scale during HTTP load testing. Together, these results demonstrate practical DevOps skills in Kubernetes operations, automation, troubleshooting, scalability, and monitoring.