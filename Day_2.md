# Day 2 - Kubernetes Fundamentals

## Overview
This document covers Kubernetes architecture, components, and basic operations including cluster setup and resource management.

## Kubernetes Architecture

### Control Plane Components

The control plane manages the Kubernetes cluster and makes global decisions about the cluster.

#### API Server
- Acts as the main controller that accepts requests
- Clients make POST requests with manifest files
- Entry point for all administrative tasks

#### etcd
- Distributed database that stores all cluster manifests
- Persistent storage for cluster state
- Ensures data consistency across the cluster

#### Scheduler
- Identifies appropriate worker nodes for pod orchestration
- Assigns manifests to worker nodes based on resource requirements
- Makes scheduling decisions based on constraints and policies

#### Controllers
- Watch the cluster state and make changes to move current state toward desired state
- Include Deployment Controller, ReplicaSet Controller, etc.

### Data Plane Components

#### Worker Nodes
- **kubelet agent**: Downloads Docker images and runs containers
- **Container runtime**: Executes containers (Docker, containerd, etc.)
- **kube-proxy**: Manages network rules for service communication

### Client Communication
- Clients communicate securely with the API server using authentication tokens
- All cluster interactions go through the API server

## Deployment Options

### Self-Managed
- **On-Premises**: You manage both control plane and data plane
- Full control but requires significant operational overhead

### Platform-Managed Control Plane
- **Managed Kubernetes** (EKS, AKS, GKE): Platform manages control plane, you manage worker nodes
- Reduced operational overhead for control plane components

### Serverless
- **Fargate/ACI**: Both control plane and data plane managed by platform
- Pay-per-use model with minimal operational overhead

## Essential Commands

### Generic Syntax
```bash
kubectl get <resource>                    # List resources
kubectl apply -f <manifest-file>          # Create/update resources
kubectl describe <resource> <name>        # Get detailed information
kubectl delete <resource> <name>          # Delete resources
```

## Installation Requirements

