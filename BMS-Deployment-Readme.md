# 🎬 Book My Show Clone — DevOps Deployment Guide

> **Author:** Kastro Kiran V  
> **Stack:** Docker · Jenkins · SonarQube · Trivy · Amazon EKS · Prometheus · Grafana

---

## 📋 Table of Contents

- [Architecture Overview](#architecture-overview)
- [Part I: Docker Deployment](#part-i-docker-deployment)
  - [Step 1: Infrastructure Setup](#step-1-infrastructure-setup)
  - [Step 2: Tools Installation](#step-2-tools-installation)
  - [Step 3: Jenkins Configuration](#step-3-jenkins-configuration)
  - [Step 4: Email Integration](#step-4-email-integration)
  - [Step 5: Pipeline Job](#step-5-pipeline-job)
- [Part II: Kubernetes Deployment & Monitoring](#part-ii-kubernetes-deployment--monitoring)
  - [Step 6: EKS Pipeline Setup](#step-6-eks-pipeline-setup)
  - [Step 7: Monitoring Stack](#step-7-monitoring-stack)
- [Pipeline Scripts](#pipeline-scripts)
- [Cleanup](#cleanup)

---

## Architecture Overview

```
Developer → GitHub (main branch)
              ↓
           Jenkins CI/CD Pipeline
              ├── SonarQube Analysis (Code Quality)
              ├── Trivy FS Scan (Security)
              ├── Docker Build & Push (DockerHub)
              └── Deploy
                    ├── Docker Container  (Part I)
                    └── Amazon EKS        (Part II)
                              ↓
                     Prometheus + Grafana (Monitoring)
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

| Type | Protocol | Port Range | Purpose |
|---|---|---|---|
| SSH | TCP | 22 | Remote access |
| HTTP | TCP | 80 | Web traffic |
| HTTPS | TCP | 443 | Secure web traffic |
| SMTP | TCP | 25 | Mail server communication |
| SMTPS | TCP | 465 | Secure email (SSL/TLS) |
| Custom TCP | TCP | 3000–10000 | Node.js, Grafana, Jenkins, apps |
| Custom TCP | TCP | 6443 | Kubernetes API server |
| Custom TCP | TCP | 30000–32767 | Kubernetes NodePort range |

---

#### 1.2 Create IAM User for EKS

> ⚠️ Do **not** use the root account to create EKS clusters.

**1.2.1** Create a new IAM user in the AWS Console.

**1.2.2** Attach the following managed policies:

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

**1.2.3** Generate **Access Keys** for the IAM user.

---

#### 1.3 Create EKS Cluster

Connect to `BMS-Server` and run:

```bash
sudo apt update
```

**1.3.1 Install AWS CLI**

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
sudo apt install unzip
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

**(a) Create cluster (no node group yet):**

```bash
eksctl create cluster --name=kastro-eks \
                      --region=us-east-1 \
                      --zones=us-east-1a,us-east-1b \
                      --version=1.30 \
                      --without-nodegroup
```

> ⏱ Takes 5–10 minutes. Verify in the EKS Console.

**(b) Associate IAM OIDC Provider** *(enables IAM Roles for Service Accounts)*:

```bash
eksctl utils associate-iam-oidc-provider \
    --region us-east-1 \
    --cluster kastro-eks \
    --approve
```

**(c) Create Node Group** *(replace `Kastro` with your PEM file name — no `.pem` extension)*:

```bash
eksctl create nodegroup --cluster=kastro-eks \
                       --region=us-east-1 \
                       --name=node2 \
                       --node-type=t3.medium \
                       --nodes=3 \
                       --nodes-min=2 \
                       --nodes-max=4 \
                       --node-volume-size=20 \
                       --ssh-access \
                       --ssh-public-key=Kastro \
                       --managed \
                       --asg-access \
                       --external-dns-access \
                       --full-ecr-access \
                       --appmesh-access \
                       --alb-ingress-access
```

> ⏱ Takes 5–10 minutes.

**(d)** In the **EKS Cluster Security Group**, open **All Traffic** to allow internal communication between the control plane and worker nodes.

---

### Step 2: Tools Installation

#### 2.1.1 Install Jenkins

```bash
vi Jenkins.sh
```

Paste the following and save:

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

> Open port **8080** and complete the Jenkins setup wizard.

---

#### 2.1.2 Install Docker

```bash
vi docker.sh
```

Paste the following and save:

```bash
#!/bin/bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
https://download.docker.com/linux/ubuntu \
$(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

```bash
sudo chmod +x docker.sh && ./docker.sh
docker --version
```

If image pulls fail, fix Docker socket permissions:

```bash
sudo chmod 666 /var/run/docker.sock
```

Login to DockerHub:

```bash
docker login -u <DockerHubUsername>
```

---

#### 2.1.3 Install Trivy

```bash
vi trivy.sh
```

Paste the following and save:

```bash
#!/bin/bash
sudo apt-get install wget apt-transport-https gnupg
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key \
  | gpg --dearmor | sudo tee /usr/share/keyrings/trivy.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] \
  https://aquasecurity.github.io/trivy-repo/deb generic main" \
  | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy
```

```bash
sudo chmod +x trivy.sh && ./trivy.sh
trivy --version
```

---

#### 2.2 Setup SonarQube

Run SonarQube as a Docker container:

```bash
docker run -d --name sonar -p 9000:9000 sonarqube:lts-community
```

> Open port **9000**, then access `http://<server-ip>:9000`.  
> Default credentials: `admin` / `admin` — set a new password on first login.

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
| Kubernetes | Kubernetes, Kubernetes CLI, Kubernetes Client API, Kubernetes Credentials, Config File Provider |
| Observability | Prometheus metrics |
| UI | Pipeline Stage View |
| Notifications | Email Extension Template |

#### 3.2 Create SonarQube Token

In SonarQube: **My Account → Security → Generate Token**

Example token: `squ_69eb05b41575c699579c6ced901eaafae66d63a2`

Add this token as a Jenkins credential (type: **Secret text**, ID: `Sonar-token`).

#### 3.3 Configure Credentials

Add the following credentials in **Manage Jenkins → Credentials → Global**:

| ID | Kind | Purpose |
|---|---|---|
| `Sonar-token` | Secret Text | SonarQube authentication |
| `docker` | Username with Password | DockerHub login |
| `email-creds` | Username with Password | Gmail SMTP |

#### 3.4 Configure Tools

Navigate to **Manage Jenkins → Tools** and configure the following:

**JDK:**
- Click **Add JDK** → Name: `jdk17`
- Check **Install automatically** → Add Installer → **Install from adoptium.net**
- Version: `jdk-17.0.x+x` (latest 17)

**NodeJS:**
- Click **Add NodeJS** → Name: `node23`
- Check **Install automatically**
- Version: `NodeJS 23.x`

**SonarQube Scanner:**
- Click **Add SonarQube Scanner** → Name: `sonar-scanner`
- Check **Install automatically** → select latest version

**Docker:**
- Click **Add Docker** → Name: `docker`
- Check **Install automatically** → **Download from docker.com** → latest version

Click **Apply → Save**.

#### 3.5 System Configuration

**Manage Jenkins → System → SonarQube Servers:**

- Name: `sonar-server`
- URL: `http://<server-ip>:9000`
- Token: select `Sonar-token` credential

---

### Step 4: Email Integration

**4.1 Generate Gmail App Password**

1. Go to **Google Account → Search "App Passwords"**
2. App name: `jenkins` → Create
3. Copy the 16-character password (remove spaces)

**4.2 Add Email Credential in Jenkins**

**Manage Jenkins → Credentials → Global → Add Credential:**

| Field | Value |
|---|---|
| Kind | Username with Password |
| Username | your-email@gmail.com |
| Password | `<app-password>` |
| ID | `email-creds` |

**4.3 Configure Extended Email Notification**

**Manage Jenkins → System → Extended Email Notification:**

| Field | Value |
|---|---|
| SMTP Server | `smtp.gmail.com` |
| SMTP Port | `465` |
| Credentials | `email-creds` |
| Use SSL | ✅ |
| Default Content Type | HTML |

**4.4 Configure Email Notification (test)**

**Manage Jenkins → System → Email Notification:**

| Field | Value |
|---|---|
| SMTP Server | `smtp.gmail.com` |
| Use SMTP Authentication | ✅ |
| Username | your-email@gmail.com |
| Password | `<app-password>` |
| Use SSL | ✅ |
| SMTP Port | `465` |

Click **Test Configuration** — verify the test email arrives.

Under **Default Triggers**: check ✅ **Always**, ✅ **Failure-Any**, ✅ **Success**.

**4.5 Install NPM**

```bash
apt install npm
```

---

### Step 5: Pipeline Job

> Before pasting the pipeline script, update:
> - Your DockerHub username in the `Docker Build & Push` and `Deploy to Container` stages
> - Your email address in the `post` block

#### Pipeline Script (Docker Only — No K8S)

```groovy
pipeline {
    agent any
    tools {
        jdk 'jdk17'
        nodejs 'node23'
    }
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }
    stages {
        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }
        stage('Checkout from Git') {
            steps {
                git branch: 'main', url: 'https://github.com/KastroVKiran/Book-My-Show.git'
                sh 'ls -la'
            }
        }
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh '''
                    $SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectName=BMS \
                        -Dsonar.projectKey=BMS
                    '''
                }
            }
        }
        stage('Quality Gate') {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token'
                }
            }
        }
        stage('Install Dependencies') {
            steps {
                sh '''
                cd bookmyshow-app
                ls -la
                if [ -f package.json ]; then
                    rm -rf node_modules package-lock.json
                    npm install
                else
                    echo "Error: package.json not found in bookmyshow-app!"
                    exit 1
                fi
                '''
            }
        }
        stage('Trivy FS Scan') {
            steps {
                sh 'trivy fs . > trivyfs.txt'
            }
        }
        stage('Docker Build & Push') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker', toolName: 'docker') {
                        sh '''
                        docker build --no-cache -t kastrov/bms:latest -f bookmyshow-app/Dockerfile bookmyshow-app
                        docker push kastrov/bms:latest
                        '''
                    }
                }
            }
        }
        stage('Trivy Image Scan') {
            steps {
                sh 'trivy image kastrov/bms:latest > trivyimage.txt'
            }
        }
        stage('Deploy to Container') {
            steps {
                sh '''
                docker stop bms || true
                docker rm bms || true
                docker run -d --restart=always --name bms -p 3000:3000 kastrov/bms:latest
                docker ps -a
                sleep 5
                docker logs bms
                '''
            }
        }
    }
    post {
        always {
            emailext attachLog: true,
                subject: "'${currentBuild.result}'",
                body: "Project: ${env.JOB_NAME}<br/>" +
                      "Build Number: ${env.BUILD_NUMBER}<br/>" +
                      "URL: ${env.BUILD_URL}<br/>",
                to: 'kastrokiran@gmail.com',
                attachmentsPattern: 'trivyfs.txt,trivyimage.txt'
        }
    }
}
```

> ✅ Access the BMS App at `http://<BMS-Server-PublicIP>:3000`

---

## Part II: Kubernetes Deployment & Monitoring

### Step 6: EKS Pipeline Setup

**Configure AWS credentials as the Jenkins user:**

```bash
# Find which user Jenkins runs as
ps aux | grep jenkins

# Switch to Jenkins user
sudo -su jenkins

# Configure AWS credentials
aws configure
# (enter Access Key ID, Secret Access Key, region: us-east-1, output: json)

# Verify credentials
aws sts get-caller-identity
```

Expected output:
```json
{
    "UserId": "EXAMPLEUSERID",
    "Account": "123456789012",
    "Arn": "arn:aws:iam::123456789012:user/example-user"
}
```

```bash
# Exit Jenkins user, restart Jenkins
exit
sudo systemctl restart jenkins

# Switch back to Jenkins user and configure kubeconfig
sudo -su jenkins
aws eks update-kubeconfig --name kastro-eks --region us-east-1
```

#### Pipeline Script (With EKS Deployment)

```groovy
pipeline {
    agent any

    tools {
        jdk 'jdk17'
        nodejs 'node23'
    }

    environment {
        SCANNER_HOME    = tool 'sonar-scanner'
        DOCKER_IMAGE    = 'kastrov/bms:latest'
        EKS_CLUSTER_NAME = 'kastro-eks'
        AWS_REGION      = 'us-east-1'
    }

    stages {
        stage('Clean Workspace') {
            steps { cleanWs() }
        }

        stage('Checkout from Git') {
            steps {
                git branch: 'main', url: 'https://github.com/KastroVKiran/Book-My-Show.git'
                sh 'ls -la'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh '''
                    $SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectName=BMS \
                        -Dsonar.projectKey=BMS
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token'
                }
            }
        }

        stage('Install Dependencies') {
            steps {
                sh '''
                cd bookmyshow-app
                ls -la
                if [ -f package.json ]; then
                    rm -rf node_modules package-lock.json
                    npm install
                else
                    echo "Error: package.json not found in bookmyshow-app!"
                    exit 1
                fi
                '''
            }
        }

        stage('Trivy FS Scan') {
            steps {
                sh 'trivy fs . > trivyfs.txt'
            }
        }

        stage('Docker Build & Push') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker', toolName: 'docker') {
                        sh '''
                        docker build --no-cache -t $DOCKER_IMAGE \
                            -f bookmyshow-app/Dockerfile bookmyshow-app
                        docker push $DOCKER_IMAGE
                        '''
                    }
                }
            }
        }

        stage('Deploy to EKS Cluster') {
            steps {
                script {
                    sh '''
                    aws sts get-caller-identity
                    aws eks update-kubeconfig --name $EKS_CLUSTER_NAME --region $AWS_REGION
                    kubectl config view
                    kubectl apply -f deployment.yml
                    kubectl apply -f service.yml
                    kubectl get pods
                    kubectl get svc
                    '''
                }
            }
        }
    }

    post {
        always {
            emailext attachLog: true,
                subject: "'${currentBuild.result}'",
                body: "Project: ${env.JOB_NAME}<br/>" +
                      "Build Number: ${env.BUILD_NUMBER}<br/>" +
                      "URL: ${env.BUILD_URL}<br/>",
                to: 'kastrokiran@gmail.com',
                attachmentsPattern: 'trivyfs.txt'
        }
    }
}
```

---

### Step 7: Monitoring Stack

**Launch a new VM:**

| Setting | Value |
|---|---|
| Name | `Monitoring-Server` |
| OS | Ubuntu 22.04 |
| Instance Type | `t2.medium` |
| Open Ports | 9090 (Prometheus), 9100 (Node Exporter), 3000 (Grafana) |

---

#### 7.1 Install Prometheus

```bash
sudo apt update

# Create Prometheus system user
sudo useradd --system --no-create-home --shell /bin/false prometheus

# Download and extract
sudo wget https://github.com/prometheus/prometheus/releases/download/v2.47.1/prometheus-2.47.1.linux-amd64.tar.gz
tar -xvf prometheus-2.47.1.linux-amd64.tar.gz
sudo mkdir -p /data /etc/prometheus
cd prometheus-2.47.1.linux-amd64/

# Move binaries and config
sudo mv prometheus promtool /usr/local/bin/
sudo mv consoles/ console_libraries/ /etc/prometheus/
sudo mv prometheus.yml /etc/prometheus/prometheus.yml

# Set ownership
sudo chown -R prometheus:prometheus /etc/prometheus/ /data/

# Clean up
cd && rm -rf prometheus-2.47.1.linux-amd64.tar.gz prometheus-2.47.1.linux-amd64/
prometheus --version
```

Create the systemd service:

```bash
sudo vi /etc/systemd/system/prometheus.service
```

```ini
[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target
StartLimitIntervalSec=500
StartLimitBurst=5

[Service]
User=prometheus
Group=prometheus
Type=simple
Restart=on-failure
RestartSec=5s
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/data \
  --web.console.templates=/etc/prometheus/consoles \
  --web.console.libraries=/etc/prometheus/console_libraries \
  --web.listen-address=0.0.0.0:9090 \
  --web.enable-lifecycle

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl enable prometheus
sudo systemctl start prometheus
sudo systemctl status prometheus
```

> Access Prometheus: `http://<monitoring-server-ip>:9090`

---

#### 7.2 Install Node Exporter

```bash
sudo useradd --system --no-create-home --shell /bin/false node_exporter

wget https://github.com/prometheus/node_exporter/releases/download/v1.6.1/node_exporter-1.6.1.linux-amd64.tar.gz
tar -xvf node_exporter-1.6.1.linux-amd64.tar.gz
sudo mv node_exporter-1.6.1.linux-amd64/node_exporter /usr/local/bin/
rm -rf node_exporter*

node_exporter --version
```

Create the systemd service:

```bash
sudo vi /etc/systemd/system/node_exporter.service
```

```ini
[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target
StartLimitIntervalSec=500
StartLimitBurst=5

[Service]
User=node_exporter
Group=node_exporter
Type=simple
Restart=on-failure
RestartSec=5s
ExecStart=/usr/local/bin/node_exporter --collector.logind

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl enable node_exporter
sudo systemctl start node_exporter
sudo systemctl status node_exporter
```

---

#### 7.3 Configure Prometheus Scrape Jobs

```bash
sudo vi /etc/prometheus/prometheus.yml
```

Append to the end of the file:

```yaml
  - job_name: 'node_exporter'
    static_configs:
      - targets: ['<MonitoringServerIP>:9100']

  - job_name: 'jenkins'
    metrics_path: '/prometheus'
    static_configs:
      - targets: ['<JenkinsServerIP>:8080']
```

> ⚠️ Use port `9100` for Node Exporter even though Prometheus runs on `9090`.

Validate and reload:

```bash
promtool check config /etc/prometheus/prometheus.yml
# Expected: SUCCESS

curl -X POST http://localhost:9090/-/reload
```

Verify targets at `http://<monitoring-ip>:9090/targets` — all three should show **UP**:

- `prometheus (1/1 up)`
- `node_exporter (1/1 up)`
- `jenkins (1/1 up)`

---

#### 7.4 Install Grafana

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https software-properties-common

cd ~
wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -

echo "deb https://packages.grafana.com/oss/deb stable main" \
  | sudo tee -a /etc/apt/sources.list.d/grafana.list

sudo apt-get update
sudo apt-get -y install grafana

sudo systemctl enable grafana-server
sudo systemctl start grafana-server
sudo systemctl status grafana-server
```

> Access Grafana: `http://<monitoring-server-ip>:3000`  
> Default credentials: `admin` / `admin`

**Add Prometheus as a Data Source:**

1. In Grafana, go to **Home → Connections → Data Sources**
2. Click **Add data source**
3. Select **Prometheus**
4. Set the URL to `http://localhost:9090`
5. Scroll down and click **Save & Test** — you should see a green "Data source is working" confirmation

**Import Dashboards:**

| Dashboard | Grafana URL |
|---|---|
| Node Exporter Full | https://grafana.com/grafana/dashboards/1860 |
| Jenkins Performance & Health | https://grafana.com/grafana/dashboards/9964 |

To import: **Dashboards → New → Import → Paste dashboard ID or URL**

---

## Pipeline Scripts

| Script | Description |
|---|---|
| [Docker-only pipeline](#pipeline-script-docker-only--no-k8s) | Build, scan, push, and deploy to a Docker container |
| [EKS pipeline](#pipeline-script-with-eks-deployment) | Full CI/CD including EKS cluster deployment |

---

## Cleanup

Once done, delete all AWS resources to avoid charges:

```bash
# Delete EKS node group
eksctl delete nodegroup --cluster=kastro-eks --name=node2 --region=us-east-1

# Delete EKS cluster
eksctl delete cluster --name=kastro-eks --region=us-east-1
```

Also terminate:
- `BMS-Server` EC2 instance
- `Monitoring-Server` EC2 instance
- Any associated Security Groups, IAM users, and Access Keys

---

> 📌 **Repository:** [https://github.com/KastroVKiran/Book-My-Show](https://github.com/KastroVKiran/Book-My-Show)
