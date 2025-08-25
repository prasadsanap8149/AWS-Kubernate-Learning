# Day 1 - AWS IAM, EC2, and Docker Fundamentals

## Overview
This document covers the setup of AWS IAM users and groups, EC2 instance configuration, Docker installation and basic operations, and container registry management.

---

## Lab 1: AWS IAM Setup

### Objective
Create IAM users and groups in AWS with appropriate permissions for EKS training.

### Services Used
- **IAM (Identity and Access Management)** - AWS equivalent of Azure Entra ID

### Steps

#### 1. Create IAM Group
1. Navigate to AWS Console: `https://classpathio.signin.aws.amazon.com/console`
2. Go to IAM service
3. Create a group called `eks-training`
4. Assign the following permissions:
   - `EC2FullAccess` - Full access to EC2 services
   - `IAMFullAccess` - Full access to IAM services

#### 2. Create IAM User
1. Create IAM user in the `eks-training` group
2. Enable Multi-Factor Authentication (MFA) for enhanced security
3. Assign IAM permissions:
   - `IAMFullAccess`
   - `EC2FullAccess`

---

## Lab 2: EC2 Setup and Docker Installation

### Prerequisites
- **Required Permission**: `EC2FullAccess` to create and manage EC2 instances

### Docker Installation on Amazon Linux 2023

```bash
# Update the system packages
sudo dnf update -y
# Explanation: Updates all installed packages to their latest versions

# Install Docker
sudo dnf install -y docker
# Explanation: Installs Docker engine from the default repository

# Start Docker service
sudo systemctl start docker
# Explanation: Starts the Docker daemon service immediately

# Enable Docker to start automatically on boot
sudo systemctl enable docker
# Explanation: Configures Docker to start automatically when the system boots

# Verify Docker installation (requires sudo initially)
sudo docker info
# Explanation: Displays Docker system information to verify installation

# Add ec2-user to docker group (eliminates need for sudo)
sudo usermod -aG docker ec2-user
# Explanation: Adds the ec2-user to the docker group for non-sudo access
# Note: Requires logout/login or browser reconnection to take effect

# Verify Docker works without sudo (after reconnection)
docker info
# Explanation: Verifies Docker access without sudo privileges
```

---

## Docker Commands Reference

### Basic Docker Operations

#### Image Management
```bash
# List all local Docker images
docker images
# Explanation: Shows all downloaded/built images with repository, tag, image ID, creation date, and size

# Download an image from Docker Hub
docker pull hello-world
# Explanation: Downloads the hello-world image from the default registry (Docker Hub)

# Download nginx web server image
docker pull nginx
# Explanation: Downloads the latest nginx image for web server functionality
```

#### Container Operations
```bash
# Run a container from an image (interactive mode)
docker container run hello-world
# Explanation: Creates and runs a new container from hello-world image, shows output and exits

# List currently running containers
docker container ls
# Explanation: Shows only active/running containers with their details

# List all containers (running and stopped)
docker container ls --all
# Explanation: Shows all containers regardless of their current state

# Run container in detached mode (background)
docker container run -d nginx
# Explanation: Runs nginx container in background, returns container ID immediately

# View container logs
docker container logs <container-id>
# Explanation: Displays the logs/output from the specified container
# Replace <container-id> with actual container ID from 'docker container ls'

# Execute commands inside a running container
docker container exec -it <container-id> /bin/bash
# Explanation: Opens an interactive bash shell inside the running container
# -i: interactive mode, -t: allocate pseudo-TTY

# Test nginx from inside container
curl http://localhost
# Explanation: Tests if nginx is serving content on localhost (inside container)

# Exit from container shell
exit
# Explanation: Exits the interactive shell, returns to host system
```

#### Container Lifecycle Management
```bash
# Stop a running container
docker container stop <container-id>
# Explanation: Gracefully stops the specified container (sends SIGTERM signal)

# Start a stopped container
docker container start <container-id>
# Explanation: Restarts a previously stopped container

# Restart a container
docker container restart <container-id>
# Explanation: Stops and starts the container in one command

# Remove a stopped container
docker container rm <container-id>
# Explanation: Permanently deletes the specified container (must be stopped first)

# Remove Docker images
docker image rmi -f nginx
# Explanation: Force removes the nginx image from local storage
# -f: force removal even if containers are using the image

docker image rmi -f hello-world
# Explanation: Force removes the hello-world image
```

---

## Docker Concepts

### Imperative vs Declarative Approaches

#### Imperative Approach
- **Definition**: Define what you need AND how it should be implemented
- **Characteristics**: 
  - More control over the process
  - Allows extensive customization
  - Step-by-step instructions
- **Example**: Using individual `docker run` commands with specific parameters

#### Declarative Approach
- **Definition**: Define what you need and let the system figure out how to implement it
- **Characteristics**:
  - Less control but simpler to manage
  - System handles implementation details
  - Configuration-based approach
- **Example**: Using Docker Compose files or Kubernetes manifests

---

## Dockerfile Best Practices

### Key Principles
- **File Name**: Must be named `Dockerfile` (no extension)
- **First Command**: Always start with `FROM` instruction
- **Entry Point**: Use `ENTRYPOINT` to define the main process
- **Location**: Place Dockerfile in the project root directory
- **Version Control**: Commit Dockerfile to repository for repeatability

### Dockerfile Structure
```dockerfile
# Base image (must be first instruction)
FROM <base-image>

# Set working directory
WORKDIR /app

# Copy dependency files first (for better caching)
COPY package.json ./

# Install dependencies
RUN npm install

# Copy application code
COPY . .

# Expose port
EXPOSE 3000

# Define the main process
ENTRYPOINT ["node", "app.js"]
```

