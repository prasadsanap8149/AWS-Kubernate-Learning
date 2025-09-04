# Day 4 - Ingress, Logging, and Access Control

## Overview
This document covers advanced Kubernetes networking with Ingress controllers, centralized logging with OpenSearch, and Role-Based Access Control (RBAC).

## Ingress: Application Gateway

### Understanding Ingress
Ingress acts as a reverse proxy and application gateway for Kubernetes clusters:
- **API Gateway**: Single entry point for external traffic
- **Layer 7 Load Balancing**: HTTP/HTTPS routing based on hostnames and paths
- **SSL Termination**: Handles TLS certificates and encryption
- **Path-based Routing**: Route requests to different services based on URL paths

### Ingress vs Ingress Controller
- **Ingress**: Kubernetes API resource defining routing rules
- **Ingress Controller**: Vendor-specific implementation that reads Ingress resources and configures the actual load balancer

## AWS Load Balancer Controller Setup

### Installation
```bash
# Install AWS Load Balancer Controller using Helm
helm upgrade -i aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=<EKS_CLUSTER_NAME> \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller
```

### Verification

#### Check Service Account
```bash
# Verify service account exists with proper IAM role
kubectl get sa aws-load-balancer-controller -n kube-system -o yaml
```

#### Verify Controller Pods
```bash
# List all pods in kube-system namespace
kubectl get pods -n kube-system

# Check specific controller pod status
kubectl get pods -n kube-system aws-load-balancer-controller-bf4945589-4qjpm 
```

## Application Deployment and Service Creation

### Deploy Microservice
```bash
# Update repository and deploy application
git pull origin
kubectl apply -f k8s-microservice.yaml

# Verify deployment
kubectl get deploy
kubectl get pods

# Test application internally
kubectl exec -it <pod-name> -- /bin/bash
curl http://localhost:8080/api/v1/orders        # Direct pod access
curl http://order-microservice-svc/api/v1/orders # Service access
exit
```

## Ingress Configuration and External Access

### Deploy Ingress Resource
```bash
# Apply Ingress configuration
kubectl apply -f k8s-microservice.yaml

# Verify Ingress creation
kubectl get ingress
kubectl describe ingress order-ingress
```

### DNS Configuration
```bash
# Get Load Balancer DNS name from Ingress description
# Look for 'Address' field in the output above

# Resolve load balancer IP addresses
nslookup k8s-<your-alb-name>

# Configure local DNS (requires root access)
sudo su -

# Add entries to hosts file (use IPs from nslookup)
echo "<ip-1> <your-name>.classpath.io" >> /etc/hosts
echo "<ip-2> <your-name>.classpath.io" >> /etc/hosts

# Verify hosts file
cat /etc/hosts
exit
```

### Test External Access
```bash
# Test application through Ingress
curl http://<your-name>.classpath.io/api/v1/orders
```

### Cleanup
```bash
# Remove Ingress resources
kubectl delete ingress --all
```

## Centralized Logging with OpenSearch

### Architecture Overview
- **Application Pods**: Generate logs to files
- **Fluent Bit**: Lightweight log processor and forwarder (sidecar container)
- **OpenSearch**: Centralized log storage and indexing
- **Dashboards**: Visualization and querying interface

### Sidecar Container Pattern
The sidecar pattern involves running a secondary container alongside the main application container:
- **Shared Volume**: Both containers share log directory
- **Log Collection**: Fluent Bit reads logs and forwards to OpenSearch
- **Isolation**: Each service has its own logging sidecar

### Deploy Application with Logging
```bash
# Deploy application with Fluent Bit sidecar
git pull origin
kubectl apply -f k8s-opensearch-order-microservice.yaml

# Verify pod deployment (should see multiple containers per pod)
kubectl get pods
```

### Test Logging
```bash
# Access the main application container
kubectl exec -it <pod-name> -c <main-container-name> -- /bin/bash

# Check log directory
ls /var/log/app/

# Generate some logs by making API calls
curl http://localhost:8080/api/v1/orders
exit

# Check Fluent Bit logs
kubectl logs -c fluent-bit <pod-name>
```

### OpenSearch Dashboard Access
```bash
# Access OpenSearch Dashboards in browser
# URL: https://search-demo-opensearch-x2g47sbaex7g34epqird2w6efee.aos.ap-south-1.on.aws/_dashboards/app/login?

# Login credentials:
# Username: admin
# Password: [provided separately]
```

