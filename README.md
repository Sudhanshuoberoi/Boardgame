# BoardgameListing WebApp

### Description

**Board Game Database Full-Stack Web Application.** This web application displays lists of board games and their reviews. While anyone can view the board game lists and reviews, they are required to log in to add/ edit the board games and their reviews. The 'users' have the authority to add board games to the list and add reviews, and the 'managers' have the authority to edit/ delete the reviews on top of the authorities of users.

### Technologies Used

- CI/CD: Jenkins
- Code Quality & Security: SonarQube
- Artifact Repository: Nexus
- Container Security: Trivy
- Container Registry: AWS ECR
- Orchestration: Kubernetes on AWS EKS
- Monitoring : Prometheus and Grafana
- Cloud Infrastructure: AWS EC2, IAM, NLB

### Architecture Overview
1. EC2 Instances<br>
- Main Web Server: To SSH into other server(such jenkins, nexus, etc.)
- Jenkins Server (t3.medium): CI/CD Orchestration
- SonarQube Server (t3.medium): Code Quality & Security Scanning
- Nexus Server (t3.medium): Artifact Repository (JAR/TAR storage)
- Monitoring Server (t3.medium): Prometheus & Grafana

2.  AWS ECR<br>
- Used to store docker images after build and are pulled for deployment.

3. AWS EKS Cluster<br>
- Managed Kubernetes control plane
- Worker Node Group (2 EC2 nodes)
- Integrated with AWS ECR for storing container images

4. Monitoring Layer<br>
- Prometheus + Grafana dashboards
- Metrics collected via node-exporter and kube-state-metrics

### End-to-End Setup:<br>
1. Jenkins Setup <br>
- Opened port 8080 in the security group to allow Jenkins web UI access.
- Allowed SSH from single EC2(Main web server)
- Installed essential Jenkins plugins.
- Installed trivy on this EC2 server itself.

2. SonarQube Setup (Code Quality)<br>
- Deployed SonarQube on a separate EC2 instance (port 9000).
- Allowed SSH from single EC2(Main web server)
- Configured webhook → sends Quality Gate results back to Jenkins.

3. Nexus Repository Setup (Artifact Management)<br>
- Installed Nexus Repository Manager on an EC2 server (port 8081).
- Allowed SSH from single EC2(Main web server)
- Created dedicated Maven hosted repositories for JARs (application builds)

4. AWS EKS Cluster (Kubernetes Deployment)<br>
- Provisioned an EKS cluster using AWS console/CLI.
- Configured IAM roles & RBAC bindings for Jenkins to deploy workloads.
- Added a managed node group with two EC2 worker nodes.
On Jenkins EC2:<br>
- Installed kubectl and AWS CLI.
- Updated aws-auth ConfigMap to allow node → cluster communication.

5. CI/CD Pipeline (Jenkinsfile Workflow)<br>
- explained separately each step in the ending.

6. Monitoring & Observability<br>
- Provisioned Prometheus & Grafana on a monitoring EC2 instance.
- Open 9090 (Prometheus) and 3000 (Grafana) in the EC2 security group.
- Allowed SSH from single EC2(Main web server)

### CI/CD Pipeline Overview (Jenkinsfile)

##### Pipeline Tools & Environment Setup
1. Maven (M3) → For building Java project

2. JDK11 → Java runtime for Maven & SonarQube

3. Environment Variables → Preconfigured for Nexus, ECR, EKS cluster, etc.

#### Pipeline Stages

**1. Git Checkout**<br>
Clones the code from GitHub (main branch) using a PAT (Personal Access Token).

**2. Code Analysis (SonarQube)**<br>
Runs SonarQube analysis with a PAT token. Checks code quality, security vulnerabilities, bugs, and code smells.

**3. Quality Gate**<br>
Waits for SonarQube’s Quality Gate result.<br>
- *If it fails* -> Pipeline stops immediately.<br>
- *If it passes* -> Pipeline continues.<br>

**4. Build**<br>
Executes mvn clean install to package the Java application into a JAR file.

**5. Deploy to Nexus**<br>
Uploads the JAR artifact into Nexus repository. Uses Nexus credentials (nexus-creds).

**6. Docker Build**<br>
- Builds a Docker image from the application source code.<br>
- Tags it with IMAGE_NAME:BUILD_NUMBER.<br>

**7. Trivy Security Scan**<br>
Runs a Trivy security scan on the Docker image to check image vulnerabilities and generates a report (trivy-report.txt) with HIGH/CRITICAL vulnerabilities.

**8. Login to AWS ECR**<br>
Logs in to AWS Elastic Container Registry (ECR) using AWS credentials (aws-creds).

**9. Push Docker Image to ECR**<br>
- This step tags the Docker image with:<br>
${AWS_ACCOUNT_NUMER}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}:${BUILD_NUMBER}<br>
- Pushes the image to AWS ECR.<br>
- Cleans up the local image after successful push of the image to ECR.<br>

**10. Deploy to EKS (Kubernetes)**
- This step first Updates Kubernetes manifest(deployment-service.yaml) with ECR registry, Repository name, Build number and Saves it as deployment.yml<br>
- Applies it to EKS cluster using:<br>
```bash
aws eks update-kubeconfig --region ${AWS_REGION} --name ${EKS_CLUSTER}
kubectl apply -f deployment.yaml
```

**11. Post-Build Actions**
- *On Failure* → Sends an email with build failure details, logs, and links.<br>
- *On Success* → Sends an email with success notification and build details.
