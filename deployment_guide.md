# CineBook End-to-End Local Deployment Guide (New Machine)

This guide provides step-by-step instructions to deploy the complete **CineBook** microservice ecosystem on a new machine using **Docker Desktop**, **Kubernetes (k8s)**, and **Helm 3**.

---

## 📋 1. Prerequisites & Tool Installation

Ensure the following tools are installed on the new machine:

1. **Git**
2. **Java Development Kit (JDK 21)**
3. **Apache Maven (3.9+)**
4. **Node.js (v18+) & npm**
5. **Docker Desktop** (with **Kubernetes** enabled in Settings -> Kubernetes -> Enable Kubernetes)
6. **Helm (v3+)**

---

## 🚀 2. Infrastructure Setup (Databases, Message Broker, Identity)

Before launching microservices, start shared infrastructure components using Docker Compose:

```powershell
# Navigate to local infrastructure folder
cd cinebook-infra/local

# Spin up PostgreSQL, Kafka, Keycloak, Apicurio Schema Registry, Axon Server, and Temporal
docker compose -f docker-compose.dev.yml up -d
```

### 📊 Infrastructure Web UI & Observability Dashboards

| Service | Web UI / Interface URL | Credentials / Details |
| :--- | :--- | :--- |
| **Axon Server** | [http://localhost:8024](http://localhost:8024) | Axon Event Store & Command Routing Dashboard |
| **Grafana** | [http://localhost:3000](http://localhost:3000) | Observability Dashboard (Default: `admin` / `admin`) |
| **OpenSearch Dashboards** | [http://localhost:5601](http://localhost:5601) | Log & Search Analytics (User: `admin`, Password: `admin`) |
| **Prometheus** | [http://localhost:9090](http://localhost:9090) | Metrics & Alerting Engine |
| **Temporal UI** | [http://localhost:8089](http://localhost:8089) | Saga Workflow Orchestration UI |
| **Keycloak (OIDC Auth)** | [http://localhost:8080](http://localhost:8080) | Realm: `cinebook` \| Admin: `admin` / `admin` |
| **Kafka UI** | [http://localhost:9091](http://localhost:9091) | Kafka Topic & Message Inspector |
| **Apicurio (Schema Registry)** | [http://localhost:8081](http://localhost:8081) | Avro Schema Repository |
| **Adminer (DB Web Client)** | [http://localhost:8082](http://localhost:8082) | PostgreSQL Web Client |

### 🔌 Backend Infrastructure Connection Endpoints

| Service | Host & Port | Description |
| :--- | :--- | :--- |
| **Axon Server (gRPC)** | `localhost:8124` | Application gRPC Connection |
| **OpenSearch API** | `https://localhost:9200` | OpenSearch Engine API (Basic Auth: `admin` / `admin`) |
| **OTEL Collector (OTLP gRPC)** | `localhost:4317` | OpenTelemetry Traces/Metrics gRPC |
| **OTEL Collector (OTLP HTTP)** | `localhost:4318` | OpenTelemetry Traces/Metrics HTTP |
| **Kafka Broker** | `localhost:9092` / `localhost:29092` | Kafka Bootstrap Server |
| **Temporal Server** | `localhost:7233` | Temporal gRPC Endpoint |
| **PostgreSQL** | `localhost:5432` | DB: `cinebook_booking_db` \| User: `postgres`, Password: `password` |
| **Redis** | `localhost:6379` | User Profile & Caching Layer |


---

## 🛠️ 3. Build & Containerize Microservices

Build executable JARs and Docker images for each microservice:

### A. Core Shared Libraries (Dependencies)
Build and install core common libraries into local Maven repository first:

```powershell
# 1. CineBook Core Common Library
cd cinebook-core-common
mvn clean install -DskipTests
cd ..

# 2. CineBook Common Messaging Library
cd cinebook-common-messaging
mvn clean install -DskipTests
cd ..
```

### B. Microservice Builds & Docker Images
Run the following commands for each microservice folder:

```powershell
# 1. User Management Service
cd user-management-service
mvn clean package -DskipTests
docker build -t user-management-service:latest .
cd ..

# 2. Cinema Service
cd cinema-service
mvn clean package -DskipTests
docker build -t cinema-service:latest .
cd ..

# 3. Booking Service
cd booking-service
mvn clean package -DskipTests
docker build -t booking-service:latest .
cd ..

# 4. Payment Service
cd payment-service
mvn clean package -DskipTests
docker build -t payment-service:latest .
cd ..

# 5. Notification Service
cd notification-service
mvn clean package -DskipTests
docker build -t notification-service:latest .
cd ..

# 6. Location Service
cd location-service
mvn clean package -DskipTests
docker build -t location-service:latest .
cd ..
```

---

## ☸️ 4. Kubernetes Deployment via Helm

### Step 1 — Create Kubernetes Namespace
```powershell
kubectl create namespace cinebook
```

### Step 2 — Deploy Services using Helm Charts
Navigate to `cinebook-infra/helm`:

```powershell
cd cinebook-infra/helm

# Deploy User Management Service
helm install user-management-service ./cinebook-microservice -f values-user-management-local.yaml -n cinebook

# Deploy Cinema Service
helm install cinema-service ./cinebook-microservice -f values-cinema-local.yaml -n cinebook

# Deploy Booking Service
helm install booking-service ./cinebook-microservice -f values-booking-local.yaml -n cinebook

# Deploy Payment Service
helm install payment-service ./cinebook-microservice -f values-payment-local.yaml -n cinebook

# Deploy Notification Service
helm install notification-service ./cinebook-microservice -f values-notification-local.yaml -n cinebook

# Deploy Location Service
helm install location-service ./cinebook-microservice -f values-location-local.yaml -n cinebook
```

### Step 3 — Verify Pod Status
```powershell
kubectl get pods -n cinebook
```
*All pods should transition to `Running` status (1/1).*

---

## 🔌 5. Port Forwarding & Services Exposure

### Start Port Forwarding (Background Jobs)
To allow local development access and frontend communication, run background port forwarding jobs:

```powershell
Start-Job -ScriptBlock { kubectl port-forward -n cinebook svc/user-management-service-svc 8086:8086 }
Start-Job -ScriptBlock { kubectl port-forward -n cinebook svc/cinema-service-svc 8083:8083 }
Start-Job -ScriptBlock { kubectl port-forward -n cinebook svc/booking-service-svc 8085:8085 }
Start-Job -ScriptBlock { kubectl port-forward -n cinebook svc/payment-service-svc 7085:7085 }
Start-Job -ScriptBlock { kubectl port-forward -n cinebook svc/notification-service-svc 7083:7083 }
Start-Job -ScriptBlock { kubectl port-forward -n cinebook svc/location-service-svc 7086:7086 }
```

### Stop Port Forwarding (Background Jobs)
To stop and clean up all active port forwarding background jobs:

```powershell
Get-Job | Stop-Job | Remove-Job
```

---

## 💻 6. Frontend Setup & Execution

1. Navigate to the frontend directory:
   ```powershell
   cd cinebook-frontend
   ```

2. Install dependencies:
   ```powershell
   npm install
   ```

3. Launch development server:
   ```powershell
   npm run dev
   ```

4. Open `http://localhost:5173` in your web browser.

---

## 🔄 7. Upgrading or Redeploying a Single Service

If you modify code or configuration for a single microservice (e.g. `notification-service`):

```powershell
# 1. Rebuild JAR & Docker Image
cd notification-service
mvn clean package -DskipTests
docker build -t notification-service:latest .

# 2. Upgrade Helm Release & Restart Pod
cd ../cinebook-infra/helm
helm upgrade notification-service ./cinebook-microservice -f values-notification-local.yaml -n cinebook
kubectl rollout restart deployment/notification-service-deployment -n cinebook
```
