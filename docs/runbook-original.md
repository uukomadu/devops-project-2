# Project 2: Deploying Microservices on Amazon EKS (original, as authored)

> Archived verbatim as the source document. **This version contains four known blockers** —
> see the gap review doc. Use the corrected runbook for actual deployment.

**Overall Explanation:** This project involves setting up an Amazon EKS (Elastic Kubernetes
Service) cluster to deploy containerized applications using Helm, a package manager for
Kubernetes. The goal is to modernize application deployment, allowing for better management,
scaling, and security. The process includes installing required tools, creating an EKS
cluster, and deploying a WordPress microservice that can dynamically scale based on user
traffic.

## Step 1: Install Required Tools

- **What this step does:** Install the AWS CLI and `eksctl`, a command-line utility for creating and managing EKS clusters.
- **Why it's important:** These tools allow you to interact with AWS services and simplify the creation and management of EKS clusters.

```bash
aws --version
```

```bash
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_windows_amd64.zip" -o "eksctl.zip"
unzip eksctl.zip -d eksctl
```

```bash
mkdir -p /c/tools/eksctl
mv ./eksctl/eksctl.exe /c/tools/eksctl/eksctl.exe
```

```bash
export PATH=$PATH:/c/tools/eksctl
```

## Step 2: Set Up AWS CLI Credentials

- **What this step does:** Configure the AWS CLI with your AWS account credentials.
- **Why it's important:** Links your local environment to your AWS account, enabling you to create and manage resources.

```bash
aws configure
```

## Step 3: Create the EKS Cluster

- **What this step does:** Create a new EKS cluster that includes a control plane and worker nodes.
- **Why it's important:** The EKS cluster serves as the foundation for deploying and managing your containerized applications.

```bash
eksctl create cluster \
  --name enterprise-cluster \
  --region us-east-1 \
  --nodegroup-name standard-workers \
  --node-type t3.medium \
  --nodes 3 \
  --nodes-min 3 \
  --nodes-max 6 \
  --managed
```

## Step 4: Verify the Cluster Creation

- **What this step does:** Verify the EKS cluster was created successfully by checking the status of the worker nodes.
- **Why it's important:** Ensuring the cluster is ready is crucial before deploying any applications.

```bash
aws eks --region us-east-1 update-kubeconfig --name enterprise-cluster
```

```bash
kubectl get nodes
```

## Step 5: Install Helm

- **What this step does:** Install Helm on your local machine to manage Kubernetes applications.
- **Why it's important:** Helm simplifies the deployment and management of applications on Kubernetes.

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

```bash
helm version
```

## Step 6: Create a Helm Chart for Your Microservices

- **What this step does:** Create a Helm chart for your microservice (e.g., WordPress).
- **Why it's important:** Helm charts define how applications are deployed on Kubernetes.

```bash
helm create my-microservice
```

Edit `values.yaml`:

```yaml
# Default values for my-microservice.
replicaCount: 1

image:
  repository: wordpress
  pullPolicy: IfNotPresent
  tag: latest

imagePullSecrets: []
nameOverride: ""
fullnameOverride: ""

serviceAccount:
  create: true
  automount: true
  annotations: {}
  name: ""

podAnnotations: {}
podLabels: {}
podSecurityContext: {}
securityContext: {}

service:
  type: LoadBalancer
  port: 80

ingress:
  enabled: false
  className: ""
  annotations: {}
  hosts:
    - host: chart-example.local
      paths:
        - path: /
          pathType: ImplementationSpecific
  tls: []

resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 300m
    memory: 256Mi

autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 100
  targetCPUUtilizationPercentage: 80

persistence:
  enabled: true
  size: 10Gi
  storageClass: ""
  accessModes:
    - ReadWriteOnce

volumes: []
volumeMounts: []
nodeSelector: {}
tolerations: []
affinity: {}
```

## Step 7: Deploy the Microservices Using Helm

```bash
helm install my-microservice ./my-microservice
```

## Step 8: Verify the Microservices Deployment

```bash
helm status my-microservice
```

## Step 9: Implement Horizontal Pod Autoscaler (HPA)

- **What this step does:** Configure HPA to allow Kubernetes to automatically scale the number of running pods based on CPU usage.
- **Why it's important:** Autoscaling ensures that your application can handle varying loads efficiently.

```bash
touch hpa.yaml
```

```yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: wordpress-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: wordpress
  minReplicas: 1
  maxReplicas: 5
  targetCPUUtilizationPercentage: 50
```

```bash
kubectl apply -f hpa.yaml
```

## Step 10: Test Autoscaling Using Siege

```bash
kubectl run siege --image=jstarcher/siege --restart=Never -- /bin/sh -c "siege -c 100 -t 5m http://<EXTERNAL-IP>"
```

```bash
kubectl get hpa -w
kubectl get pods -w
```

## Connecting to the WordPress Application

```bash
kubectl get svc
```

```
NAME                TYPE           CLUSTER-IP       EXTERNAL-IP      PORT(S)        AGE
my-microservice    LoadBalancer   10.100.200.1     a1b2c3d4e5f6g7   80:30956/TCP   10m
```

Open `http://<EXTERNAL-IP>` in a browser.

## Final Result

After completing these steps, your WordPress microservice will be fully operational on an
Amazon EKS cluster. The setup allows for efficient management, scaling, and modernization
of your applications.

## How To Delete Your Resources & Cluster

```bash
kubectl delete all --all
```

```bash
eksctl delete cluster --name enterprise-cluster --region us-east-1
```
