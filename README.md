# ☕ Starbucks Clone — CI/CD Pipeline with Jenkins & Docker

A hands-on DevOps project demonstrating a complete, automated CI/CD pipeline for a front-end web application clone — from a GitHub push all the way to a live, running container — using **Jenkins** for orchestration, **Docker** for packaging, and **AWS EC2** as the build and runtime server.

This isn't just a "steps that worked" writeup — it documents every real error encountered along the way (missing shared libraries, Docker Hub auth failures, firewall misconfigurations) along with the root cause and fix for each, because that's where most real DevOps learning happens.

---

## 🧰 Tech Stack

| Category | Tool |
|---|---|
| Cloud | AWS EC2 (Ubuntu 24.04 LTS) |
| CI/CD | Jenkins (Pipeline / Jenkinsfile) |
| Containerization | Docker & Docker Hub |
| Source Control | GitHub |
| Runtime | Node.js / npm |
| SSH Client | MobaXterm |

---

## 🏗️ High-Level Pipeline Flow
GitHub Repo
│
▼
Jenkins CI Job → checkout → npm install → docker build → push to Docker Hub
│
▼ (auto-triggered on success)
Jenkins CD Job → pull latest image → stop/remove old container → run new container
│
▼
Live App served from EC2 (port 3000)



A single `git push` to `main` results in a freshly deployed container with **zero manual steps** in between.

---

## 📋 Table of Contents

