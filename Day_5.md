# Day 5 - Advanced Scheduling, Resource Management, and Package Management

## Overview
This document covers advanced Kubernetes scheduling mechanisms, resource management, health checks, and package management with Helm.

## Advanced Pod Scheduling

### Overview of Scheduling Options
Kubernetes provides several mechanisms to influence where pods are scheduled:

1. **Taints and Tolerations**: Control which pods can be scheduled on specific nodes
2. **NodeSelector**: Simple node selection based on labels
3. **Node Affinity/Anti-Affinity**: Advanced node selection with flexible rules
4. **Pod Affinity/Anti-Affinity**: Schedule pods relative to other pods

## Taints and Tolerations

### Understanding Taints and Tolerations
- **Taints**: Applied to nodes to repel pods that don't tolerate the taint
- **Tolerations**: Applied to pods to allow scheduling on tainted nodes
- **Use Cases**: Dedicated nodes, hardware-specific workloads, maintenance windows

### Taint Effects
- **NoSchedule**: Prevents new pods from being scheduled
- **PreferNoSchedule**: Tries to avoid scheduling pods (soft preference)
- **NoExecute**: Evicts existing pods that don't tolerate the taint

### Lab: Taints and Tolerations

#### Apply Taints to Nodes
```bash
# List all nodes
kubectl get nodes

# Apply taint to a specific node
kubectl taint node <node-name> name=your-name:NoSchedule

# Try to schedule a regular pod (will fail due to taint)
kubectl run nginx --image nginx
kubectl get po
kubectl describe po nginx
```

#### Deploy Pod with Tolerations
```bash
# Deploy pod that tolerates the taint
git pull origin
kubectl apply -f taints-demo.yaml

# Verify pod is scheduled on tainted node
kubectl get pods -o wide
```

#### Remove Taints
```bash
# Remove taint from node (note the minus sign at the end)
kubectl taint node <node-name> name=your-name:NoSchedule-
```

## NodeSelector: Label-Based Scheduling

### Understanding NodeSelector
- Simplest way to constrain pods to nodes with specific labels
- Uses key-value label matching
- Pod only schedules if node has matching labels

### Lab: NodeSelector

#### Clean Environment and Apply Configuration
```bash
# Remove existing pods
kubectl delete po --all

# Apply NodeSelector configuration
git pull origin
kubectl apply -f taints-demo.yaml

# Check pod scheduling
kubectl get pods
kubectl describe po nginx

# Verify node has required label
kubectl get node <node-name> --show-labels | grep -i disk=ssd
```

#### Add Required Label to Node
```bash
# Add label to node if missing
kubectl label node <node-name> disk=ssd

# Verify pod can now be scheduled
kubectl get pods
```

## Pod and Node Affinity/Anti-Affinity

### Node Affinity
More expressive than NodeSelector with support for:
- **Required**: Hard requirements (requiredDuringSchedulingIgnoredDuringExecution)
- **Preferred**: Soft preferences (preferredDuringSchedulingIgnoredDuringExecution)
- **Operators**: In, NotIn, Exists, DoesNotExist, Gt, Lt

### Pod Affinity/Anti-Affinity
Schedule pods relative to other pods:
- **Pod Affinity**: Schedule pods near other pods (same zone, same node)
- **Pod Anti-Affinity**: Schedule pods away from other pods (different zones, different nodes)

### Use Cases
- **High Availability**: Spread replicas across zones
- **Performance**: Co-locate related services
- **Security**: Isolate sensitive workloads

## Health Checks: Liveness and Readiness Probes

### Understanding Probes
- **Liveness Probe**: Determines if container needs to be restarted
- **Readiness Probe**: Determines if container is ready to receive traffic
- **Startup Probe**: Gives slow-starting containers time to initialize

### Probe Types
1. **HTTP GET**: Check HTTP endpoint
2. **TCP Socket**: Check if port is open
3. **Exec**: Run command inside container

### Lab: Health Checks
```bash
# Deploy application with health checks
git pull origin
kubectl apply -f liveness-readiness.yaml

# Monitor pod status
kubectl get pods

# Simulate application failure
kubectl exec -it liveness-nginx -- /bin/bash
rm -rf /usr/share/nginx/html/index.html
exit

# Watch pod restart due to failed liveness probe
kubectl get pods liveness-nginx --watch
```

### Health Check Best Practices
1. **Separate Concerns**: Different endpoints for liveness and readiness
2. **Lightweight Checks**: Avoid expensive operations in probes
3. **Appropriate Timeouts**: Balance responsiveness with false positives
4. **Dependency Checks**: Include critical dependencies in readiness probes

## Resource Management

### Understanding Resource Requests and Limits
- **Requests**: Minimum resources guaranteed to container
- **Limits**: Maximum resources container can consume
- **Quality of Service (QoS)**: Determined by requests and limits configuration

### Resource Types
- **CPU**: Measured in millicores (m) or cores
- **Memory**: Measured in bytes with suffixes (Ki, Mi, Gi, Ti, Pi, Ei)

### Resource Configuration Example
```yaml
resources:
  limits:
    cpu: 250m      # 0.25 CPU cores
    memory: 500Mi  # 500 Mebibytes
  requests:
    cpu: 100m      # 0.1 CPU cores  
    memory: 200Mi  # 200 Mebibytes
```

