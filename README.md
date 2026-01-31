# Go Web App DevOps

A complete DevOps pipeline for a Go web application with Kubernetes deployment, including local development setup and AWS EKS deployment instructions.

## Project Structure
```
go-web-app-devops/
├── k8s/
│   └── manifests/
│       ├── deployment.yml
│       ├── service.yml
│       └── ingress.yml
├── Dockerfile
├── main.go
└── README.md
```

## Prerequisites
- Docker
- Kubernetes (Minikube or Kind for local)
- kubectl
- AWS CLI (for AWS deployment)
- Helm (optional)

## Quick Start

### 1. Local Development with Kind

```bash
# Clone the repository
git clone https://github.com/Aditya-das-4707-e/go-web-app-devops.git
cd go-web-app-devops

# Build the Docker image
docker build -t go-web-app:v1 .

# Create Kind cluster
kind create cluster --name go-web-app-cluster

# Load image into Kind
kind load docker-image go-web-app:v1 --name go-web-app-cluster

# Deploy to Kubernetes
kubectl apply -f k8s/manifests/
```

### 2. Access the Application
For local development, use port-forwarding:

```bash
# Forward Ingress controller port
kubectl port-forward -n ingress-nginx service/ingress-nginx-controller 8080:80

# Access the application
curl -H "Host: go-web-app.local" http://localhost:8080/
# Or open browser: http://go-web-app.local:8080
```

## Common Issues & Troubleshooting

### Issue 1: Ingress Not Working on Local Kind Cluster
**Problem:** Ingress shows `<pending>` for external IP and NodePort is not accessible.

**Root Cause:** Kind cluster doesn't provide a cloud LoadBalancer. NodePorts aren't automatically mapped to localhost.

**Solutions:**

**Solution A: Use Port-Forwarding (Recommended for Local Dev)**
```bash
kubectl port-forward -n ingress-nginx service/ingress-nginx-controller 8080:80
```
Access at: http://go-web-app.local:8080

**Solution B: Configure Kind with Port Mapping**

Create `kind-config.yaml`:
```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    protocol: TCP
  - containerPort: 443
    hostPort: 443
    protocol: TCP
```

Create cluster:
```bash
kind delete cluster
kind create cluster --config kind-config.yaml
```

**Solution C: Change Ingress Controller to NodePort**
```bash
kubectl patch svc ingress-nginx-controller -n ingress-nginx -p '{"spec": {"type": "NodePort"}}'
NODE_PORT=$(kubectl get svc -n ingress-nginx ingress-nginx-controller -o jsonpath='{.spec.ports[0].nodePort}')
curl -H "Host: go-web-app.local" http://localhost:$NODE_PORT/
```

### Issue 2: Service Has No Endpoints
**Problem:** Service shows no active endpoints even though Pod is running.

**Root Cause:** Service selector doesn't match Pod labels.

**Solution:** Ensure labels match in both Service and Deployment:

`service.yml`:
```yaml
spec:
  selector:
    app: go-web-app  # Must match Pod labels
```

`deployment.yml`:
```yaml
template:
  metadata:
    labels:
      app: go-web-app  # Must match Service selector
```

Verify with:
```bash
kubectl get endpoints go-web-app
kubectl describe service go-web-app
kubectl describe pod <pod-name>
```

### Issue 3: Application Returns 404
**Problem:** Application responds but returns "404 page not found".

**Root Cause:** The Go application doesn't handle the root path `/` or needs specific endpoints.

**Solution:**

Check what endpoints your app supports:
```bash
kubectl port-forward pod/<pod-name> 8080:8080
curl http://localhost:8080/health
curl http://localhost:8080/api
curl http://localhost:8080/hello
```

Update Ingress to match your app's endpoints:
```yaml
paths:
- path: /api  # Change to match your app
  pathType: Prefix
  backend:
    service:
      name: go-web-app
      port:
        number: 80
```

### Issue 4: Connection Refused on NodePort
**Problem:** `curl: (7) Failed to connect to localhost port 31767`

**Root Cause:** Kind doesn't automatically map NodePorts to localhost.

**Solution:**
```bash
# Check if port is mapped
docker port $(docker ps --filter "name=kind-control-plane" --format "{{.ID}}") | grep 31767

# If not mapped, use port-forward instead
kubectl port-forward -n ingress-nginx service/ingress-nginx-controller 8080:80
```

### Issue 5: kubectl port-forward Fails with Permission Error
**Problem:** `The connection to the server localhost:8080 was refused`

**Root Cause:** Running kubectl with sudo loses kubeconfig context.

**Solution:**
```bash
# Don't use sudo, or pass kubeconfig
kubectl port-forward -n ingress-nginx service/ingress-nginx-controller 8080:80

# Or if you need port 80:
sudo KUBECONFIG=$HOME/.kube/config kubectl port-forward -n ingress-nginx service/ingress-nginx-controller 80:80
```

