# 🏏 IPL Live Score Platform

> **Real-time IPL match scores** powered by Spring Boot, Redis, Kubernetes on AWS EKS — built with production-grade DevOps practices.

[![CI/CD](https://github.com/naveenvelanati/ipl-live-score-platform/actions/workflows/ci-cd.yml/badge.svg)](https://github.com/naveenvelanati/ipl-live-score-platform/actions)
[![Java](https://img.shields.io/badge/Java-21-orange)](https://openjdk.org/)
[![Spring Boot](https://img.shields.io/badge/Spring%20Boot-3.2.5-brightgreen)](https://spring.io/projects/spring-boot)
[![Redis](https://img.shields.io/badge/Redis-7.2-red)](https://redis.io/)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-EKS-326CE5)](https://aws.amazon.com/eks/)
[![Terraform](https://img.shields.io/badge/IaC-Terraform-7B42BC)](https://www.terraform.io/)

---

## 📐 Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                        AWS (ap-south-1)                             │
│                                                                     │
│   ┌─────────────────────────────────────────────────────────────┐   │
│   │                    VPC (10.0.0.0/16)                        │   │
│   │                                                             │   │
│   │  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐   │   │
│   │  │ Public Subnet│   │ Public Subnet│   │ Public Subnet│   │   │
│   │  │ ap-south-1a  │   │ ap-south-1b  │   │ ap-south-1c  │   │   │
│   │  │  (NAT GW)    │   │  (NAT GW)    │   │  (NAT GW)    │   │   │
│   │  └──────┬───────┘   └──────┬───────┘   └──────┬───────┘   │   │
│   │         │                  │                   │           │   │
│   │  ┌──────▼───────────────────▼───────────────────▼───────┐  │   │
│   │  │           AWS Network Load Balancer (NLB)            │  │   │
│   │  │              ipl-live-score-service                  │  │   │
│   │  └──────────────────────────┬───────────────────────────┘  │   │
│   │                             │                               │   │
│   │  ┌──────────────┐   ┌───────┴──────┐   ┌──────────────┐   │   │
│   │  │Private Subnet│   │Private Subnet│   │Private Subnet│   │   │
│   │  │ ap-south-1a  │   │ ap-south-1b  │   │ ap-south-1c  │   │   │
│   │  │              │   │              │   │              │   │   │
│   │  │  ┌────────┐  │   │  ┌────────┐  │   │            │   │   │
│   │  │  │EKS Pod │  │   │  │EKS Pod │  │   │            │   │   │
│   │  │  │(Spring)│  │   │  │(Spring)│  │   │            │   │   │
│   │  │  └───┬────┘  │   │  └───┬────┘  │   │            │   │   │
│   │  └──────┼────────┘   └──────┼────────┘   └────────────┘   │   │
│   │         │                  │                               │   │
│   │         └────────┬─────────┘                               │   │
│   │                  │                                         │   │
│   │           ┌──────▼──────┐                                  │   │
│   │           │    Redis     │                                  │   │
│   │           │  (ClusterIP) │                                  │   │
│   │           │  TTL: 30s   │                                  │   │
│   │           └─────────────┘                                  │   │
│   └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│   ┌─────────┐   ┌──────────┐   ┌───────────────────────────────┐   │
│   │   ECR   │   │    S3    │   │      DynamoDB (State Lock)    │   │
│   │  Images │   │TF State  │   │    ipl-terraform-locks        │   │
│   └─────────┘   └──────────┘   └───────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘

External:
  Browser ──► NLB ──► Spring Boot Pods ──► [Redis Cache]
                                       └──► CricAPI (on cache miss)

GitHub Actions:
  Push to main ──► Maven Build ──► Docker Build ──► ECR Push ──► EKS Deploy
```

---

## 🚀 Request Flow

```
Browser
  │
  │ GET /api/ipl-matches
  ▼
Spring Boot Controller
  │
  ├─► Redis Cache (ipl_matches key)
  │       │
  │   HIT ├──► Return cached list (< 1ms)
  │       │
  │  MISS └──► WebClient ──► CricAPI /v1/currentMatches
  │                              │
  │                              ├──► Filter IPL matches
  │                              ├──► Store in Redis (TTL = 30s)
  │                              └──► Return to browser
  │
  └─► JSON Response { status, message, data: [...matches] }
```

---

## 🧱 Tech Stack

| Layer         | Technology                          |
|---------------|-------------------------------------|
| Backend       | Java 21 · Spring Boot 3.2.5         |
| HTTP Client   | Spring WebFlux WebClient            |
| Cache         | Redis 7.2 · Spring Cache            |
| Monitoring    | Micrometer · Prometheus · Actuator  |
| Container     | Docker (multi-stage build)          |
| Orchestration | Kubernetes 1.29 · AWS EKS           |
| IaC           | Terraform 1.7+ · AWS Modules        |
| CI/CD         | GitHub Actions                      |
| Registry      | AWS ECR                             |
| Frontend      | Vanilla HTML · CSS · JavaScript     |
| External API  | CricAPI v1 (`/currentMatches`)      |

---

## 📁 Project Structure

```
ipl-live-score-platform/
│
├── backend/                          # Spring Boot application
│   ├── src/main/java/com/naveen/ipl/
│   │   ├── IplApplication.java       # Entry point + @EnableCaching
│   │   ├── controller/
│   │   │   └── MatchController.java  # GET /api/ipl-matches
│   │   ├── service/
│   │   │   └── MatchService.java     # Business logic + Redis caching
│   │   ├── config/
│   │   │   ├── RedisConfig.java      # RedisCacheManager (TTL 30s)
│   │   │   ├── WebClientConfig.java  # Timeout-aware WebClient bean
│   │   │   └── CorsConfig.java       # CORS for frontend
│   │   ├── model/
│   │   │   ├── Match.java            # Domain model (Serializable)
│   │   │   └── CricApiResponse.java  # API response envelope
│   │   └── exception/
│   │       └── GlobalExceptionHandler.java
│   ├── src/main/resources/
│   │   └── application.yml
│   └── pom.xml
│
├── docker/
│   └── Dockerfile                    # Multi-stage build (Maven → JRE)
│
├── k8s/
│   ├── deployment.yaml               # 2 replicas + probes + HPA + RBAC
│   └── service.yaml                  # NLB LoadBalancer + Redis + HPA
│
├── terraform/
│   ├── main.tf                       # VPC · EKS · ECR · S3 · DynamoDB
│   ├── variables.tf                  # All input variables with validation
│   └── outputs.tf                    # Cluster endpoint · ECR URL · etc.
│
├── frontend/
│   └── index.html                    # Single-page score dashboard
│
├── .github/workflows/
│   └── ci-cd.yml                     # Build → Docker → ECR → EKS
│
└── README.md
```

---

## ⚙️ Prerequisites

| Tool        | Version  | Install                                                       |
|-------------|----------|---------------------------------------------------------------|
| Java        | 21+      | https://adoptium.net                                          |
| Maven       | 3.9+     | https://maven.apache.org                                      |
| Docker      | 24+      | https://docs.docker.com/get-docker/                           |
| kubectl     | 1.29+    | https://kubernetes.io/docs/tasks/tools/                       |
| Terraform   | 1.7+     | https://developer.hashicorp.com/terraform/downloads           |
| AWS CLI     | 2.x      | https://aws.amazon.com/cli/                                   |
| Redis       | 7.x      | https://redis.io/download (local dev)                         |

---

## 🛠️ Local Development Setup

### 1. Clone the repository
```bash
git clone https://github.com/naveenvelanati/ipl-live-score-platform.git
cd ipl-live-score-platform
```

### 2. Start Redis locally
```bash
# Using Docker (recommended)
docker run -d --name redis-ipl -p 6379:6379 redis:7.2-alpine

# Or install natively
brew install redis && redis-server   # macOS
```

### 3. Get your CricAPI key
Sign up at https://cricapi.com and copy your API key.

### 4. Run the Spring Boot backend
```bash
cd backend

# Export required environment variables
export CRICAPI_KEY="your_cricapi_key_here"
export REDIS_HOST="localhost"
export REDIS_PORT="6379"

# Build and run
mvn spring-boot:run
```

Backend starts at → http://localhost:8080

### 5. Open the frontend
```bash
# Simply open in your browser
open frontend/index.html

# Or serve with any static server
npx serve frontend/
```

### 6. Test the API
```bash
# Fetch IPL matches
curl http://localhost:8080/api/ipl-matches | jq .

# Force refresh cache
curl -X POST http://localhost:8080/api/ipl-matches/refresh | jq .

# Health check
curl http://localhost:8080/actuator/health | jq .

# Prometheus metrics
curl http://localhost:8080/actuator/prometheus
```

---

## 🐳 Docker Build

```bash
# Build from project root
docker build -f docker/Dockerfile -t ipl-live-score:local ./backend

# Run with environment variables
docker run -d \
  --name ipl-app \
  -p 8080:8080 \
  -e CRICAPI_KEY="your_key" \
  -e REDIS_HOST="host.docker.internal" \
  -e REDIS_PORT="6379" \
  ipl-live-score:local

# Check logs
docker logs -f ipl-app
```

---

## ☁️ AWS Deployment

### Step 1 — Configure AWS CLI
```bash
aws configure
# Enter: Access Key ID, Secret Access Key, Region (ap-south-1), Output (json)
```

### Step 2 — Bootstrap Terraform remote state (one-time)
```bash
# Create S3 bucket and DynamoDB table manually first, then:
cd terraform
terraform init
```

### Step 3 — Review and apply infrastructure
```bash
# Preview what will be created
terraform plan -var="environment=production"

# Apply (creates VPC, EKS, ECR, S3, DynamoDB)
terraform apply -var="environment=production"

# Configure kubectl
$(terraform output -raw kubectl_config_command)
```

### Step 4 — Push Docker image to ECR
```bash
# Get ECR URL from Terraform output
ECR_URL=$(terraform output -raw ecr_repository_url)
AWS_REGION="ap-south-1"

# Authenticate Docker to ECR
aws ecr get-login-password --region $AWS_REGION | \
  docker login --username AWS --password-stdin $ECR_URL

# Build, tag, push
docker build -f docker/Dockerfile -t ipl-live-score ./backend
docker tag ipl-live-score:latest $ECR_URL:latest
docker push $ECR_URL:latest
```

### Step 5 — Deploy to EKS
```bash
# Update image in deployment
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml

# Create secret with CricAPI key
kubectl create secret generic ipl-secrets \
  --namespace=ipl \
  --from-literal=cricapi-key="YOUR_KEY"

# Wait for rollout
kubectl rollout status deployment/ipl-live-score -n ipl

# Get the LoadBalancer DNS
kubectl get svc ipl-live-score-service -n ipl
```

---

## 🔄 CI/CD Pipeline

### GitHub Secrets Required
Add these in **Repository → Settings → Secrets → Actions**:

| Secret Name             | Description                            |
|-------------------------|----------------------------------------|
| `AWS_ACCESS_KEY_ID`     | IAM user access key                    |
| `AWS_SECRET_ACCESS_KEY` | IAM user secret key                    |
| `CRICAPI_KEY`           | Your CricAPI.com API key               |

### Pipeline Stages

```
┌──────────────┐    ┌────────────────────┐    ┌───────────────────┐
│ 1. Build &   │───►│ 2. Docker Build &  │───►│ 3. Deploy to EKS  │
│    Test      │    │    Push to ECR     │    │    (main only)    │
│              │    │                    │    │                   │
│ mvn verify   │    │ docker buildx      │    │ kubectl apply     │
│ JUnit tests  │    │ ECR push           │    │ Rolling update    │
│              │    │ Image scan         │    │ Rollback on fail  │
└──────────────┘    └────────────────────┘    └───────────────────┘
```

Trigger: **Push to `main`** or **Pull Request** (build/test only on PR).

---

## 📊 Monitoring & Observability

### Prometheus Metrics
The app exposes metrics at `/actuator/prometheus`. Key metrics:

| Metric                        | Description                        |
|-------------------------------|------------------------------------|
| `ipl.matches.get_seconds`     | Latency of GET /api/ipl-matches    |
| `ipl.matches.refresh_seconds` | Latency of cache force-refresh     |
| `redis.commands.duration`     | Redis operation latency            |
| `jvm.memory.used`             | JVM heap usage                     |
| `http.server.requests`        | All HTTP request metrics           |

### Health Endpoints
```bash
# Overall health
GET /actuator/health

# Liveness (used by K8s livenessProbe)
GET /actuator/health/liveness

# Readiness (used by K8s readinessProbe)
GET /actuator/health/readiness

# App info
GET /actuator/info
```

---

## 🔐 Security Best Practices Applied

- ✅ API key injected via environment variable (never in source code)
- ✅ Non-root container user (UID 1000)
- ✅ Read-only root filesystem
- ✅ All Linux capabilities dropped
- ✅ ECR image scanning on push
- ✅ S3 state bucket encrypted + public access blocked
- ✅ EKS nodes in private subnets only
- ✅ K8s Secrets for sensitive values
- ✅ Resource limits on all containers
- ✅ CORS restricted to API paths

---

## 🧹 Cleanup

```bash
# Delete Kubernetes resources
kubectl delete namespace ipl

# Destroy AWS infrastructure (⚠️ DESTRUCTIVE)
cd terraform
terraform destroy -var="environment=production"
```

---

## 🤝 Contributing

1. Fork the repo
2. Create a feature branch: `git checkout -b feature/my-feature`
3. Commit: `git commit -m 'Add my feature'`
4. Push: `git push origin feature/my-feature`
5. Open a Pull Request

---

## 👤 Author

**Velanati Naveen Kumar**
**DevOps Engineer**

> Skills: **CI/CD** | **AWS** | **Kubernetes** | **Terraform** | **GitOps** | **Automation**

- 🌐 GitHub: [@naveenvelanati](https://github.com/naveenvelanati)
- 💼 LinkedIn: [Velanati Naveen Kumar](https://linkedin.com/in/naveenvelanati)

---

## 📄 License

MIT License — feel free to use this project as a template for your own DevOps portfolio.