### Optimization Tips
- **Layer Caching**: Frequently changing instructions should be placed at the bottom
- **Build Efficiency**: Once a layer changes, all subsequent layers are rebuilt
- **Best Practice**: Keep stable instructions (like dependency installation) at the top

---

## Lab Exercises

### Lab Exercise 1: Basic Docker Application

```bash
# Clone the lab repository
sudo dnf -y install git
# Explanation: Installs Git version control system

git clone https://gitlab.com/classpath-docker/docker-lab.git
# Explanation: Downloads the lab exercises repository

cd docker-lab/01-hello-world
# Explanation: Navigate to the first exercise directory

# View the Dockerfile
cat Dockerfile
# Explanation: Display the contents of the Dockerfile

# Build custom image
docker build -t custom-hello-world .
# Explanation: Builds Docker image from current directory's Dockerfile
# -t: tags the image with name 'custom-hello-world'
# .: specifies current directory as build context

# Verify image creation
docker images
# Explanation: Confirm the new image appears in local registry

# Run the custom container
docker container run custom-hello-world
# Explanation: Execute the custom-built container
```

### Lab Exercise 2: Web Application

```bash
cd ../02-hello-web
# Explanation: Move to web application exercise

# Build web application image
docker build -t hello-web .
# Explanation: Creates image for web application

# Run web container in background
docker container run -d hello-web
# Explanation: Starts web server container in detached mode

# Test web application from inside container
docker container exec -it <container-id> curl http://localhost
# Explanation: Tests web server accessibility from within container
```

### Lab Exercise 3: Express.js CRUD API

```bash
cd ../03-express-crud-app
# Explanation: Navigate to Express.js application directory

# Build Express API image
docker build -t items-api .
# Explanation: Creates image for Node.js Express application

# Run API container
docker container run -d items-api
# Explanation: Starts API server in background

# Test API endpoint
docker container exec -it <container-id> curl http://localhost:3000/items
# Explanation: Tests the REST API endpoint for items
```

### Lab Exercise 4: Spring Boot Multi-stage Build

```bash
# Clone Spring Boot microservice
cd ~/
git clone https://gitlab.com/12-12-vmware-microservices/orders-microservice.git
cd orders-microservice

# Pull latest changes
git pull origin
# Explanation: Ensures repository is up to date

# Build Spring Boot application
docker build -t orders-microservice .
# Explanation: Builds Spring Boot application using multi-stage Dockerfile

# Test the application
docker container exec -it <container-id> curl http://localhost:8080/api/v1/orders
# Explanation: Tests Spring Boot REST API endpoint

# Check Java version inside container
docker container exec -it <container-id> java -version
# Explanation: Verifies Java runtime version in container
```

### Important Fix for Spring Boot Dockerfile
```dockerfile
# Correct ENTRYPOINT for Spring Boot applications
ENTRYPOINT ["java", "org.springframework.boot.loader.launch.JarLauncher"]
```

---

## Container Registry Operations

### Overview
Container registries store and distribute Docker images:
- **Public Registry**: Docker Hub (docker.io) - no credentials needed for pulling
- **Private Registry**: AWS ECR, Azure ACR - requires authentication

### Docker Image Naming Convention
```
<registry-url>/<repository-name>/<image-name>:<tag>
```
- **registry-url**: Defaults to `docker.io` if not specified
- **repository-name**: Organization or user namespace
- **image-name**: Application name
- **tag**: Version identifier (defaults to `latest`)

### AWS ECR Push Process

#### 1. Configure AWS CLI
```bash
aws configure
# Prompts for:
# Access Key ID: Your AWS access key
# Secret Access Key: Your AWS secret key
# Default region: ap-south-1 (Mumbai region)
# Default output format: json
```

#### 2. Authenticate with ECR
```bash
aws ecr get-login-password --region ap-south-1 | docker login --username AWS --password-stdin 831955480324.dkr.ecr.ap-south-1.amazonaws.com
# Explanation: 
# - Gets temporary login token from ECR
# - Pipes it to docker login for authentication
# - 831955480324 is the AWS account ID
# - ap-south-1 is the AWS region
```

#### 3. Tag Image for ECR
```bash
docker tag orders-microservice 831955480324.dkr.ecr.ap-south-1.amazonaws.com/eks-training/order-microservice:pradeep
# Explanation:
# - Tags local image 'orders-microservice' with ECR repository path
# - Repository: eks-training/order-microservice
# - Tag: pradeep (custom version identifier)
```

#### 4. Push to ECR
```bash
docker push 831955480324.dkr.ecr.ap-south-1.amazonaws.com/eks-training/order-microservice:pradeep
# Explanation: Uploads the tagged image to AWS ECR repository
```

---

## Key Takeaways

1. **IAM Security**: Always use least privilege principle and enable MFA
2. **Docker Efficiency**: Optimize Dockerfile layer ordering for better build performance
3. **Container Management**: Use appropriate lifecycle commands for different scenarios
4. **Registry Operations**: Understand authentication requirements for different registry types
5. **Best Practices**: Follow naming conventions and version control for reproducible builds

---

## Useful References

- [Docker Command Reference](https://docs.docker.com/reference/dockerfile/)
- [Lab Repository](https://gitlab.com/classpath-docker/docker-lab)
- [AWS ECR Documentation](https://docs.aws.amazon.com/ecr/)
- [Docker Best Practices](https://docs.docker.com/develop/dev-best-practices/)