### Resource Units
```bash
# CPU Units
# 1 CPU = 1000m (millicores)
# 500m = 0.5 CPU cores
# 100m = 0.1 CPU cores

# Memory Units
# 1 KiB = 1024 bytes
# 1 MiB = 1024 KiB = 1,048,576 bytes
# 1 GiB = 1024 MiB
# 1 TiB = 1024 GiB
# 1 PiB = 1024 TiB
# 1 EiB = 1024 PiB
```

### Quality of Service Classes
1. **Guaranteed**: Requests = Limits for all containers
2. **Burstable**: Some containers have requests < limits
3. **BestEffort**: No requests or limits specified

## Horizontal Pod Autoscaler (HPA)

### Understanding HPA
Automatically scales deployment based on resource utilization:
- **CPU Utilization**: Scale based on CPU usage percentage
- **Memory Utilization**: Scale based on memory usage
- **Custom Metrics**: Scale based on application-specific metrics

### Lab: HPA Setup
```bash
# Create deployment
kubectl create deploy nginx-deployment --replicas=2 --image=nginx

# Configure autoscaler
kubectl autoscale deployment nginx-deployment --cpu-percent=50 --min=1 --max=5

# Verify HPA
kubectl get hpa
```

### HPA Best Practices
1. **Set Appropriate Targets**: Avoid constant scaling
2. **Consider Scale-up/Scale-down Delays**: Prevent thrashing
3. **Monitor Metrics**: Ensure metrics are available and accurate
4. **Test Thoroughly**: Validate scaling behavior under load

## Helm: Kubernetes Package Manager

### Understanding Helm
- **Package Manager**: Like apt, yum, or npm for Kubernetes
- **Charts**: Packages containing Kubernetes manifests
- **Templates**: Parameterized Kubernetes manifests
- **Releases**: Installed instances of charts

### Helm Benefits
- **Reusability**: Share common application patterns
- **Parameterization**: Customize deployments for different environments
- **Versioning**: Track and manage application versions
- **Rollbacks**: Easy rollback to previous versions

### Helm Installation
```bash
# Download and install Helm
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh

# Verify installation
helm version
```

### Working with Helm Repositories
```bash
# Add popular chart repositories
helm repo add stable https://charts.helm.sh/stable
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx

# Update repository information
helm repo update

# List configured repositories
helm repo list
```

### Installing Charts
```bash
# Install nginx from Bitnami repository
helm install my-custom-nginx bitnami/nginx

# Verify installation
kubectl get deploy
kubectl get rs
kubectl get pod
kubectl get sa
kubectl get svc
kubectl get cm
```

### Managing Helm Releases
```bash
# List installed releases
helm list

# Get release status
helm status my-custom-nginx

# Upgrade release
helm upgrade my-custom-nginx bitnami/nginx --set replicaCount=3

# Rollback release
helm rollback my-custom-nginx 1

# Uninstall release
helm uninstall my-custom-nginx
```

### Exploring Charts
```bash
# Download and extract chart for examination
helm pull bitnami/nginx --untar
cd nginx
ls

# Chart structure:
# Chart.yaml     - Chart metadata
# values.yaml    - Default configuration values
# templates/     - Kubernetes manifest templates
# charts/        - Dependency charts
```

## Creating Custom Helm Charts

### Chart Structure
```
mychart/
├── Chart.yaml          # Chart metadata
├── values.yaml         # Default values
├── templates/          # Kubernetes manifests
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   └── _helpers.tpl    # Template helpers
└── charts/             # Chart dependencies
```

### Lab: Custom Microservice Chart
```bash
# Clone example microservice chart
git clone https://github.com/pradeepkl/order-microservices-helm.git
cd order-microservices-helm

# Examine chart structure
ls -la

# Install custom chart
helm install order-microservice . -f values.yaml

# Verify deployment
kubectl get all
```

### Helm Templating
```yaml
# Example template with values
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Values.app.name }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Values.app.name }}
  template:
    metadata:
      labels:
        app: {{ .Values.app.name }}
    spec:
      containers:
      - name: {{ .Values.app.name }}
        image: {{ .Values.image.repository }}:{{ .Values.image.tag }}
        ports:
        - containerPort: {{ .Values.service.port }}
```

## Best Practices Summary

### Scheduling
1. **Use Taints and Tolerations** for dedicated nodes
2. **Implement Anti-Affinity** for high availability
3. **Consider Resource Requirements** when placing pods
4. **Test Scheduling Constraints** before production deployment

### Resource Management
1. **Always Set Resource Requests** for predictable scheduling
2. **Set Appropriate Limits** to prevent resource exhaustion
3. **Monitor Resource Usage** and adjust as needed
4. **Use HPA** for dynamic scaling based on load

### Health Checks
1. **Implement Both Probes** (liveness and readiness)
2. **Use Lightweight Checks** to avoid overhead
3. **Set Appropriate Timeouts** and thresholds
4. **Test Failure Scenarios** to validate probe behavior

### Helm Usage
1. **Use Established Charts** when possible
2. **Customize via Values** rather than forking charts
3. **Version Your Charts** for reproducible deployments
4. **Test Charts** in staging before production
5. **Document Custom Values** for team collaboration

### Security Considerations
1. **Limit Resource Consumption** to prevent DoS attacks
2. **Use Pod Security Policies** or Pod Security Standards
3. **Regularly Update Charts** for security patches
4. **Audit Chart Dependencies** for vulnerabilities

This completes Day 5 content covering advanced scheduling, resource management, and package management with Helm in Kubernetes.
