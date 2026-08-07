# CI/CD Pipeline Automation with Jenkins, Docker & AWS

> A portfolio-ready DevOps project — automated software delivery from code commit to live deployment, no manual steps.

---

## What This Is

This project builds a production-style CI/CD pipeline on AWS that takes a code push on GitHub all the way to a running container on a live server — automatically. No SSHing in to restart things. No manually uploading files. Just push code and the pipeline handles the rest.

Every stage is defined in a `Jenkinsfile`, version-controlled alongside the application code, and observable through Jenkins logs and CloudWatch.

This is exactly how real engineering teams ship software.

---

## Why I Built This

Manual deployments are slow, error-prone, and don't scale. At any serious tech company, code goes through an automated pipeline before it ever touches a server. Jenkins, Docker, and EC2 are the tools that make that happen — and they show up on virtually every DevOps and Cloud Engineer job description.

I built this project to understand the full delivery lifecycle end to end, not just the theory.

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                     DEVELOPER                           │
│              git push → GitHub Repo                     │
└─────────────────────┬───────────────────────────────────┘
                      │ Webhook
                      ▼
┌─────────────────────────────────────────────────────────┐
│              JENKINS SERVER (EC2)                       │
│                                                         │
│  Stage 1: Clone  → pulls code from GitHub               │
│  Stage 2: Build  → docker build -t appname .            │
│  Stage 3: Push   → docker push to DockerHub             │
│  Stage 4: Deploy → SSH into App Server, run container   │
│  Stage 5: Notify → SNS email (success or failure)       │
└──────────────┬──────────────────┬───────────────────────┘
               │                  │
               ▼                  ▼
        ┌─────────────┐    ┌─────────────┐
        │  DockerHub  │    │  App Server │
        │  Registry   │    │   (EC2)     │
        │             │◄───│  pulls image│
        │ image:build │    │  runs app   │
        └─────────────┘    └─────────────┘
                                  │
                                  ▼
                        App live on public IP
