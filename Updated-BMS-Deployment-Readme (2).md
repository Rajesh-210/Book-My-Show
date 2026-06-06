# 🎬 Book My Show Clone — DevSecOps Deployment Guide

> **Stack:** Docker · Jenkins · SonarQube · OWASP · Trivy · Amazon EKS · ArgoCD · Prometheus · Grafana

---

## 📋 Table of Contents

- [Architecture Overview](#architecture-overview)
- [Part I: Docker Deployment](#part-i-docker-deployment)
  - [Step 1: Infrastructure Setup](#step-1-infrastructure-setup)
  - [Step 2: Tools Installation](#step-2-tools-installation)
  - [Step 3: Jenkins Configuration](#step-3-jenkins-configuration)
  - [Step 4: Email Integration](#step-4-email-integration)
  - [Step 5: Pipeline Job — Docker Only](#step-5-pipeline-job--docker-only)
- [Part II: Kubernetes Deployment](#part-ii-kubernetes-deployment)
  - [Step 6: EKS Pipeline Setup](#step-6-eks-pipeline-setup)
  - [Step 7: Production Dockerfile](#step-7-production-dockerfile)
  - [Step 8: Kubernetes Manifests](#step-8-kubernetes-manifests)
  - [Step 9: Final Jenkins Pipeline with ArgoCD](#step-9-final-jenkins-pipeline-with-argocd)
- [Part III: GitOps with ArgoCD](#part-iii-gitops-with-argocd)
  - [Step 10: Install ArgoCD on EKS](#step-10-install-argocd-on-eks)
  - [Step 11: Create ArgoCD Application](#step-11-create-argocd-application)
- [Part IV: Monitoring Stack](#part-iv-monitoring-stack)
  - [Step 12: Install kube-prometheus-stack via Helm](#step-12-install-kube-prometheus-stack-via-helm)
  - [Step 13: Grafana Dashboards](#step-13-grafana-dashboards)
- [Cleanup](#cleanup)

---

## Architecture Overview

```
Developer → GitHub (main branch)
              ↓
           Jenkins CI/CD Pipeline
              ├── SonarQube Analysis     (Code Quality)
              ├── OWASP Dependency Check (CVE Scanning)
              ├── Trivy FS Scan          (Filesystem Security)
              ├── Docker Build + Tag     (Versioned Image)
              ├── Trivy Image Scan       (Container Security)
              ├── Docker Push            (DockerHub Registry)
              └── Update Image Tag → Git commit
                              ↓
                         ArgoCD (GitOps)
                              ↓
                       Amazon EKS Cluster
                         ├── bms-app (2 replicas)
                         └── bms-service (LoadBalancer)
                              ↓
                    Prometheus + Grafana (Helm — in-cluster)
                         ├── Node Exporter (all EKS nodes)
                         ├── kube-state-metrics
                         └── Jenkins scrape job
```

---

## Part I: Docker Deployment

### Step 1: Infrastructure Setup

#### 1.1 Launch EC2 VM

| Setting | Value |
|---|---|
| Name | `BMS-Server` |
| OS | Ubuntu 24.04 |
| Instance Type | `t2.large` |
| Storage | 28 GB |

**Security Group — Open the following ports:**

| Type | Protocol | Port | Purpose |
|---|---|---|---|
| SSH | TCP | 22 | Remote access |
| HTTP | TCP | 80 | Web traffic |
| HTTPS | TCP | 443 | Secure web traffic |
| SMTP | TCP | 25 | Mail server |
| SMTPS | TCP | 465 | Secure email |
| Custom TCP | TCP | 3000–10000 | Jenkins, SonarQube, apps |
| Custom TCP | TCP | 6443 | Kubernetes API |
| Custom TCP | TCP | 30000–32767 | Kubernetes NodePort |

---

#### 1.2 Create IAM User for EKS

> ⚠️ Do **not** use the root account to create EKS clusters.

Attach the following managed policies to your IAM user:

- `AmazonEC2FullAccess`
- `AmazonEKS_CNI_Policy`
- `AmazonEKSClusterPolicy`
- `AmazonEKSWorkerNodePolicy`
- `AWSCloudFormationFullAccess`
- `IAMFullAccess`

Also attach the following **inline policy**:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": "eks:*",
            "Resource": "*"
        }
    ]
}
```

Generate **Access Keys** for the IAM user.

---

#### 1.3 Create EKS Cluster

**1.3.1 Install AWS CLI**

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
sudo apt install unzip -y
unzip awscliv2.zip
sudo ./aws/install
aws configure
```

**1.3.2 Install kubectl**

```bash
curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.19.6/2021-01-05/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin
kubectl version --short --client
```

**1.3.3 Install eksctl**

```bash
curl --silent --location \
  "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" \
  | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
eksctl version
```

**1.3.4 Create the EKS Cluster**

```bash
# (a) Create cluster
eksctl create cluster --name=Bms-eks \
                      --region=us-east-1 \
                      --zones=us-east-1a,us-east-1b \
                      --version=1.31 \
                      --without-nodegroup
```

```bash
# (b) Associate IAM OIDC Provider
eksctl utils associate-iam-oidc-provider \
    --region us-east-1 \
    --cluster Bms-eks \
    --approve
```

```bash
# (c) Create Node Group (replace KeyPairName with your PEM file name — no .pem)
eksctl create nodegroup --cluster=Bms-eks \
                       --region=us-east-1 \
                       --name=node2 \
                       --node-type=t3.medium \
                       --nodes=3 \
                       --nodes-min=2 \
                       --nodes-max=4 \
                       --node-volume-size=20 \
                       --ssh-access \
                       --ssh-public-key=KeyPairName \
                       --managed \
                       --asg-access \
                       --external-dns-access \
                       --full-ecr-access \
                       --appmesh-access \
                       --alb-ingress-access
```

> ⏱ Both commands take 5–10 minutes each.

**1.3.5 Set gp2 as the default StorageClass** *(required for Helm monitoring stack)*

```bash
kubectl patch storageclass gp2 \
  -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

kubectl get storageclass   # should show gp2 (default)
```

---

### Step 2: Tools Installation

#### 2.1 Install Jenkins

```bash
vi Jenkins.sh
```

```bash
#!/bin/bash
sudo apt install openjdk-17-jre-headless -y

sudo wget -O /usr/share/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key

echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null

sudo apt-get update
sudo apt-get install jenkins -y
```

```bash
sudo chmod +x Jenkins.sh && ./Jenkins.sh
```

> Access Jenkins at `http://<server-ip>:8080` and complete the setup wizard.

---

#### 2.2 Install Docker

```bash
vi docker.sh
```

```bash
#!/bin/bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

echo "deb [arch=$(dpkg --print-architecture) \
  signed-by=/etc/apt/keyrings/docker.asc] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io \
  docker-buildx-plugin docker-compose-plugin
```

```bash
sudo chmod +x docker.sh && ./docker.sh
sudo chmod 666 /var/run/docker.sock
docker --version
```

---

#### 2.3 Install Trivy

```bash
vi trivy.sh
```

```bash
#!/bin/bash
sudo apt-get install wget apt-transport-https gnupg -y
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key \
  | gpg --dearmor | sudo tee /usr/share/keyrings/trivy.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] \
  https://aquasecurity.github.io/trivy-repo/deb generic main" \
  | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy -y
```

```bash
sudo chmod +x trivy.sh && ./trivy.sh
trivy --version
```

---

#### 2.4 Setup SonarQube

```bash
docker run -d --name sonar -p 9000:9000 sonarqube:lts-community
```

> Access at `http://<server-ip>:9000` — default credentials: `admin` / `admin`

---

### Step 3: Jenkins Configuration

#### 3.1 Install Plugins

Navigate to **Manage Jenkins → Plugins → Available** and install:

| Category | Plugins |
|---|---|
| Java | Eclipse Temurin Installer |
| Code Quality | SonarQube Scanner |
| Runtime | NodeJS |
| Containers | Docker, Docker Commons, Docker Pipeline, Docker API, docker-build-step |
| Security | OWASP Dependency Check |
| Kubernetes | Kubernetes, Kubernetes CLI, Kubernetes Client API, Kubernetes Credentials |
| Observability | Prometheus metrics |
| Notifications | Email Extension Template |
| Pipeline | Pipeline Stage View |

#### 3.2 Configure Credentials

**Manage Jenkins → Credentials → Global → Add Credential** for each:

| ID | Kind | Purpose |
|---|---|---|
| `Sonar-token` | Secret Text | SonarQube authentication token |
| `docker` | Username with Password | DockerHub login |
| `email-creds` | Username with Password | Gmail SMTP |
| `nvd-api-key` | Secret Text | NVD API key for OWASP DC |
| `github` | Username with Password | GitHub PAT for ArgoCD GitOps |

> **GitHub PAT:** GitHub → Settings → Developer Settings → Personal Access Tokens (classic) → Generate with `repo` scope.

#### 3.3 Configure Tools

**Manage Jenkins → Tools:**

| Tool | Name | Version |
|---|---|---|
| JDK | `jdk17` | jdk-17 (adoptium.net) |
| NodeJS | `node18` | NodeJS 18.x |
| SonarQube Scanner | `sonar-scanner` | Latest |
| Docker | `docker` | Latest |
| OWASP DC | `DC` | Latest |

#### 3.4 System Configuration

**Manage Jenkins → System → SonarQube Servers:**

- Name: `sonar-server`
- URL: `http://<server-ip>:9000`
- Token: `Sonar-token`

---

### Step 4: Email Integration

**4.1 Generate Gmail App Password**

1. Go to **Google Account → Search "App Passwords"**
2. App name: `jenkins` → Create
3. Copy the 16-character password

**4.2 Configure Extended Email Notification**

**Manage Jenkins → System → Extended Email Notification:**

| Field | Value |
|---|---|
| SMTP Server | `smtp.gmail.com` |
| SMTP Port | `465` |
| Credentials | `email-creds` |
| Use SSL | ✅ |
| Default Content Type | HTML |

---

### Step 5: Pipeline Job — Docker Only

> Use this pipeline for local Docker container deployment only (no Kubernetes).

```groovy
pipeline {
    agent any
    tools {
        jdk 'jdk17'
        nodejs 'node18'
    }
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }
    stages {
        stage('Clean Workspace') {
            steps { cleanWs() }
        }
        stage('Checkout from Git') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/Rajesh-210/Book-My-Show.git'
            }
        }
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh '''
                    $SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectName=BMS \
                        -Dsonar.projectKey=BMS \
                        -Dsonar.sources=bookmyshow-app/src
                    '''
                }
            }
        }
        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: false
                }
            }
        }
        stage('Install Dependencies') {
            steps {
                dir('bookmyshow-app') {
                    sh 'npm install --no-audit --prefer-offline'
                }
            }
        }
        stage('OWASP Dependency Check') {
            steps {
                withCredentials([string(credentialsId: 'nvd-api-key', variable: 'NVD_API_KEY')]) {
                    dependencyCheck(
                        odcInstallation: 'DC',
                        additionalArguments: "--scan . --format XML --format HTML --nvdApiKey ${NVD_API_KEY}",
                        stopBuild: false
                    )
                }
            }
        }
        stage('Publish Dependency Check Report') {
            steps {
                dependencyCheckPublisher(pattern: '**/dependency-check-report.xml')
            }
        }
        stage('Trivy FS Scan') {
            steps {
                sh 'trivy fs --no-progress . > trivyfs.txt 2>&1 || true'
            }
            post {
                always {
                    archiveArtifacts artifacts: 'trivyfs.txt', allowEmptyArchive: true
                }
            }
        }
        stage('Docker Build & Push') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'docker',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
                    docker build --no-cache \
                        -t chilukurir/bms:latest \
                        -f bookmyshow-app/Dockerfile \
                        bookmyshow-app
                    echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                    docker push chilukurir/bms:latest
                    '''
                }
            }
        }
        stage('Trivy Image Scan') {
            steps {
                sh 'trivy image --no-progress chilukurir/bms:latest > trivyimage.txt 2>&1 || true'
            }
            post {
                always {
                    archiveArtifacts artifacts: 'trivyimage.txt', allowEmptyArchive: true
                }
            }
        }
        stage('Deploy to Container') {
            steps {
                sh '''
                docker stop bms || true
                docker rm bms || true
                docker run -d --restart=always --name bms -p 80:80 chilukurir/bms:latest
                docker ps -a
                '''
            }
        }
    }
    post {
        always {
            emailext(
                attachLog: true,
                subject: "${currentBuild.currentResult}: ${env.JOB_NAME}",
                body: "Project: ${env.JOB_NAME}<br/>Build Number: ${env.BUILD_NUMBER}<br/>URL: ${env.BUILD_URL}<br/>",
                to: 'chilukurirajesh@priaccinnovations.ai',
                attachmentsPattern: 'trivyfs.txt, trivyimage.txt, **/dependency-check-report.xml'
            )
        }
    }
}
```

---

## Part II: Kubernetes Deployment

### Step 6: EKS Pipeline Setup

Configure AWS credentials as the Jenkins user:

```bash
# Find which user Jenkins runs as
ps aux | grep jenkins

# Switch to Jenkins user
sudo -su jenkins

# Configure AWS credentials
aws configure
# Enter: Access Key ID, Secret Access Key, region: us-east-1, output: json

# Verify
aws sts get-caller-identity

# Configure kubeconfig
aws eks update-kubeconfig --name Bms-eks --region us-east-1

# Exit and restart Jenkins
exit
sudo systemctl restart jenkins
```

---

### Step 7: Production Dockerfile

> ⚠️ **Never use `npm start` (`react-scripts start`) in a container.** It is a development server that requires file watchers (`chokidar`) and will always crash in Kubernetes with `CrashLoopBackOff`.

The correct approach is a **multi-stage build**: Stage 1 builds static files, Stage 2 serves them with nginx.

Save this as `bookmyshow-app/Dockerfile`:

```dockerfile
# ─────────────────────────────────────────────────────
# Stage 1 — Build
# ─────────────────────────────────────────────────────
FROM node:18 AS builder

WORKDIR /app

COPY package.json package-lock.json ./

# Fix stale lock file / peer dependency issues
RUN npm install postcss@8.4.21 postcss-safe-parser@6.0.0 --legacy-peer-deps
RUN npm install --legacy-peer-deps

COPY . .

ENV NODE_OPTIONS=--openssl-legacy-provider

# Produces /app/build with static HTML/CSS/JS
RUN npm run build

# ─────────────────────────────────────────────────────
# Stage 2 — Serve with nginx (~50MB final image)
# ─────────────────────────────────────────────────────
FROM nginx:stable-alpine

RUN rm -rf /usr/share/nginx/html/*

COPY --from=builder /app/build /usr/share/nginx/html

# Handle React Router client-side routing
RUN echo 'server { \
    listen 80; \
    root /usr/share/nginx/html; \
    index index.html; \
    location / { try_files $uri $uri/ /index.html; } \
}' > /etc/nginx/conf.d/default.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

> **Why this matters:** The original `CMD ["npm", "start"]` Dockerfile caused `CrashLoopBackOff` on EKS because `chokidar` (a dev-only file watcher) wasn't installed. The multi-stage build eliminates Node.js entirely from the final image — no webpack, no dev server, nothing that can crash.

---

### Step 8: Kubernetes Manifests

Create a `k8s/` folder in your repo root for GitOps:

```
Book-My-Show/
├── bookmyshow-app/
│   └── Dockerfile
├── k8s/               ← GitOps manifests (ArgoCD watches this)
│   ├── deployment.yml
│   └── service.yml
└── Jenkinsfile
```

**k8s/deployment.yml:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: bms-app
  labels:
    app: bms
spec:
  replicas: 2
  selector:
    matchLabels:
      app: bms
  template:
    metadata:
      labels:
        app: bms
    spec:
      containers:
        - name: bms-container
          image: chilukurir/bms:latest
          ports:
            - containerPort: 80
          resources:
            requests:
              memory: "128Mi"
              cpu: "250m"
            limits:
              memory: "256Mi"
              cpu: "500m"
          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 10
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 15
            periodSeconds: 10
```

**k8s/service.yml:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: bms-service
spec:
  type: LoadBalancer
  selector:
    app: bms
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

Commit and push:

```bash
git add k8s/ bookmyshow-app/Dockerfile
git commit -m "ci: add k8s manifests and production Dockerfile"
git push
```

---

### Step 9: Final Jenkins Pipeline with ArgoCD

> This is the complete production pipeline. The `Deploy to EKS` stage is replaced by `Update Image Tag in Git` — Jenkins commits the new image tag to Git, and ArgoCD handles the actual deployment to EKS automatically.

```groovy
pipeline {
    agent any

    tools {
        jdk    'jdk17'
        nodejs 'node18'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        IMAGE_NAME   = 'chilukurir/bms'
        IMAGE_TAG    = "1.0.${BUILD_NUMBER}"
    }

    stages {

        stage('Clean Workspace') {
            steps { cleanWs() }
        }

        stage('Checkout from Git') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/Rajesh-210/Book-My-Show.git'
            }
        }

        stage('Install Dependencies') {
            steps {
                dir('bookmyshow-app') {
                    sh 'npm install --no-audit --prefer-offline'
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh """
                    \$SCANNER_HOME/bin/sonar-scanner \
                    -Dsonar.projectName=BMS \
                    -Dsonar.projectKey=BMS \
                    -Dsonar.sources=bookmyshow-app/src
                    """
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: false
                }
            }
        }

        stage('OWASP Dependency Check') {
            steps {
                withCredentials([
                    string(credentialsId: 'nvd-api-key', variable: 'NVD_API_KEY')
                ]) {
                    dependencyCheck(
                        odcInstallation: 'DC',
                        additionalArguments: "--scan . --format XML --format HTML --nvdApiKey ${NVD_API_KEY}",
                        stopBuild: false
                    )
                }
            }
        }

        stage('Publish Dependency Check Report') {
            steps {
                dependencyCheckPublisher(pattern: '**/dependency-check-report.xml')
            }
        }

        stage('Trivy File System Scan') {
            steps {
                sh '''
                trivy fs \
                    --exit-code 1 \
                    --severity HIGH,CRITICAL \
                    --no-progress \
                    . > trivyfs.txt 2>&1 || true
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: 'trivyfs.txt', allowEmptyArchive: true
                }
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
                docker build --no-cache \
                    -t $IMAGE_NAME:$IMAGE_TAG \
                    -t $IMAGE_NAME:latest \
                    -f bookmyshow-app/Dockerfile \
                    bookmyshow-app

                # Smoke test — verify container starts before pushing
                docker run --rm -d --name bms-test -p 8081:80 $IMAGE_NAME:$IMAGE_TAG
                sleep 5
                curl -f http://localhost:8081 || (docker stop bms-test && exit 1)
                docker stop bms-test
                '''
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh '''
                trivy image \
                    --exit-code 1 \
                    --severity HIGH,CRITICAL \
                    --no-progress \
                    $IMAGE_NAME:$IMAGE_TAG > trivyimage.txt 2>&1 || true
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: 'trivyimage.txt', allowEmptyArchive: true
                }
            }
        }

        stage('Docker Push') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'docker',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )
                ]) {
                    sh '''
                    echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                    docker push $IMAGE_NAME:$IMAGE_TAG
                    docker push $IMAGE_NAME:latest
                    '''
                }
            }
        }

        stage('Update Image Tag in Git') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'github',
                        usernameVariable: 'GIT_USER',
                        passwordVariable: 'GIT_TOKEN'
                    )
                ]) {
                    sh '''
                    rm -rf gitops-repo
                    git clone https://${GIT_USER}:${GIT_TOKEN}@github.com/Rajesh-210/Book-My-Show.git gitops-repo
                    cd gitops-repo

                    sed -i "s|image: $IMAGE_NAME:.*|image: $IMAGE_NAME:$IMAGE_TAG|g" k8s/deployment.yml

                    grep "image:" k8s/deployment.yml

                    git config user.email "jenkins@bms.com"
                    git config user.name "Jenkins"
                    git add k8s/deployment.yml
                    git commit -m "ci: update image tag to $IMAGE_TAG [skip ci]"
                    git push
                    '''
                }
            }
        }

    }

    post {
        always {
            emailext(
                attachLog: true,
                subject: "${currentBuild.currentResult}: ${env.JOB_NAME}",
                body: """
                Project: ${env.JOB_NAME}<br/>
                Build Number: ${env.BUILD_NUMBER}<br/>
                Build URL: ${env.BUILD_URL}<br/>
                Image Tag: ${env.IMAGE_TAG}<br/>
                """,
                to: 'chilukurirajesh@priaccinnovations.ai',
                attachmentsPattern: 'trivyfs.txt, trivyimage.txt, **/dependency-check-report.xml'
            )
        }
    }

}
```

---

## Part III: GitOps with ArgoCD

### Step 10: Install ArgoCD on EKS

```bash
# Create namespace
kubectl create namespace argocd

# Install ArgoCD
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Watch pods come up (wait until all show 1/1 Running)
kubectl get pods -n argocd -w
```

Expose the UI via LoadBalancer:

```bash
kubectl patch svc argocd-server -n argocd \
  -p '{"spec": {"type": "LoadBalancer"}}'

# Get external URL (wait ~2 mins for ELB)
kubectl get svc argocd-server -n argocd
```

Get the initial admin password:

```bash
kubectl get secret argocd-initial-admin-secret \
  -n argocd \
  -o jsonpath="{.data.password}" | base64 -d; echo
```

> Access ArgoCD at `http://<argocd-external-ip>` — login: `admin` / `<password above>`

---

### Step 11: Create ArgoCD Application

```bash
cat > argocd-app.yml << 'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: bms-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/Rajesh-210/Book-My-Show.git
    targetRevision: main
    path: k8s
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true       # removes resources deleted from Git
      selfHeal: true    # reverts manual kubectl changes
    syncOptions:
      - CreateNamespace=true
EOF

kubectl apply -f argocd-app.yml

# Verify
kubectl get application -n argocd
```

Expected output:
```
NAME      SYNC STATUS   HEALTH STATUS
bms-app   Synced        Healthy
```

**How GitOps works end-to-end:**

```
Jenkins pushes new image tag to k8s/deployment.yml in Git
              ↓
ArgoCD detects the change (polls every 3 minutes)
              ↓
ArgoCD syncs the new deployment to EKS automatically
              ↓
Rolling update — zero downtime pod replacement
```

---

## Part IV: Monitoring Stack

> The monitoring stack runs **inside EKS** as pods managed by Helm. No separate Monitoring Server VM is needed.

### Step 12: Install kube-prometheus-stack via Helm

**12.1 Install Helm**

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-sh | bash
helm version
```

**12.2 Set gp2 as default StorageClass** *(skip if already done in Step 1.3.5)*

```bash
kubectl patch storageclass gp2 \
  -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

**12.3 Add Prometheus Helm repo**

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

**12.4 Create values file**

```bash
cat > prometheus-values.yaml << 'EOF'
grafana:
  adminPassword: "Admin@123"
  service:
    type: LoadBalancer
  persistence:
    enabled: false

prometheus:
  service:
    type: LoadBalancer
  prometheusSpec:
    retention: 24h
    storageSpec: {}
    additionalScrapeConfigs:
      - job_name: 'jenkins'
        metrics_path: '/prometheus'
        static_configs:
          - targets: ['<JenkinsPublicIP>:8080']   # replace with Jenkins public IP

nodeExporter:
  enabled: true

alertmanager:
  enabled: false
EOF
```

> ⚠️ Replace `<JenkinsPublicIP>` with your Jenkins server's actual public IP. Also ensure port `8080` is open in the Jenkins security group to the EKS node security group.

**12.5 Install the stack**

```bash
helm install prometheus-stack prometheus-community/kube-prometheus-stack \
  -f prometheus-values.yaml \
  --namespace monitoring \
  --create-namespace

# Verify all pods are Running
kubectl get pods -n monitoring
```

**12.6 Get external URLs**

```bash
kubectl get svc -n monitoring | grep LoadBalancer
```

| Service | Port | Access |
|---|---|---|
| Grafana | 80 | `http://<grafana-external-ip>` |
| Prometheus | 9090 | `http://<prometheus-external-ip>:9090` |

Grafana login: `admin` / `Admin@123`

**12.7 Enable Jenkins Prometheus endpoint**

In Jenkins: **Manage Jenkins → Plugins → Available** → search `Prometheus` → install **Prometheus metrics plugin**.

Verify: `http://<jenkins-ip>:8080/prometheus` should return metrics.

**12.8 Upgrade config after changing values**

```bash
helm upgrade prometheus-stack prometheus-community/kube-prometheus-stack \
  -f prometheus-values.yaml \
  --namespace monitoring
```

---

### Step 13: Grafana Dashboards

In Grafana: **Dashboards → New → Import → enter ID → Load → select Prometheus datasource → Import**

| Dashboard | ID | Shows |
|---|---|---|
| Node Exporter Full | `1860` | CPU, Memory, Disk, Network per EKS node |
| Kubernetes Cluster | `6417` | Pod count, deployments, resource usage |
| Jenkins Performance & Health | `9964` | Build stats, JVM memory, executor usage |

**Verify Prometheus targets at** `http://<prometheus-ip>:9090/targets`:

- `kubernetes-nodes` — UP
- `node-exporter` — UP (one per EKS node)
- `jenkins` — UP (requires Prometheus plugin installed)

---

## Cleanup

Once done, delete all AWS resources to avoid charges:

```bash
# Remove ArgoCD application
kubectl delete -f argocd-app.yml

# Remove monitoring stack
helm uninstall prometheus-stack -n monitoring
kubectl delete namespace monitoring

# Remove ArgoCD
kubectl delete namespace argocd

# Delete EKS node group
eksctl delete nodegroup --cluster=Bms-eks --name=node2 --region=us-east-1

# Delete EKS cluster
eksctl delete cluster --name=Bms-eks --region=us-east-1
```

Also terminate:
- `BMS-Server` EC2 instance
- Any associated Security Groups, IAM users, and Access Keys

---

> 📌 **Repository:** [https://github.com/Rajesh-210/Book-My-Show](https://github.com/Rajesh-210/Book-My-Show)
