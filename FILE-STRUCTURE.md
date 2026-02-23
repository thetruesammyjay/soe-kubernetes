# Project File Structure

This document outlines the file structure for the **Enterprise Patient Management System -- Kubernetes Edition**. The system is designed as a multi-module Maven project where the root `pom.xml` acts as an aggregator. Kubernetes manifests are organized under the `k8s/` directory, replacing Docker Compose and AWS CDK.

## Directory Tree

```
soe-kubernetes/
│
├── README.md                                   <-- Project overview and getting started guide
├── FILE-STRUCTURE.md                           <-- This architecture map
├── pom.xml                                     <-- [Root Aggregator POM] Manages all sub-modules, Java 21, Spring Cloud versions
├── docker-compose.yml                          <-- [Legacy] Local development fallback without Kubernetes
│
├── k8s/                                        <-- All Kubernetes manifests
│   ├── namespace.yaml                          <-- Defines the `patient-management` namespace
│   │
│   ├── secrets/                                <-- Kubernetes Secrets for credentials
│   │   ├── db-secrets.yaml                     <-- PostgreSQL credentials (username, password)
│   │   └── jwt-secret.yaml                     <-- JWT signing key
│   │
│   ├── databases/                              <-- StatefulSets and Services for PostgreSQL
│   │   ├── patient-db-statefulset.yaml         <-- StatefulSet for patient-service PostgreSQL
│   │   ├── patient-db-service.yaml             <-- ClusterIP Service exposing patient-service-db
│   │   ├── auth-db-statefulset.yaml            <-- StatefulSet for auth-service PostgreSQL
│   │   └── auth-db-service.yaml                <-- ClusterIP Service exposing auth-service-db
│   │
│   ├── kafka/                                  <-- Kafka and Zookeeper deployments
│   │   ├── zookeeper-deployment.yaml           <-- Deployment for Zookeeper coordination service
│   │   ├── zookeeper-service.yaml              <-- ClusterIP Service for Zookeeper
│   │   ├── kafka-statefulset.yaml              <-- StatefulSet for Kafka broker
│   │   └── kafka-service.yaml                  <-- ClusterIP Service exposing Kafka on port 9092
│   │
│   ├── services/                               <-- Deployments and Services for application microservices
│   │   ├── patient-service-deployment.yaml     <-- Deployment for patient-service (replicas, env vars, probes)
│   │   ├── patient-service-svc.yaml            <-- ClusterIP Service on port 4000
│   │   ├── billing-service-deployment.yaml     <-- Deployment for billing-service (HTTP + gRPC ports)
│   │   ├── billing-service-svc.yaml            <-- ClusterIP Service on ports 4001 (HTTP) and 9001 (gRPC)
│   │   ├── analytics-service-deployment.yaml   <-- Deployment for analytics-service (Kafka consumer)
│   │   ├── auth-service-deployment.yaml        <-- Deployment for auth-service (JWT issuer)
│   │   └── auth-service-svc.yaml               <-- ClusterIP Service on port 4005
│   │
│   └── gateway/                                <-- API Gateway deployment
│       ├── api-gateway-deployment.yaml         <-- Deployment for Spring Cloud Gateway
│       └── api-gateway-service.yaml            <-- NodePort or LoadBalancer Service on port 4004
│
├── patient-service/                            <-- Core patient data schema management
│   ├── pom.xml                                 <-- [POM] Spring Web, Data JPA, Validation, PostgreSQL, gRPC, Kafka
│   ├── Dockerfile                              <-- Multi-stage Docker build utilizing Maven compiler dependencies
│   └── src/
│       ├── main/
│       │   ├── java/com/pm/patientservice/
│       │   │   ├── PatientServiceApplication.java
│       │   │   ├── controller/
│       │   │   │   └── PatientController.java  <-- REST endpoints for CRUD patient operations
│       │   │   ├── service/
│       │   │   │   └── PatientService.java     <-- Business logic, triggers gRPC and Kafka calls
│       │   │   ├── repository/
│       │   │   │   └── PatientRepository.java  <-- Spring Data JPA interface for PostgreSQL
│       │   │   ├── model/
│       │   │   │   └── Patient.java            <-- JPA Entity mapped to the patients table
│       │   │   ├── dto/
│       │   │   │   ├── PatientRequestDTO.java  <-- Inbound request validation DTO
│       │   │   │   ├── PatientResponseDTO.java <-- Outbound response DTO
│       │   │   │   └── validators/
│       │   │   │       └── CreatePatientValidationGroup.java
│       │   │   ├── mapper/
│       │   │   │   └── PatientMapper.java      <-- Entity-to-DTO conversion utility
│       │   │   ├── grpc/
│       │   │   │   └── BillingServiceGrpcClient.java  <-- gRPC client stub calling billing-service
│       │   │   ├── kafka/
│       │   │   │   └── KafkaProducer.java      <-- Publishes patient events to Kafka topic
│       │   │   └── exception/
│       │   │       ├── GlobalExceptionHandler.java
│       │   │       ├── EmailAlreadyExistsException.java
│       │   │       └── PatientNotFoundException.java
│       │   ├── proto/
│       │   │   ├── billing_service.proto       <-- gRPC service definition for billing calls
│       │   │   └── patient_event.proto         <-- Protobuf schema for Kafka event payloads
│       │   └── resources/
│       │       └── application.properties      <-- Port 4000, Kafka serializer config
│       └── test/java/com/pm/patientservice/
│           └── PatientServiceApplicationTests.java
│
├── billing-service/                            <-- Provisions financial accounts via gRPC
│   ├── pom.xml                                 <-- [POM] gRPC, Protobuf, Spring Boot, os-maven-plugin, protobuf-maven-plugin
│   ├── Dockerfile
│   └── src/
│       ├── main/
│       │   ├── java/com/pm/billingservice/
│       │   │   ├── BillingServiceApplication.java
│       │   │   └── grpc/
│       │   │       └── BillingGrpcService.java <-- gRPC server implementation processing billing requests
│       │   ├── proto/
│       │   │   └── billing_service.proto       <-- gRPC service and message definitions
│       │   └── resources/
│       │       └── application.properties      <-- Port 4001 (HTTP), Port 9001 (gRPC)
│       └── test/java/com/pm/billingservice/
│           └── BillingServiceApplicationTests.java
│
├── analytics-service/                          <-- Listens to Kafka event streams
│   ├── pom.xml                                 <-- [POM] Kafka, Protobuf data serialization
│   ├── Dockerfile
│   └── src/
│       ├── main/
│       │   ├── java/com/pm/analyticsservice/
│       │   │   ├── AnalyticsServiceApplication.java
│       │   │   └── kafka/
│       │   │       └── KafkaConsumer.java      <-- Subscribes to patient event topic and logs events
│       │   ├── proto/
│       │   │   └── patient_event.proto         <-- Protobuf schema for deserializing Kafka messages
│       │   └── resources/
│       │       └── application.properties
│       └── test/java/com/pm/analyticsservice/
│           └── AnalyticsServiceApplicationTests.java
│
├── auth-service/                               <-- User authentication and JWT issuer
│   ├── pom.xml                                 <-- [POM] Spring Security, JWT (jjwt), PostgreSQL, SpringDoc OpenAPI
│   ├── Dockerfile
│   └── src/main/java/com/pm/authservice/
│       ├── AuthServiceApplication.java
│       ├── config/
│       │   └── SecurityConfig.java             <-- Spring Security filter chain configuration
│       ├── controller/
│       │   └── AuthController.java             <-- Login endpoint issuing JWTs
│       ├── service/
│       │   ├── AuthService.java                <-- Authentication logic and JWT generation
│       │   └── UserService.java                <-- User lookup from PostgreSQL
│       ├── repository/
│       │   └── UserRepository.java             <-- Spring Data JPA interface for users table
│       ├── model/
│       │   └── User.java                       <-- JPA Entity for user credentials
│       ├── dto/
│       │   ├── LoginRequestDTO.java            <-- Inbound login payload
│       │   └── LoginResponseDTO.java           <-- JWT response payload
│       └── util/
│           └── JwtUtil.java                    <-- JWT signing, parsing, and validation utility
│
├── api-gateway/                                <-- Secure entry point validating security tokens
│   ├── pom.xml                                 <-- [POM] Spring Cloud Gateway, WebFlux
│   ├── Dockerfile
│   └── src/main/
│       ├── java/com/pm/apigateway/
│       │   ├── ApiGatewayApplication.java
│       │   ├── filter/
│       │   │   └── JwtValidationGatewayFilterFactory.java  <-- Custom filter validating JWT on protected routes
│       │   └── exception/
│       │       └── JwtValidationException.java
│       └── resources/
│           ├── application.yml                 <-- Route definitions: /auth/**, /api/patients/**, /api-docs/**
│           └── application-prod.yml            <-- Production profile overrides
│
├── integration-tests/                          <-- Cross-service communication testing module
│   ├── pom.xml                                 <-- [POM] RestAssured, JUnit 5 with <scope>test</scope>
│   └── src/test/java/
│       ├── AuthIntegrationTest.java            <-- Tests auth-service login and token generation
│       └── PatientIntegrationTest.java         <-- Tests patient CRUD through the api-gateway
│
└── infrastructure/                             <-- Cloud Infrastructure as Code (for reference)
    ├── pom.xml                                 <-- [POM] aws-cdk-lib (legacy, replaced by k8s/ manifests)
    ├── localstack-deploy.sh                    <-- LocalStack testing script
    └── src/main/java/com/pm/stack/
        └── LocalStack.java                     <-- AWS CDK stack definition (reference only)
```