## Debugging Checklist
When Ingress is not working, follow this checklist:

**1. Check Pod Status:**
```bash
kubectl get pods -l app=go-web-app
kubectl describe pod <pod-name>
kubectl logs <pod-name>
```

**2. Check Service and Endpoints:**
```bash
kubectl get service go-web-app
kubectl get endpoints go-web-app
kubectl describe service go-web-app
```

**3. Check Ingress Configuration:**
```bash
kubectl get ingress
kubectl describe ingress go-web-app-ingress
```

**4. Check Ingress Controller:**
```bash
kubectl get pods -n ingress-nginx
kubectl logs -n ingress-nginx deployment/ingress-nginx-controller
```

**5. Test Direct Access:**
```bash
# Test pod directly
kubectl port-forward pod/<pod-name> 8080:8080
curl http://localhost:8080/

# Test service
kubectl port-forward service/go-web-app 8080:80
curl http://localhost:8080/
```

## Deployment on AWS EKS
Deploying to AWS EKS is much simpler because AWS provides a real LoadBalancer.

### 1. Setup EKS Cluster
```bash
# Create EKS cluster
eksctl create cluster \
  --name go-web-app-cluster \
  --region us-east-1 \
  --node-type t3.medium \
  --nodes 2

# Update kubeconfig
aws eks update-kubeconfig --region us-east-1 --name go-web-app-cluster
```

### 2. Deploy Application
```bash
# Apply manifests (same as local)
kubectl apply -f k8s/manifests/

# Check Ingress - it will get a real LoadBalancer IP
kubectl get ingress

# Output will show:
# NAME                 CLASS   HOSTS              ADDRESS                                 PORTS   AGE
# go-web-app-ingress   nginx   go-web-app.local   a1b2c3d4e5f6g-1234567890.elb.amazonaws.com   80      10m
```

### 3. Configure DNS
Add a CNAME record in Route53:
```
go-web-app.local CNAME a1b2c3d4e5f6g-1234567890.elb.amazonaws.com
```

Or update `/etc/hosts` for testing:
```
<ELB-IP> go-web-app.local
```

### 4. Access Application
```bash
# Get the LoadBalancer URL
ELB_URL=$(kubectl get ingress go-web-app-ingress -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
echo "Access at: http://$ELB_URL"

# Or if using host header
curl -H "Host: go-web-app.local" http://$ELB_URL
```

## Key Differences: Local vs AWS EKS

| Aspect | Local (Kind) | AWS EKS |
|--------|--------------|---------|
| LoadBalancer | Stays `<pending>` | Gets real ELB IP |
| Access Method | Port-forwarding | Direct DNS/ELB |
| NodePort | Not mapped to localhost | Not needed |
| DNS | `/etc/hosts` file | Route53 or ALIAS |
| Cost | Free | Pay for EKS + EC2 + ELB |

## Quick Fix Script for Local Development

Save this as `local-dev.sh`:

```bash
#!/bin/bash

echo "=== Go Web App Local Development Setup ==="

# Check if Kind cluster exists
if ! kind get clusters | grep -q "kind"; then
    echo "Creating Kind cluster..."
    kind create cluster
fi

# Install ingress-nginx if not present
if ! kubectl get ns ingress-nginx >/dev/null 2>&1; then
    echo "Installing ingress-nginx..."
    kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
    sleep 30
fi

# Deploy application
echo "Deploying application..."
kubectl apply -f k8s/manifests/

# Wait for pods
echo "Waiting for pods to be ready..."
kubectl wait --for=condition=ready pod -l app=go-web-app --timeout=60s

# Start port-forward
echo "Starting port-forward..."
echo "Access at: http://go-web-app.local:8080"
echo "Press Ctrl+C to stop"
kubectl port-forward -n ingress-nginx service/ingress-nginx-controller 8080:80
```

## Summary

This project demonstrates a complete Go web application deployment pipeline. The main challenges were:

- **Local Kind cluster networking limitations** - Solved with port-forwarding
- **Service selector mismatches** - Fixed by ensuring label consistency
- **Ingress controller configuration** - Different setup needed for local vs cloud

For production on AWS EKS, the deployment is straightforward as AWS handles LoadBalancer creation automatically. For local development, port-forwarding provides a simple workaround.

## Useful Commands

```bash
# Debugging
kubectl get all -l app=go-web-app
kubectl describe ingress go-web-app-ingress
kubectl logs -n ingress-nginx deployment/ingress-nginx-controller

# Cleanup
kubectl delete -f k8s/manifests/
kind delete cluster

# Update deployment
kubectl rollout restart deployment go-web-app
```

## Contributing
Feel free to submit issues and enhancement requests!
