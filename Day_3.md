# Day 3 - Security, Secrets Management, and Deployments

## Overview
This document covers advanced Kubernetes concepts including Sealed Secrets, private image registries, Deployments, and AWS Secrets Manager integration.

## Initial Setup

### AWS Configuration
```bash
# Configure AWS CLI with default profile
aws configure

# Add EKS admin profile configuration
echo "[profile eks-admin]" >> ~/.aws/config
echo "region = ap-south-1" >> ~/.aws/config
echo "output = json" >> ~/.aws/config 
echo "role_arn = arn:aws:iam::831955480324:role/eks-admin" >> ~/.aws/config
echo "source_profile = default" >> ~/.aws/config 

# Update kubeconfig for EKS cluster
aws eks update-kubeconfig --region ap-south-1 --name eks-cluster --profile eks-admin

# Verify cluster connection
kubectl get nodes

# Create and set namespace
kubectl create ns pradeep-ns
kubectl config set-context --current --namespace pradeep-ns
kubectl get pods
```

## Sealed Secrets: Secure Secret Management

### What are Sealed Secrets?
Sealed Secrets provide a way to encrypt secrets that can be stored in Git repositories safely. The controller decrypts them inside the cluster.

### Benefits
- **GitOps Friendly**: Encrypted secrets can be stored in version control
- **Security**: Only the cluster can decrypt the secrets
- **Automation**: Integrates with CI/CD pipelines

### Installation and Setup

#### 1. Verify kubeseal CLI
```bash
# Check if kubeseal is installed
kubeseal --version
```

#### 2. Install Sealed Secrets Controller
```bash
# Install the controller in kube-system namespace
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.31.0/controller.yaml

# Reference: https://github.com/bitnami-labs/sealed-secrets/releases
```

#### 3. Verify Controller Installation
```bash
# Check if controller pod is running
kubectl get pods -n kube-system | grep sealed-secrets-controller

# View controller logs
kubectl logs -n kube-system sealed-secrets-controller-86865f5c75-j7rxv 
```

#### 4. Extract Public Key Certificate
```bash
# Fetch the public key from the controller
kubeseal --fetch-cert > public-key-cert.pem
```

### Working with Repository
```bash
# Clone the workshop repository
git clone https://gitlab.com/26-08-25-eks-workshop/k8s-manifests.git
cd k8s-manifests
git pull origin

# Fetch the public key certificate
kubeseal --fetch-cert > public-key-cert.pem
```

### Creating Sealed Secrets
```bash
# Encrypt a regular secret using the public key
kubeseal --cert=public-key-cert.pem --format=yaml < db-credentials.yaml > sealed-db-credentials.yaml

# View the encrypted secret
cat sealed-db-credentials.yaml
```

## Private Container Registry Access

### The Challenge
When using private Docker images, Kubernetes needs credentials to pull images from the registry.

### Creating Registry Credentials
```bash
# Create Docker registry secret
kubectl create secret docker-registry docker-credentials \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=classpathio \
  --docker-password=Welcome444 \
  --docker-email=pradeep.kumar44@gmail.com \
  --dry-run=client -o yaml
```

### Deploying with Private Images
```bash
# Update repository and deploy
git pull origin
kubectl apply -f k8s-microservice.yaml

# Check deployment status
kubectl get pods
kubectl describe po <pod-name>

# Verify secrets
kubectl get secrets
kubectl get secret docker-credentials
```

### Service Account Integration

#### Create Service Account
```bash
# Create service account for Docker operations
kubectl create sa docker-sa --dry-run=client -o yaml
```

#### Link Secret to Service Account
The service account can be configured to automatically use the Docker credentials for image pulls.

```bash
# Apply updated configuration
git pull origin 
kubectl delete rs --all
kubectl apply -f k8s-microservice.yaml

# Verify pod status
kubectl get pods
kubectl describe po <pod-name>
```

## Deployments: Production-Ready Pod Management

### Why Deployments?
- **Rolling Updates**: Update applications without downtime
- **Rollback Capability**: Revert to previous versions
- **Version History**: Track deployment changes
- **Scaling**: Horizontal pod scaling
- **Self-Healing**: Automatic pod replacement

### Deployment Operations

#### Basic Deployment
```bash
# Deploy application
git pull origin
kubectl apply -f k8s-microservice.yaml

# Monitor deployment
kubectl get deploy
kubectl rollout status deploy order-microservice-deployment

# View associated resources
kubectl get rs    # ReplicaSets created by deployment
kubectl get pods  # Pods managed by ReplicaSet
```

#### Deployment History
```bash
# View rollout history
kubectl rollout history deploy order-microservice-deployment
```