### OpenSearch Dashboard Features
- **Discover**: Search and filter logs in real-time
- **Visualizations**: Create charts and graphs from log data
- **Dashboards**: Combine multiple visualizations
- **Alerts**: Set up notifications based on log patterns
- **Index Management**: Configure log retention and storage

## Role-Based Access Control (RBAC)

### Understanding RBAC
RBAC provides fine-grained access control for Kubernetes resources:
- **Subjects**: Users, Groups, or Service Accounts
- **Resources**: Kubernetes objects (pods, services, deployments, etc.)
- **Verbs**: Actions (get, list, create, update, delete, etc.)
- **Scope**: Namespace-level (Role) or cluster-level (ClusterRole)

### RBAC Components

#### Role
Defines permissions within a specific namespace:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: pradeep-ns
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
```

#### RoleBinding
Links a Role to subjects (users/service accounts):
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: pradeep-ns
subjects:
- kind: ServiceAccount
  name: pod-reader-sa
  namespace: pradeep-ns
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

### Implement RBAC
```bash
# Deploy RBAC configuration
git pull origin
kubectl apply -f k8s-role-binding.yaml

# Verify RBAC resources
kubectl get role
kubectl get rolebinding
kubectl get sa
```

### Test RBAC Permissions
```bash
# Test if user can list pods in namespace
kubectl auth can-i list pods --namespace=pradeep-ns --as=eks-admin

# Test if user can delete pods in namespace
kubectl auth can-i delete pods --namespace=pradeep-ns --as=eks-admin
```

### RBAC Best Practices

#### 1. Principle of Least Privilege
- Grant minimum necessary permissions
- Use namespace-scoped roles when possible
- Regularly audit permissions

#### 2. Service Account Strategy
```bash
# Create dedicated service accounts for applications
kubectl create serviceaccount app-service-account

# Bind specific roles to service accounts
kubectl create rolebinding app-binding \
  --role=app-role \
  --serviceaccount=default:app-service-account
```

#### 3. Testing Access
```bash
# Test permissions before deployment
kubectl auth can-i <verb> <resource> --as=<user> --namespace=<namespace>

# Examples:
kubectl auth can-i create pods --as=developer --namespace=dev
kubectl auth can-i delete deployments --as=developer --namespace=prod
```

## Troubleshooting Guide

### Ingress Issues
```bash
# Check Ingress controller pods
kubectl get pods -n kube-system | grep alb

# Verify Ingress configuration
kubectl describe ingress <ingress-name>

# Check controller logs
kubectl logs -n kube-system deployment/aws-load-balancer-controller
```

### Logging Issues
```bash
# Check if all containers in pod are running
kubectl get pods
kubectl describe pod <pod-name>

# Verify Fluent Bit configuration
kubectl logs -c fluent-bit <pod-name>

# Check log file permissions
kubectl exec -it <pod-name> -c <main-container> -- ls -la /var/log/app/
```

### RBAC Issues
```bash
# Check if role exists
kubectl get role -n <namespace>

# Verify role binding
kubectl describe rolebinding <binding-name> -n <namespace>

# Test permissions
kubectl auth can-i <verb> <resource> --as=<subject>
```

## Monitoring and Observability

### Key Metrics to Monitor
1. **Application Performance**
   - Response times
   - Error rates
   - Throughput

2. **Infrastructure Health**
   - CPU and memory usage
   - Network traffic
   - Storage utilization

3. **Security Events**
   - Failed authentication attempts
   - Unauthorized access attempts
   - Certificate expiration

### Log Analysis Best Practices
1. **Structured Logging**: Use JSON format for easier parsing
2. **Correlation IDs**: Track requests across services
3. **Log Levels**: Use appropriate levels (DEBUG, INFO, WARN, ERROR)
4. **Retention Policies**: Balance storage costs with compliance requirements

## Security Considerations

### Network Security
- Use TLS for all external communications
- Implement network policies to control pod-to-pod traffic
- Regular security scans of container images

### Access Control
- Regular RBAC audits
- Use strong authentication mechanisms
- Implement service mesh for advanced security features

### Monitoring and Alerting
- Set up alerts for security events
- Monitor unusual traffic patterns
- Track privileged access

This completes Day 4 content covering Ingress networking, centralized logging, and access control mechanisms in Kubernetes.

