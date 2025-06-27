# Kubernetes Cluster Deployment and Application Management

A comprehensive guide covering containerized application deployment from Docker basics to Kubernetes orchestration, including practical implementations of web applications and MySQL databases.

## 📋 Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Assignment 1: Docker Containerization](#assignment-1-docker-containerization)
- [Assignment 2: Kubernetes Deployment](#assignment-2-kubernetes-deployment)
- [Project Structure](#project-structure)
- [Troubleshooting](#troubleshooting)
- [Key Learnings](#key-learnings)
- [References](#references)

## 🎯 Overview

This repository demonstrates end-to-end containerized application deployment using Docker and Kubernetes. The project covers:

- **Assignment 1**: Docker containerization of web applications and databases
- **Assignment 2**: Kubernetes cluster deployment with Kind, application orchestration, and lifecycle management

**Technologies Used:**
- Docker & Docker Compose
- Kubernetes (Kind)
- Amazon EC2
- Amazon ECR
- MySQL Database
- Node.js Web Application

## 🛠️ Prerequisites

### System Requirements
- Amazon Linux EC2 instance (t3.medium recommended)
- 4GB RAM minimum
- 20GB storage
- Security groups configured for SSH (22) and application ports

### Software Dependencies
```bash
# Update system
sudo yum update -y

# Install Docker
sudo yum install -y docker
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -a -G docker ec2-user

# Install Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/download/v2.20.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

# Install kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Install Kind
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind

# Install AWS CLI (if not present)
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

## 🐳 Assignment 1: Docker Containerization

### Step 1: Application Setup

**Web Application (Node.js)**
```dockerfile
FROM node:16-alpine

WORKDIR /app

COPY package*.json ./
RUN npm install

COPY . .

EXPOSE 3000

CMD ["npm", "start"]
```

**MySQL Database**
```dockerfile
FROM mysql:8.0

ENV MYSQL_ROOT_PASSWORD=rootpassword
ENV MYSQL_DATABASE=webapp_db
ENV MYSQL_USER=webapp_user
ENV MYSQL_PASSWORD=webapp_password

COPY init.sql /docker-entrypoint-initdb.d/

EXPOSE 3306
```

### Step 2: Docker Compose Configuration

```yaml
version: '3.8'

services:
  webapp:
    build: ./webapp
    ports:
      - "3000:3000"
    environment:
      - DB_HOST=mysql
      - DB_USER=webapp_user
      - DB_PASSWORD=webapp_password
      - DB_NAME=webapp_db
    depends_on:
      - mysql
    networks:
      - app-network

  mysql:
    build: ./mysql
    environment:
      - MYSQL_ROOT_PASSWORD=rootpassword
      - MYSQL_DATABASE=webapp_db
      - MYSQL_USER=webapp_user
      - MYSQL_PASSWORD=webapp_password
    volumes:
      - mysql_data:/var/lib/mysql
    networks:
      - app-network

volumes:
  mysql_data:

networks:
  app-network:
    driver: bridge
```

### Step 3: Build and Deploy

```bash
# Build and run with Docker Compose
docker-compose up -d

# Verify deployment
docker-compose ps
docker-compose logs webapp
docker-compose logs mysql

# Test application
curl http://localhost:3000
```

### Step 4: Push to ECR

```bash
# Create ECR repositories
aws ecr create-repository --repository-name webapp --region us-east-1
aws ecr create-repository --repository-name mysql-custom --region us-east-1

# Get login token
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin <account-id>.dkr.ecr.us-east-1.amazonaws.com

# Tag and push images
docker tag webapp:latest <account-id>.dkr.ecr.us-east-1.amazonaws.com/webapp:latest
docker tag mysql-custom:latest <account-id>.dkr.ecr.us-east-1.amazonaws.com/mysql-custom:latest

docker push <account-id>.dkr.ecr.us-east-1.amazonaws.com/webapp:latest
docker push <account-id>.dkr.ecr.us-east-1.amazonaws.com/mysql-custom:latest
```

## ⚓ Assignment 2: Kubernetes Deployment

### Step 1: Kind Cluster Setup

```yaml
# kind-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "ingress-ready=true"
  extraPortMappings:
  - containerPort: 30000
    hostPort: 30000
    protocol: TCP
```

```bash
# Create Kind cluster
kind create cluster --config=kind-config.yaml --name k8s-assignment

# Verify cluster
kubectl cluster-info
kubectl get nodes
```

### Step 2: ECR Authentication

```bash
# Create ECR secret
kubectl create secret docker-registry ecr-secret \
  --docker-server=<account-id>.dkr.ecr.us-east-1.amazonaws.com \
  --docker-username=AWS \
  --docker-password=$(aws ecr get-login-password --region us-east-1)
```

### Step 3: Pod Deployments

**Web Application Pod**
```yaml
# webapp-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: webapp-pod
  labels:
    app: webapp
    tier: frontend
spec:
  imagePullSecrets:
  - name: ecr-secret
  containers:
  - name: webapp
    image: <account-id>.dkr.ecr.us-east-1.amazonaws.com/webapp:latest
    ports:
    - containerPort: 3000
    env:
    - name: DB_HOST
      value: "mysql-service"
    - name: DB_USER
      value: "webapp_user"
    - name: DB_PASSWORD
      value: "webapp_password"
    - name: DB_NAME
      value: "webapp_db"
```

**MySQL Pod**
```yaml
# mysql-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: mysql-pod
  labels:
    app: mysql
    tier: database
spec:
  imagePullSecrets:
  - name: ecr-secret
  containers:
  - name: mysql
    image: <account-id>.dkr.ecr.us-east-1.amazonaws.com/mysql-custom:latest
    ports:
    - containerPort: 3306
    env:
    - name: MYSQL_ROOT_PASSWORD
      value: "rootpassword"
    - name: MYSQL_DATABASE
      value: "webapp_db"
    - name: MYSQL_USER
      value: "webapp_user"
    - name: MYSQL_PASSWORD
      value: "webapp_password"
```

### Step 4: ReplicaSet Configuration

```yaml
# webapp-replicaset.yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: webapp-replicaset
  labels:
    app: webapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webapp
      tier: frontend
  template:
    metadata:
      labels:
        app: webapp
        tier: frontend
    spec:
      imagePullSecrets:
      - name: ecr-secret
      containers:
      - name: webapp
        image: <account-id>.dkr.ecr.us-east-1.amazonaws.com/webapp:latest
        ports:
        - containerPort: 3000
        env:
        - name: DB_HOST
          value: "mysql-service"
        - name: DB_USER
          value: "webapp_user"
        - name: DB_PASSWORD
          value: "webapp_password"
        - name: DB_NAME
          value: "webapp_db"
```

### Step 5: Deployment Management

```yaml
# webapp-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-deployment
  labels:
    app: webapp
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  selector:
    matchLabels:
      app: webapp
      tier: frontend
  template:
    metadata:
      labels:
        app: webapp
        tier: frontend
    spec:
      imagePullSecrets:
      - name: ecr-secret
      containers:
      - name: webapp
        image: <account-id>.dkr.ecr.us-east-1.amazonaws.com/webapp:v2
        ports:
        - containerPort: 3000
        readinessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 10
          periodSeconds: 5
        env:
        - name: DB_HOST
          value: "mysql-service"
        - name: DB_USER
          value: "webapp_user"
        - name: DB_PASSWORD
          value: "webapp_password"
        - name: DB_NAME
          value: "webapp_db"
```

### Step 6: Service Configuration

**NodePort Service for Web Application**
```yaml
# webapp-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: webapp-service
  labels:
    app: webapp
spec:
  type: NodePort
  selector:
    app: webapp
    tier: frontend
  ports:
  - port: 80
    targetPort: 3000
    nodePort: 30000
```

**ClusterIP Service for MySQL**
```yaml
# mysql-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql-service
  labels:
    app: mysql
spec:
  type: ClusterIP
  selector:
    app: mysql
    tier: database
  ports:
  - port: 3306
    targetPort: 3306
```

### Step 7: Deployment Commands

```bash
# Deploy in sequence
kubectl apply -f mysql-pod.yaml
kubectl apply -f mysql-service.yaml
kubectl apply -f webapp-pod.yaml
kubectl apply -f webapp-replicaset.yaml
kubectl apply -f webapp-deployment.yaml
kubectl apply -f webapp-service.yaml

# Verify deployments
kubectl get all
kubectl get pods -o wide
kubectl get services

# Test connectivity
kubectl exec -it webapp-pod -- curl localhost:3000
curl http://<EC2-Public-IP>:30000
```

### Step 8: Rolling Updates

```bash
# Update deployment with new image
kubectl set image deployment/webapp-deployment webapp=<account-id>.dkr.ecr.us-east-1.amazonaws.com/webapp:v2

# Monitor rollout
kubectl rollout status deployment/webapp-deployment
kubectl rollout history deployment/webapp-deployment

# Rollback if needed
kubectl rollout undo deployment/webapp-deployment
```

## 📁 Project Structure

```
k8s-assignment/
├── README.md
├── assignment1/
│   ├── webapp/
│   │   ├── Dockerfile
│   │   ├── package.json
│   │   ├── server.js
│   │   └── public/
│   ├── mysql/
│   │   ├── Dockerfile
│   │   └── init.sql
│   └── docker-compose.yml
├── assignment2/
│   ├── cluster/
│   │   └── kind-config.yaml
│   ├── pods/
│   │   ├── webapp-pod.yaml
│   │   └── mysql-pod.yaml
│   ├── replicasets/
│   │   └── webapp-replicaset.yaml
│   ├── deployments/
│   │   └── webapp-deployment.yaml
│   ├── services/
│   │   ├── webapp-service.yaml
│   │   └── mysql-service.yaml
│   └── secrets/
│       └── ecr-secret.yaml
├── scripts/
│   ├── setup.sh
│   ├── deploy.sh
│   └── cleanup.sh
└── docs/
    ├── assignment1-report.md
    └── assignment2-report.md
```

## 🔧 Troubleshooting

### Common Issues and Solutions

**1. ECR Authentication Failures**
```bash
# Recreate ECR secret with fresh token
kubectl delete secret ecr-secret
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin <account-id>.dkr.ecr.us-east-1.amazonaws.com
kubectl create secret docker-registry ecr-secret --docker-server=<account-id>.dkr.ecr.us-east-1.amazonaws.com --docker-username=AWS --docker-password=$(aws ecr get-login-password --region us-east-1)
```

**2. Service Connectivity Issues**
```bash
# Debug service connectivity
kubectl get endpoints
kubectl describe service mysql-service
kubectl exec -it webapp-pod -- nslookup mysql-service
kubectl logs webapp-pod
```

**3. Resource Constraints**
```bash
# Check cluster resources
kubectl top nodes
kubectl describe nodes
kubectl get pods --all-namespaces
```

**4. NodePort Access Issues**
```bash
# Verify security group allows port 30000
aws ec2 describe-security-groups --group-ids <security-group-id>
# Test local connectivity first
kubectl exec -it webapp-pod -- curl webapp-service
```

### Debugging Commands

```bash
# Pod debugging
kubectl describe pod <pod-name>
kubectl logs <pod-name>
kubectl exec -it <pod-name> -- /bin/bash

# Service debugging
kubectl get endpoints
kubectl describe service <service-name>

# Cluster debugging
kubectl get events --sort-by=.metadata.creationTimestamp
kubectl cluster-info dump
```

## 📚 Key Learnings

### Docker Concepts
- Container isolation and networking
- Multi-stage builds and image optimization
- Docker Compose for multi-container applications
- Volume management and data persistence

### Kubernetes Concepts
- Pod lifecycle and networking
- ReplicaSet vs Deployment differences
- Service types and use cases
- Rolling updates and rollback strategies
- Resource management and troubleshooting

### Best Practices
- Security: Never expose databases externally
- Labeling: Consistent resource labeling strategy
- Updates: Use readiness probes for zero-downtime deployments
- Monitoring: Implement comprehensive logging and monitoring

### Architecture Decisions
- **NodePort vs ClusterIP**: Based on external access requirements
- **Deployment vs ReplicaSet**: Deployments provide additional rollout capabilities
- **Resource Sizing**: Right-size instances based on workload requirements

## 🎯 Assignment Questions & Answers

### Q1: What is the IP of the K8s API server in your cluster?
**A:** `127.0.0.1:6443` - Kind creates a local cluster exposing the API server on localhost.

### Q2: Can both MySQL and web applications listen on the same port inside containers?
**A:** Yes, each pod has its own network namespace, preventing port conflicts between different pods.

### Q3: Are standalone pods governed by ReplicaSets?
**A:** No, ReplicaSets only manage pods they create, not pre-existing standalone pods.

### Q4: Are manual ReplicaSets part of Deployments?
**A:** No, Deployments create and manage their own ReplicaSets with unique naming conventions.

### Q5: Why use different service types for web and MySQL applications?
**A:** Security and access requirements - web apps need external access (NodePort), databases should remain internal (ClusterIP).

## 📖 References

- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [Kind Documentation](https://kind.sigs.k8s.io/)
- [Docker Documentation](https://docs.docker.com/)
- [AWS ECR Documentation](https://docs.aws.amazon.com/ecr/)

---

**Repository maintained by:** Ishan Patel  
**Last updated:** June 2025  
