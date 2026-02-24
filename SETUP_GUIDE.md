# Secure Health API - Setup & Running Guide

This document provides complete step-by-step instructions to run the Secure Health API project and set up Jenkins CI/CD pipeline.

---

## Table of Contents
1. [Prerequisites](#prerequisites)
2. [Project Overview](#project-overview)
3. [Running the Project](#running-the-project)
4. [Testing the API](#testing-the-api)
5. [Key Management](#key-management)
6. [Jenkins CI/CD Pipeline Setup](#jenkins-cicd-pipeline-setup)
7. [Accessing Services](#accessing-services)

---

## 1. Prerequisites

Before running the project, ensure you have the following installed:

| Requirement | Version | Purpose |
|-------------|---------|---------|
| **Docker** | Latest | Containerization |
| **Docker Compose** | Latest | Orchestrating services |
| **Python** | 3.11+ | Running key generation scripts |
| **Git** | Latest | Version control |

### Verify Installation

```
bash
# Check Docker
docker --version

# Check Docker Compose
docker compose version

# Check Python
python --version
```

---

## 2. Project Overview

The Secure Health API is a HIPAA-compliant health data API with:

- **Flask API** (Port 8443) - RESTful endpoints for patient records
- **Keycloak** (Port 8081) - Authentication/Authorization (OIDC)
- **Prometheus** (Port 9090) - Metrics collection
- **Grafana** (Port 3000) - Visualization dashboards
- **Loki** (Port 3100) - Log aggregation
- **Jenkins** (Port 8080) - CI/CD automation

---

## 3. Running the Project

### Step 1: Navigate to Project Directory

```
bash
cd secure-health-api
```

### Step 2: Generate Encryption Key (First Time Only)

The data encryption key is required for storing patient records securely:

```
bash
# Using Python (creates keys/data.key)
python keygen.py
```

Or use the PowerShell script:

```
powershell
powershell -ExecutionPolicy Bypass -File scripts/setup.ps
```

### Step 3: Start All Services

Using Docker Compose:

```
bash
docker compose up -d
```

This will start:
- Keycloak (Identity Provider)
- App (Health API)
- Prometheus (Metrics)
- Grafana (Dashboards)
- Loki (Logs)
- Promtail (Log Agent)
- Jenkins (CI/CD)

### Step 4: Verify Services are Running

```
bash
docker compose ps
```

Expected output:
```
NAME                IMAGE                           COMMAND                  SERVICE             CREATED             STATUS              PORTS
secure-health-api-app-1        health-api:local         "python server.py"      app                 ...                 Up                 0.0.0.0:8443->8443/tcp
secure-health-api-keycloak-1    quay.io/keycloak/keycloak:24.0   "start-dev"          keycloak            ...                 Up                 0.0.0.0:8081->8080/tcp
secure-health-api-prometheus-1  prom/prometheus          "/bin/prometheus ..."  prometheus          ...                 Up                 0.0.0.0:9090->9090/tcp
secure-health-api-grafana-1     grafana/grafana          "/run.sh"              grafana              ...                 Up                 0.0.0.0:3000->3000/tcp
secure-health-api-loki-1        grafana/loki             "-config.file=/etc…"   loki                 ...                 Up                 0.0.0.0:3100->3100/tcp
secure-health-api-jenkins-1     jenkins/jenkins:lts      "/usr/bin/tini -- ..." jenkins              ...                 Up                 0.0.0.0:8081->8080/tcp
```

---

## 4. Testing the API

### Step 1: Access Keycloak Console

1. Open browser: http://localhost:8081
2. Login with credentials:
   - Username: `admin`
   - Password: `admin`

### Step 2: Create Realm and Client

1. Create a new realm named: `health`
2. Create a client:
   - Client ID: `health-api`
   - Client Protocol: `openid-connect`
   - Access Type: `confidential`
   - Valid Redirect URIs: `http://localhost:*`
3. Create roles: `viewer`, `editor`
4. Create a test user with roles

### Step 3: Get JWT Token

```
bash
# Using curl (replace with your credentials)
curl -X POST http://localhost:8081/realms/health/protocol/openid-connect/token \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=password" \
  -d "client_id=health-api" \
  -d "client_secret=YOUR_CLIENT_SECRET" \
  -d "username=YOUR_USERNAME" \
  -d "password=YOUR_PASSWORD"
```

### Step 4: Test API Endpoints

```
bash
# Get patient record (requires JWT token)
curl -k https://localhost:8443/records/patient456 \
  -H "Authorization: Bearer YOUR_JWT_TOKEN"

# Create new patient record
curl -k -X POST https://localhost:8443/records \
  -H "Authorization: Bearer YOUR_JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"patient_id":"patient789","name":"Jane Doe","diagnosis":"Flu"}'

# View metrics
curl -k https://localhost:8443/metrics
```

---

## 5. Key Management

### Generate New Key

```
bash
python keygen.py
```

### Rotate Keys (Production)

```
bash
python keyrotate.py
```

This script:
1. Generates a new encryption key
2. Re-encrypts all existing encrypted data files
3. Replaces the old key with the new one

---

## 6. Jenkins CI/CD Pipeline Setup

### Option A: Run Jenkins via Docker Compose

Jenkins is already included in `docker-compose.yml`:

```
bash
# Start Jenkins
docker compose up -d jenkins
```

### Option B: Manual Jenkins Setup

#### Step 1: Start Jenkins Container

```
bash
docker run -d \
  --name jenkins \
  -p 8080:8080 \
  -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  jenkins/jenkins:lts
```

#### Step 2: Get Initial Admin Password

```
bash
# Get Jenkins initial admin password
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

#### Step 3: Complete Jenkins Setup

1. Open browser: http://localhost:8080
2. Enter the initial admin password
3. Install suggested plugins
4. Create admin user
5. Click "Manage Jenkins" → "Manage Plugins" → "Available"
6. Install: "Docker Pipeline", "Pipeline Utility Steps"

#### Step 4: Create Pipeline Job

1. Click "New Item" → Select "Pipeline"
2. Name: `secure-health-api`
3. Scroll down to "Pipeline" section
4. Select "Pipeline script from SCM"
5. Configure:
   - SCM: Git
   - Repository URL: Your git repository URL
   - Branches to build: `*/main` or `*/master`
   - Script Path: `Jenkinsfile` (for Windows) or `jenkins/Jenkinsfile` (for Linux)

#### Step 5: Run the Pipeline

1. Click "Build Now"
2. Watch the pipeline stages:
   - **Checkout** - Pulls code from repository
   - **Build** - Builds Docker image
   - **Test** - Runs pytest tests
   - **Security basics** - Placeholder for security validation
   - **Deploy** - Deploys the application

### Jenkins Pipeline Stages Explained

```
groovy
pipeline {
  agent any
  stages {
    // 1. Pull latest code
    stage('Checkout') {
      steps { checkout scm }
    }
    
    // 2. Build Docker image
    stage('Build') {
      steps {
        bat 'docker build -t health-api:local ./app'  // Windows
        // sh 'docker build -t health-api:local ./app'  // Linux
      }
    }
    
    // 3. Run tests
    stage('Test') {
      steps {
        bat 'docker run --rm -v "%cd%:/app" health-api:local python -m pytest --junitxml=/app/pytest.xml -q'
        // sh 'docker run --rm health-api:local python -m pytest -q'
      }
      post {
        always { junit 'pytest.xml' }
      }
    }
    
    // 4. Security checks (placeholder)
    stage('Security basics') {
      steps {
        bat 'echo "Placeholder: validate dependencies (offline)"'
      }
    }
    
    // 5. Deploy application
    stage('Deploy') {
      steps {
        bat 'docker compose up -d app'
      }
    }
  }
}
```

---

## 7. Accessing Services

| Service | URL | Credentials |
|---------|-----|-------------|
| **Health API** | https://localhost:8443 | JWT Token |
| **Keycloak** | http://localhost:8081 | admin/admin |
| **Prometheus** | http://localhost:9090 | - |
| **Grafana** | http://localhost:3000 | admin/admin |
| **Loki** | http://localhost:3100 | - |
| **Jenkins** | http://localhost:8080 | admin/admin |

---

## Quick Start Commands Summary

```
bash
# Clone and navigate
cd secure-health-api

# Generate encryption key
python keygen.py

# Start all services
docker compose up -d

# View logs
docker compose logs -f app

# Stop services
docker compose down

# Rebuild and restart
docker compose up -d --build

# Run tests locally
docker build -t health-api:local ./app
docker run --rm health-api:local python -m pytest -q
```

---

## Troubleshooting

### Port Already in Use

If ports are already in use, modify `docker-compose.yml`:

```
yaml
ports:
  - "8082:8081"  # Change host port from 8081 to 8082
```

### Docker Permission Denied

```
bash
# Linux - add user to docker group
sudo usermod -aG docker $USER
```

### Keycloak Not Starting

```
bash
# Check Keycloak logs
docker compose logs keycloak

# Increase memory if needed
```

---

## Additional Resources

- **Flask**: https://flask.palletsprojects.com/
- **Keycloak**: https://www.keycloak.org/documentation
- **Jenkins**: https://www.jenkins.io/doc/
- **Docker**: https://docs.docker.com/
- **Prometheus**: https://prometheus.io/docs/
- **Grafana**: https://grafana.com/docs/