## Kubernetes Architecture Overview

### Resource Hierarchy

```
Namespace: patient-management
│
├── Secrets
│   ├── db-secrets          (PostgreSQL credentials)
│   └── jwt-secret          (JWT signing key)
│
├── StatefulSets (Databases & Kafka)
│   ├── patient-service-db  (PostgreSQL 15)
│   ├── auth-service-db     (PostgreSQL 15)
│   ├── kafka               (Bitnami Kafka broker)
│   └── zookeeper           (Bitnami Zookeeper)
│
├── Deployments (Application Services)
│   ├── patient-service     (REST + gRPC Client + Kafka Producer)
│   ├── billing-service     (gRPC Server)
│   ├── analytics-service   (Kafka Consumer)
│   ├── auth-service        (JWT Issuer)
│   └── api-gateway         (Spring Cloud Gateway)
│
└── Services (Networking)
    ├── patient-service-db   ClusterIP  :5432
    ├── auth-service-db      ClusterIP  :5432
    ├── kafka                ClusterIP  :9092
    ├── zookeeper            ClusterIP  :2181
    ├── patient-service      ClusterIP  :4000
    ├── billing-service      ClusterIP  :4001, :9001
    ├── auth-service         ClusterIP  :4005
    └── api-gateway          NodePort   :4004
```

### Key Design Decisions

1. **Namespace Isolation:** All resources are scoped under the `patient-management` namespace for clear separation from other cluster workloads.
2. **StatefulSets for Stateful Components:** PostgreSQL databases and Kafka use StatefulSets with PersistentVolumeClaims to ensure data durability across Pod restarts.
3. **Deployments for Stateless Services:** All application microservices use Kubernetes Deployments, enabling rolling updates and horizontal scaling.
4. **ClusterIP for Internal Traffic:** All inter-service communication uses ClusterIP services (internal DNS), keeping services unreachable from outside the cluster.
5. **NodePort/LoadBalancer for Gateway:** Only the api-gateway is exposed externally, maintaining a single secure entry point.
6. **Secrets for Credentials:** Database passwords and JWT keys are stored as Kubernetes Secrets and injected as environment variables rather than hardcoded in manifests.
