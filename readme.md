# 🚀 Expense Tracker DevOps Deployment Guide

This document contains all the important commands and steps required to run, deploy, and maintain the Expense Tracker project using Docker, Kubernetes (K3s), AWS EC2, and CI/CD.

---

# 🔐 1. Connect to EC2

```bash
chmod 400 yourkey.pem
ssh -i yourkey.pem ubuntu@<EC2-PUBLIC-IP>
```

---

# 🔁 2. After EC2 Restart (VERY IMPORTANT)

```bash
sudo systemctl start k3s
sudo chmod 644 /etc/rancher/k3s/k3s.yaml
sudo chown ubuntu:ubuntu /etc/rancher/k3s/k3s.yaml
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
```

---

# 🔍 3. Check Kubernetes Cluster

```bash
kubectl get nodes
kubectl get pods -A
```

---

# 🚀 4. Deploy Application (if needed)

```bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
```

---

# 🔍 5. Check Application Status

```bash
kubectl get pods
kubectl get svc
```

---

# 🌐 6. Access Application

```
http://<EC2-PUBLIC-IP>:30007
```

---

# 🔄 7. Restart Pods (Manual Fix)

```bash
kubectl rollout restart deployment expense-tracker
```

---

# 🛠️ 8. Debug Commands

```bash
kubectl logs <pod-name>
kubectl describe pod <pod-name>
kubectl get pods -w
```

---

# 📦 9. Check Deployment/Image

```bash
kubectl get pods -o wide
```

---

# ⚙️ 10. Trigger CI/CD Pipeline

```bash
git add .
git commit -m "update"
git push
```

✔ Automatically:

* Builds Docker image
* Pushes to Docker Hub
* Updates Kubernetes deployment

---

# 🔑 11. If Public IP Changes

Update the following:

* Browser URL
* GitHub Secret → `EC2_HOST`

---

# 💾 12. Swap Memory (One-time Setup)

```bash
sudo fallocate -l 1G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

Check:

```bash
free -h
```

---


# 🔄 Complete Flow

```
Code → Git Push → GitHub Actions → Docker Hub → EC2 → Kubernetes → Live App
```

---

