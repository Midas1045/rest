# Bank Backend API — CI/CD Pipeline Documentation

## Table of Contents
1. [Project Overview](#project-overview)
2. [Architecture](#architecture)
3. [Repositories](#repositories)
4. [Pipeline Overview](#pipeline-overview)
5. [Workflows](#workflows)
   - [SecOps Workflow](#1-secops-workflow)
   - [Bank Backend Pipeline](#2-bank-backend-pipeline)
6. [Self-Hosted Runner Setup](#self-hosted-runner-setup)
7. [Required Secrets](#required-secrets)
8. [K6 Stress Testing](#k6-stress-testing)
9. [K6 Baseline Health Check Results](#k6-baseline-health-check-results)
10. [Access Requirements](#access-requirements)
11. [Troubleshooting](#troubleshooting)
12. [Ingress Configuration](#ingress-configuration)

---

## Project Overview

This project implements a fully automated CI/CD pipeline for the **Bank Backend API** — a Java/Spring Boot application. The pipeline covers:

- Security scanning with **SonarQube**
- Docker image build and push to **Amazon ECR**
- Kubernetes manifest auto-update
- Stress testing with **K6**
- Deployment via **ArgoCD** or equivalent GitOps tooling

---

## Architecture

```
Developer Push
      │
      ▼
┌─────────────┐
│  SecOps     │  → Maven Build → SonarQube Scan
│  Workflow   │
└──────┬──────┘
       │ on success
       ▼
┌──────────────────────┐
│  Bank Backend        │
│  Pipeline            │
│                      │
│  1. Build Docker     │  → Push to Amazon ECR
│  2. Update K8s       │  → Push to kubernetes-manifest repo
│  3. K6 Stress Test   │  → Run against deployed backend
└──────────────────────┘
       │
       ▼
┌─────────────┐
│  K8s        │  → ArgoCD detects manifest change → Deploys new image
│  Cluster    │
└─────────────┘
```

---

## Repositories

| Repository | Purpose |
|---|---|
| `Pod-4-project/<backend-repo>` | Spring Boot backend source code + CI/CD workflows |
| `Pod-4-project/kubernetes-manifest` | Kubernetes deployment manifests (updated by pipeline) |

> ⚠️ **Note:** The `kubernetes-manifest` repo default branch is `Main` (capital M) — not `main`. Always use `git push origin Main` or `git push origin HEAD:Main` to avoid push failures.

---

## Pipeline Overview

The pipeline is split into two workflows that run sequentially:

| Workflow | Trigger | Purpose |
|---|---|---|
| `SecOps` | Push to `main` or `feature/backend_pretest` | Build and security scan |
| `Bank Backend Pipeline` | After SecOps completes successfully | Build image, push to ECR, update manifest, stress test |

---

## Workflows

### 1. SecOps Workflow

**File:** `.github/workflows/secops.yml`

**Trigger:**
```yaml
on:
  push:
    branches:
      - main
      - feature/backend_pretest
```

**Jobs:**

| Job | Steps |
|---|---|
| `build` | Checkout → Set up JDK 17 → Maven clean verify → SonarQube scan |

**Required Secrets:**

| Secret | Description |
|---|---|
| `NVD_API_KEY_01` | NVD API key for dependency checks |
| `SONAR_TOKEN` | SonarQube authentication token |
| `SONAR_HOST_URL` | SonarQube server URL |

---

### 2. Bank Backend Pipeline

**File:** `.github/workflows/bank-backend-pipeline.yml`

**Trigger:**
```yaml
on:
  workflow_run:
    workflows: ["SecOps"]
    types:
      - completed
```

> ⚠️ **Important:** This workflow file MUST exist on the `main` branch for the `workflow_run` trigger to work. GitHub only reads `workflow_run` listeners from the default branch.

**Jobs:**

#### Job 1: `Continues-Integration_build_and_push`
| Step | Description |
|---|---|
| Checkout repository | Pull latest source code |
| Set up Docker Buildx | Configure Docker build environment |
| Configure AWS credentials | Authenticate with AWS using secrets |
| Log in to Amazon ECR | Docker login to ECR registry |
| Build Docker image | Build and tag with run number + latest |
| Push Docker images to ECR | Push both versioned and latest tags |
| Clean up local images | Remove local images to free disk space |

#### Job 2: `Continues-Update_k8s_manifest`
| Step | Description |
|---|---|
| Clone `kubernetes-manifest` repo | Using PAT for authentication |
| Update image tag in `backendapi.yaml` | `sed` replaces old image tag with new run number |
| Verify update | `grep` confirms the new tag is present |
| Commit and push | Pushes updated manifest back to `main` |

#### Job 3: `Stress-Test` *(to be added)*
| Step | Description |
|---|---|
| Install K6 | Download and install K6 binary |
| Run stress test | Execute `./k6/stress-test.js` against the backend |

**Environment Variables:**

| Variable | Value |
|---|---|
| `ECR_REGISTRY` | `737600811222.dkr.ecr.us-east-1.amazonaws.com` |
| `ECR_REPOSITORY` | `bank-backendapi` |
| `AWS_REGION` | `us-east-1` |
| `IMAGE_TAG` | `${{ github.run_number }}` |

**Required Secrets:**

| Secret | Description |
|---|---|
| `AWS_ACCESS_KEY_ID` | AWS IAM access key |
| `AWS_SECRET_ACCESS_KEY` | AWS IAM secret key |
| `GIT_USERNAME` | GitHub username for manifest repo access |
| `GIT_PASSWORD` | GitHub PAT with `repo` and `workflow` scope |

---

## Self-Hosted Runner Setup

The pipeline runs on a self-hosted GitHub Actions runner deployed as a Docker container on an AWS EC2 instance.

### Prerequisites
- AWS EC2 instance (Ubuntu 22.04 recommended)
- SSH access (`.pem` key + public IP)
- Docker and Docker Compose installed on EC2
- GitHub PAT with `repo` and `workflow` scope
- Repo admin access on the backend repo

### Files

| File | Purpose |
|---|---|
| `Dockerfile` | Builds the runner image (Ubuntu 22.04 + Docker + Terraform + Node.js + GitHub runner) |
| `docker-compose.yaml` | Runs the runner container with Docker socket mount |
| `entrypoint.sh` | Fetches registration token and registers runner with GitHub on startup |
| `.env` | Environment variables for the runner (not committed to Git) |

### Step-by-Step Setup

**1. SSH into EC2**
```bash
ssh -i your-key.pem ubuntu@<ec2-public-ip>
```

**2. Install Docker**
```bash
sudo apt update && sudo apt install -y docker.io docker-compose
sudo systemctl enable docker && sudo systemctl start docker
sudo usermod -aG docker ubuntu
newgrp docker
```

**3. Transfer Runner Files to EC2**

From your local machine:
```bash
scp -i your-key.pem Dockerfile docker-compose.yaml entrypoint.sh ubuntu@<ec2-ip>:~/github-runner/
```

**4. Create `.env` File**
```bash
cd ~/github-runner
nano .env
```

```env
REPO=pod9project/<backend-repo-name>
GITHUB_PAT=ghp_yourPersonalAccessTokenHere
RUNNER_NAME=backend-ec2-runner
```

> ⚠️ Never commit the `.env` file to Git. Add it to `.gitignore`.

**5. Build and Start the Runner**
```bash
docker-compose up -d --build
```

**6. Verify Registration**
```bash
docker logs github-runner
```

Expected output:
```
Requesting registration token for pod9project/<repo>...
Registering runner: backend-ec2-runner
√ Connected to GitHub
Starting runner....
```

Then confirm on GitHub: **Repo → Settings → Actions → Runners** — runner should show as **Idle ✅**

**7. Update Pipeline to Use Self-Hosted Runner**

In `bank-backend-pipeline.yml`, change:
```yaml
runs-on: ubuntu-latest
```
to:
```yaml
runs-on: [self-hosted, docker]
```

### Runner Labels

The runner registers with these labels (defined in `entrypoint.sh`):

| Label | Purpose |
|---|---|
| `self-hosted` | Identifies as self-hosted runner |
| `docker` | Has Docker available |
| `terraform` | Has Terraform installed |

---

## Required Secrets

All secrets are stored under **Repo → Settings → Secrets and Variables → Actions**

| Secret Name | Used In | Description |
|---|---|---|
| `AWS_ACCESS_KEY_ID` | Backend Pipeline | AWS IAM key for ECR login and image push |
| `AWS_SECRET_ACCESS_KEY` | Backend Pipeline | AWS IAM secret for ECR login and image push |
| `BACKEND_URL` | K6 Stress Test | Full backend URL e.g. `https://bankapi.rivetrecords.online` |
| `DB_PASSWORD` | App properties + K8s Secret | Database password for Spring Boot DB connection |
| `DB_URL` | App properties + K8s ConfigMap | Full JDBC connection string e.g. `jdbc:mysql://host:3306/db` |
| `DB_USERNAME` | App properties + K8s Secret | Database username for Spring Boot DB connection |
| `GIT_FGPAT` | K8s manifest checkout | Fine-grained PAT scoped to `Pod-4-project` org for cloning and pushing `kubernetes-manifest` |
| `GIT_PASSWORD` | Pipeline — manifest clone/push | Classic PAT (backup) for git operations |
| `GIT_USERNAME` | Pipeline — manifest clone/push | GitHub username e.g. `Midas1045` |
| `NVD_API_KEY_01` | SecOps | NVD API key for Maven dependency vulnerability scanning |
| `SONAR_HOST_URL` | SecOps | SonarQube server URL |
| `SONAR_TOKEN` | SecOps | SonarQube authentication token |

> **Note:** `GIT_FGPAT` is the primary token used for manifest repo operations. It must be a Fine-grained PAT with `Pod-4-project` as the resource owner and `Contents: Read and write`, `Actions: Read and write`, `Workflows: Read and write` permissions.

---

## Where Secrets Are Referenced

### `application.properties`
```properties
# DB credentials injected at runtime from environment variables
spring.datasource.url=jdbc:mysql://${DB_URL}
spring.datasource.username=${DB_USERNAME}
spring.datasource.password=${DB_PASSWORD}
```
These placeholders are populated by the pipeline injecting GitHub secrets as environment variables at build/run time. Never hardcode credentials in this file.

---

### `configmap.yaml`
Non-sensitive database config stored here:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: bank-backend-config
data:
  DB_HOST: production-mysql-db.cax8262mgyle.us-east-1.rds.amazonaws.com
  DB_PORT: "3306"
  DB_NAME: online_banking_system
```

---

### `secret.yaml`
Sensitive credentials stored as a Kubernetes Secret — never pushed to GitHub:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: bank-backend-secret
type: Opaque
stringData:
  DB_USERNAME: your-db-username
  DB_PASSWORD: your-db-password
```
> ⚠️ `secret.yaml` must be in `.gitignore`. Apply it to the cluster manually via `kubectl apply -f secret.yaml`. The file is handed to the K8s team securely — not via GitHub.

---

### `backendapi.yaml` (in `kubernetes-manifest`)
The deployment references both ConfigMap and Secret:
```yaml
env:
  - name: DB_HOST
    valueFrom:
      configMapKeyRef:
        name: bank-backend-config
        key: DB_HOST
  - name: DB_USERNAME
    valueFrom:
      secretKeyRef:
        name: bank-backend-secret
        key: DB_USERNAME
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: bank-backend-secret
        key: DB_PASSWORD
```
The image tag in this file is automatically updated by the pipeline on every successful build:
```yaml
image: 737600811222.dkr.ecr.us-east-1.amazonaws.com/bank-backendapi:latest
```
`sed` replaces `:latest` with the actual run number e.g. `:23`.

---

### K6 Stress Test
```yaml
env:
  BACKEND_URL: ${{ secrets.BACKEND_URL }}
run: |
  k6 run --insecure-skip-tls-verify --env BACKEND_URL=$BACKEND_URL ./k6/health-check.js
```
`BACKEND_URL` must include the protocol: `https://bankapi.rivetrecords.online`

---

## K6 Stress Testing

K6 stress tests run as the final job in the Backend Pipeline after the k8s manifest is updated. Three scripts run sequentially — light to heavy.

### Folder Structure
```
k6/
├── health-check.js        # 10 virtual users — confirms app is alive
├── single-endpoint.js     # 50 virtual users — targets login endpoint
└── multiple-endpoints.js  # 100 virtual users — login, account, transactions
```

### Pipeline Job
```yaml
Stress-Test:
  runs-on: ubuntu-latest
  needs: Continues-Update_k8s_manifest
  steps:
    - name: Checkout repository
      uses: actions/checkout@v4

    - name: Install K6
      run: |
        curl https://github.com/grafana/k6/releases/download/v0.49.0/k6-v0.49.0-linux-amd64.tar.gz -L | tar xvz
        sudo cp k6-v0.49.0-linux-amd64/k6 /usr/local/bin

    - name: Health Check
      env:
        BACKEND_URL: ${{ secrets.BACKEND_URL }}
      run: |
        echo "Running health check against $BACKEND_URL"
        k6 run --insecure-skip-tls-verify --env BACKEND_URL=$BACKEND_URL ./k6/health-check.js

    - name: Single Endpoint Test
      env:
        BACKEND_URL: ${{ secrets.BACKEND_URL }}
        API_TOKEN: ${{ secrets.API_TOKEN }}
      run: k6 run --insecure-skip-tls-verify --env BACKEND_URL=$BACKEND_URL --env API_TOKEN=$API_TOKEN ./k6/single-endpoint.js

    - name: Multiple Endpoint Test
      env:
        BACKEND_URL: ${{ secrets.BACKEND_URL }}
        API_TOKEN: ${{ secrets.API_TOKEN }}
      run: k6 run --insecure-skip-tls-verify --env BACKEND_URL=$BACKEND_URL --env API_TOKEN=$API_TOKEN ./k6/multiple-endpoints.js
```

### Required Secrets for K6

| Secret | Value |
|---|---|
| `BACKEND_URL` | `https://bankapi.rivetrecords.online` |
| `API_TOKEN` | JWT token from login endpoint (for authenticated requests) |

### Understanding K6 Metrics

| Metric | What it means |
|---|---|
| `http_req_duration avg` | Average time to complete one full request end to end |
| `p(95)<500` | 95% of all requests completed in under 500ms |
| `http_req_failed rate` | Percentage of requests that returned an error |
| `vus` | Virtual users — simulated concurrent users hitting the backend |
| `iterations` | Total number of complete test cycles run |

---

## Access Requirements

| Access | Who Grants It | Why Needed |
|---|---|---|
| Repo admin on backend repo | Org owner | Register runner, manage secrets, manage Actions |
| Write access on `kubernetes-manifest` | Org owner | Pipeline pushes updated image tags to this repo |
| EC2 IP + `.pem` SSH key | AWS account owner | To deploy and manage the self-hosted runner |
| AWS IAM credentials | AWS account owner | To push Docker images to ECR |

---

## Troubleshooting

### `workflow_run` trigger not firing
- Confirm the Bank Backend Pipeline YAML exists on the **`main` branch** — GitHub only reads `workflow_run` listeners from the default branch
- Confirm the SecOps `name:` field matches exactly: `SecOps`
- Check the Actions tab — did SecOps complete with ✅ success?
- **Alternative fix:** Switch to `push` trigger on the same branches as SecOps to run both pipelines in parallel

---

### `git clone` failing with exit code 1 — Authentication failed
Encountered during `Continues-Update_k8s_manifest` job.

**Cause:** Classic PAT did not have org-level access to `Pod-4-project`.

**Fix:** Generate a **Fine-grained PAT** with `Pod-4-project` as the resource owner instead of personal account. Scopes needed: `Contents: Read and write`, `Actions: Read and write`, `Workflows: Read and write`.

---

### `fatal: destination path already exists`
Encountered when both `actions/checkout@v4` and a manual `git clone` were used in the same job.

**Cause:** `actions/checkout@v4` already cloned the repo — manual `git clone` then tried to clone into the same directory.

**Fix:** Use `actions/checkout@v4` with the `repository` and `token` fields instead of a manual clone:
```yaml
- name: Checkout manifest repo
  uses: actions/checkout@v4
  with:
    repository: Pod-4-project/kubernetes-manifest
    token: ${{ secrets.GIT_FGPAT }}
    path: kubernetes-manifest
```

---

### `sed` not replacing image tag — exit code 1
Encountered during manifest update step.

**Cause:** The image line in `backendapi.yaml` had `http://` prefix and no tag:
```yaml
image: http://737600811222.dkr.ecr.us-east-1.amazonaws.com/bank-backendapi
```
`sed` looks for `:.*` pattern — without a tag there is nothing to match.

**Fix:** Update the image line in `backendapi.yaml` to the correct format before running the pipeline:
```yaml
image: 737600811222.dkr.ecr.us-east-1.amazonaws.com/bank-backendapi:latest
```

---

### `git push` failing — `src refspec main does not match any`
Encountered when pushing manifest update to `kubernetes-manifest`.

**Cause:** The default branch of `kubernetes-manifest` is `Main` (capital M) not `main`.

**Fix:**
```bash
git push origin Main
# or
git push origin HEAD:Main
```

---

### K6 — `unsupported protocol scheme ""`
Encountered during health check stress test.

**Cause:** `BACKEND_URL` secret was empty or not passed correctly to K6.

**Fix:** Declare the secret in the `env` block first, then reference it as a shell variable:
```yaml
- name: Health Check
  env:
    BACKEND_URL: ${{ secrets.BACKEND_URL }}
  run: |
    echo "Running health check against $BACKEND_URL"
    k6 run --insecure-skip-tls-verify --env BACKEND_URL=$BACKEND_URL ./k6/health-check.js
```
Always include `https://` in the `BACKEND_URL` secret value e.g. `https://bankapi.rivetrecords.online`.

---

### K6 — `accepts 1 arg(s), received 2`
Encountered when running K6 with `--insecure-skip-tls-verify`.

**Cause:** Missing `--` prefix on the flag — K6 treated `insecure-skip-tls-verify` as the script file argument.

**Fix:**
```bash
# Wrong
k6 run insecure-skip-tls-verify --env BACKEND_URL=$BACKEND_URL ./k6/health-check.js

# Correct
k6 run --insecure-skip-tls-verify --env BACKEND_URL=$BACKEND_URL ./k6/health-check.js
```

---

### K6 — `TypeError: Cannot read property 'includes' of undefined`
Encountered in health check script.

**Cause:** Response body was `null` — the server returned a response with no body content.

**Fix:** Add a null check before calling `.includes()`:
```javascript
'app is UP': (r) => r.body !== null && r.body.includes('UP'),
```

---

### K6 — 404 from nginx on `/actuator/health`
Encountered during health check against `https://bankapi.rivetrecords.online`.

**Cause 1:** Spring Boot Actuator not configured in `application.properties`.
**Fix:** Add:
```properties
management.endpoints.web.exposure.include=health
management.endpoint.health.show-details=always
```
And add to `pom.xml`:
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

**Cause 2:** Backend pod not deployed to the cluster yet — nginx ingress is running but routing to nothing.
**Fix:** Team must apply the deployment manifests to the cluster:
```bash
kubectl apply -f bank-project/backendapi.yaml
kubectl apply -f bank-project/configmap.yaml
kubectl apply -f bank-project/secret.yaml
```

---

### CORS — Frontend showing "Unable to contact the server"

**Cause:** Two problems found in the backend code:

**Problem 1** — CORS was completely disabled in `SecurityConfig.java`:
```java
// ❌ Before
http.csrf(csrf -> csrf.disable())
    .cors(cors -> cors.disable())
```

**Problem 2** — Wrong allowed origin hardcoded in five files:
```java
// ❌ Before
"https://bank.cloudwitches.online"
```

**Fix:** Removed `.cors(cors -> cors.disable())` to allow `WebConfig.java` to handle CORS. Updated allowed origin in all five files:
```java
// ✅ After
"https://bank.rivetrecords.online"
```

Files updated: `OnlineBankingSystemApplication.java`, `WebConfig.java`, `BankAccountController.java`, `BankAccountTransactionController.java`, `BankController.java`

---

### Actuator Health Endpoint — 403 Access Denied

**Cause:** Spring Security was blocking `/actuator/health` — not listed in `permitAll()`.

**Fix:** Added `/actuator/health` to `permitAll()` in `SecurityConfig.java`:
```java
// ❌ Before
.requestMatchers("/api/user/login", "/api/user/admin/register").permitAll()

// ✅ After
.requestMatchers("/api/user/login", "/api/user/admin/register", "/actuator/health").permitAll()
```

**Verification:** After fix, hitting `https://bankapi.rivetrecords.online/actuator/health` returned:
```json
{
  "status": "UP",
  "components": {
    "db": { "status": "UP" },
    "diskSpace": { "status": "UP" },
    "livenessState": { "status": "UP" },
    "readinessState": { "status": "UP" }
  }
}
```

---

### ConfigMap Mismatch — 503 Service Unavailable

Backend pod was starting successfully but returning 503. Environment variables were not being injected into the pod.

**Three problems found:**

**Problem 1** — ConfigMap name mismatch:
```yaml
# ❌ configmap.yaml had:
name: bank-backend-config

# backendapi.yaml expected:
name: bank-config
```

**Problem 2** — Wrong variable name:
```yaml
# ❌ configmap.yaml had:
DB_URL: production-mysql-db.cax8262mgyle.us-east-1.rds.amazonaws.com

# application.properties expected:
DB_HOST: ...
```

**Problem 3** — `DB_URL` incomplete — missing port and database name.

**Fix:** Updated `configmap.yaml`:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: bank-config
data:
  DB_HOST: "production-mysql-db.cax8262mgyle.us-east-1.rds.amazonaws.com"
  DB_PORT: "3306"
  DB_NAME: "production_db"
```

Updated `application.properties`:
```properties
# ❌ Before
spring.datasource.url=jdbc:mysql://${DB_URL}

# ✅ After
spring.datasource.url=jdbc:mysql://${DB_HOST}:${DB_PORT}/${DB_NAME}
```

---

### K6 — 100% failure after app confirmed healthy

**Cause:** `BACKEND_URL` was stored under **Repository Variables** but the pipeline referenced it as a secret:
```yaml
# Pipeline was looking here:
BACKEND_URL: ${{ secrets.BACKEND_URL }}

# But the value was stored here:
Repository Variables → BACKEND_URL
```

**Fix:** Moved `BACKEND_URL` from Repository Variables to **Repository Secrets**:
- GitHub → Settings → Secrets and Variables → Actions → New repository secret
- Name: `BACKEND_URL`
- Value: `https://bankapi.rivetrecords.online`

---

### K6 — TLS certificate mismatch
Encountered: `x509: certificate is valid for ingress.local, not bankapi.rivetrecords.online`

**Cause:** cert-manager issued the certificate for the wrong domain.

**Temporary fix:** Add `--insecure-skip-tls-verify` flag to K6 run command while team fixes the certificate.

**Permanent fix:** Team deletes and reissues the certificate:
```bash
kubectl delete certificate bank-Rivetrecords.online
kubectl delete secret bank-Rivetrecords.online
```
cert-manager will automatically request a new certificate for the correct domain.

---

### ECR push failing
- Confirm `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` secrets are correctly set
- Confirm the IAM user has `AmazonEC2ContainerRegistryFullAccess` policy

---

### Runner shows as offline
```bash
docker logs github-runner
docker-compose restart
```
Check that `GITHUB_PAT` in `.env` has not expired. The `.env` file must never be committed to Git.

---

### SonarQube scan failing
- Confirm `SONAR_TOKEN` and `SONAR_HOST_URL` are correctly set
- Check that the SonarQube server is reachable from the runner

---

## K6 Baseline Health Check Results

First successful run after backend pod deployment — May 2026.

| Metric | Result | Status |
|---|---|---|
| `status is 200` | 100% — 2034 checks passed | ✅ |
| `app is UP` | 100% | ✅ |
| `http_req_failed` | 0.00% | ✅ |
| `http_req_duration avg` | 6.33ms | ✅ Well under 500ms threshold |
| `http_req_duration p(95)` | 8.37ms | ✅ Well under 500ms threshold |
| `http_reqs` | 1017 at 7.25 req/s | ✅ |
| `data_received` | 780 kB | ✅ Real data returning |
| Virtual users | 10 max | ✅ |

All thresholds passed. Backend confirmed healthy and performant.

---

## Ingress Configuration

The backend ingress is managed centrally by the team in `kubernetes-manifest`. Do not apply a separate ingress from the backend repo — it will conflict.

**Team ingress covers:**

| Host | Service | Port |
|---|---|---|
| `bank.rivetrecords.online` | `bank-frontend-service` | 3000 |
| `bankapi.rivetrecords.online` | `bank-backend-service` | 8080 |

**Ingress details:**
- Ingress class: `external-nginx`
- TLS: managed by cert-manager with `http-01-production` cluster issuer
- SSL redirect: enabled

---

*Last updated: May 2026*
