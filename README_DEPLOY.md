# AWS Deployment & Engineering Guide

This document details the architecture, infrastructure setup, and deployment pipeline for the Managed ChatKit application on AWS.

## 1. Architecture Overview

The application runs as a containerized workload on **AWS ECS Fargate**.

- **Frontend**: React/Vite app served via Nginx (Port 80).
- **Backend**: FastAPI Python app (Port 8000).
- **Load Balancer**: Application Load Balancer (ALB) handles SSL termination and path-based routing.

### Traffic Flow
1.  User accesses `https://app.domain.com`.
2.  **ALB Listener 443** receives traffic.
3.  **Path Routing**:
    - `/api/*` -> Forwards to **Backend Target Group** (Port 8000).
    - `/*` (Default) -> Forwards to **Frontend Target Group** (Port 80).
4.  **ECS Task**: Both containers run in the same Fargate Task Definition, sharing the same network interface (awsvpc).

---

## 2. Infrastructure Setup

### A. Networking & Security Groups

1.  **ALB Security Group (`alb-sg`)**
    - **Inbound**:
        - TCP 80 (HTTP) from `0.0.0.0/0` (Redirect to HTTPS).
        - TCP 443 (HTTPS) from `0.0.0.0/0`.
    - **Outbound**: All traffic.

2.  **Fargate Task Security Group (`fargate-sg`)**
    - **Inbound**:
        - TCP 80 from `alb-sg` (Frontend).
        - TCP 8000 from `alb-sg` (Backend).
    - **Outbound**: All traffic (needed for pulling images from GHCR and reaching OpenAI API).

### B. Application Load Balancer (ALB)

1.  **Target Groups**:
    - `tg-frontend`: Protocol HTTP, Port 80, Target Type IP, Health Check `/`.
    - `tg-backend`: Protocol HTTP, Port 8000, Target Type IP, Health Check `/health`.
2.  **Listeners**:
    - **HTTP:80**: Redirect to HTTPS:443.
    - **HTTPS:443**:
        - Rule 1: If Path is `/api/*` -> Forward to `tg-backend`.
        - Default: Forward to `tg-frontend`.

### C. ECS Cluster & Service

1.  **Cluster**: Fargate (Serverless).
2.  **Task Definition**:
    - Network Mode: `awsvpc`.
    - CPU/Memory: e.g., 256 CPU / 512 MB Memory.
    - Containers:
        - `frontend`: Image `ghcr.io/OWNER/REPO-frontend:latest`.
        - `backend`: Image `ghcr.io/OWNER/REPO-backend:latest`.
3.  **Service**:
    - Launch Type: Fargate.
    - Public IP: ENABLED (unless using NAT Gateway).
    - Load Balancing: Attached to the ALB Target Groups created above.

---

## 3. CI/CD Pipeline

Deployments are automated via **GitHub Actions** defined in `.github/workflows/docker-publish.yml`.

### Workflow Steps
1.  **Trigger**: Push to `main` branch.
2.  **Build**:
    - Builds Docker images for Frontend and Backend.
    - **Frontend Build Arg**: Injects `VITE_CHATKIT_WORKFLOW_ID` at build time.
3.  **Push**: Pushes images to **GitHub Container Registry (GHCR)**.
    - *Note*: GHCR packages must be set to **Public** visibility to allow ECS to pull them without credentials.
4.  **Deploy**:
    - Updates the ECS Task Definition with new image tags.
    - Injects `OPENAI_API_KEY` into the backend container environment.
    - Triggers a rolling update on the ECS Service.

---

## 4. Configuration & Secrets

### GitHub Repository Secrets
These must be set in GitHub (Settings -> Secrets and variables -> Actions):

| Secret Name | Description | Usage |
| :--- | :--- | :--- |
| `AWS_ACCESS_KEY_ID` | IAM User Key | AWS Authentication |
| `AWS_SECRET_ACCESS_KEY` | IAM User Secret | AWS Authentication |
| `OPENAI_API_KEY` | OpenAI API Key | Injected into Backend at runtime |
| `VITE_CHATKIT_WORKFLOW_ID` | ChatKit Workflow ID | Injected into Frontend at build time |

### Environment Variables (ECS Task Definition)
These can be hardcoded in `.aws/task-definition.json` or injected via the workflow.

| Variable | Container | Value / Description |
| :--- | :--- | :--- |
| `ENVIRONMENT` | Backend | `production` (Enables secure cookies) |
| `ALLOWED_ORIGINS` | Backend | e.g., `https://app.yourdomain.com` (CORS) |
| `OPENAI_API_KEY` | Backend | Injected by GitHub Actions |

---

## 5. Troubleshooting

### Common Issues

1.  **"crypto.randomUUID is not a function"**
    - **Cause**: Accessing the site via HTTP.
    - **Fix**: Ensure ALB redirects HTTP to HTTPS. The app requires a Secure Context (HTTPS or localhost).

2.  **CORS Errors**
    - **Cause**: Backend rejecting requests from the frontend domain.
    - **Fix**: Update `ALLOWED_ORIGINS` in the Backend container environment to match your ALB/Domain URL.

3.  **502 Bad Gateway**
    - **Cause**: Container failed to start or health check failed.
    - **Fix**: Check ECS Task logs in CloudWatch. Ensure Security Groups allow traffic from ALB to Task on ports 80/8000.

4.  **Deployment Fails in GitHub**
    - **Cause**: IAM permissions or ECS Service not found.
    - **Fix**: Verify `ECS_SERVICE` and `ECS_CLUSTER` names in `.github/workflows/docker-publish.yml` match AWS resources. Ensure IAM user has `ecs:UpdateService` and `ecs:RegisterTaskDefinition` permissions.
