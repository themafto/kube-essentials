
# Kube-Essentials

This project demonstrates deploying a full-stack application on Kubernetes using Kind. The application consists of a React frontend, a FastAPI backend, and a PostgreSQL database. The project showcases various Kubernetes concepts, including deployments, services, ingress, secrets, resource management, health checks, persistent volumes, and cron jobs. It provides experience for managing applications in a Kubernetes environment.

## Project Overview

The application architecture is a standard three-tier setup:

- **Frontend:** A React application served by an Nginx Ingress controller.
- **Backend:** A FastAPI application that interacts with the database.
- **Database:** A PostgreSQL database deployed using a Helm chart.

The project utilizes Kind to create a local Kubernetes cluster for development purposes. The application components are deployed in a dedicated namespace (`dev`) and managed using Kubernetes Deployments and Services.  An Ingress resource routes external traffic to the correct service based on the request path.  Secrets management is implemented to securely store sensitive data like database credentials. Resource limits and requests are configured for the deployments, ensuring optimal resource allocation. Liveness and readiness probes monitor the health of the applications.  Finally, a CronJob demonstrates how to automate database backups and persist them using a PersistentVolumeClaim. The cluster setup includes a deliberate distribution of workloads across nodes to illustrate the use of node selectors.


## Prerequisites

- **Docker:** [Docker installation guide](https://docs.docker.com/get-docker/)
- **Kind:** [Kind installation guide](https://kind.sigs.k8s.io/docs/user/quick-start/)
- **kubectl:** [kubectl installation guide](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
- **Helm:** [Helm installation guide](https://helm.sh/docs/intro/install/)


## Cluster Setup

1. **Create the Kind Cluster:**
   ```bash
   kind create cluster --config kind-config.yaml
   ```
   This creates a cluster with 1 control-plane and 3 worker nodes (node-basic, node-fast, node-db).

2. **Install the Nginx Ingress Controller:**
   ```bash
   kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
   ```
   Verifies installation with: `kubectl get pods -n ingress-nginx`

3. **PostgreSQL Provisioning:**
   ```bash
   helm repo add bitnami https://charts.bitnami.com/bitnami
   helm install postgres bitnami/postgresql --version 16.0.3 -n postgres --create-namespace \
     --set architecture=standalone \
     --set "primary.tolerations[0].key=postgres-only" \
     --set "primary.tolerations[0].operator=Exists" \
     --set "primary.tolerations[0].effect=NoSchedule" \
     --set "primary.nodeSelector.kubernetes\.io/hostname=node-db" \
     --set primary.persistence.size=2Gi \
     --set primary.resourcesPreset=none \
     --set primary.resources.requests.cpu=125m \
     --set primary.resources.requests.memory=256Mi \
     --set primary.resources.limits.cpu=250m \
     --set primary.resources.limits.memory=512Mi
   ```
   This deploys PostgreSQL to the `node-db` node with resource limitations suitable for a development environment.


4. **Prepare the Frontend and Backend Images:**
   ```bash
   docker build -t frontend:0.7.1 https://github.com/fastapi/full-stack-fastapi-template.git#0.7.1:frontend
   docker build -t backend:0.7.1 https://github.com/fastapi/full-stack-fastapi-template.git#0.7.1:backend
   kind load docker-image frontend:0.7.1
   kind load docker-image backend:0.7.1
   ```
5. **Create a Development Namespace:**
   ```bash
   kubectl create ns dev
   ```

6. **(Optional) Rename Containers (for clarity):**
   ```bash
   docker rename kind-control-plane control-plane
   docker rename kind-worker node-basic
   docker rename kind-worker2 node-fast
   docker rename kind-worker3 node-db
   ```

## Deployment

All Kubernetes configuration files are located in the `manifests` directory.  Apply them to the `dev` namespace:

```bash
kubectl apply -f manifests --recursive -n dev
```


The manifests directory contains the configurations for:

* **Backend Deployment (`manifests/backend/deployment.yaml`):**  Deploys the backend application, includes init containers for database readiness check (using `cgr.dev/chainguard/wait-for-it`) and database migrations.  Resource requests/limits, liveness, and readiness probes are configured. Deployed to `node-fast` using a node selector.
* **Backend Secret (`manifests/backend/secret.yaml`):** Contains sensitive data like database credentials and superuser password. Instructions for generating the `FIRST_SUPERUSER_PASSWORD` and retrieving the `POSTGRES_PASSWORD` are within the file.
* **Backend Service (`manifests/backend/service.yaml`):** Exposes the backend deployment.
* **Frontend Deployment (`manifests/frontend/deployment.yaml`):** Deploys the frontend application with resource requests/limits and liveness/readiness probes.  Can be scheduled on `node-fast` or `node-basic`.
* **Frontend Service (`manifests/frontend/service.yaml`):** Exposes the frontend deployment.
* **Ingress (`manifests/ingress.yaml`):**  Configures the Nginx Ingress controller to route traffic to the frontend and backend services based on the request path (`/`, `/api`, `/docs`). Uses `localhost` as the host.
* **Persistent Volume Claim (PVC) (`manifests/pvc.yaml`):**  Defines a PVC named `postgres-backup` for storing database backups.
* **CronJob (`manifests/cronjob.yaml`):** Schedules daily database backups using `pg_dump`. Backups are stored in the PVC mounted at `/backups`. The job runs on `node-basic` and uses environment variables from `backend-secrets`.


## Accessing the Application

Access the application at `localhost` in your browser. API documentation is available at `localhost/docs`. Login credentials are configured in the `backend-secrets` secret.


## Locating Database Backups

Accessing the backups from `node-basic` requires accessing the Docker volume where the data is stored.  The command will look similar to this, but the volume name will vary depending on your Kind setup:


```bash
docker exec -it node-basic bash -c 'cat /mnt/data/postgres-backup/backup-YYYYMMDDHHMMSS.sql' # Replace YYYYMMDDHHMMSS with the actual timestamp 
```


You'll need to identify the correct volume name. One approach to identify the mount path would be inspecting the  PostgreSQL Pod on `node-db` to see where its persistence volume is mounted and then correlating it to a Docker volume using `docker inspect <node-db container ID>`. The process could vary, and tools might make it easier. It can be difficult if you are unfamiliar with Docker.

## Cleanup

```bash
kind delete cluster
```
