# Kubernetes and ArgoCD Setup Guide

This guide provides instructions for setting up a Kubernetes cluster and ArgoCD to deploy the webapp application.

## Prerequisites

- GitHub Codespaces or a Linux environment
- Docker installed
- Git installed

## Option 1: Using kind (Kubernetes in Docker)

### 1. Install kind

```bash
# For AMD64 / x86_64
[ $(uname -m) = x86_64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64
# For ARM64
[ $(uname -m) = aarch64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-arm64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
```

### 2. Install kubectl

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
```

### 3. Create kind cluster

```bash
# Create cluster configuration
cat << EOF > kind-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "ingress-ready=true"
  extraPortMappings:
  - containerPort: 80
    hostPort: 8080
    protocol: TCP
  - containerPort: 443
    hostPort: 8443
    protocol: TCP
EOF

# Create the cluster
kind create cluster --config=kind-config.yaml --name webapp-cluster
```

### 4. Verify cluster

```bash
kubectl cluster-info
kubectl get nodes
```

## Option 2: Using k3d (k3s in Docker)

### 1. Install k3d

```bash
curl -s https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | bash
```

### 2. Create k3d cluster

```bash
k3d cluster create webapp-cluster \
  --port "8080:80@loadbalancer" \
  --port "8443:443@loadbalancer" \
  --agents 2
```

### 3. Verify cluster

```bash
kubectl cluster-info
kubectl get nodes
```

## Installing ArgoCD

### 1. Create ArgoCD namespace and install

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### 2. Wait for ArgoCD to be ready

```bash
kubectl wait --for=condition=available --timeout=300s deployment/argocd-server -n argocd
```

### 3. Get ArgoCD admin password

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

### 4. Access ArgoCD UI

#### For kind cluster:
```bash
kubectl port-forward svc/argocd-server -n argocd 8081:443
```

#### For k3d cluster:
```bash
kubectl port-forward svc/argocd-server -n argocd 8081:443
```

Then access: https://localhost:8081 (accept the self-signed certificate)

> **Note:** ArgoCD UI and the webapp cannot both run on port 8080. Use port 8081 for both to avoid conflicts.

Login with:
- Username: `admin`
- Password: (from step 3)

## Installing NGINX Ingress Controller

### For kind:
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=90s
```

### For k3d:
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=90s
```

## Deploy the Application

### 1. Apply the ArgoCD Application

```bash
kubectl apply -f argo-app.yaml
```

### 2. Sync the application in ArgoCD UI

1. Go to ArgoCD UI (https://localhost:8080)
2. Login with admin credentials
3. Click on the "webapp" application
4. Click "SYNC" to deploy the application

### 3. Verify deployment

```bash
# Check if pods are running
kubectl get pods -n default

# Check services
kubectl get svc -n default

# Check ingress
kubectl get ingress -n default
```

### 4. Access the application

#### Via port-forward:
```bash
kubectl port-forward svc/webapp 8081:80
```
Then access: http://localhost:8081

> **Note:** Ensure you are not running ArgoCD UI and the webapp on the same port.

#### Via ingress (if configured):

If running on WSL 2, update your hosts file in both Windows and WSL environments:
```
127.0.0.1 webapp.local
```
Then access: http://webapp.local:8080

## Monitoring

### View application logs:
```bash
kubectl logs -f deployment/webapp
```

### Check application health:
```bash
kubectl get pods -l app=webapp
kubectl describe pod <pod-name>
```

### Access metrics endpoint:
```bash
kubectl port-forward svc/webapp 3000:80
curl http://localhost:3000/metrics
```

## Cleanup

### Remove the application:
```bash
kubectl delete -f argo-app.yaml
```

### Remove ArgoCD:
```bash
kubectl delete namespace argocd
```

### Remove the cluster:

#### For kind:
```bash
kind delete cluster --name webapp-cluster
```

#### For k3d:
```bash
k3d cluster delete webapp-cluster
```

## Troubleshooting

### Browser Compatibility

ArgoCD UI and the webapp are recommended for Chromium-based browsers (e.g., Brave) to avoid security certificate issues. Firefox may require strict permissions when running both interfaces.

### Debugging Tips

- Include screenshots of errors or browser issues when reporting problems.
- Check browser console for certificate warnings.
- Use `kubectl describe` and `kubectl logs` for deeper diagnostics.

### Common issues:

1. **Pods not starting**: Check logs with `kubectl logs <pod-name>`
2. **Image pull errors**: Ensure the image exists and is accessible
3. **Service not accessible**: Check service and ingress configuration
4. **ArgoCD sync issues**: Check the ArgoCD application status and events

### Useful commands:
```bash
# Get all resources
kubectl get all

# Describe a resource
kubectl describe <resource-type> <resource-name>

# Get events
kubectl get events --sort-by=.metadata.creationTimestamp

# Check ArgoCD application status
kubectl get applications -n argocd
