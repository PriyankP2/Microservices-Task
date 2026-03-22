# Microservices on Kubernetes (Minikube + ECR)

A containerized Node.js microservices application deployed on Kubernetes using Minikube, with images hosted on Amazon ECR.

Architecture

                    ┌─────────────────────────────────────────────┐
                    │               Minikube Cluster              │
                    │                                             │
    Bowser/curl     │  ┌──────────────────┐                       │
    ──────────────►  NodePort :30003      │                       │
                    │  │  gateway-service │                       │
                    │  │     port 3003    │                       │
                    │  └────────┬─────────┘                       │
                    │           │  ClusterIP DNS                  │
                    │    ┌──────┼──────┐                          │
                    │    ▼      ▼      ▼                          │
                    │  user  product  order                       │
                    │  :3000  :3001   :3002                       │
                    └─────────────────────────────────────────────┘

| Service         | ECR Image                                                                | Port | Service Type     |
|-----------------|--------------------------------------------------------------------------|------|------------------|
| user-service    | 975050024946.dkr.ecr.us-west-1.amazonaws.com/user-service:latest         | 3000 | ClusterIP        |
| product-service | 975050024946.dkr.ecr.us-west-1.amazonaws.com/product-service:latest      | 3001 | ClusterIP        |
| order-service   | 975050024946.dkr.ecr.us-west-1.amazonaws.com/order-service:latest        | 3002 | ClusterIP        |
| gateway-service | 975050024946.dkr.ecr.us-west-1.amazonaws.com/gateway-service:latest      | 3003 | NodePort (30003) |

---

## Prerequisites

- EC2 instance (Ubuntu 22.04) with Docker installed
- Minikube v1.38+
- kubectl v1.35+
- AWS CLI v2 configured with ECR access in `us-west-1`

---

## 1. Minikube Setup

### Install Minikube
```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
minikube version
```

### Install kubectl
```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client
```

### Start Minikube
```bash
minikube start --driver=docker --memory=4096 --cpus=2
minikube status
kubectl get nodes
```

Expected output:
```
NAME       STATUS   ROLES           AGE   VERSION
minikube   Ready    control-plane   1m    v1.35.1
```

---

## 2. ECR Image Pull Secret

Minikube needs credentials to pull images from Amazon ECR.

```bash
ECR_PASSWORD=$(aws ecr get-login-password --region us-west-1)

kubectl create secret docker-registry ecr-secret \
  --docker-server=975050024946.dkr.ecr.us-west-1.amazonaws.com \
  --docker-username=AWS \
  --docker-password="$ECR_PASSWORD" \
  --namespace=default

kubectl patch serviceaccount default \
  --namespace default \
  -p '{"imagePullSecrets": [{"name": "ecr-secret"}]}'
```

Verify the secret was created:
```bash
kubectl get secret ecr-secret
```

> **Note:** ECR tokens expire after 12 hours. Re-run the above commands if pods show `ImagePullBackOff`.

---

## 3. Deploy All Services

```bash
kubectl apply -f deployments/
kubectl apply -f services/
```

### Verify everything is running
```bash
kubectl get pods
kubectl get services
kubectl get deployments
```

Expected output:
```
NAME                               READY   STATUS    RESTARTS   AGE
gateway-service-7c96d4c8bf-p7qt2   1/1     Running   0          1m
order-service-6fd645fd5f-22b4z     1/1     Running   0          1m
product-service-6676567859-96hjf   1/1     Running   0          1m
user-service-85fd8bd8f9-lndpm      1/1     Running   0          1m

NAME              TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
gateway-service   NodePort    10.106.215.216   <none>        3003:30003/TCP   1m
order-service     ClusterIP   10.101.103.133   <none>        3002/TCP         1m
product-service   ClusterIP   10.100.156.155   <none>        3001/TCP         1m
user-service      ClusterIP   10.101.32.161    <none>        3000/TCP         1m
```

---

## 4. Testing Services

### Option A — kubectl port-forward

Open a second terminal and run:
```bash
kubectl port-forward svc/gateway-service 3003:3003
```

Then test in your current terminal:
```bash
curl http://localhost:3003/api/users
curl http://localhost:3003/api/products
curl http://localhost:3003/api/orders
```

Expected responses:
```json
[{"id":1,"name":"John Doe"},{"id":2,"name":"Jane Smith"}]
[{"id":1,"name":"Laptop","price":999},{"id":2,"name":"Phone","price":699}]
[]
```

### Option B — Direct service endpoints via port-forward
```bash
kubectl port-forward svc/user-service 3000:3000
curl http://localhost:3000/users

kubectl port-forward svc/product-service 3001:3001
curl http://localhost:3001/products

kubectl port-forward svc/order-service 3002:3002
curl http://localhost:3002/orders
```

### Option C — Validate inter-service communication
```bash
kubectl run curl-test --image=curlimages/curl:latest --rm -it --restart=Never -- sh

# Inside the pod:
curl http://user-service:3000/users
curl http://product-service:3001/products
curl http://order-service:3002/orders
curl http://gateway-service:3003/api/users
```

---

## 5. Bonus: Ingress Configuration

### Enable NGINX Ingress addon
```bash
minikube addons enable ingress

# Wait for ingress controller to be ready
kubectl get pods -n ingress-nginx
```

### Apply Ingress manifest
```bash
kubectl apply -f ingress/ingress.yaml
```

### Remove Ingress
```bash
kubectl delete -f ingress/ingress.yaml
```

---

## 6. Viewing Logs

```bash
# View logs per service
kubectl logs -l app=gateway-service --tail=20
kubectl logs -l app=user-service --tail=20
kubectl logs -l app=product-service --tail=20
kubectl logs -l app=order-service --tail=20

# Stream live logs
kubectl logs -f deployment/gateway-service

# Describe a pod if it won't start
kubectl describe pod -l app=user-service
```

---

## 7. Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| `ImagePullBackOff` | ECR token expired or secret missing | Re-run ECR secret creation commands |
| `CrashLoopBackOff` | App crash on startup | `kubectl logs <pod-name>` to see error |
| Gateway 502/503 | Downstream pods not ready | Wait for all pods `1/1 Running` |
| Port-forward refused | Service not running | Check `kubectl get pods` and `kubectl get svc` |
| Ingress 404 | Wrong host header | Confirm `/etc/hosts` has correct Minikube IP |

### Quick redeploy
```bash
kubectl delete -f services/ -f deployments/
kubectl apply -f deployments/ -f services/
```

### Full Minikube reset
```bash
minikube delete
minikube start --driver=docker --memory=4096 --cpus=2
# Re-create ECR secret, then re-apply manifests
```

---

## Directory Structure

```
submission/
├── deployments/
│   ├── user-service.yaml       # Deployment: ECR image, probes, resource limits
│   ├── product-service.yaml
│   ├── order-service.yaml
│   └── gateway-service.yaml    # Includes downstream service env vars
├── services/
│   ├── user-service.yaml       # ClusterIP :3000
│   ├── product-service.yaml    # ClusterIP :3001
│   ├── order-service.yaml      # ClusterIP :3002
│   └── gateway-service.yaml    # NodePort :3003 / nodePort 30003
├── ingress/
│   └── ingress.yaml            # Bonus: NGINX path-based routing
├── screenshots/
│   ├── pods.png                # kubectl get pods
│   ├── logs.png                # Service communication logs
│   └── service-test.png        # curl test results
└── README.md
