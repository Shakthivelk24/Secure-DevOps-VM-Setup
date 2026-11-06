# Secure DevOps VM Setup

A step-by-step implementation of a secure DevOps environment on an **Azure Virtual Machine (Ubuntu 24.04 LTS)** using Jenkins, Docker, SonarQube, Trivy, Prometheus, and Grafana.

This project demonstrates how to configure a cloud-based DevOps environment, automate application deployment, integrate security checks, and monitor application and infrastructure performance.

## 🚀 Project Overview

The objective of this project is to establish a centralized DevOps environment on an Azure Virtual Machine that supports:

- Continuous Integration and Continuous Deployment (CI/CD).
- Automated application builds and deployments.
- Static Application Security and Code Quality Analysis.
- Container Image Vulnerability Scanning.
- Containerized Application Deployment.
- Infrastructure and Application Monitoring.
- Performance visualization and alerting.

## 🏗️ Architecture

```text
                 Developer
                     |
                     v
                GitHub Repo
                     |
                     v
                  Jenkins
                     |
          +----------+----------+
          |                     |
          v                     v
      Build App           SonarQube
          |               Code Analysis
          v                     |
      Docker Build              |
          |                     |
          v                     |
       Trivy Scan <-------------+
          |
          v
    Docker Compose
          |
     +----+----+
     |         |
     v         v
  Backend   Frontend
     |
     v
  Application
     |
     v
 Monitoring Stack
     |
  +--+----------------+
  |                   |
Prometheus          Grafana
  |                   |
  v                   v
Metrics           Dashboards
```

## ☁️ Cloud Infrastructure

| Component | Configuration |
|---|---|
| Cloud Provider | Microsoft Azure |
| Virtual Machine | Azure Linux VM |
| Operating System | Ubuntu 24.04 LTS |
| VM Username | `azureuser` |
| Region | Central India |
| Access Method | SSH |
| Deployment Platform | Docker |

## 🛠️ Technologies Used

| Category | Tools |
|---|---|
| Cloud Platform | Microsoft Azure |
| Operating System | Ubuntu Linux |
| CI/CD | Jenkins |
| Version Control | Git, GitHub |
| Containerization | Docker |
| Container Orchestration | Docker Compose |
| Code Quality Analysis | SonarQube |
| Vulnerability Scanning | Trivy |
| Monitoring | Prometheus |
| Visualization | Grafana |
| Infrastructure Metrics | Node Exporter |
| Container Metrics | cAdvisor |
| Application Runtime | Node.js |

## 📋 Prerequisites

Before starting the setup, ensure you have:

- An Azure account.
- An Ubuntu 24.04 LTS virtual machine.
- SSH access to the VM.
- A GitHub repository.
- A Docker Hub account.
- Basic knowledge of Linux commands, Git, Docker, and Jenkins.

## 1. Create an Azure Virtual Machine

