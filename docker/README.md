# Flask Calculator – Docker Single‑Stage vs Multi‑Stage vs Distroless Deployment on AWS

*An end‑to‑end DevOps portfolio project showcasing Docker image‑size optimization, security‑hardening with Distroless containers, and deployment to an Ubuntu EC2 instance via Amazon ECR.*

---

## Table of Contents

1. [Project Overview](#project-overview)
2. [Technology Stack](#technology-stack)
3. [Repository Structure](#repository-structure)
4. [Application Details](#application-details)
5. [Docker Build Strategies](#docker-build-strategies)
   - 5.1 [Single‑Stage Dockerfile](#single‑stage-dockerfile)
   - 5.2 [Multi‑Stage Dockerfile](#multi‑stage-dockerfile)
   - 5.3 [Distroless Dockerfile](#distroless-dockerfile)
6. [Image Size & Security Comparison](#image-size--security-comparison)
7. [Local Testing](#local-testing)
8. [Publishing to Docker Hub](#publishing-to-docker-hub)
9. [Publishing to Amazon ECR](#publishing-to-amazon-ecr)
10. [EC2 Configuration](#ec2-configuration)
11. [EC2 Deployment Steps](#ec2-deployment-steps)
12. [Container Management Commands](#container-management-commands)
13. [Architecture Diagram](#architecture-diagram)
14. [Key Learnings](#key-learnings)
15. [Future Improvements (non‑CI/CD)](#future-improvements)
16. [Project Outcomes](#project-outcomes)
17. [Troubleshooting & FAQ](#troubleshooting--faq)

---

## Project Overview

A **Flask‑based web calculator** that performs addition, subtraction, multiplication, and division (with graceful handling of division‑by‑zero). The app is containerized using three Docker approaches:

| Approach | Base Image | Build Technique | Image Tag |
|----------|------------|----------------|-----------|
| **Single‑Stage** | `python:3.14-slim` | All steps in one layer | `flask-calculator:single` |
| **Multi‑Stage** | `python:3.14-slim` (builder) → `python:3.14-slim` (runtime) | Separate builder & runtime layers | `flask-calculator:multi` |
| **Distroless** | `python:3.13-slim-trixie` → `gcr.io/distroless/python3-debian13:nonroot` | Minimal runtime, non‑root user | `flask-calculator:distroless` |

Only the **Distroless** image (≈ 50 % smaller than the single‑stage image) was pushed to **Docker Hub** and **Amazon ECR**.

---

## Technology Stack

| Category | Tool / Technology | Version / Tag |
|----------|-------------------|---------------|
| Programming Language | Python | 3.14 (single/multi) • 3.13 (Distroless) |
| Web Framework | Flask | 3.0 |
| Container Runtime | Docker Engine | 27.x |
| Registries | Docker Hub, Amazon ECR | — |
| Cloud Provider | AWS (EC2, IAM, ECR) | — |
| OS (EC2) | Ubuntu 26.04.1 LTS (x86_64) | — |
| CI/CD | *Not used in this project* | — |

---

## Repository Structure

```
.
├─ app.py                     # Flask entry point
├─ requirements.txt           # Python dependencies
├─ templates/
│   └─ index.html            # Simple HTML calculator UI
├─ Dockerfile.single          # Single‑stage build
├─ Dockerfile.multistage      # Multi‑stage build
├─ Dockerfile.distroless      # Distroless build (exact content below)
├─ README.md                  # This documentation
└─ .gitignore
```

---

## Application Details

- **Routes**
  - `GET /` – Render the calculator UI.
  - `POST /calculate` – Accept two numbers and an operator, return the result.

- **Error handling** – Division‑by‑zero returns a friendly message instead of a server error.

- **Network** – Flask listens on `0.0.0.0:5000` so Docker can expose the port to the host.

---

## Docker Build Strategies

### Single‑Stage Dockerfile

```dockerfile
# Dockerfile.single
FROM python:3.14-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .
COPY templates/ ./templates/

EXPOSE 5000
CMD ["python", "app.py"]
```

All steps occur in a single image, which is easy to read but retains build‑time files, resulting in a larger final image.

---

### Multi‑Stage Dockerfile

```dockerfile
# Dockerfile.multistage
# ---- Builder stage -------------------------------------------------
FROM python:3.14-slim AS builder
WORKDIR /install

COPY requirements.txt .
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt

# ---- Runtime stage -------------------------------------------------
FROM python:3.14-slim
WORKDIR /app

COPY --from=builder /install /usr/local
COPY app.py .
COPY templates/ ./templates/

EXPOSE 5000
CMD ["python", "app.py"]
```

The builder installs dependencies into a custom prefix (`/install`). The runtime stage copies only those files, shedding build‑time layers and reducing size by ~10 % compared with the single‑stage build.

---

### Distroless Dockerfile

```dockerfile
# Dockerfile.distroless  (exact content)
FROM python:3.13-slim-trixie AS builder

WORKDIR /build

COPY requirements.txt .

# Install deps into /install so they can be copied into the distroless image
RUN pip install --no-cache-dir --target=/install -r requirements.txt

FROM gcr.io/distroless/python3-debian13:nonroot

WORKDIR /app

# Runtime gets only the compiled dependencies
COPY --from=builder /install /app/deps

# Application source
COPY app.py .
COPY templates/ ./templates/

# Tell Python where to find the vendored packages
ENV PYTHONPATH=/app/deps

EXPOSE 5000

# Distroless images lack a shell; we run the script directly.
CMD ["app.py"]
```

**Why this works**

| Aspect | Explanation |
|--------|--------------|
| Builder Python version | Matches the Distroless runtime to avoid ABI mismatches. |
| `--target=/install` | Creates a self‑contained site‑packages directory that can be copied without the full Python runtime. |
| `COPY --from=builder` | Pulls only the needed files, dramatically shrinking the final image. |
| `PYTHONPATH=/app/deps` | Adds the vendored packages to Python’s import path (Distroless lacks the standard site‑packages location). |
| `nonroot` variant | Runs the container as a non‑root UID/GID (65532), following the principle of least privilege. |
| No shell / package manager | Removes `bash`, `apt`, etc., shrinking the attack surface. |

---

## Image Size & Security Comparison

| Build Strategy | Approx. Image Size (observed) | Security Highlights |
|----------------|------------------------------|---------------------|
| **Single‑Stage** | Baseline (≈ 200 MB) | Full OS utilities present; larger attack surface. |
| **Multi‑Stage** | ~10 % smaller than single‑stage (≈ 180 MB) | Strips build‑time layers; fewer packages. |
| **Distroless** | **≈ 50 % smaller than single‑stage** (≈ 100 MB) | No shell, no package manager, runs as non‑root → minimal runtime attack surface. |

> **Important:** These sizes are **observed on the author’s environment** and serve only to illustrate the relative differences. They are **not universal benchmarks**.

### Visual size comparison

![Distroless image size reduction (≈ 50 % smaller than single‑stage)](file:///C:/Users/user/.gemini/antigravity/brain/c9bf48e9-9c5e-456b-9edf-18c0e827c50f/.user_uploaded/media_1794155936788_30c995ac.jpg)

---

## Local Testing

```bash
# Single‑stage
docker build -f Dockerfile.single -t flask-calculator:single .
docker run -d --name calc-single -p 5000:5000 flask-calculator:single

# Multi‑stage
docker build -f Dockerfile.multistage -t flask-calculator:multi .
docker run -d --name calc-multi -p 5001:5000 flask-calculator:multi

# Distroless
docker build -f Dockerfile.distroless -t flask-calculator:distroless .
docker run -d --name calc-distroless -p 5002:5000 flask-calculator:distroless
```

Visit the following URLs to verify each container:

- `http://localhost:5000` – Single‑stage
- `http://localhost:5001` – Multi‑stage
- `http://localhost:5002` – Distroless

---

## Publishing to Docker Hub

| Item | Value |
|------|-------|
| Docker Hub username | `mayursureshnair` |
| Repository | `flask-calculator` |
| Distroless Tag | `distroless` |

```bash
docker login                 # prompts for credentials
docker tag flask-calculator:distroless \
  mayursureshnair/flask-calculator:distroless
docker push mayursureshnair/flask-calculator:distroless
```

**Docker Hub push log**

![Docker Hub push output for the Distroless image](file:///C:/Users/user/.gemini/antigravity/brain/c9bf48e9-9c5e-456b-9edf-18c0e827c50f/.user_uploaded/media_1794155953984_c2c07b3e.jpg)

---

## Publishing to Amazon ECR

| Parameter | Value |
|-----------|-------|
| AWS Region | `us-east-1` |
| Account ID | `457980076200` |
| Repository | `flask-calculator` |
| Full URI | `457980076200.dkr.ecr.us-east-1.amazonaws.com/flask-calculator` |
| Tag | `distroless` |

### Steps

1. **Create the repository** (private) via the AWS Console – name: `flask-calculator`.
2. **Verify identity**

   ```bash
   aws sts get-caller-identity
   ```
3. **Authenticate Docker with ECR**

   ```bash
   aws ecr get-login-password --region us-east-1 \
     | sudo docker login --username AWS --password-stdin \
       457980076200.dkr.ecr.us-east-1.amazonaws.com
   ```
4. **Tag & push**

   ```bash
   docker tag flask-calculator:distroless \
     457980076200.dkr.ecr.us-east-1.amazonaws.com/flask-calculator:distroless

   docker push 457980076200.dkr.ecr.us-east-1.amazonaws.com/flask-calculator:distroless
   ```

**ECR push log**

![ECR push output showing the Distroless image being uploaded](file:///C:/Users/user/.gemini/antigravity/brain/c9bf48e9-9c5e-456b-9edf-18c0e827c50f/.user_uploaded/media_1794155975813_91389c50.png)

---

## EC2 Configuration

| Component | Details |
|-----------|---------|
| OS | Ubuntu 26.04.1 LTS (x86_64) |
| Docker | `sudo apt-get install -y docker.io` |
| AWS CLI v2 | Manual ZIP install (see below) |
| IAM Role | `AmazonEC2ContainerRegistryReadOnly` attached (no static keys) |
| Docker permissions | Commands run with `sudo` (default `ubuntu` user not in `docker` group) |

### Install AWS CLI v2 (x86_64)

```bash
sudo apt-get update && sudo apt-get install -y curl unzip

curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Verify
aws --version   # e.g., aws-cli/2.x.x Python/3.x.x Linux/5.x.x
```

---

## EC2 Deployment Steps

```bash
# 1️⃣ Verify IAM identity (optional)
aws sts get-caller-identity

# 2️⃣ Authenticate Docker with ECR
aws ecr get-login-password --region us-east-1 \
  | sudo docker login --username AWS --password-stdin \
    457980076200.dkr.ecr.us-east-1.amazonaws.com

# 3️⃣ Pull the Distroless image
sudo docker pull 457980076200.dkr.ecr.us-east-1.amazonaws.com/flask-calculator:distroless

# 4️⃣ Verify the image is present locally
sudo docker images | grep flask-calculator

# 5️⃣ Run the container, mapping container port 5000 to host port 80
sudo docker run -d \
  --name calculator-ecr \
  -p 80:5000 \
  457980076200.dkr.ecr.us-east-1.amazonaws.com/flask-calculator:distroless
```

- **Port mapping** `80:5000` makes the calculator reachable via the standard HTTP port (80) on the EC2 instance.
- The instance’s **Security Group** must allow inbound TCP traffic on port **80**.

### EC2 deployment screenshot

![Running the Distroless container on EC2](file:///C:/Users/user/.gemini/antigravity/brain/c9bf48e9-9c5e-456b-9edf-18c0e827c50f/.user_uploaded/media_1794156163451_73083386.jpg)

---

## Container Management Commands

| Command | Purpose |
|---------|----------|
| `sudo docker ps` | List running containers. |
| `sudo docker ps -a` | List all containers (including stopped). |
| `sudo docker images` | Show local images. |
| `sudo docker logs calculator-ecr` | View container logs. |
| `sudo docker stop calculator-ecr` | Stop the container. |
| `sudo docker start calculator-ecr` | Start a stopped container. |
| `sudo docker restart calculator-ecr` | Restart the container. |
| `sudo docker rm calculator-ecr` | Remove the container (must be stopped). |
| `sudo docker rmi <image-id>` | Delete a local image to free space. |

---

## Architecture Diagram

```mermaid
flowchart TD
    A[Local Development Machine] --> B[Flask Application Code]
    B --> C{Docker Build Strategies}
    C -->|Single‑Stage| D[flask-calculator:single]
    C -->|Multi‑Stage| E[flask-calculator:multi]
    C -->|Distroless| F[flask-calculator:distroless]
    D --> G[Docker Hub]
    E --> G
    F --> G
    F --> H[Amazon ECR]
    G --> I[EC2 Ubuntu Instance]
    H --> I
    I --> J[Docker Daemon (sudo)]
    J --> K[Pull Distroless Image]
    K --> L[Run Container (non‑root)]
    L --> M[Flask Calculator – HTTP Port 80]
    style G fill:#f9f,stroke:#333,stroke-width:2px
    style H fill:#bbf,stroke:#333,stroke-width:2px
```

### Diagram rendering screenshot

![Rendered architecture diagram](file:///C:/Users/user/.gemini/antigravity/brain/c9bf48e9-9c5e-456b-9edf-18c0e827c50f/.user_uploaded/media_1794156326463_46313b0b.jpg)

---

## Key Learnings

- **Image optimization** – Multi‑stage and Distroless builds can halve container size, improving download time and surface area.
- **Security hardening** – Non‑root execution and removal of shells/package managers drastically reduce the runtime attack surface.
- **Registry differences** – Docker Hub is public by default; Amazon ECR provides private, IAM‑controlled storage.
- **IAM role usage** – Supplies temporary credentials to the EC2 instance, eliminating static secrets.
- **Port mapping & security groups** – Correct mapping (`80:5000`) and SG rule enable public HTTP access.
- **Debugging Distroless** – Without a shell, rely on logs or temporary entrypoint overrides for troubleshooting.
- **Linux Docker permissions** – Using `sudo` avoids permission errors on fresh Ubuntu installations.

---

## Future Improvements (non‑CI/CD)

| Area | Planned Enhancement |
|------|---------------------|
| **Automated Builds** | Use a local script or Makefile to rebuild and push all three images with a single command. |
| **Health Checks** | Add a `HEALTHCHECK` instruction to each Dockerfile so Docker can monitor container health. |
| **Restart Policies** | Deploy with `--restart unless-stopped` for better resilience. |
| **Image Scanning** | Enable Amazon ECR vulnerability scanning and act on findings. |
| **Monitoring** | Export container metrics to CloudWatch (CPU, memory, request latency). |
| **Load Balancing** | Front the EC2 instance with an Application Load Balancer for production‑grade availability. |
| **Security Enhancements** | Experiment with `gVisor` or custom `seccomp` profiles for additional isolation. |

---

## Project Outcomes

- Demonstrated **Docker image‑size reduction** (≈ 50 % with Distroless).
- Gained hands‑on experience with **multi‑stage builds**, **Distroless containers**, and **non‑root execution**.
- Delivered a **complete, manual deployment pipeline**: code → container → Docker Hub → Amazon ECR → EC2.
- Strengthened **AWS IAM** and **ECR authentication** knowledge, avoiding static credentials.
- Produced a **portfolio‑ready README** and Dockerfiles suitable for junior DevOps or Cloud Engineer roles.

---

## Troubleshooting & FAQ

| Symptom | Likely Cause | Fix |
|---------|--------------|-----|
| `docker: permission denied while trying to connect to the Docker daemon socket` | Running Docker without `sudo` and not in `docker` group. | Prefix Docker commands with `sudo` or add the user to the `docker` group (`sudo usermod -aG docker $USER`). |
| Container exits with `Error: No module named app` (Distroless) | `PYTHONPATH` missing or mistyped. | Verify `ENV PYTHONPATH=/app/deps` is present in the Dockerfile. |
| `401 Unauthorized` when pushing to ECR | Not logged in or IAM role lacks permission. | Re‑run the ECR login command and ensure the attached role includes `AmazonEC2ContainerRegistryReadOnly`. |
| Division‑by‑zero still crashes | Inputs not cast to numeric types. | Convert form values to `float` in `app.py` before performing calculations. |
| Port 80 already in use on EC2 | Another service (e.g., Apache) is bound to port 80. | Stop the conflicting service or map the container to a different host port (e.g., `-p 8080:5000`). |
| Image size larger than expected | `.dockerignore` missing, copying unnecessary files. | Add a `.dockerignore` with entries like `__pycache__/`, `.git/`, `*.pyc`. |
| No response from the calculator after deployment | Security Group does not allow inbound HTTP traffic. | Open inbound TCP port 80 in the EC2 instance’s Security Group. |

---

*Feel free to reach out if you need any further adjustments or additional documentation.*