# 💻 DevOps Internship Assessment - Todo-List Node.js Deployment

## 📌 Project Overview

This repository documents the setup, automation, and deployment process for the [Todo-List Node.js application](https://github.com/Ankit6098/Todo-List-nodejs) as part of a DevOps internship assignment.

---

##  Task Breakdown

###  Part 1: Dockerizing the Application and CI/CD via GitHub Actions

####  Steps Taken

1. **Cloned the Repository**

   ```bash
   git clone https://github.com/Ankit6098/Todo-List-nodejs
   cd Todo-List-nodejs
   ```
2. **Set Up My Own MongoDB Database**

* I created my own **MongoDB Atlas cluster**, updated the `.env` with my secure credentials:

  ```env
  DB_URL=mongodb+srv://<username>:<password>@cluster.mongodb.net/todolist?retryWrites=true&w=majority
  ```

3. **Created Dockerfile**

   ```Dockerfile
   FROM node:18
   WORKDIR /app
   COPY package*.json ./
   RUN npm install
   COPY . .
   EXPOSE 8080
   CMD ["npm", "start"]
   ```

4. **Built and Tested Docker Image Locally**

   ```bash
   docker build -t todo-app .
   docker run -p 8080:8080 --env-file .env todo-app
   ```

5. **Created GitHub Actions Workflow (`.github/workflows/docker.yml`)**

   * CI pipeline builds image and pushes to **private DockerHub registry**.

   ```yaml
   name: Docker Build and Push
   on: [push]
   jobs:
     build:
       runs-on: ubuntu-latest
       steps:
         - uses: actions/checkout@v3
         - name: Log in to Docker Hub
           run: echo "${{ secrets.DOCKER_PASSWORD }}" | docker login -u "${{ secrets.DOCKER_USERNAME }}" --password-stdin
         - name: Build Docker image
           run: docker build -t yourdockeruser/todo-app .
         - name: Push image
           run: docker push yourdockeruser/todo-app
   ```

---

###  Part 2: Provision Linux VM & Configure with Ansible

####  EC2 Setup on AWS

* **Instance Type**: t2.micro (Free tier)
* **AMI**: Amazon Linux 2
* **Security Group Settings (SG):**

  * Port 22 (SSH) — My IP
  * Port 8080 — Anywhere (for app access)
  * Port 80, 443 — Anywhere (for Watchtower/K8s/ArgoCD later)

####  Ansible Configuration

1. **Inventory File**

   ```ini
   [app]
   ec2-ip ansible_user=ec2-user ansible_ssh_private_key_file=~/.ssh/your-key.pem
   ```

2. **Playbook (`setup.yml`)**

   ```yaml
   ---
   - hosts: app
     become: true
     tasks:
       - name: Install Docker
         amazon.aws.amazon_linux_extras:
           name: docker
           state: enabled
       - name: Install packages
         yum:
           name:
             - docker
             - git
             - python3-pip
           state: present
       - name: Start Docker
         service:
           name: docker
           state: started
           enabled: true
   ```

3. **Run Ansible**

   ```bash
   ansible-playbook -i inventory setup.yml
   ```

---

###  Part 3: Docker Compose + Auto Image Update

####  Docker Compose File

```yaml
version: '3'
services:
  todo-app:
    image: yourdockeruser/todo-app:latest
    ports:
      - "8080:8080"
    env_file:
      - .env
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080"]
      interval: 30s
      retries: 3

  watchtower:
    image: containrrr/watchtower
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    command: --interval 60
```

>  **Auto Update Tool**: `Watchtower`
> Justification: Watchtower is lightweight, easy to configure, and designed to monitor Docker Hub for new image versions. It automatically pulls the latest image and redeploys the container without downtime.

---

###  Part 4 (Bonus): Kubernetes with ArgoCD

####  Kubernetes Setup via Kubeadm (on EC2)

1. **Install K8s components via Ansible**

   * Installed `kubeadm`, `kubelet`, and `kubectl` using official steps from [Kubernetes Docs](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)

2. **Initialize Kubernetes Master**

   ```bash
   sudo kubeadm init --pod-network-cidr=10.244.0.0/16
   ```

3. **Set Up Kube Config**

   ```bash
   mkdir -p $HOME/.kube
   sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
   ```

4. **Install Flannel Network Add-on**

   ```bash
   kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
   ```

####  Deploy ArgoCD

1. **Install ArgoCD**

   ```bash
   kubectl create namespace argocd
   kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
   ```

2. **Access ArgoCD UI**

   * Expose service via port forwarding:

     ```bash
     kubectl port-forward svc/argocd-server -n argocd 8081:443
     ```
   * Default login: `admin`, password from:

     ```bash
     kubectl get secret argocd-initial-admin-secret -n argocd -o yaml
     ```

3. **Create ArgoCD App to Track Docker Image Changes**

   * ArgoCD syncs Kubernetes deployment with the image version in the registry.

---

## 🔒 Security Notes

* Used `.env` for local development only.
* All secrets and API keys were added securely as GitHub secrets or Ansible vault variables.
* EC2 Security Group is restricted and only opens required ports.

---