```

---

## How the Pipeline Works

1. Developer pushes code to GitHub
2. GitHub fires a Webhook to Jenkins
3. Jenkins triggers the pipeline defined in the `Jenkinsfile`

| Stage | What Happens |
|-------|-------------|
| Clone | Jenkins pulls the latest code from GitHub |
| Build | `docker build` creates a new versioned image |
| Push | Jenkins authenticates to DockerHub and pushes the image |
| Deploy | Jenkins SSHs into the App Server, pulls the new image, stops the old container, starts the new one |
| Notify | Jenkins calls SNS — success or failure email lands in your inbox |

Every run is logged in the Jenkins Console Output for full auditability.

---

## What Gets Built

### Source Control (GitHub)
- Repository holds the app code, `Dockerfile`, and `Jenkinsfile`
- GitHub Webhook triggers Jenkins automatically on every push

### CI/CD Server (Jenkins on EC2)
- EC2 instance running Jenkins (open-source automation server)
- Docker installed on the same instance for building images
- Credentials configured for GitHub, DockerHub, and the App Server
- `Jenkinsfile` defines all pipeline stages as code

### Container Registry (DockerHub)
- Every successful build produces a tagged image: `appname:build-1`, `appname:build-2`, etc.
- DockerHub is the handoff point between build and deploy
- Every image version is stored — rollback is always possible

### Application Server (EC2)
- Second EC2 instance as the live deployment target
- Docker installed to run the application container
- Jenkins SSHs in during the Deploy stage, pulls the latest image, and runs it
- Application accessible on the server's public IP on port 80

### Notification Layer (SNS)
- SNS topic with an email subscription
- Jenkins calls the SNS API at the end of every pipeline run
- Success and failure notifications delivered automatically

### Observability
- Every pipeline stage logged in Jenkins Build Console Output
- EC2 instance metrics visible in CloudWatch
- Full pipeline history tracked in the Jenkins dashboard

---

## Tools & Services Used

| Tool / Service | Purpose |
|----------------|---------|
| GitHub | Source code repository and webhook trigger |
| Jenkins | CI/CD automation server (self-hosted on EC2) |
| Docker | Application containerisation |
| DockerHub | Container image registry |
| Amazon EC2 (×2) | Jenkins server + Application server |
| Amazon SNS | Pipeline success/failure email notifications |
| AWS IAM | EC2 instance roles, least-privilege permissions |
| Security Groups | Network access control for both instances |
| Amazon CloudWatch | EC2 monitoring and logging |

---

## Getting Started

### Prerequisites
- AWS account with EC2 access
- GitHub account
- DockerHub account (free tier is fine)
- Basic familiarity with SSH and Linux

### Setup Overview

**1. GitHub**
```bash
# Create a repo with your app code, Dockerfile, and Jenkinsfile
# Configure a Webhook pointing to: http://<jenkins-ec2-ip>:8080/github-webhook/
```

**2. Jenkins EC2**
```bash
# Launch a t2.micro EC2 instance
# Install Jenkins and Docker
# Access Jenkins at http://<ec2-public-ip>:8080
# Configure credentials: GitHub token, DockerHub login, App Server SSH key
```

**3. App Server EC2**
```bash
# Launch a second t2.micro EC2 instance
# Install Docker
# Add Jenkins public key to authorized_keys for SSH access
```

**4. Jenkinsfile (Pipeline as Code)**

The real `Jenkinsfile` in this repo takes the image name, app server host, and SNS topic ARN as pipeline parameters (`params.IMAGE_NAME`, `params.APP_SERVER_HOST`, `params.SNS_TOPIC_ARN`) rather than hardcoding them, so no real infrastructure details are committed to a public repo. Set your real values when triggering a build in Jenkins, or as defaults in your own fork. Simplified illustration:

```groovy
pipeline {
    agent any
    stages {
        stage('Clone') {
            steps { git 'https://github.com/[your-username]/[repo-name]' }
        }
        stage('Build') {
            steps { sh 'docker build -t appname:${BUILD_NUMBER} .' }
        }
        stage('Push') {
            steps { sh 'docker push [dockerhub-username]/appname:${BUILD_NUMBER}' }
        }
        stage('Deploy') {
            steps { sh 'ssh ec2-user@<app-server-ip> "docker pull ... && docker run ..."' }
        }
    }
    post {
        always { sh 'aws sns publish --topic-arn <arn> --message "Pipeline ${currentBuild.result}"' }
    }
}
```

**5. Trigger It**
```bash
git add .
git commit -m "trigger pipeline"
git push
```

Watch Jenkins pick it up automatically, build the image, push to DockerHub, deploy to the App Server, and send you an email.

---

## Project Structure

```
.
├── app.py                # Flask application source
├── requirements.txt      # Python dependencies
├── Dockerfile            # Container image definition
├── Jenkinsfile           # Pipeline stages as code (parameterized — no hardcoded infra details)
└── README.md
```

---

## Estimated Cost

| Resource | Free Tier | Estimated Cost |
|----------|-----------|----------------|
| EC2 t2.micro (×2) | 750 hrs/month | ~$0.00 |
| DockerHub | Free tier | $0.00 |
| GitHub | Free | $0.00 |
| SNS notifications | Free tier | ~$0.00 |

Total estimated cost: **$0.00 – $0.50** for the duration of this project. Stop or terminate both EC2 instances when done to avoid ongoing charges.

---

## Student Checklist

- [x] GitHub repository created with app code, Dockerfile, and Jenkinsfile
- [x] Jenkins EC2 launched, Jenkins installed and accessible on port 8080
- [x] Docker installed on Jenkins EC2
- [x] DockerHub account created and repository set up
- [x] Jenkins connected to GitHub with webhook configured
- [x] Jenkins credentials configured (GitHub, DockerHub, App Server SSH)
- [x] Jenkinsfile written with all pipeline stages
- [x] App Server EC2 launched with Docker installed
- [x] SSH key pair configured for Jenkins → App Server communication
- [x] Full pipeline triggered by a GitHub push and completed successfully
- [x] Application accessible on App Server public IP
- [x] SNS email notification received on pipeline completion
- [x] Jenkins Console Output reviewed for all stages
- [x] Both EC2 instances stopped/terminated after project
- [x] Project documented on GitHub

---

## The Problem This Solves

Manual deployments create real, costly problems for engineering teams:

- Slow delivery — every deploy requires someone to SSH in, pull code, restart services, and verify
- Human error — one missed command can bring down production
- No audit trail — no record of who deployed what, when, or whether it succeeded
- Team bottlenecks — only the person who knows the steps can ship code

This pipeline eliminates all of that. Push code, everything else is automatic.

---

## What I Learned

- How to design and build a multi-server CI/CD pipeline from scratch
- How Jenkins pipelines work and how to define them as code with a `Jenkinsfile`
- How Docker containerisation fits into the delivery lifecycle
- How to wire together GitHub, Jenkins, DockerHub, EC2, and SNS into a single automated system
- Why pipeline-as-code matters — it's versioned, reviewable, and reproducible

---

*Built as part of a cloud engineering curriculum. Feedback welcome.*
