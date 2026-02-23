# Enterprise Patient Management System -- Kubernetes Edition

This repository contains the source code for a distributed Enterprise Patient Management System, rebuilt and deployed using **Kubernetes** as the container orchestration platform. The original architecture -- designed by [Chris Blakely](https://github.com/chrisblakely01/java-spring-microservices) and built with Java Spring Boot -- has been adapted from Docker Compose to a fully declarative Kubernetes deployment strategy.

> **Original Course:** [Build & Deploy a Production-Ready Patient Management System with Microservices: Java Spring Boot](https://youtu.be/tseqdcFfTUY)
>
> **Original Source Code:** [github.com/chrisblakely01/java-spring-microservices](https://github.com/chrisblakely01/java-spring-microservices)

## Architecture Overview

The system decomposes a traditional monolithic health management application into distinct, independently deployable microservices. Each service is packaged as a Docker container and scheduled as a Kubernetes Pod, communicating over a secure cluster network via gRPC, REST HTTP, and asynchronous event streams using Apache Kafka.

### Microservices

| Service | Port | Protocol | Description |
|---|---|---|---|
| **patient-service** | 4000 | REST, gRPC Client, Kafka Producer | Core service for persisting and retrieving patient data using PostgreSQL and Hibernate ORM. Publishes patient events to Kafka and invokes billing via gRPC. |
| **billing-service** | 4001 (HTTP), 9001 (gRPC) | gRPC Server | Provisions financial accounts. Receives synchronous gRPC calls from the patient-service when a new patient is registered. |
| **analytics-service** | -- | Kafka Consumer | Subscribes to Apache Kafka event streams to asynchronously log critical patient registration events. |
| **auth-service** | 4005 | REST | Manages user credentials, password encryption (BCrypt), and issues JSON Web Tokens (JWTs) for stateless authentication. Backed by PostgreSQL. |
| **api-gateway** | 4004 | REST (Spring Cloud Gateway) | The singular entry point for external traffic. Validates JWTs before routing requests to internal services. |

### Supporting Infrastructure

| Component | Image | Description |
|---|---|---|
| **patient-service-db** | `postgres:15` | Dedicated PostgreSQL instance for the patient-service. |
| **auth-service-db** | `postgres:15` | Dedicated PostgreSQL instance for the auth-service. |
| **kafka** | `bitnami/kafka` | Apache Kafka broker for asynchronous event streaming between patient-service and analytics-service. |
| **zookeeper** | `bitnami/zookeeper` | Coordination service required by the Kafka broker. |

### Communication Flow

```
                                    +--------------------+
                                    |   api-gateway      |
                                    |   (Port 4004)      |
                                    +--------+-----------+
                                             |
                          JWT Validation     |     JWT Validation
                         +-------------------+-------------------+
                         |                                       |
                +--------v---------+                   +---------v--------+
                |   auth-service   |                   |  patient-service |
                |   (Port 4005)    |                   |  (Port 4000)     |
                +--------+---------+                   +---+---------+----+
                         |                                 |         |
                   [PostgreSQL]                    gRPC    |         | Kafka
                   auth-service-db                         |         |
                                               +-----------+   +----v-----------+
                                               |               |                |
                                      +--------v--------+   +--v---------------+
                                      | billing-service  |   | analytics-service|
                                      | (HTTP: 4001)     |   | (Kafka Consumer) |
                                      | (gRPC: 9001)     |   +------------------+
                                      +-----------------+
                                                          [PostgreSQL]
                                                          patient-service-db
```

## Why Kubernetes?

The original project used Docker Compose for local orchestration and AWS CDK for cloud deployment. This edition replaces both with Kubernetes, providing:

- **Declarative Infrastructure:** All services, databases, and networking are defined as YAML manifests stored alongside the application code.
- **Self-Healing:** Kubernetes automatically restarts failed containers and reschedules Pods to healthy nodes.
- **Service Discovery:** Internal DNS resolution allows services to communicate by name (e.g., `patient-service`, `kafka`) without hardcoded IP addresses.
- **Scaling:** Horizontal Pod Autoscaling (HPA) can scale individual microservices based on load.
- **Rolling Updates:** Zero-downtime deployments with configurable rollout strategies.
- **Secrets Management:** Kubernetes Secrets securely store database credentials and JWT signing keys, injected as environment variables at runtime.
- **Environment Parity:** The same manifests work on local clusters (Minikube, kind, Docker Desktop) and production cloud providers (GKE, EKS, AKS).

## Technology Stack

| Layer | Technology |
|---|---|
| Language | Java 21 |
| Framework | Spring Boot 3.4.1, Spring Cloud 2024.0.0 |
| Build Tool | Apache Maven 3.9+ (Multi-Module) |
| RPC | gRPC 1.69.0 with Protocol Buffers |
| Messaging | Apache Kafka (Bitnami) |
| Authentication | Spring Security + JWT (jjwt 0.12.6) |
| API Gateway | Spring Cloud Gateway |
| Database | PostgreSQL 15 |
| Containerization | Docker (Multi-Stage Builds) |
| Orchestration | Kubernetes |
| Testing | JUnit 5, RestAssured |

## Prerequisites

Ensure the following tools are installed on your development machine:

- **Docker Desktop** (with Kubernetes enabled) or **Minikube**
- **kubectl** -- Kubernetes CLI
- **Java Development Kit (JDK) 21** (only if building outside Docker)
- **Apache Maven 3.9+** (only if building outside Docker)

## Getting Started

### 1. Build the Docker Images

From the project root directory, build all service images:

```bash
docker build -t patient-service:latest -f patient-service/Dockerfile .
docker build -t billing-service:latest -f billing-service/Dockerfile .
docker build -t analytics-service:latest -f analytics-service/Dockerfile .
docker build -t auth-service:latest -f auth-service/Dockerfile .
docker build -t api-gateway:latest -f api-gateway/Dockerfile .
```

### 2. Deploy to Kubernetes

Apply all Kubernetes manifests:

```bash
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/secrets/
kubectl apply -f k8s/databases/
kubectl apply -f k8s/kafka/
kubectl apply -f k8s/services/
kubectl apply -f k8s/gateway/
```

### 3. Verify the Deployment

```bash
kubectl get pods -n patient-management
kubectl get services -n patient-management
```

### 4. Access the System

Once all pods are running, access the API Gateway:

```bash
kubectl port-forward -n patient-management svc/api-gateway 4004:4004
```

The system is now accessible at `http://localhost:4004`.

## API Endpoints

### Authentication

| Method | Endpoint | Description |
|---|---|---|
| POST | `/auth/login` | Authenticate and receive a JWT |

### Patients (Requires JWT)

| Method | Endpoint | Description |
|---|---|---|
| GET | `/api/patients` | Retrieve all patients |
| POST | `/api/patients` | Register a new patient |
| GET | `/api/patients/{id}` | Retrieve a specific patient |
| PUT | `/api/patients/{id}` | Update a patient record |
| DELETE | `/api/patients/{id}` | Delete a patient record |

### API Documentation

| Method | Endpoint | Description |
|---|---|---|
| GET | `/api-docs/patients` | Patient Service OpenAPI spec |
| GET | `/api-docs/auth` | Auth Service OpenAPI spec |

## Default Credentials

| Field | Value |
|---|---|
| Email | `testuser@test.com` |
| Password | `password` |
| Role | `ADMIN` |

## Maven Build Architecture

This is a Multi-Module Maven project. The root `pom.xml` acts as an aggregator that drives the build lifecycle for the entire architecture.

- **Protobuf Code Generation:** Running `mvn compile` triggers the `os-maven-plugin` and `protobuf-maven-plugin` to automatically download Protocol Buffer compilers and generate Java networking stubs for gRPC and Kafka serialization.
- **Multi-Stage Docker Builds:** Each Dockerfile uses a heavyweight `maven:3.9.9-eclipse-temurin-21` builder stage to compile the Fat JAR offline, then copies only the `.jar` into a minimal `openjdk:21-jdk` runtime image.

## Credits

- **Original Course & Source Code:** [Chris Blakely / Code Jackal](https://www.youtube.com/@codejackal)
- **Kubernetes Adaptation:** Rebuilt for Kubernetes deployment as part of the Software Engineering (SOE) coursework.
