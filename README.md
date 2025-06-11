
# 🚀 Deploying an Ansible Cluster on Kubernetes (AWS EKS)

A complete walkthrough to deploy an **Ansible Master-Slave Cluster** inside Docker containers, orchestrated via **Kubernetes on AWS EKS**.

---

## 📍 Table of Contents

- [🔧 Step 1: Create EKS Cluster on AWS](#-step-1-create-eks-cluster-on-aws)
- [🐳 Step 2: Build Docker Images for Master & Slaves](#-step-2-build-docker-images-for-master--slaves)
- [📦 Step 3: Kubernetes Deployment YAMLs](#-step-3-kubernetes-deployment-yamls)
- [☸️ Step 4: Deploy to EKS via kubectl](#️-step-4-deploy-to-eks-via-kubectl)
- [🔐 Step 5: SSH Communication Setup](#-step-5-ssh-communication-setup)
- [✅ Step 6: Run Ansible Commands](#-step-6-run-ansible-commands)
- [🧰 Useful Commands & Resources](#-useful-commands--resources)

---

## 🔧 Step 1: Create EKS Cluster on AWS

Use `eksctl` to provision a 3-node managed EKS cluster in `ap-south-1`:

```bash
eksctl create cluster \
  --name ansible-cluster \
  --region ap-south-1 \
  --nodegroup-name ansible-nodes \
  --nodes 3 \
  --node-type t3.medium \
  --managed
```
Set your namespace:
```bash
kubectl create namespace ansible-cluster
kubectl config set-context --current --namespace=ansible-cluster
```
## 🐳Step 2: Build Docker Images for Master & Slaves
### 📁 Directory Structure

```text
.
├── ansible-master-node/
│   ├── Dockerfile
│   ├── ansible.cfg
│   └── inventory.ini
├── ansible-slave-node/
│   └── Dockerfile
```
### 🐳 Dockerfile: Ansible Master
```dockerfile
FROM alpine:3.15

# Install dependencies
RUN apk add --no-cache \
    python3 \
    py3-pip \
    openssh \
    sshpass \
    && pip3 install --upgrade pip \
    && pip3 install ansible

# Create ansible user
RUN adduser -D ansible && \
    echo "ansible:ansible" | chpasswd && \
    mkdir -p /home/ansible/.ssh && \
    chown -R ansible:ansible /home/ansible/.ssh

# Switch to ansible user
USER ansible
WORKDIR /home/ansible

# Generate SSH key
RUN ssh-keygen -t rsa -f /home/ansible/.ssh/id_rsa -q -N ""

# Prepare ansible directories
RUN mkdir -p /home/ansible/ansible/playbooks

COPY ansible.cfg /home/ansible/ansible/ansible.cfg
COPY inventory.ini /home/ansible/ansible/inventory.ini

CMD ["sh", "-c", "tail -f /dev/null"]
```
### 🐳 Dockerfile: Ansible Slave
```dockerfile

FROM alpine:3.15

# Install dependencies
RUN apk add --no-cache \
    python3 \
    openssh \
    sudo

# Create ansible user
RUN adduser -D ansible && \
    echo "ansible:ansible" | chpasswd && \
    mkdir -p /home/ansible/.ssh && \
    chown -R ansible:ansible /home/ansible/.ssh && \
    echo "ansible ALL=(ALL) NOPASSWD: ALL" >> /etc/sudoers

# Switch to ansible user
USER ansible
WORKDIR /home/ansible

# Prepare SSH directory
RUN mkdir -p /home/ansible/.ssh && \
    touch /home/ansible/.ssh/authorized_keys && \
    chmod 700 /home/ansible/.ssh && \
    chmod 600 /home/ansible/.ssh/authorized_keys

CMD ["sh", "-c", "sudo /usr/sbin/sshd -D -e"]
```
### 🔧 Build & Push to Docker Hub

```bash

docker build -t harshj2003/ansible-master:v1 ./ansible-master-node
docker push harshj2003/ansible-master:v1

docker build -t harshj2003/ansible-slave:v1 ./ansible-slave-node
docker push harshj2003/ansible-slave:v1
```
## 📦 Step 3: Kubernetes Deployment YAMLs

### Master Deployment
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ansible-master
  namespace: ansible-cluster
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ansible-master
  template:
    metadata:
      labels:
        app: ansible-master
    spec:
      containers:
      - name: ansible-master
        image: harshj2003/ansible-master:v1
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 22
        volumeMounts:
        - name: ansible-config
          mountPath: /home/ansible/ansible
      volumes:
      - name: ansible-config
        emptyDir: {}
```
### Master Service
```yaml

# k8s/ansible-master-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: ansible-master-service
spec:
  selector:
    app: ansible-master
  ports:
    - protocol: TCP
      port: 22
      targetPort: 22
  type: ClusterIP
```
### Slave Deployment
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ansible-slave
  namespace: ansible-cluster
spec:
  replicas: 3  # Create 3 nodes for ansible testing
  selector:
    matchLabels:
      app: ansible-slave
  template:
    metadata:
      labels:
        app: ansible-slave
    spec:
      containers:
      - name: ansible-slave
        image: harshj2003/ansible-slave:v1
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 22
```
### Slave Service
```yaml
# k8s/ansible-slave-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: ansible-slave-service
spec:
  selector:
    app: ansible-slave
  ports:
    - protocol: TCP
      port: 22
      targetPort: 22
  type: ClusterIP
```
## ☸️ Step 4: Deploy to EKS via kubectl

Apply all deployments and services:

```bash
kubectl apply -f k8s/
```
Get pods and IPs:
```bash
kubectl get pods -o wide
```
Exec into master:
```bash
kubectl exec -it <ansible-master-pod> -- bash
```
## 🔐 Step 5: SSH Communication Setup

### 🔎 Get Pod IPs of Slaves
```bash
kubectl get pods -l app=ansible-slave -o wide
```
Example  `inventory.ini`:
```ini
[all]
10.0.0.23
10.0.0.45
10.0.0.67

Update  `/home/ansible/inventory.ini`  in master pod accordingly.
```
### 📤 Copy SSH Key to Slaves

```bash

ssh-copy-id ansible@10.0.0.23
ssh-copy-id ansible@10.0.0.45
ssh-copy-id ansible@10.0.0.67
```
## ✅ Step 6: Run Ansible Commands

### 🔎 Test connectivity:

```bash


ansible all -i /home/ansible/inventory.ini -m ping
```
Expected:

```text

10.0.0.23 | SUCCESS => {...}
10.0.0.45 | SUCCESS => {...}
10.0.0.67 | SUCCESS => {...}
```
### 💡 Run a custom command:

```bash

ansible all -i /home/ansible/inventory.ini -m command -a "date"
```
## 🧰 Useful Commands & Resources

Purpose

Command

Get pods

`kubectl get pods -o wide`

Get services

`kubectl get svc`

Get node info

`kubectl get nodes -o wide`

Exec into pod

`kubectl exec -it <pod-name> -- bash`

Apply all resources

`kubectl apply -f k8s/`

Delete all

`kubectl delete -f k8s/`

## 📦 Docker Images Used

Role

Image

Master Node

`harshj2003/ansible-master:v1`

Slave Node

`harshj2003/ansible-slave:v1`

## 📌 GitHub Repository

👉  [View Repository](https://github.com/your-repo-link)

## 🙌 Author

**Harshwardhan Jadhav**  
☁️ DevOps & Cloud Engineer
<<<<<<< HEAD
LinkedIN: https://www.linkedin.com/posts/jadhavharshwardhan_devops-ansible-kubernetes-activity-7338274579619950592-FWtp?utm_source=share&utm_medium=member_desktop&rcm=ACoAAFQgnCoBgMH2PuaEzuGok5YyLx2BNyWlB8E# n-Ansible-Cluster-in-Docker-Container-Kubernetes-Pod
=======
LinkedIN: https://www.linkedin.com/posts/jadhavharshwardhan_devops-ansible-kubernetes-activity-7338274579619950592-FWtp?utm_source=share&utm_medium=member_desktop&rcm=ACoAAFQgnCoBgMH2PuaEzuGok5YyLx2BNyWlB8E
>>>>>>> 78a56574bb685ec2d0ee20f77a716ea150539b32
