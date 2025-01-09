# Kubernetes Demo Project

## Overview
This project demonstrates a basic Kubernetes deployment setup using k3d, a lightweight wrapper to run k3s (Rancher Lab's minimal Kubernetes distribution) in docker. It includes configuration for local development with a private registry.

## Prerequisites
- Docker Desktop
- kubectl CLI tool
- k3d installed
- Windows PowerShell or Command Prompt

## Installation & Setup

### 1. Install Required Tools
```powershell
# Install Chocolatey (Windows package manager) if not already installed
Set-ExecutionPolicy Bypass -Scope Process -Force; [System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072; iex ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))

# Install kubectl
choco install kubernetes-cli

# Install k3d
choco install k3d
```

### 2. Create a k3d Cluster with Registry
```powershell
# Create a new cluster with a local registry
k3d cluster create demo --registry-create mycluster-registry.localhost:5000
```

### 3. Configure Local Environment
Add the following entry to your hosts file (`C:\Windows\System32\drivers\etc\hosts`):
```
127.0.0.1 mycluster-registry.localhost
```

## Project Structure
```
kubernetes-demo/
├── Dockerfile
├── deployment.yaml
├── service.yaml
└── README.md
```

## Configuration Files

### Dockerfile
The Dockerfile uses a multi-stage build process to create an optimized production image:

```dockerfile
# Stage 1: Build
FROM node:19 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build

# Stage 2: Production
FROM node:19 AS runner
WORKDIR /app
COPY --from=builder /app/next.config.ts ./
COPY --from=builder /app/public ./public
COPY --from=builder /app/.next ./.next
COPY --from=builder /app/package*.json ./
RUN npm install --only=production
CMD ["npm", "start"]
```

### Deployment Configuration
The `deployment.yaml` file contains the Kubernetes deployment configuration:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kubernetes-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kubernetes-demo
  template:
    metadata:
      labels:
        app: kubernetes-demo
    spec:
      containers:
      - name: kubernetes-demo
        image: mycluster-registry.localhost:5000/kubernetes-demo:latest
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 3000
```

### Service Configuration
The `service.yaml` file exposes the application using NodePort:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: kubernetes-demo
spec:
  type: NodePort
  ports:
    - port: 3000
      targetPort: 3000
      nodePort: 30001
  selector:
    app: kubernetes-demo
```

## Usage

### 1. Build and Tag the Docker Image
```powershell
# Build the image
docker build -t kubernetes-demo:latest .

# Tag the image for local registry
docker tag kubernetes-demo:latest mycluster-registry.localhost:5000/kubernetes-demo:latest

# Push to local registry
docker push mycluster-registry.localhost:5000/kubernetes-demo:latest
```

### 2. Deploy to Kubernetes
```powershell
# Apply the deployment and service configurations
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml

# Verify the deployment and service
kubectl get deployments
kubectl get services
kubectl get pods
```

### 3. Accessing the Application
After deployment, the application will be available at:
```
http://localhost:30001
```

### 4. Monitoring and Troubleshooting
```powershell
# Check pod status
kubectl get pods

# View pod logs
kubectl logs [pod-name]

# Describe pod for detailed information
kubectl describe pod [pod-name]

# Check service status
kubectl get services
```

## Common Issues and Solutions

### Image Pull Errors
If you encounter `ErrImageNeverPull` or image pull issues:
1. Verify the registry is running:
   ```powershell
   docker ps | findstr registry
   ```
2. Ensure hosts file entry is correct
3. Confirm image is properly tagged and pushed to registry
4. Check `imagePullPolicy` in deployment.yaml

### Registry Connection Issues
If unable to push to registry:
1. Verify registry is running on port 5000
2. Check hosts file entry
3. Ensure Docker Desktop is running
4. Try restarting Docker Desktop

### Service Access Issues
If unable to access the application:
1. Verify the service is running: `kubectl get services`
2. Check if pods are running: `kubectl get pods`
3. Ensure port 30001 is not being used by another application
4. Try accessing the application using the node IP address

## Contributing
Feel free to submit issues and enhancement requests.

## License
[MIT License](LICENSE)