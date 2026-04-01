# Expense Tracker

[![Python](https://img.shields.io/badge/python-3.11+-blue.svg)](https://www.python.org/downloads/)
[![Flask](https://img.shields.io/badge/flask-latest-green.svg)](https://flask.palletsprojects.com/)
[![Docker](https://img.shields.io/badge/docker-ready-blue.svg)](https://www.docker.com/)
[![Kubernetes](https://img.shields.io/badge/kubernetes-k3s-326CE5.svg)](https://k3s.io/)
[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)

A lightweight, web-based personal finance management application for recording, filtering, and visualizing expenses across categories. The application is built on Flask with SQLite persistence and ships with full containerization and Kubernetes deployment support.

---

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Tech Stack](#tech-stack)
- [Critical Concepts](#critical-concepts)
- [Project Structure](#project-structure)
- [Local Development Setup](#local-development-setup)
- [Application Usage](#application-usage)
- [Docker Deployment](#docker-deployment)
- [Kubernetes Deployment](#kubernetes-deployment)
- [CI/CD Pipeline](#cicd-pipeline)
- [Debugging and Troubleshooting](#debugging-and-troubleshooting)
- [Contributing](#contributing)
- [License](#license)

---

## Architecture Overview

```
[Browser Client]
      |
      | HTTP
      v
[Flask Application - Gunicorn WSGI]
      |
      | SQL Queries
      v
[SQLite Database - expenses.db]
```

For production deployments, the stack extends to:

```
[Git Push]
    |
    v
[GitHub Actions - CI/CD]
    |
    v
[Docker Hub - Image Registry]
    |
    v
[EC2 Instance - K3s Node]
    |
    v
[Kubernetes Pod - expense-tracker container]
    |
    v
[NodePort Service - Port 30007]
```

---

## Tech Stack

| Layer | Technology | Purpose |
|---|---|---|
| Language | Python 3.11+ | Application runtime |
| Web Framework | Flask | HTTP routing, templating, request handling |
| WSGI Server | Gunicorn | Production-grade HTTP server |
| Database | SQLite3 | Lightweight embedded SQL persistence |
| Frontend | HTML5, CSS3, JavaScript | UI rendering and chart interactivity |
| Containerization | Docker | Image build and portability |
| Orchestration | Kubernetes (K3s) | Pod scheduling, service exposure, rollout management |
| CI/CD | GitHub Actions | Automated build, push, and deploy pipeline |
| Registry | Docker Hub | Container image storage and versioning |
| Cloud | AWS EC2 | Host node for K3s cluster |

---

## Critical Concepts

### 1. WSGI and Gunicorn

Flask's built-in development server is single-threaded and not suitable for production. Gunicorn acts as a WSGI-compliant HTTP server that spawns multiple worker processes to handle concurrent requests. The `Dockerfile` uses Gunicorn as the entrypoint rather than `python app.py`.

### 2. SQLite Auto-initialization

The database file (`instance/expenses.db`) is created automatically on application startup via Flask's `with app.app_context(): db.create_all()` pattern. No manual schema migration is required for fresh deployments. The `instance/` directory is excluded from version control and persists locally.

> **Note:** In a Kubernetes deployment, SQLite data does not persist across pod restarts unless a PersistentVolume (PV) is mounted. For production workloads with data durability requirements, migrate to PostgreSQL or MySQL with a PersistentVolumeClaim.

### 3. Docker Image Layering

The `Dockerfile` follows a layered build strategy. Dependencies (`requirements.txt`) are copied and installed before application source code. This ensures that the dependency layer is cached by the Docker build daemon and only rebuilt when `requirements.txt` changes, significantly reducing build times on subsequent pushes.

### 4. Kubernetes NodePort Service

The application is exposed via a `NodePort` service on port `30007`. NodePort allocates a static port on every node in the cluster, making the application accessible at `http://<EC2-PUBLIC-IP>:30007` without requiring an external load balancer. For production multi-node setups, replace NodePort with a `LoadBalancer` service or an Ingress controller.

### 5. K3s as a Lightweight Kubernetes Distribution

K3s is a CNCF-certified, production-ready Kubernetes distribution optimized for resource-constrained environments such as single EC2 instances. It bundles `containerd`, `CoreDNS`, and `Flannel` CNI into a single binary, eliminating the overhead of a full kubeadm cluster while remaining fully `kubectl`-compatible.

### 6. GitHub Actions CI/CD Pipeline

The pipeline triggers on every `git push` to the configured branch and executes three sequential stages: building the Docker image, pushing it to Docker Hub, and issuing a `kubectl rollout restart` on the EC2-hosted K3s cluster via SSH. The EC2 host and Docker credentials are stored as GitHub Secrets and injected as environment variables at runtime.

### 7. Swap Memory on EC2

Free-tier EC2 instances (e.g., `t2.micro`) ship with 1 GB RAM. Building Docker images or running multiple pods can exhaust physical memory. Allocating a swap file provides virtual memory overflow, preventing OOM kills during builds and deployments. Swap is configured once and persisted across reboots via `/etc/fstab`.

---

## Project Structure

```
expense-tracker/
|
|-- app.py                  # Flask application factory, route definitions, DB models
|-- requirements.txt        # Pinned Python dependencies
|-- Dockerfile              # Multi-stage container build definition
|-- deployment.yaml         # Kubernetes Deployment manifest (replica set, image, resources)
|-- service.yaml            # Kubernetes Service manifest (NodePort exposure)
|
|-- templates/
|   |-- base.html           # Base layout with shared navigation and asset imports
|   |-- index.html          # Main dashboard: expense form, filter controls, charts
|   |-- edit.html           # Expense edit form
|
|-- instance/
|   |-- expenses.db         # SQLite database (auto-created, not committed to VCS)
|
|-- docs/
    |-- DEPLOYMENT.md       # Extended production deployment guide (EC2, K3s, TLS)
```

---

## Local Development Setup

### Prerequisites

- Python 3.11 or higher
- pip
- git

### Step 1 - Clone the Repository

```bash
git clone https://github.com/yourusername/expense-tracker.git
cd expense-tracker
```

### Step 2 - Create and Activate a Virtual Environment

```bash
# Linux / macOS
python3 -m venv venv
source venv/bin/activate

# Windows
python -m venv venv
.\venv\Scripts\activate
```

### Step 3 - Install Dependencies

```bash
pip install -r requirements.txt
```

### Step 4 - Run the Development Server

```bash
python app.py
```

### Step 5 - Access the Application

Open a browser and navigate to:

```
http://localhost:5000
```

---

## Application Usage

### Adding an Expense

1. Navigate to the home page.
2. Complete the expense form with the following fields:
   - **Description** - A brief label for the transaction.
   - **Amount** - A positive numeric value.
   - **Category** - One of: Food, Transport, Rent, Utilities, Health, Other.
   - **Date** - Defaults to the current date; can be set retroactively.
3. Submit the form to persist the record.

### Filtering Expenses

- **Date Range** - Use the start and end date inputs to scope the expense list and chart data to a specific period.
- **Category** - Select a single category from the dropdown to isolate category-specific spending.
- Totals and charts update in real time based on the active filter state.

### Editing and Deleting Records

- Click **Edit** on any expense row to open the edit form pre-populated with existing values.
- Click **Delete** to remove a record. Deletion is handled via an HTTP POST request (not GET) to prevent accidental removal via link prefetching or browser history traversal.

### Analytics

- **Pie Chart** - Displays proportional expense distribution by category for the active filter period.
- **Line Chart** - Plots cumulative or daily spending over time, enabling trend identification.

---

## Docker Deployment

### Step 1 - Build the Image

```bash
docker build -t expense-tracker:latest .
```

### Step 2 - Run the Container

```bash
docker run -p 5000:5000 expense-tracker:latest
```

### Step 3 - Access the Application

```
http://localhost:5000
```

> To persist data across container restarts, mount a volume to the `instance/` directory:
> ```bash
> docker run -p 5000:5000 -v $(pwd)/instance:/app/instance expense-tracker:latest
> ```

---

## Kubernetes Deployment

### Prerequisites

- A running K3s cluster on EC2 (or equivalent)
- `kubectl` configured with the cluster kubeconfig
- The Docker image pushed to Docker Hub

### Step 1 - Apply the Deployment Manifest

```bash
kubectl apply -f deployment.yaml
```

### Step 2 - Apply the Service Manifest

```bash
kubectl apply -f service.yaml
```

### Step 3 - Verify the Deployment

```bash
# Check pod status
kubectl get pods

# Check service and assigned NodePort
kubectl get svc
```

### Step 4 - Access the Application

```
http://<EC2-PUBLIC-IP>:30007
```

### Step 5 - Restart Pods (Force Redeploy)

```bash
kubectl rollout restart deployment expense-tracker
```

---

## CI/CD Pipeline

The pipeline automates the full build-to-deploy lifecycle on every push to the main branch.

### Pipeline Flow

```
git push
   |
   v
GitHub Actions triggered
   |
   v
Docker image built from Dockerfile
   |
   v
Image pushed to Docker Hub
   |
   v
SSH into EC2 and issue: kubectl rollout restart deployment expense-tracker
   |
   v
Kubernetes pulls updated image and replaces running pods
```

### Required GitHub Secrets

| Secret Key | Description |
|---|---|
| `DOCKER_USERNAME` | Docker Hub account username |
| `DOCKER_PASSWORD` | Docker Hub account password or access token |
| `EC2_HOST` | Public IP address of the EC2 instance |
| `EC2_SSH_KEY` | Private SSH key for EC2 access |

> If the EC2 public IP changes (e.g., after instance stop/start), update the `EC2_HOST` secret and the browser URL accordingly.

---

## Debugging and Troubleshooting

### View Pod Logs

```bash
kubectl logs <pod-name>
```

### Describe a Pod (Events, Resource Issues)

```bash
kubectl describe pod <pod-name>
```

### Watch Pod Status in Real Time

```bash
kubectl get pods -w
```

### Inspect Node Assignment and Image in Use

```bash
kubectl get pods -o wide
```

### One-Time Swap Memory Setup (EC2 Free Tier)

Run the following once after provisioning the EC2 instance to prevent OOM errors during builds:

```bash
sudo fallocate -l 1G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

Verify swap allocation:

```bash
free -h
```

---

## Contributing

1. Fork the repository.
2. Create a feature branch:
   ```bash
   git checkout -b feature/your-feature-name
   ```
3. Commit your changes with a descriptive message:
   ```bash
   git commit -m "feat: add export to CSV functionality"
   ```
4. Push the branch:
   ```bash
   git push origin feature/your-feature-name
   ```
5. Open a Pull Request against the `main` branch with a clear description of the change and its motivation.

Refer to [CONTRIBUTING.md](CONTRIBUTING.md) for code style guidelines and review expectations.

---

## License

This project is licensed under the MIT License. See the [LICENSE](LICENSE) file for full terms.

---
