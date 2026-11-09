# Docker Image Optimization & Deployment using Multi‑Stage and Distroless Builds

## Overview

This repository demonstrates how to **build**, **optimize**, and **deploy** a simple Flask calculator application using Docker and AWS.  The main goals are:

- Compare three Docker image strategies: **Single‑Stage**, **Multi‑Stage**, and **Distroless**.
- Show a ~50 % size reduction when using a Distroless base compared to the Single‑Stage build.
- Publish images to **Docker Hub** and a private **Amazon ECR** repository.
- Deploy the container on an AWS EC2 instance using an **IAM role** for secure ECR access (no long‑lived AWS credentials stored).

The project is organized as follows:

```
├─ app/                     # Flask application source
├─ Dockerfile.single        # Single‑stage build
├─ Dockerfile.multi         # Multi‑stage build
├─ Dockerfile.distroless    # Distroless build
├─ images/                  # Screenshots (add the provided images here)
└─ README.md
```

---

## Image Size Comparison

| Build Type | Image Size |
|------------|-----------|
| Single‑Stage | ![Single‑Stage Size](images/media_1794199125716_64dbad8d.jpg) |
| Multi‑Stage | ![Multi‑Stage Size](images/media_1794199159371_e2d7fa2f.jpg) |
| Distroless | ![Distroless Size](images/media_1794199182412_a48ae917.png) |

> **Note:** The Distroless image is roughly **50 %** smaller than the Single‑Stage image.

---

## Building the Images

```bash
# Single‑stage
docker build -f Dockerfile.single -t mayur/calculator:single .

# Multi‑stage
docker build -f Dockerfile.multi -t mayur/calculator:multi .

# Distroless
docker build -f Dockerfile.distroless -t mayur/calculator:distroless .
```

## Publishing to Docker Hub

```bash
docker login            # Authenticate to Docker Hub
docker tag mayur/calculator:distroless mayur/calculator:latest
docker push mayur/calculator:latest
```

## Publishing to Amazon ECR

1. **Create a private ECR repository** (e.g., `calculator`).
2. **Authenticate Docker to ECR** using the AWS CLI:

```bash
aws ecr get-login-password --region <region> |
  docker login --username AWS --password-stdin <aws_account_id>.dkr.ecr.<region>.amazonaws.com
```

3. **Tag and push** the Distroless image:

```bash
docker tag mayur/calculator:distroless \
  <aws_account_id>.dkr.ecr.<region>.amazonaws.com/calculator:distroless

docker push <aws_account_id>.dkr.ecr.<region>.amazonaws.com/calculator:distroless
```

---

## Deploying on an EC2 Instance

1. **Attach an IAM role** with `AmazonEC2ContainerRegistryReadOnly` permission to the EC2 instance.
2. **Install Docker** on the instance (if not already installed).
3. **Pull and run** the image directly from ECR:

```bash
aws ecr get-login-password --region <region> |
  docker login --username AWS --password-stdin <aws_account_id>.dkr.ecr.<region>.amazonaws.com

docker run -d -p 80:5000 \
  <aws_account_id>.dkr.ecr.<region>.amazonaws.com/calculator:distroless
```

4. Verify the deployment by visiting the public DNS of the EC2 instance (e.g., `http://<ec2-public-dns>/`).

![Deployment Verification](images/media_1794199194638_986f1a0e.jpg)

---

## Repository Structure

```
multistage-docker/
│
├─ app/                # Flask calculator source code
│   ├─ __init__.py
│   └─ routes.py
│
├─ Dockerfile.single   # Single‑stage Dockerfile
├─ Dockerfile.multi    # Multi‑stage Dockerfile
├─ Dockerfile.distroless # Distroless Dockerfile
│
├─ images/             # <-- place the four screenshots here
│   ├─ media_1794199125716_64dbad8d.jpg
│   ├─ media_1794199159371_e2d7fa2f.jpg
│   ├─ media_1794199182412_a48ae917.png
│   └─ media_1794199194638_986f1a0e.jpg
│
└─ README.md           # This file
```

---

## License

This project is licensed under the **MIT License** – see the `LICENSE` file for details.

---

*Happy containerizing!*