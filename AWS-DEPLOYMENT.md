# AWS Kubernetes Deployment Strategy

This document outlines the complete DevOps pipeline required to deploy the Maven-orchestrated Microservices system to **Amazon Elastic Kubernetes Service (EKS)**.

Unlike the original project which used AWS CDK and ECS Fargate, this Kubernetes edition deploys infrastructure declaratively using `kubectl` and the manifests defined in the `k8s/` directory.

---

## Architecture Overview

```
                        Internet
                           |
                    [ AWS ALB / NLB ]
                           |
                  [ api-gateway:4004 ]
                     /            \
          [ auth-service ]    [ patient-service ]
              :4005              :4000
                               /     \
                     [ billing ]   [ kafka ]
                       :9001        :9092
                                      |
                              [ analytics-service ]

    Persistent Storage:
    [ patient-db ]  [ auth-db ]  [ kafka-data ]
      (EBS PV)       (EBS PV)      (EBS PV)
```

---

## Prerequisites

Before proceeding, ensure the following tools are installed on your local machine:

| Tool | Version | Purpose |
|---|---|---|
| AWS CLI | v2+ | AWS account authentication |
| eksctl | latest | EKS cluster provisioning |
| kubectl | v1.28+ | Kubernetes resource management |
| Docker Desktop | latest | Building container images |
| Java JDK | 21 | Maven compilation (optional if using Docker builds) |
| Maven | 3.9+ | Project builds (optional if using Docker builds) |

---

## Phase 1: AWS Account Setup

### 1.1 Create and Authenticate AWS Account

```bash
# Install AWS CLI (if not already installed)
# https://aws.amazon.com/cli/

# Configure credentials
aws configure
```

You will be prompted for:
- **AWS Access Key ID** (from IAM Dashboard)
- **AWS Secret Access Key**
- **Default region** (e.g., `us-east-1`)
- **Output format** (`json`)

### 1.2 Install eksctl

```bash
# Windows (via Chocolatey)
choco install eksctl

# macOS
brew tap weaveworks/tap
brew install weaveworks/tap/eksctl

# Verify installation
eksctl version
```

---

## Phase 2: Provision the EKS Cluster

### 2.1 Create the Cluster

```bash
eksctl create cluster \
  --name patient-management-cluster \
  --region us-east-1 \
  --nodegroup-name worker-nodes \
  --node-type t3.medium \
  --nodes 3 \
  --nodes-min 2 \
  --nodes-max 4 \
  --managed
```

This provisions:
- A managed EKS control plane
- 3 `t3.medium` EC2 worker nodes with autoscaling (2-4 nodes)
- VPC, subnets, and security groups

> Cluster creation takes approximately 15-20 minutes.

### 2.2 Verify Cluster Access

```bash
# Update kubeconfig
aws eks update-kubeconfig --name patient-management-cluster --region us-east-1

# Verify connectivity
kubectl get nodes
```

Expected output: 3 nodes in `Ready` state.

---

## Phase 3: Build and Push Docker Images to ECR

### 3.1 Create ECR Repositories

```bash
# Create a repository for each microservice
aws ecr create-repository --repository-name patient-service --region us-east-1
aws ecr create-repository --repository-name billing-service --region us-east-1
aws ecr create-repository --repository-name analytics-service --region us-east-1
aws ecr create-repository --repository-name auth-service --region us-east-1
aws ecr create-repository --repository-name api-gateway --region us-east-1
```

### 3.2 Authenticate Docker to ECR

```bash
# Replace <ACCOUNT_ID> with your 12-digit AWS account ID
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin <ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com
```

### 3.3 Build Docker Images

From the project root directory (`soe-kubernetes/`):

```bash
# Build all service images using Docker Compose
docker compose build
```

Or build individually:

```bash
docker build -t patient-service -f patient-service/Dockerfile .
docker build -t billing-service -f billing-service/Dockerfile .
docker build -t analytics-service -f analytics-service/Dockerfile .
docker build -t auth-service -f auth-service/Dockerfile .
docker build -t api-gateway -f api-gateway/Dockerfile .
```

### 3.4 Tag and Push Images

```bash
# Set your account ID as a variable
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export AWS_REGION=us-east-1
export ECR_REGISTRY=${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com

# Tag and push each image
for SERVICE in patient-service billing-service analytics-service auth-service api-gateway; do
  docker tag ${SERVICE}:latest ${ECR_REGISTRY}/${SERVICE}:latest
  docker push ${ECR_REGISTRY}/${SERVICE}:latest
done
```

---

## Phase 4: Update Kubernetes Manifests for ECR

Before deploying, update all Deployment manifests to reference your ECR image URIs instead of local images.

Replace `image: <service-name>:latest` with your ECR path in each deployment YAML:

```yaml
# Example: k8s/services/patient-service-deployment.yaml
# Change:
#   image: patient-service:latest
#   imagePullPolicy: IfNotPresent
# To:
#   image: <ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/patient-service:latest
#   imagePullPolicy: Always
```

Files to update:
- `k8s/services/patient-service-deployment.yaml`
- `k8s/services/billing-service-deployment.yaml`
- `k8s/services/analytics-service-deployment.yaml`
- `k8s/services/auth-service-deployment.yaml`
- `k8s/gateway/api-gateway-deployment.yaml`

---

## Phase 5: Deploy to EKS

### 5.1 Install the EBS CSI Driver (for Persistent Volumes)

StatefulSets (PostgreSQL, Kafka) require EBS-backed persistent volumes.