#### Rollback Operations
```bash
# Rollback to previous version
kubectl rollout undo deploy order-microservice-deployment

# Pause deployment (stops rollout)
kubectl rollout pause deploy order-microservice-deployment

# Resume paused deployment
kubectl rollout resume deploy order-microservice-deployment
```

#### Cleanup
```bash
# Delete deployment (also deletes ReplicaSets and Pods)
kubectl delete deploy order-microservice-deployment

# Verify cleanup
kubectl get deploy
kubectl get rs
kubectl get po
```

### Lab Exercise: Complete Deployment Lifecycle
```bash
# Clean slate
kubectl delete deploy --all

# Deploy fresh application
git pull origin
kubectl apply -f k8s-microservice.yaml

# Monitor deployment process
kubectl get deploy
kubectl get rs
kubectl get pods
kubectl rollout status deploy order-microservice-deployment

# Check deployment history
kubectl rollout history deploy order-microservice-deployment

# Make changes and redeploy
git pull origin
kubectl apply -f k8s-microservice.yaml

# Monitor updated deployment
kubectl get deploy
kubectl get rs
kubectl get pods
kubectl rollout status deploy order-microservice-deployment
kubectl rollout history deploy order-microservice-deployment
```

## Persistent Storage with Dynamic Provisioning

### Storage Classes
Storage Classes define different types of storage that can be dynamically provisioned.

```bash
# Apply storage class configuration
git pull origin
kubectl apply -f ebs-storage-class.yaml

# Verify storage resources
kubectl get sc   # Storage Classes
kubectl get pv   # Persistent Volumes
kubectl get pvc  # Persistent Volume Claims
kubectl get pods # Pods using storage
```

### Testing Persistent Storage
```bash
# Access pod and verify mounted storage
kubectl exec -it <pod-name> -- /bin/bash
df -k           # Check disk usage
cd /data/db     # Navigate to mounted directory
ls              # List contents
exit

# Cleanup storage resources
kubectl delete deploy --all
kubectl delete pvc --all
```

## AWS Secrets Manager Integration

### What is AWS Secrets Manager?
AWS Secrets Manager helps protect access to applications, services, and IT resources without the upfront cost and complexity of managing your own hardware security module (HSM) infrastructure.

### Benefits
- **Centralized Management**: Single place for all secrets
- **Automatic Rotation**: Built-in rotation for supported services
- **Fine-grained Access**: IAM-based access control
- **Audit Trail**: CloudTrail integration for compliance

### Implementation
```bash
# Deploy AWS Secrets Manager integration
git pull origin
kubectl apply -f nginx-aws-secrets.yaml

# Verify resources
kubectl get secrets                    # Kubernetes secrets
kubectl get secretproviderclass       # AWS Secrets Manager provider
kubectl get pods                      # Application pods

# Check pod details
kubectl describe po nginx-secrets-demo
```

### Testing Secret Access
```bash
# Access pod and verify environment variables
kubectl exec -it <pod-name> -- /bin/bash
env | grep -i OPEN*  # Check for injected secrets
exit
```

## Security Best Practices

### 1. Secret Management
- **Never store secrets in plain text** in configuration files
- **Use Sealed Secrets** for GitOps workflows
- **Integrate with external secret stores** like AWS Secrets Manager
- **Rotate secrets regularly**

### 2. Image Security
- **Use private registries** for proprietary applications
- **Scan images** for vulnerabilities
- **Use minimal base images** to reduce attack surface
- **Keep images updated** with latest security patches

### 3. Access Control
- **Use Service Accounts** for pod-level permissions
- **Implement RBAC** for user access control
- **Follow principle of least privilege**
- **Regular access reviews**

### 4. Network Security
- **Use Network Policies** to control traffic flow
- **Implement ingress controllers** for external access
- **Use TLS** for encrypted communication
- **Segment networks** appropriately

## Troubleshooting Common Issues

### Secret Access Issues
```bash
# Check secret existence
kubectl get secrets

# Verify secret content (base64 encoded)
kubectl get secret <secret-name> -o yaml

# Check pod events for secret mounting issues
kubectl describe pod <pod-name>
```

### Image Pull Issues
```bash
# Check image pull secrets
kubectl get secrets | grep docker

# Verify service account configuration
kubectl describe sa <service-account-name>

# Check pod events for image pull errors
kubectl describe pod <pod-name>
```

### Deployment Issues
```bash
# Check deployment status
kubectl get deploy
kubectl describe deploy <deployment-name>

# Monitor rollout progress
kubectl rollout status deploy <deployment-name>

# Check ReplicaSet events
kubectl describe rs <replicaset-name>
```

This completes Day 3 content covering security, secrets management, and production-ready deployments in Kubernetes.