1. Sign in to the [Azure Portal](https://portal.azure.com/).
2. Navigate to **Virtual Machines**.
3. Select **Create Virtual Machine**.
4. Choose Ubuntu 24.04 LTS as the operating system.
5. Configure the VM size, authentication, disk, and networking.
6. Download and securely store the SSH private key.
7. Create the virtual machine.

### Connect to the VM

From Windows PowerShell:

```powershell
ssh -i "path-to-private-key.pem" azureuser@YOUR_VM_PUBLIC_IP
```

Replace `YOUR_VM_PUBLIC_IP` with the public IP address assigned to your VM.

## 2. Update the System

```bash
sudo apt update
sudo apt upgrade -y
```

Install essential utilities:

```bash
sudo apt install -y curl wget git unzip \
    ca-certificates gnupg fontconfig
```

## 3. Install Java

Java is required to run Jenkins.

```bash
sudo apt install -y openjdk-25-jre
```

Verify installation:

```bash
java -version
javac -version
```

## 4. Install Git

```bash
sudo apt install -y git
```

Verify:

```bash
git --version
```

## 5. Install Node.js and npm

Install Node.js 22:

```bash
sudo apt install -y curl ca-certificates

curl -fsSL https://deb.nodesource.com/setup_22.x \
    -o nodesource_setup.sh

sudo -E bash nodesource_setup.sh

sudo apt install -y nodejs build-essential
```

Verify:

```bash
node --version
npm --version
```

## 6. Install Docker Engine

Configure the official Docker repository:

```bash
sudo install -m 0755 -d /etc/apt/keyrings

sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
    -o /etc/apt/keyrings/docker.asc

sudo chmod a+r /etc/apt/keyrings/docker.asc
```

Add the Docker repository:

```bash
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" \
    | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

Install Docker:

```bash
sudo apt update

sudo apt install -y docker-ce docker-ce-cli \
    containerd.io docker-buildx-plugin docker-compose-plugin
```

Enable Docker:

```bash
sudo systemctl enable --now docker
```

Verify:

```bash
sudo docker run hello-world
docker --version
docker compose version
```

## 7. Install Jenkins

Add the Jenkins repository:

```bash
sudo mkdir -p /etc/apt/keyrings

sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc \
    https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key

echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/" \
    | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
```

Install Jenkins:

```bash
sudo apt update
sudo apt install -y jenkins
```

Start and enable Jenkins:

```bash
sudo systemctl enable --now jenkins
```

Check status:

```bash
sudo systemctl status jenkins
```

Retrieve the initial administrator password:

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

Access Jenkins:

```text
http://YOUR_VM_PUBLIC_IP:8080
```

Complete the initial configuration and install the required plugins.

## 8. Configure Docker Access for Jenkins

Allow the Jenkins service account to access Docker:

```bash
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
```

Verify Docker access using the Jenkins service account:

```bash
sudo -u jenkins docker version
```

If permissions are not updated, verify group membership and restart the Jenkins service.

**Security note:** Membership in the Docker group effectively grants root-level control over the host. Use this configuration only when appropriate for your environment.

## 9. Install SonarQube

Configure the required Linux kernel parameters:

```bash
sudo sysctl -w vm.max_map_count=524288
sudo sysctl -w fs.file-max=131072
```

Persist the configuration:

```bash
echo "vm.max_map_count=524288" | \
    sudo tee /etc/sysctl.d/99-sonarqube.conf

echo "fs.file-max=131072" | \
    sudo tee -a /etc/sysctl.d/99-sonarqube.conf

sudo sysctl --system
```

Create persistent Docker volumes:

```bash
sudo docker volume create sonarqube_data
sudo docker volume create sonarqube_extensions
sudo docker volume create sonarqube_logs
```

Pull and run SonarQube:

```bash
sudo docker pull sonarqube:community

sudo docker run -d \
    --name sonarqube \
    --restart unless-stopped \
    -p 9000:9000 \
    -v sonarqube_data:/opt/sonarqube/data \
    -v sonarqube_extensions:/opt/sonarqube/extensions \
    -v sonarqube_logs:/opt/sonarqube/logs \
    sonarqube:community
```

Check the container:

```bash
sudo docker ps
sudo docker logs -f sonarqube
```

Access SonarQube:

```text
http://YOUR_VM_PUBLIC_IP:9000
```

Generate a SonarQube token from the account security settings and store it securely in Jenkins Credentials.

## 10. Install Trivy

Add the Trivy repository:

```bash
sudo apt install -y wget gnupg

wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key \
    | sudo gpg --dearmor \
    -o /usr/share/keyrings/trivy.gpg

echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb generic main" \
    | sudo tee /etc/apt/sources.list.d/trivy.list
```

Install:

```bash
sudo apt update
sudo apt install -y trivy
```

Verify:

```bash
trivy --version
```

Scan a Docker image:

```bash
trivy image IMAGE_NAME:TAG
```

## 11. Configure Prometheus and Grafana Monitoring

The monitoring stack is used to observe VM resources, Docker containers, and application metrics.

### Monitoring Components

- **Prometheus:** Collects and stores metrics.
- **Grafana:** Visualizes metrics through dashboards.
- **Node Exporter:** Exposes Linux system metrics.
- **cAdvisor:** Exposes Docker container metrics.

### Monitoring Directory Structure

```text
monitoring/
│
├── docker-compose.yml
│
├── prometheus/
│   ├── prometheus.yml
│   └── alerts.yml
│
└── grafana/
    ├── dashboards/
    │   ├── dashboard.json
    │   └── dashboard.yaml
    │
    └── datasources/
        └── datasource.yml
```

### Start Monitoring Services

Navigate to the monitoring directory:

```bash
cd monitoring
```

Validate the Compose configuration:

```bash
sudo docker compose config
```

Pull images:

```bash
sudo docker compose pull
```

Start services:

```bash
sudo docker compose up -d
```

Check running containers:

```bash
sudo docker compose ps
```

### Access Monitoring Dashboards

| Service | URL |
|---|---|
| Grafana | `http://YOUR_VM_PUBLIC_IP:3000` |
| Prometheus | `http://localhost:9090` |

For Prometheus access from your local computer, use an SSH tunnel:

```powershell
ssh -L 9090:127.0.0.1:9090 -i "path-to-private-key.pem" azureuser@YOUR_VM_PUBLIC_IP
```

Then open:

```text
http://localhost:9090
```

Configure Grafana with Prometheus as its data source.

## 12. Jenkins CI/CD Pipeline

The Jenkins pipeline automates application integration, security analysis, and deployment.

### Pipeline Stages

1. **Checkout:** Retrieve application source code from GitHub.
2. **Install Dependencies:** Install required application dependencies.
3. **Build:** Build the application.
4. **SonarQube Analysis:** Perform code quality and static analysis.
5. **Docker Build:** Build frontend and backend container images.
6. **Trivy Scan:** Scan Docker images for known vulnerabilities.
7. **Docker Push:** Push approved images to Docker Hub.
8. **Deploy:** Deploy the application using Docker Compose.
9. **Monitoring:** Monitor application and infrastructure metrics.

### Example Deployment Stage

```groovy
stage('Deploy') {
    steps {
        script {
            if (isUnix()) {
                sh '''
                    set -e

                    docker rm -f backend frontend 2>/dev/null || true

                    docker compose up -d --build --remove-orphans

                    docker compose ps
                '''
            } else {
                bat '''
                    @echo off
                    docker rm -f backend frontend 2>nul
                    docker compose up -d --build --remove-orphans
                    if errorlevel 1 exit /b 1
                    docker compose ps
                '''
            }
        }
    }
}
```

**Note:** This example assumes the containers are safe to replace and use the exact names `backend` and `frontend`. For production deployments, prefer project-scoped Compose services and avoid hardcoded container names.

## 13. Jenkins Credentials Configuration

Configure the following credentials in:

**Jenkins → Manage Jenkins → Credentials**

| Credential | Purpose |
|---|---|
| GitHub Credentials | Private repository access, if required |
| SonarQube Token | Code quality analysis |
| Docker Hub Credentials | Push and pull Docker images |
| Gmail SMTP Credentials | Email notifications |

Use Jenkins credential IDs in the pipeline instead of hardcoding passwords, tokens, or other secrets.

## 14. Application and Infrastructure Monitoring

The monitoring setup can collect the following metrics:

### System Metrics
- CPU Utilization
- Memory Utilization
- Disk Usage
- System Availability
- Container CPU and Memory Usage

### Application Metrics
- Total HTTP Requests
- HTTP Request Rate
- HTTP Request Duration
- Successful Logins
- Failed Logins
- User Registrations
- Active Users
- Database Queries
- Server Errors
- AI Assistant Requests
- Cloudinary Uploads

### Example Prometheus Queries

HTTP Request Rate:

```promql
sum(rate(http_requests_total[5m]))
```

HTTP Request Duration (95th percentile):

```promql
histogram_quantile(
  0.95,
  sum by (le) (
    rate(http_request_duration_seconds_bucket[5m])
  )
)
```

Server Errors:

```promql
sum(server_errors_total)
```

Server Errors in the Last 5 Minutes:

```promql
sum(increase(server_errors_total[5m]))
```

These queries require the application to expose the corresponding Prometheus metrics.

## 15. Security Best Practices

- Restrict SSH access to trusted IP addresses.
- Do not expose Jenkins, SonarQube, Prometheus, or Grafana publicly without appropriate access controls.
- Use strong passwords for administrative accounts.
- Store credentials and tokens in Jenkins Credentials.
- Never commit `.env` files, private keys, or passwords to GitHub.
- Scan Docker images with Trivy before deployment.
- Review SonarQube analysis results.
- Apply operating system and software security updates.
- Use HTTPS and a reverse proxy for production-facing services.
- Configure firewall and Azure Network Security Group rules carefully.
- Back up important application data and monitoring configuration.

## 16. Useful Docker Commands

Check running containers:

```bash
sudo docker ps
```

List all containers:

```bash
sudo docker ps -a
```

View container logs:

```bash
sudo docker logs -f CONTAINER_NAME
```

Stop a container:

```bash
sudo docker stop CONTAINER_NAME
```

Remove a container:

```bash
sudo docker rm -f CONTAINER_NAME
```

View Docker images:

```bash
sudo docker images
```

Check Docker Compose services:

```bash
sudo docker compose ps
```

View Compose logs:

```bash
sudo docker compose logs -f
```

Restart Compose services:

```bash
sudo docker compose restart
```

## 17. Troubleshooting

| Issue | Possible Solution |
|---|---|
| Jenkins service not starting | Check Java compatibility and Jenkins logs |
| Docker permission denied | Verify Jenkins Docker group membership |
| Docker container name conflict | Check existing containers and resolve conflicting names |
| SonarQube not starting | Verify kernel parameters and container logs |
| Trivy scan failure | Check network access and vulnerability database updates |
| Grafana not accessible | Check container status and Azure NSG rules |
| Prometheus targets down | Verify scrape configuration and exporter availability |
| Application deployment failure | Inspect Docker Compose and Jenkins stage logs |

Useful diagnostic commands:

```bash
sudo systemctl status jenkins
sudo journalctl -u jenkins -f
sudo docker ps -a
sudo docker logs sonarqube
sudo docker compose logs -f
df -h
free -h
```

## 🎯 Project Outcomes

This setup provides a cloud-based DevOps environment with:

- Jenkins-based CI/CD automation.
- Docker-based application containerization.
- SonarQube code quality analysis.
- Trivy vulnerability scanning.
- Automated application deployment.
- Prometheus metrics collection.
- Grafana monitoring dashboards.
- Infrastructure and application observability.

## 🔮 Future Enhancements

- Kubernetes-based application deployment.
- OWASP ZAP dynamic application security testing.
- Terraform-based infrastructure provisioning.
- Centralized logging with an ELK or Loki stack.
- Advanced security alerts and notifications.
- HTTPS configuration with Nginx and Certbot.
- Automated backup and recovery.

## 👨‍💻 Author

**Shakthi Vel K**

Computer Science and Engineering

GitHub: [Your GitHub Profile](https://github.com/Shakthivelk24)

---

**Project:** Secure DevOps VM Setup

**Domain:** DevOps | DevSecOps | Cloud Computing | CI/CD | Monitoring

**Platform:** Microsoft Azure

**Operating System:** Ubuntu 24.04 LTS