1. [Provisioning the AWS EC2 Server](#1-provisioning-the-aws-ec2-server)
2. [Configuring the Security Group](#2-configuring-the-security-group)
3. [Connecting via MobaXterm](#3-connecting-via-mobaxterm)
4. [Installing Jenkins](#4-installing-jenkins)
5. [Unlocking Jenkins & Creating Admin User](#5-unlocking-jenkins--creating-admin-user)
6. [Installing Docker](#6-installing-docker)
7. [Jenkins Plugins & Tool Configuration](#7-jenkins-plugins--tool-configuration)
8. [Creating the CI Pipeline Job](#8-creating-the-ci-pipeline-job)
9. [Troubleshooting: Real Pipeline Failures](#9-troubleshooting-real-pipeline-failures)
10. [Building the CD Pipeline](#10-building-the-cd-pipeline)
11. [Troubleshooting: App Not Reachable](#11-troubleshooting-app-not-reachable)
12. [Chaining CI → CD Automatically](#12-chaining-ci--cd-automatically)
13. [Key Takeaways](#13-key-takeaways)

---

## 1. Provisioning the AWS EC2 Server

- **AMI:** Ubuntu Server 24.04 LTS (stable, long-term support, strong `apt` ecosystem)
- **Instance type:** `m7i-flex.large` (2 vCPU / 8 GiB RAM) — chosen over free-tier micro, since Jenkins + Docker + Node.js builds running together are memory-hungry, and an under-powered instance is a common cause of silently killed/hanging pipelines (OOM killer).
- **Key pair:** generated at launch for secure SSH access (no password auth).

---

## 2. Configuring the Security Group

AWS Security Groups block all inbound traffic by default. The following ports were opened:

| Port | Purpose |
|---|---|
| 22 | SSH access (MobaXterm/terminal) |
| 80 / 443 | Standard HTTP/HTTPS web traffic |
| 8080 | Jenkins web dashboard |
| 3000 | Containerized app (added later, see [Section 11](#11-troubleshooting-app-not-reachable)) |

> ⚠️ **Note:** Rules use `0.0.0.0/0` for simplicity in this learning project. For production, restrict source IPs to known/trusted ranges.

---

## 3. Connecting via MobaXterm

SSH session configured with:
- **Host:** EC2 public IP
- **Username:** `ubuntu` (default for Ubuntu AMIs)
- **Auth:** private key (`.pem`) downloaded at instance creation

---

## 4. Installing Jenkins

Installed via shell script over SSH:

```bash
#!/bin/bash
sudo apt update -y
sudo apt install -y fontconfig openjdk-21-jre
sudo apt install -y curl wget gnupg

sudo mkdir -p /etc/apt/keyrings
sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key

echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/" | \
  sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null

sudo apt update -y
sudo apt install -y jenkins

sudo systemctl enable jenkins
sudo systemctl start jenkins
sudo systemctl status jenkins --no-pager

sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

- **Java 21** is required as the current Jenkins JVM prerequisite.
- The initial admin password is retrieved from `/var/lib/jenkins/secrets/initialAdminPassword`.

---

## 5. Unlocking Jenkins & Creating Admin User

1. Navigate to `http://<public-ip>:8080`
2. Paste the initial admin password
3. Create the first admin account (permanent login for all future config)

---

## 6. Installing Docker

```bash
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y

sudo usermod -aG docker ubuntu
sudo chmod 777 /var/run/docker.sock
newgrp docker
sudo systemctl status docker
```

The `ubuntu` user is added to the `docker` group, and socket permissions are relaxed so Jenkins can run Docker commands without repeated `sudo`.

---

## 7. Jenkins Plugins & Tool Configuration

**Plugins installed:**
- Eclipse Temurin Installer
- Docker, Docker Commons, Docker Pipeline, Docker API
- NodeJS

**Global Tools configured** (auto-install on first pipeline run):
| Tool | Version |
|---|---|
| JDK | `jdk-17.0.8.1+1` (Adoptium) |
| NodeJS | Node 25.2.1 (labeled `node16` in pipeline) |
| Docker | latest (docker.com) |

---

## 8. Creating the CI Pipeline Job

A Jenkins **Pipeline** job (`CI`) was configured with **"Pipeline script from SCM"**, pointing to the GitHub repo (`main` branch) — keeping the pipeline definition version-controlled rather than pasted into Jenkins' UI.

### CI Jenkinsfile

```groovy
pipeline {
    agent any
    tools {
        jdk 'jdk17'
        nodejs 'node16'
    }
    stages {
        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }
        stage('Git Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/<your-username>/Starbucks-Application.git'
            }
        }
        stage('Install NPM Dependencies') {
            steps {
                sh "npm install"
            }
        }
        stage('Build Docker Image') {
            steps {
                sh "docker build -t starbucks ."
            }
        }
        stage('Tag & Push to DockerHub') {
            steps {
                script {
                    withDockerRegistry(credentialsId: '<your-dockerhub-credential-id>') {
                        sh "docker tag starbucks <dockerhub-username>/starbucks:latest"
                        sh "docker push <dockerhub-username>/starbucks:latest"
                    }
                }
            }
        }
    }
}
```

| Stage | Purpose |
|---|---|
| `Clean Workspace` | Wipes leftover files from previous builds |
| `Git Checkout` | Clones latest source from GitHub |
| `Install NPM Dependencies` | Runs `npm install` |
| `Build Docker Image` | Builds image from the Dockerfile |
| `Tag & Push to DockerHub` | Authenticates and pushes the image |

---

## 9. Troubleshooting: Real Pipeline Failures

### ❌ Error 1 — `npm install` fails with a missing shared library
npm install
node: error while loading shared libraries: libatomic.so.1:
cannot open shared object file: No such file or directory


**Root cause:** The auto-installed Node.js binary depends on `libatomic`, missing on the minimal Ubuntu image. Downstream stages were auto-skipped as a result.

**Fix:**
```bash
sudo apt-get update
sudo apt-get install -y libatomic1
```

---

### ❌ Error 2 — Docker Hub login fails with `unauthorized`
$ docker login -u <username> -p ********
Error response from daemon: Get "https://registry-1.docker.io/v2/":
unauthorized: incorrect username or password
ERROR: docker login failed


**Root cause:** Two issues —
1. The Jenkins credential ID referenced in the Jenkinsfile hadn't been created yet.
2. Docker Hub rejects account passwords for CLI/API logins — a **Personal Access Token** is required.

**Fix:**
1. In Jenkins: **Manage Jenkins → Credentials → Global → Add Credentials** → type "Username with password", using your Docker Hub username as both the username and credential ID.
2. Generate a Docker Hub Access Token:
   - Go to `hub.docker.com` → Account Settings → **Security → Personal Access Tokens**
   - Click **Generate New Token**, name it (e.g. `jenkins-ci`), set permissions to **Read & Write**
   - Copy the token immediately (shown only once)
3. Paste the token into the **Password** field of the Jenkins credential (replacing the account password).
4. Save and re-run the pipeline.

> 🔒 Access tokens are secrets — never commit or share them.

---

## 10. Building the CD Pipeline

A second Jenkins **Pipeline** job (`CD`) handles deployment: pulling the freshly built image and (re)starting the container.

### CD Jenkinsfile

```groovy
pipeline {
    agent any
    tools {
        jdk 'jdk17'
        nodejs 'node16'
    }
    stages {
        stage('Deploy to Docker Container') {
            steps {
                sh '''
                    # Stop existing container if it exists
                    docker stop starbucks || true

                    # Remove existing container
                    docker rm starbucks || true

                    # Pull latest image
                    docker pull <dockerhub-username>/starbucks:latest

                    # Run new container
                    docker run -d \
                        --name starbucks \
                        -p 3000:3000 \
                        <dockerhub-username>/starbucks:latest
                '''
            }
        }
    }
    post {
        success {
            echo 'Application deployed successfully!'
        }
        failure {
            echo 'Deployment failed!'
        }
    }
}
```

| Command | Purpose |
|---|---|
| `docker stop starbucks \|\| true` | Stops existing container (`\|\| true` avoids failure on first-ever deploy) |
| `docker rm starbucks \|\| true` | Removes stopped container to avoid naming conflicts |
| `docker pull ...:latest` | Pulls the newest image pushed by CI |
| `docker run -d --name starbucks -p 3000:3000 ...` | Starts the new container, mapping host port 3000 |

---

## 11. Troubleshooting: Application Not Reachable

Even with the container confirmed running, `http://<public-ip>:3000` timed out (`ERR_CONNECTION_TIMED_OUT`).

**Root cause:** The container was healthy — the issue was at the network boundary. AWS Security Groups block all inbound traffic by default, and port 3000 had never been opened on the firewall (unlike ports 22/80/443/8080).

**Fix:** Add an inbound rule on the EC2 Security Group:
- **Type:** Custom TCP
- **Port range:** 3000
- **Source:** `0.0.0.0/0` (or restrict to a known IP for better security)

✅ After this fix, the app loaded successfully in the browser.

---

## 12. Chaining CI → CD Automatically

To remove the manual hand-off between jobs, the `CD` job was configured with a build trigger:

- **Build after other projects are built** → watching `CI`
- **Trigger only if build is stable**

Now a single push to `main` runs CI, and on success, automatically triggers CD — verified via the CD console log:

Started by upstream project "CI" build number 4

This completes a fully automated, hands-off CI/CD pipeline.

---

## ✅ Key Takeaways

- A CI/CD pipeline is only as reliable as its infrastructure — right-sizing the EC2 instance and opening the correct Security Group ports upfront avoids a lot of downstream debugging.
- Keeping the pipeline definition as a **Jenkinsfile in source control** (rather than pasted into Jenkins' UI) makes every change to the build/deploy process auditable through Git.
- Almost every real pipeline fails at least once before it fully works — missing OS libraries, wrong credential IDs, and firewall gaps are among the most common first-run issues. The Jenkins console log is the fastest way to diagnose them.
- Docker Hub (and most registries) require a **Personal Access Token** for automated logins, not an account password — and tokens should always be treated as secrets.
- Automating the hand-off between CI and CD (trigger on "stable" status) is what turns two separate manual jobs into one true continuous delivery pipeline.

---

## 📌 Project Info

**Author:** Akanksha Singh
**Duration:** 29 June 2026 – 1 July 2026
**LinkedIn:** [linkedin.com/in/akanksha-singh-2780a6253](https://linkedin.com/in/akanksha-singh-2780a6253)

---

⭐ If you found this project helpful, consider giving it a star!



