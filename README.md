# DevOps Project: Automated CI/CD Pipeline for a MEAN Stack Application on AWS

**Author:** Parker  
**Date:** February 2026

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Architecture Diagram](#2-architecture-diagram)
3. [Step 1: AWS EC2 Instance Setup](#3-step-1-aws-ec2-instance-setup)
4. [Step 2: Install Dependencies on EC2](#4-step-2-install-dependencies-on-ec2)
5. [Step 3: Jenkins Installation and Setup](#5-step-3-jenkins-installation-and-setup)
6. [Step 4: GitHub Repository Configuration](#6-step-4-github-repository-configuration)
7. [Step 5: Docker Images and Docker Hub](#7-step-5-docker-images-and-docker-hub)
8. [Step 6: Jenkins CI/CD Pipeline](#8-step-6-jenkins-cicd-pipeline)
9. [Step 7: Deploy to EC2 with Docker Compose](#9-step-7-deploy-to-ec2-with-docker-compose)
10. [Step 8: Nginx Reverse Proxy](#10-step-8-nginx-reverse-proxy)
11. [Challenges Faced & Solutions](#11-challenges-faced--solutions)
12. [Conclusion](#12-conclusion)

---

## 1. Project Overview

This report documents the end-to-end deployment of a MEAN stack web application (MongoDB, Express, Angular, Node.js) on AWS EC2. The application is fully containerized using Docker and orchestrated with Docker Compose. A CI/CD pipeline is implemented using Jenkins to automate the build, push, and deployment process whenever code changes are pushed to GitHub.

**Tech Stack:**
- **Frontend:** Angular 15
- **Backend:** Node.js / Express
- **Database:** MongoDB 6
- **Reverse Proxy:** Nginx
- **Containerization:** Docker & Docker Compose
- **CI/CD:** Jenkins (running in Docker)
- **Cloud:** AWS EC2 (t2.small, Ubuntu 22.04, ap-southeast-2)

---

## 2. Architecture Diagram

```
+------------------+      +----------------------+      +-----------------------------+
|   Developer      |----->|     GitHub Repo      |----->|        Jenkins Server       |
|  (git push)      |      | parkervijay/Jenkins  |      |   (Docker on AWS EC2)       |
+------------------+      +----------------------+      |                             |
                                   |                    | 1. Clones Repository        |
                           Webhook |                    | 2. Builds Docker Images     |
                           Trigger |                    | 3. Pushes to Docker Hub     |
                                   |                    | 4. SSH Deploy to EC2        |
                                   |                    +-------------+---------------+
                                   |                                  |
                                   v                                  v
                          +--------+---------+          +------------+------------+
                          |   Docker Hub     |<-------->|    AWS EC2 (t2.small)   |
                          | parkervijay/     |  pull    |    Ubuntu 22.04 LTS     |
                          | mean-frontend    |          |                         |
                          | mean-backend     |          | +---------------------+ |
                          +------------------+          | |  Nginx (Port 80)    | |
                                                        | +---------------------+ |
                                                        | |  Angular Frontend   | |
                                                        | +---------------------+ |
                                                        | |  Node.js Backend    | |
                                                        | +---------------------+ |
                                                        | |  MongoDB            | |
                                                        | +---------------------+ |
                                                        +-------------------------+
```

---

## 3. Step 1: AWS EC2 Instance Setup

### Launch EC2 Instance

- Navigate to the AWS EC2 console
- Launch a **Ubuntu 22.04 LTS** instance
- Select **t2.small** instance type
- Create and assign a key pair for SSH access

### Configure Security Group

| Type       | Protocol | Port  | Source         |
|------------|----------|-------|----------------|
| SSH        | TCP      | 22    | Your IP        |
| HTTP       | TCP      | 80    | 0.0.0.0/0      |
| Custom TCP | TCP      | 3000  | 0.0.0.0/0      |
| Custom TCP | TCP      | 8080  | 0.0.0.0/0      |
| Custom TCP | TCP      | 27017 | 0.0.0.0/0      |

### Connect to EC2

```bash
ssh -i /path/to/your-key.pem ubuntu@3.104.76.153
```

### Screenshot — AWS EC2 Instance Running

![EC2 Instance](screenshots/Screenshot%202026-02-25%20013502.png)

---

## 4. Step 2: Install Dependencies on EC2

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install docker.io docker-compose-v2 -y
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker $USER
newgrp docker
```

---

## 5. Step 3: Jenkins Installation and Setup

Jenkins is deployed as a Docker container for portability and isolation.

### Run Jenkins Container

```bash
docker run -d \
  --name jenkins \
  -p 8080:8080 \
  -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -u root \
  jenkins/jenkins:lts
```

| Parameter | Purpose |
|-----------|---------|
| `-p 8080:8080` | Jenkins Web UI |
| `-p 50000:50000` | Jenkins Agent Communication |
| `-v jenkins_home` | Persistent Jenkins Data |
| `-v docker.sock` | Host Docker Access |
| `-u root` | Docker execution permissions |

### Get Initial Admin Password

```bash
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

### Install Docker CLI Inside Jenkins

```bash
docker exec -u 0 -it jenkins bash
apt update && apt install docker.io -y
apt install docker-compose-plugin -y
exit
```

### Required Jenkins Plugins

- Git Plugin
- Pipeline Plugin
- SSH Agent Plugin
- GitHub Plugin

---

## 6. Step 4: GitHub Repository Configuration

### Repository Structure

```
Jenkins/
├── backend/
│   ├── app/
│   ├── Dockerfile
│   ├── package.json
│   └── server.js
├── frontend/
│   ├── src/
│   ├── Dockerfile
│   ├── angular.json
│   └── package.json
├── nginx/
│   └── nginx.conf
├── screenshots/
├── docker-compose.yml
└── Jenkinsfile
```

### Screenshot — GitHub Repository

![GitHub Repository](screenshots/Screenshot%202026-02-25%20012859.png)

### Screenshot — Backend Directory

![Backend Directory](screenshots/Screenshot%202026-02-25%20013015.png)

### Screenshot — Frontend Directory

![Frontend Directory](screenshots/Screenshot%202026-02-25%20013112.png)

### Backend Dockerfile

```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 3000
CMD ["node", "server.js"]
```

### Frontend Dockerfile

```dockerfile
# Build stage
FROM node:18-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npx ng build --configuration production

# Production stage
FROM nginx:alpine
COPY --from=build /app/dist/angular-15-crud /usr/share/nginx/html
EXPOSE 80
```

### Docker Compose File

```yaml
version: "3.8"

services:
  mongo:
    image: mongo:6
    container_name: mongo
    restart: unless-stopped
    ports:
      - "27017:27017"
    networks:
      - mean-network

  backend:
    image: parkervijay/mean-backend:latest
    container_name: backend
    restart: unless-stopped
    ports:
      - "3000:3000"
    depends_on:
      - mongo
    networks:
      - mean-network

  frontend:
    image: parkervijay/mean-frontend:latest
    container_name: frontend
    restart: unless-stopped
    ports:
      - "4200:80"
    networks:
      - mean-network

  nginx:
    image: nginx:alpine
    container_name: nginx
    restart: unless-stopped
    ports:
      - "80:80"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf
    depends_on:
      - frontend
      - backend
    networks:
      - mean-network

networks:
  mean-network:
    driver: bridge
```

---

## 7. Step 5: Docker Images and Docker Hub

### Build and Push Images

```bash
docker build -t parkervijay/mean-backend:latest ./backend
docker build -t parkervijay/mean-frontend:latest ./frontend
docker login
docker push parkervijay/mean-backend:latest
docker push parkervijay/mean-frontend:latest
```

### Screenshot — Docker Hub Images

![Docker Hub](screenshots/Screenshot%202026-02-25%20013352.png)

---

## 8. Step 6: Jenkins CI/CD Pipeline

### Jenkinsfile

```groovy
pipeline {
    agent any
    triggers {
        githubPush()
    }
    environment {
        DOCKERHUB_CREDENTIALS = credentials('dockerhub-creds')
        DOCKERHUB_USERNAME = 'parkervijay'
        EC2_HOST = '3.104.76.153'
        EC2_USER = 'ubuntu'
    }
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        stage('Build Images') {
            steps {
                sh 'docker build -t $DOCKERHUB_USERNAME/mean-backend:latest ./backend'
                sh 'docker build -t $DOCKERHUB_USERNAME/mean-frontend:latest ./frontend'
            }
        }
        stage('Push to Docker Hub') {
            steps {
                sh 'echo $DOCKERHUB_CREDENTIALS_PSW | docker login -u $DOCKERHUB_CREDENTIALS_USR --password-stdin'
                sh 'docker push $DOCKERHUB_USERNAME/mean-backend:latest'
                sh 'docker push $DOCKERHUB_USERNAME/mean-frontend:latest'
            }
        }
        stage('Deploy to EC2') {
            steps {
                sshagent(['ec2-ssh-key']) {
                    sh """
                        ssh -o StrictHostKeyChecking=no $EC2_USER@$EC2_HOST '
                            cd /home/ubuntu/app &&
                            docker compose pull &&
                            docker compose up -d
                        '
                    """
                }
            }
        }
    }
    post {
        always {
            sh 'docker logout'
        }
    }
}
```

### Jenkins Credentials Setup

| ID | Kind | Purpose |
|----|------|---------|
| `dockerhub-creds` | Username with password | Docker Hub login |
| `ec2-ssh-key` | SSH Username with private key | EC2 SSH access |

### GitHub Webhook Setup

Go to your GitHub repo **Settings → Webhooks → Add webhook**:

- **Payload URL:** `http://3.104.76.153:8080/github-webhook/`
- **Content type:** `application/json`
- **Event:** Just the push event

### Screenshot — Jenkins Build History

![Jenkins Build History](screenshots/Screenshot%202026-02-25%20013146.png)

### Screenshot — Jenkins Successful Build

![Jenkins Success](screenshots/Screenshot%202026-02-25%20013226.png)

---

## 9. Step 7: Deploy to EC2 with Docker Compose

```bash
ssh -i your-key.pem ubuntu@3.104.76.153
cd /home/ubuntu/app
docker compose ps
```

### Screenshot — Docker Compose Running on EC2

![Docker Compose PS](screenshots/Screenshot%202026-02-25%20013757.png)

| Container | Image | Port |
|-----------|-------|------|
| mongo | mongo:6 | 27017 |
| backend | parkervijay/mean-backend:latest | 3000 |
| frontend | parkervijay/mean-frontend:latest | 80 |
| nginx | nginx:alpine | 0.0.0.0:80 |

---

## 10. Step 8: Nginx Reverse Proxy

Nginx acts as the single entry point, routing all traffic on port 80 to the appropriate internal service.

### Screenshot — Nginx Configuration

![Nginx Config](screenshots/Screenshot%202026-02-25%20014007.png)

| Route | Destination |
|-------|-------------|
| `/` | Angular Frontend (port 80) |
| `/api` | Node.js Backend (port 3000) |

Application accessible at: **http://3.104.76.153**

---

## 11. Challenges Faced & Solutions

### Challenge 1: Docker CLI Not Available Inside Jenkins Container

**Error:**
```
docker: not found
```

**Root Cause:** Jenkins was running inside a Docker container without Docker CLI installed and without access to the host Docker daemon socket.

**Solution:** Mounted the Docker socket and installed Docker CLI inside the Jenkins container:
```bash
docker run -v /var/run/docker.sock:/var/run/docker.sock -u root jenkins/jenkins:lts
docker exec -u 0 -it jenkins bash
apt install docker.io -y
```

---

### Challenge 2: Nginx Reverse Proxy Configuration

**Challenge:** Limited prior exposure to Nginx made it unclear how to implement a reverse proxy routing traffic from a single port to multiple internal services.

**Solution:** Researched Nginx reverse proxy concepts and implemented path-based routing in `nginx.conf`, routing `/` to the Angular frontend and `/api` to the Node.js backend using Docker's internal DNS.

**Learning:** Understood single-entry-point architecture, Docker service discovery via container names, and how reverse proxies decouple public-facing ports from internal service ports.

---

### Challenge 3: Jenkins Credentials Management

**Challenge:** Initially hardcoded sensitive credentials directly into the Jenkinsfile, which is a significant security risk.

**Solution:** Replaced hardcoded values with Jenkins Credentials Manager, storing Docker Hub credentials under `dockerhub-creds` and the EC2 SSH key under `ec2-ssh-key`, referencing them securely via `credentials()` and `sshagent()`.

**Learning:** Understood the importance of secrets management in CI/CD pipelines and how Jenkins Credentials Manager prevents sensitive data from being exposed in source code.

---

### Challenge 4: SSH Agent Plugin Not Recognized

**Error:**
```
No such DSL method 'sshagent' found
```

**Root Cause:** The SSH Agent plugin was installed but required a full Jenkins restart to activate.

**Solution:** Performed a full Jenkins restart via `http://<jenkins-ip>:8080/restart`, after which the `sshagent` step was recognized correctly.

---

### Challenge 5: Incorrect EC2 Host Format Breaking SSH

**Error:**
```
ssh: Could not resolve hostname http://3.104.76.153/
```

**Root Cause:** The `EC2_HOST` variable included `http://` and a trailing `/`, which is invalid for SSH connections.

**Solution:**
```groovy
EC2_HOST = '3.104.76.153'
```

---

### Challenge 6: Disk Space Exhaustion Causing Build Failures

**Error:**
```
ENOSPC: no space left on device
```

**Root Cause:** Accumulated Docker images, build cache, and Jenkins workspace data exhausted available disk space on the EC2 instance.

**Solution:**
```bash
docker system prune -a --volumes -f
rm -rf /var/jenkins_home/workspace/*
sudo apt-get clean && sudo apt-get autoremove -y
```

---

### Challenge 7: Accidental Exposure of Private Key in Repository

**Challenge:** A `.pem` private key file was accidentally committed and pushed to the GitHub repository.

**Solution:** Immediately deleted the repository, created a new one, and rotated the compromised AWS key pair. All secrets are now managed through Jenkins Credentials Manager and `.pem` files are excluded via `.gitignore`.

**Learning:** Reinforced the importance of configuring `.gitignore` before the first commit and never storing credentials in source control.

---

## 12. Conclusion

The CI/CD pipeline is now fully operational. Every `git push` to the `main` branch automatically triggers the Jenkins pipeline, which builds updated Docker images, pushes them to Docker Hub, and deploys the latest version to AWS EC2.

**Pipeline Flow:**
```
git push → GitHub Webhook → Jenkins Build → Docker Build → Docker Hub Push → EC2 Deploy → Live App
```

Application live at: **http://3.104.76.153**
