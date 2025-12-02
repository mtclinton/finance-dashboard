# Finance Dashboard on Minikube (Linux)
![](demo.gif)

This guide walks through:
- Installing and starting Minikube on Linux
- Building and pushing the backend and frontend images to Docker Hub
- Deploying the Kubernetes manifest (`finance-dashboard.yaml`)
- Port‑forwarding to view the Finance Dashboard locally

The manifest assumes the namespace `finance` and the images:
- Backend: `docker.io/mtclinton/finance-dashboard-backend:latest`
- Frontend: `docker.io/mtclinton/finance-dashboard-frontend:latest`

If your Docker Hub username or repository names differ, substitute accordingly.

---

## 1) Prerequisites (Linux)

Install:
- Docker (or another container runtime)
- kubectl
- Minikube

On Ubuntu/Debian (example):
```bash
sudo apt-get update
sudo apt-get install -y curl apt-transport-https ca-certificates gnupg lsb-release

# Docker
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
newgrp docker

# kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl && sudo mv kubectl /usr/local/bin/

# Minikube
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
```

Start Minikube (Docker driver):
```bash
minikube start --driver=docker --cpus=4 --memory=6g
kubectl get nodes
```

---

## 2) Build and push images to Docker Hub (optional)

You will need to be logged in to Docker Hub locally:
```bash
docker login
```

### Backend (optional)
Clone the [backend project](https://github.com/mtclinton/finance-dashboard-backend) and substitute the below dockerhub username with your own:
```bash
# Build and push
cd finance-dashboard-backend
docker build -t docker.io/mtclinton/finance-dashboard-backend:latest .
docker push docker.io/mtclinton/finance-dashboard-backend:latest
```
Update finance-dashboard.yaml backend image to your image

### Frontend (optional)
Clone the [frontend project](https://github.com/mtclinton/finance-dashboard-frontend) and substitute the below dockerhub username with your own:
```bash
cd finance-dashboard-frontend
docker build -t docker.io/mtclinton/finance-dashboard-frontend:latest .
docker push docker.io/mtclinton/finance-dashboard-frontend:latest
```
Update finance-dashboard.yaml frontend image to your image

Optional (faster iteration): instead of pushing, you can load local images into Minikube:
```bash
minikube image load docker.io/mtclinton/finance-dashboard-backend:latest
minikube image load docker.io/mtclinton/finance-dashboard-frontend:latest
```

---

## 3) Apply the Kubernetes manifest

From the directory that contains `finance-dashboard.yaml`:
```bash
kubectl apply -f finance-dashboard.yaml
```

Wait for Pods:
```bash
kubectl get pods -n finance -w
```
You should see a Deployment named `finance-stack` (backend+postgres+frontend in one Pod) and a Service `finance-frontend`.

If you change an image, restart to pick it up:
```bash
kubectl rollout restart deploy/finance-stack -n finance
kubectl rollout status deploy/finance-stack -n finance
```

---

## 4) Port‑forward and open the app

Port‑forward the frontend Service (ClusterIP) to your localhost:
```bash
kubectl port-forward -n finance svc/finance-frontend 8081:80
```
Then open:
```
http://localhost:8081
```

API quick checks (through the frontend proxy):
```bash
curl -s http://localhost:8081/health
curl -s http://localhost:8081/api/health
curl -s http://localhost:8081/api/categories | jq
curl -s http://localhost:8081/api/analytics  | jq
```

Direct backend (optional):
```bash
kubectl port-forward -n finance svc/finance-backend 8080:8080
curl -s http://localhost:8080/health
```

---

## 5) Seed demo data (optional)

The backend includes a demo seeder that populates realistic income/expense transactions for the last ~30 days and a few budgets. It’s idempotent and only runs if there are no transactions.

Run inside the backend container in Kubernetes:
```bash
kubectl exec -it -n finance deploy/finance-stack -c backend -- ./main -seed-demo
```

If you want to reset and reseed:
```bash
kubectl exec -it -n finance deploy/finance-stack -c backend -- sh -lc '\
  psql "$DATABASE_URL" -c "TRUNCATE transactions, budgets RESTART IDENTITY;" && \
  ./main -seed-demo \
'
```

Refresh the dashboard afterward. You should see analytics populate and recent transactions listed.

---

## 6) Troubleshooting

- Pods status:
```bash
kubectl get pods -n finance -o wide
```
- Logs:
```bash
kubectl logs -n finance deploy/finance-stack -c backend --tail=200
kubectl logs -n finance deploy/finance-stack -c postgres --tail=200
kubectl logs -n finance deploy/finance-stack -c frontend --tail=200
```
- Services and endpoints:
```bash
kubectl get svc,endpoints -n finance
```
- Rebuild and redeploy workflow (after backend/frontend changes):
```bash
# build & push
docker build -t docker.io/mtclinton/finance-dashboard-backend:latest .
docker push docker.io/mtclinton/finance-dashboard-backend:latest

cd finance-dashboard-frontend
docker build -t docker.io/mtclinton/finance-dashboard-frontend:latest .
docker push docker.io/mtclinton/finance-dashboard-frontend:latest

# restart
kubectl rollout restart deploy/finance-stack -n finance
```

---

## 7) Clean up

```bash
kubectl delete -f finance-dashboard.yaml
minikube delete
```