```bash
# Create an IAM OIDC provider for the cluster
eksctl utils associate-iam-oidc-provider \
  --cluster patient-management-cluster \
  --region us-east-1 \
  --approve

# Install the EBS CSI driver addon
eksctl create addon \
  --name aws-ebs-csi-driver \
  --cluster patient-management-cluster \
  --region us-east-1 \
  --service-account-role-arn arn:aws:iam::${AWS_ACCOUNT_ID}:role/AmazonEKS_EBS_CSI_DriverRole \
  --force
```

### 5.2 Create a StorageClass (if not using default gp2)

```bash
kubectl apply -f - <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-sc
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
parameters:
  type: gp3
EOF
```

### 5.3 Deploy Resources in Order

Deploy resources sequentially to respect dependencies:

```bash
# Step 1: Namespace
kubectl apply -f k8s/namespace.yaml

# Step 2: Secrets
kubectl apply -f k8s/secrets/

# Step 3: Databases (wait for them to become Ready)
kubectl apply -f k8s/databases/
kubectl -n patient-management wait --for=condition=ready pod -l app=patient-service-db --timeout=120s
kubectl -n patient-management wait --for=condition=ready pod -l app=auth-service-db --timeout=120s

# Step 4: Kafka infrastructure
kubectl apply -f k8s/kafka/
kubectl -n patient-management wait --for=condition=ready pod -l app=zookeeper --timeout=120s
kubectl -n patient-management wait --for=condition=ready pod -l app=kafka --timeout=120s

# Step 5: Application services
kubectl apply -f k8s/services/

# Step 6: API Gateway
kubectl apply -f k8s/gateway/
```

### 5.4 Verify Deployment

```bash
# Check all pods are running
kubectl -n patient-management get pods

# Check all services
kubectl -n patient-management get svc

# View pod logs for any service
kubectl -n patient-management logs -f deployment/patient-service
kubectl -n patient-management logs -f deployment/api-gateway
```

Expected output: All pods in `Running` state with `1/1` containers ready.

---

## Phase 6: Expose the Gateway via AWS Load Balancer

### Option A: NodePort (already configured)

The `api-gateway-service.yaml` is configured as `NodePort` on port `30004`. Access via any worker node's public IP:

```bash
# Get node external IPs
kubectl get nodes -o wide

# Access: http://<NODE_EXTERNAL_IP>:30004
```

### Option B: AWS Load Balancer (production recommended)

Install the AWS Load Balancer Controller, then change the gateway service type:

```bash
# Install AWS Load Balancer Controller
helm repo add eks https://aws.github.io/eks-charts
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=patient-management-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller
```

Then modify `k8s/gateway/api-gateway-service.yaml`:

```yaml
spec:
  type: LoadBalancer   # Change from NodePort
  selector:
    app: api-gateway
  ports:
    - port: 80
      targetPort: 4004
      protocol: TCP
```

Apply and get the ALB URL:

```bash
kubectl apply -f k8s/gateway/api-gateway-service.yaml
kubectl -n patient-management get svc api-gateway
```

The `EXTERNAL-IP` column will show your ALB DNS name.

---

## Phase 7: Validate the Deployment

### 7.1 Test Authentication

```bash
# Login (replace <GATEWAY_URL> with your ALB DNS or NodeIP:30004)
curl -X POST http://<GATEWAY_URL>/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email": "testuser@test.com", "password": "password123"}'
```

Expected: `{"token": "eyJhbGciOi..."}`

### 7.2 Test Patient API (Authenticated)

```bash
# Use the token from the login response
curl -X GET http://<GATEWAY_URL>/api/patients \
  -H "Authorization: Bearer <TOKEN>"
```

Expected: `200 OK` with patient data.

### 7.3 Test Unauthorized Access

```bash
curl -X GET http://<GATEWAY_URL>/api/patients
```

Expected: `401 Unauthorized`

---

## Phase 8: Cleanup

To avoid ongoing AWS charges, tear down all resources:

```bash
# Delete all Kubernetes resources
kubectl delete namespace patient-management

# Delete the EKS cluster (this also removes worker nodes)
eksctl delete cluster --name patient-management-cluster --region us-east-1
```

---

## Cost Estimation

| Resource | Type | Estimated Monthly Cost |
|---|---|---|
| EKS Control Plane | Managed | $73/month |
| 3x t3.medium Nodes | EC2 | ~$100/month |
| 3x EBS Volumes (10Gi) | gp3 | ~$2.50/month |
| ECR Storage | Per GB | ~$1/month |
| Load Balancer | ALB | ~$22/month |
| **Total** | | **~$199/month** |

> Use `eksctl delete cluster` immediately after presentations to avoid charges.

---

## Presentation Talking Points

When presenting this deployment, highlight these key aspects:

1. **Kubernetes-Native Infrastructure**: All infrastructure is defined declaratively in YAML manifests (`k8s/`), enabling version-controlled, reproducible deployments across any Kubernetes cluster.

2. **Apache Maven Orchestration**: Maven manages complex dependency trees across 6 modules and automatically generates thousands of lines of gRPC networking code via the `protobuf-maven-plugin`.

3. **Multi-Stage Docker Optimization**: Dockerfiles use `maven:3.9.9` to build artifacts offline (`mvn dependency:go-offline`), then copy only the `.jar` files into minimal `openjdk:21` runtime images — reducing final image sizes by 80%+.

4. **StatefulSet Persistence**: Databases and Kafka use Kubernetes StatefulSets with PersistentVolumeClaims backed by AWS EBS, ensuring data survives pod restarts.

5. **Init Container Dependencies**: Services use init containers to wait for databases and Kafka before starting, replacing Docker Compose's `depends_on` with cloud-native dependency management.

6. **CI/CD Pipeline**: GitHub Actions workflows (`.github/workflows/`) automatically verify Maven compilation on every push, with path-filtered triggers for each microservice.