### Required Tools
1. **AWS CLI**: [Installation Guide](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
2. **kubectl**: [Installation Guide](https://kubernetes.io/docs/tasks/tools/)

### Local Development Options
- **Minikube**: Single-node cluster for development
- **Docker Desktop**: Built-in Kubernetes support
- **Digital Ocean**: $200 credit for 2 months
- **Cloud Providers**: EKS, AKS, GKE

### Production Setup Tools
- **kubeadm**: For on-premises installations
- **eksctl**: AWS-specific cluster creation
- **Terraform**: Infrastructure as Code
- **CloudFormation**: AWS native IaC

## Cluster Setup with eksctl

### Create EKS Cluster
```bash
eksctl create cluster \
  --name eks-training \
  --region ap-south-1 \
  --nodegroup-name worker-nodes \
  --node-type t2.small \
  --nodes 3
```

## Step-by-Step Setup Guide

### 1. Install kubectl on Amazon Linux
```bash
# Update system packages
sudo yum -y update

# Download kubectl binary
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

# Install kubectl
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Verify installation
kubectl version

# Install git for repository management
sudo yum install -y git
```

### 2. Configure AWS Access
```bash
# Configure AWS profile
aws configure --profile eks-admin
# Enter: access key, secret key, region (ap-south-1), output (json)

# Verify configuration
cat ~/.aws/config
```

### 3. Setup AWS Profile for EKS
```bash
# Edit AWS config file
vi ~/.aws/config

# Add the following configuration:
[profile eks-admin]
region = ap-south-1
output = json
role_arn = arn:aws:iam::831955480324:role/eks-admin
source_profile = eks-admin
```

### 4. Connect to EKS Cluster
```bash
# Update kubeconfig
aws eks update-kubeconfig --region ap-south-1 --name eks-cluster --profile eks-admin

# Verify connection
kubectl get nodes
```

## Namespace Management

### Understanding Namespaces
Namespaces provide logical separation within a cluster, allowing multiple teams or applications to share the same cluster.

### Namespace Operations
```bash
# List existing namespaces
kubectl get ns

# Create a new namespace
kubectl create ns pradeep-ns

# Set default namespace for current context
kubectl config set-context --current --namespace pradeep-ns

# Delete a namespace (be careful - this deletes all resources in the namespace)
kubectl delete namespace <old-namespace>

# Check pods in current namespace
kubectl get pods
```

## Working with Pods

### Clone Workshop Repository
```bash
cd ~/
git clone https://gitlab.com/26-08-25-eks-workshop/k8s-manifests.git
cd k8s-manifests
```

### Pod Operations
```bash
# Create a pod from manifest
kubectl apply -f nginx-pod.yaml

# List pods
kubectl get pods

# Get detailed pod information
kubectl describe po nginx

# View pod logs
kubectl logs nginx

# Execute commands inside pod
kubectl exec -it nginx -- /bin/bash
curl http://localhost  # Test nginx inside container
exit

# Delete pod
kubectl delete po nginx
```

## Labels: The Kubernetes Glue

### Understanding Labels
- Key-value pairs that act as selectors to bind resources
- Can be attached to any Kubernetes resource
- Keys must be unique within a resource
- Used for resource selection and organization

### Label Operations
```bash
# View node labels
kubectl get nodes --show-labels

# Create pod and view labels
kubectl apply -f nginx-pod.yaml
kubectl get po --show-labels

# Add multiple labels to a pod
kubectl label po nginx-pod env=dev version=1.0.0 bu=bajaj-finserve tier=backend disk=ssd

# Update existing label (requires --overwrite)
kubectl label po nginx-pod version=2.0.0 --overwrite

# Filter pods by labels
kubectl get pods -l disk=ssd

# View pods across all namespaces
kubectl get pods --all-namespaces

# Filter pods across all namespaces by label
kubectl get pods --all-namespaces -l env=dev

# Remove a label (note the minus sign)
kubectl label po nginx-pod disk-
```

## ReplicaSets: Managing Pod Lifecycle

### Why ReplicaSets?
- Pods should not be managed directly in production
- ReplicaSets ensure desired number of pods are always running
- Provide self-healing and scaling capabilities
- Use labels as selectors to manage pods

### ReplicaSet Operations
```bash
# Apply ReplicaSet manifest
git pull origin
kubectl apply -f nginx-rs.yaml

# View ReplicaSet and pods
kubectl get rs
kubectl get pods
kubectl get pods --show-labels

# Test self-healing - delete a pod
kubectl delete po <pod-name>
kubectl get pods  # New pod should be created automatically

# Delete all pods (ReplicaSet will recreate them)
kubectl delete po --all
kubectl get pods

# Scale up
kubectl scale rs nginx-rs --replicas=4
kubectl get pods

# Scale down
kubectl scale rs nginx-rs --replicas=1
kubectl get pods

# Test label selector - change pod label
kubectl label po <pod-name> app=nginx-debug --overwrite
kubectl get pods  # New pod created as old one no longer matches selector

# Restore label
kubectl label po <pod-name> app=nginx --overwrite
kubectl get pods

# Clean up
kubectl delete rs nginx-rs
kubectl get rs
kubectl get pods  # All pods deleted with ReplicaSet
```

## Services: Network Abstraction

### Understanding Services
- Provide stable network endpoints for pods
- Create proxy with cluster IP for load balancing
- Three main types: ClusterIP, NodePort, LoadBalancer
- Use labels to select backend pods
- No direct relationship with ReplicaSets

### Service Operations
```bash
# Create service and ReplicaSet
git pull origin
kubectl apply -f nginx-svc.yaml
kubectl apply -f nginx-rs.yaml

# View resources
kubectl get svc
kubectl get rs
kubectl get pods

# Examine service details
kubectl describe svc nginx-svc

# Test service connectivity
kubectl exec -it <pod-name> -- /bin/bash
curl http://localhost                    # Direct pod access
curl http://<cluster-ip>                # Access via service (load balanced)
exit
```

### Microservice Communication
```bash
# Deploy microservices
kubectl apply -f k8s-microservice.yaml
kubectl apply -f k8s-inventory-microservice.yaml

# Verify deployment
kubectl get rs
kubectl get pods
kubectl get svc

# Test inter-service communication
kubectl exec -it <order-microservice-pod-name> -- /bin/bash
curl -vv http://inventory-microservice-svc/api/inventory
exit
```

## ConfigMaps: Application Configuration

### Understanding ConfigMaps
- Kubernetes resource for injecting application configuration
- Store key-value pairs (key: string, value: string or file content)
- Can be consumed as environment variables or mounted as files
- Provide configuration consistency across pods

### ConfigMap as Environment Variables
```bash
git pull origin
kubectl delete rs --all
kubectl apply -f k8s-microservice.yaml

# Verify resources
kubectl get rs
kubectl get pods
kubectl get cm

# Check environment variables in pod
kubectl exec -it <order-microservice-pod-name> -- /bin/bash
env | grep -i SPRING_*
exit
```

### ConfigMap as Volume Mount
```bash
git pull origin
kubectl apply -f k8s-microservice.yaml

# Verify resources
kubectl get cm
kubectl get pods

# Check mounted configuration files
kubectl exec -it <pod-name> -- /bin/bash
cd /app/config
ls
cat application-dev.yaml
cat application-qa.yaml
env | grep -i SPRING_*
exit

# Edit ConfigMap (changes will be reflected in mounted files)
kubectl edit cm order-microservice-cm
```

## Best Practices Summary

1. **Always use namespaces** for logical separation
2. **Use labels consistently** for resource organization
3. **Never manage pods directly** - use ReplicaSets or Deployments
4. **Use ConfigMaps** for application configuration
5. **Test connectivity** between services before deploying
6. **Monitor resource usage** and scale appropriately
