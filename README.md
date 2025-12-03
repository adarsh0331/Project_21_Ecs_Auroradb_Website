# 🚀 Deploying a Website on AWS ECS with Aurora using Terraform

## 🧭 Overview

This project demonstrates a **complete DevOps setup** where a **containerized website** is deployed on **Amazon ECS (Fargate)** backed by an **Amazon Aurora Database Cluster**, with all infrastructure provisioned using **Terraform (Infrastructure as Code - IaC)**.

It follows **best practices** for:
- Environment isolation (Dev & Staging)
- High availability and scalability
- Secure networking and IAM policies
- Centralized logging and monitoring using CloudWatch

---
## Project Architecture Diagram

![Architecture](images/architecture-diagram.png)

---

## 🎯 Project Goals / Requirements

1. Use **Terraform** for Infrastructure as Code (IaC) to provision AWS resources.  
2. Deploy the **website on Amazon ECS** using the **Fargate launch type**.  
3. Set up an **Aurora Database Cluster** with **multi-AZ availability** for fault tolerance.  
4. Implement **separate environments** for **Development** and **Staging**.  
5. Configure **Route53** for domain routing for both environments.  
6. Integrate **CloudWatch** for logging and monitoring (ECS & Aurora).  
7. Ensure **secure configurations** — proper **VPC**, **subnet isolation**, **security groups**, and **restricted database access**.

---

## 🏗️ Architecture Overview

```plaintext
                    ┌────────────────────────┐
                    │        Route53         │
                    │ dev.myapp.com          │
                    │ staging.myapp.com      │
                    └──────────┬─────────────┘
                               │
                          ┌────▼────┐
                          │  ALB    │
                          └────┬────┘
                               │
                      ┌────────▼────────┐
                      │ ECS Fargate     │
                      │ (Web Containers)│
                      └────────┬────────┘
                               │
                      ┌────────▼────────┐
                      │  Aurora Cluster │
                      │ (Multi-AZ RDS)  │
                      └─────────────────┘
                               │
                      ┌────────▼────────┐
                      │ CloudWatch + SNS│
                      │  Alerts & Logs  │
                      └─────────────────┘
```
---

## 🧩 **Project Structure**

```plaintext
app
├── Dockerfile
├── index.html
terraform
├── envs
│ ├── dev
│ │ ├── backend.tf
│ │ └── main.tf
│ ├── staging
│ │ ├── backend.tf
│ │ └── main.tf
│ └── global
│ └── backend
│ └── main.tf
└── modules
├── aurora
│ ├── main.tf
│ ├── outputs.tf
│ └── variables.tf
├── cloudwatch
│ ├── main.tf
│ ├── outputs.tf
│ └── variables.tf
├── ecs
│ ├── main.tf
│ ├── outputs.tf
│ └── variables.tf
├── route53
│ ├── main.tf
│ ├── outputs.tf
│ └── variables.tf
├── sns
│ ├── main.tf
│ ├── outputs.tf
│ └── variables.tf
└── vpc
├── main.tf
├── outputs.tf
└── variables.tf
├── Jenkinsfile

```

---

## ⚙️ **Prerequisites**

* **AWS Account** with IAM permissions
* **Terraform ≥ v1.5**
* **AWS CLI** configured (`aws configure`)
* **Docker** installed
* **Domain registered in Route53 (optional)** for DNS setup
* **S3 Setup** create your own S3 bucket in console
---

## 🪜 **Setup Steps to deploy with jenkins CI/CD flow**

## 🚀 Project Overview

## 🎯 Goal

Deploy a sample HTML website using:

- **Terraform** → to provision AWS infrastructure  
- **Jenkins** → to automate CI/CD pipeline  
- **AWS ECS (Fargate)** → to host the containerized web app  
- **AWS ECR** → to store Docker images  
- **CloudWatch** → for monitoring and logs
---

# 🧩 Project Components

| Component     | Purpose                                      |
|----------------|----------------------------------------------|
| **index.html** | Sample web page                              |
| **Dockerfile** | Builds the website image                     |
| **Terraform**  | Creates ECS, VPC, ECR, ALB, etc.             |
| **Jenkinsfile**| Defines the CI/CD pipeline                   |
| **AWS**        | Target cloud platform (ECS Fargate)          |

---

# 🌐 1. Sample Webpage (`app/index.html`)

```html
<!DOCTYPE html>
<html>
<head>
  <title>Welcome to My Website</title>
  <style>
    body {
      font-family: Arial;
      text-align: center;
      margin-top: 10%;
      background-color: #f4f4f4;
    }
    h1 {
      color: #0078d7;
    }
  </style>
</head>
<body>
  <h1>🚀 Deployed via Jenkins on AWS ECS Fargate</h1>
  <p>This is a sample web page deployed automatically using CI/CD.</p>
</body>
</html>
```
---
# 🐳 2. Dockerfile (`app/Dockerfile`)

```FROM nginx:latest

# Remove default nginx files
RUN rm -rf /usr/share/nginx/html/*

# Copy everything from the current directory (app/) into nginx html
COPY . /usr/share/nginx/html/

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```
---
# 🧱 4. Jenkinsfile (CI/CD Pipeline)

Here’s the main automation logic 👇

```groovy
pipeline {
    agent any

    parameters {
        choice(name: 'ENV', choices: ['dev', 'staging'], description: 'Select environment to deploy')
    }

    environment {
        AWS_REGION      = 'us-east-1'
        AWS_CREDENTIALS = credentials('aws-jenkins-creds')
        ECR_REPO        = '141559732042.dkr.ecr.us-east-1.amazonaws.com/mywebsite'
        // IMAGE_TAG will be set after checkout (contains BUILD_NUMBER + git short sha)
    }

    stages {

        stage('Checkout Code') {
            steps {
                git branch: 'main', credentialsId: 'git', url: 'https://github.com/Jithendarramagiri1998/ecs-aurora-website.git'
                script {
                    // capture short git sha and set a unique image tag
                    def gitShort = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
                    env.IMAGE_TAG = "v${env.BUILD_NUMBER}-${gitShort}"
                    echo "Using IMAGE_TAG = ${env.IMAGE_TAG}"
                }
            }
        }

        stage('Terraform Init & Validate') {
            steps {
                script {
                    def terraformRoot = "${env.WORKSPACE}/terraform"
                    def backendPath   = "${terraformRoot}/global/backend"
                    def envPath       = "${terraformRoot}/envs/${params.ENV}"

                    dir(envPath) {
                        sh '''
                        set -eux
                        if ! aws s3api head-bucket --bucket my-terraform-states-1234 2>/dev/null; then
                            echo "🚀 Creating backend S3 & DynamoDB..."
                            cd ../../global/backend
                            terraform init -input=false
                            terraform apply -auto-approve
                            cd -
                        else
                            echo "✅ Backend S3 bucket already exists."
                        fi

                        terraform init \
                          -backend-config="bucket=my-terraform-states-1234" \
                          -backend-config="key=${ENV}/terraform.tfstate" \
                          -backend-config="region=us-east-1" \
                          -backend-config="dynamodb_table=terraform-locks" \
                          -input=false

                        terraform validate
                        terraform workspace select ${ENV} || terraform workspace new ${ENV}
                        '''
                    }
                }
            }
        }

        stage('Terraform Plan & Apply Infra') {
            steps {
                dir("terraform/envs/${params.ENV}") {
                    withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-jenkins-creds']]) {
                        sh '''
                        set -eux
                        echo "📦 Running Terraform Plan for ${ENV}..."
                        terraform plan -input=false -out=tfplan -var="env=${ENV}"
                        echo "🚀 Applying Terraform Changes..."
                        terraform apply -input=false -auto-approve tfplan
                        '''
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    echo "🔧 Building Docker image with app code..."
                    sh '''
                    set -eux
                    cd app
                    echo "📁 Checking files inside app/"
                    ls -la
                    echo "🐳 Building Docker image..."
                    docker build --no-cache -t ${ECR_REPO}:${IMAGE_TAG} .
                    echo "✅ Docker image built successfully: ${ECR_REPO}:${IMAGE_TAG}"
                    '''
                }
            }
        }

        stage('Push Docker Image to ECR') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-jenkins-creds']]) {
                    sh '''
                    set -eux
                    echo "🔐 Logging in to Amazon ECR..."
                    aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REPO}

                    # ensure repo exists (idempotent)
                    REPO_NAME=$(basename "${ECR_REPO}")
                    if ! aws ecr describe-repositories --repository-names "${REPO_NAME}" --region ${AWS_REGION} >/dev/null 2>&1; then
                        aws ecr create-repository --repository-name "${REPO_NAME}" --region ${AWS_REGION} || true
                    fi

                    echo "🚀 Pushing Docker image to ECR..."
                    docker push ${ECR_REPO}:${IMAGE_TAG}
                    '''
                }
            }
        }

        stage('Deploy to ECS (safe rolling with immutable digest)') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-jenkins-creds']]) {
                    sh '''
                    set -eux
                    echo "🚀 Deploy: building immutable image reference and updating ECS..."

                    REPO_URI="${ECR_REPO}"                        # e.g. 141559.../mywebsite
                    REPO_NAME=$(basename "${REPO_URI}")          # mywebsite
                    IMAGE_TAG="${IMAGE_TAG}"                     # v123-abcd

                    # Wait for ECR image to be available
                    echo "🔎 Waiting for image ${REPO_NAME}:${IMAGE_TAG} in ECR..."
                    for i in 1 2 3 4 5 6; do
                      aws ecr describe-images --repository-name "${REPO_NAME}" --image-ids imageTag="${IMAGE_TAG}" --region ${AWS_REGION} && break || sleep 2
                    done

                    # Get the image digest (sha256:...)
                    IMAGE_DIGEST=$(aws ecr describe-images \
                      --repository-name "${REPO_NAME}" \
                      --image-ids imageTag="${IMAGE_TAG}" \
                      --query 'imageDetails[0].imageDigest' --output text --region ${AWS_REGION})

                    if [ -z "${IMAGE_DIGEST}" ] || [ "${IMAGE_DIGEST}" = "None" ]; then
                      echo "ERROR: Could not find image digest in ECR for ${REPO_NAME}:${IMAGE_TAG}"
                      exit 1
                    fi

                    IMMUTABLE_IMAGE="${REPO_URI}@${IMAGE_DIGEST}"
                    echo "✅ Using immutable image: ${IMMUTABLE_IMAGE}"

                    # Task family name (should match your existing family)
                    TASK_FAMILY="${ENV}-app-task"

                    # Fetch current task definition JSON for the family (taskDefinition object)
                    echo "📦 Fetching current task definition for family ${TASK_FAMILY}..."
                    CURRENT_TASK_JSON=$(aws ecs describe-task-definition --task-definition "${TASK_FAMILY}" --region ${AWS_REGION} --query 'taskDefinition' --output json)

                    if [ -z "${CURRENT_TASK_JSON}" ] || [ "${CURRENT_TASK_JSON}" = "null" ]; then
                        echo "ERROR: Could not fetch current task definition for ${TASK_FAMILY}"
                        exit 1
                    fi

                    # Build new task definition JSON - remove fields not allowed during register, replace image with immutable image
                    echo "${CURRENT_TASK_JSON}" \
                      | jq --arg img "${IMMUTABLE_IMAGE}" 'del(.status, .revision, .taskDefinitionArn, .requiresAttributes, .compatibilities, .registeredAt, .registeredBy) |
                        .containerDefinitions = (.containerDefinitions | map(.image = $img))' \
                      > new-task-def.json

                    echo "📝 Registering new task definition..."
                    REGISTER_OUT=$(aws ecs register-task-definition --cli-input-json file://new-task-def.json --region ${AWS_REGION})
                    NEW_TASK_DEF_ARN=$(echo "${REGISTER_OUT}" | jq -r '.taskDefinition.taskDefinitionArn')
                    echo "✅ Registered: ${NEW_TASK_DEF_ARN}"

                    echo "🚀 Updating ECS service ${ENV}-ecs-service in cluster ${ENV}-ecs-cluster..."
                    aws ecs update-service --cluster ${ENV}-ecs-cluster --service ${ENV}-ecs-service --task-definition "${NEW_TASK_DEF_ARN}" --region ${AWS_REGION}

                    echo "⏳ Waiting for service to stabilize..."
                    aws ecs wait services-stable --cluster ${ENV}-ecs-cluster --services ${ENV}-ecs-service --region ${AWS_REGION}
                    echo "🎉 Deployment finished and stable."
                    '''
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                script {
                    echo "✅ Deployment completed for ${params.ENV} environment!"
                    echo "🌐 Check website URL after Route53 setup: https://${params.ENV}.yourdomain.com"
                }
            }
        }
    }

    post {
        success {
            echo "🎉 ${params.ENV} deployment successful!"
            mail to: 'ramagirijithendar1998@gmail.com',
                 subject: "✅ Jenkins Build Success: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                 body: "The build succeeded!\nCheck details: ${env.BUILD_URL}"
        }
        failure {
            echo "❌ Deployment failed. Check Jenkins logs and CloudWatch for details."
            mail to: 'ramagirijithendar1998@gmail.com',
                 subject: "❌ Jenkins Build Failed: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                 body: "The build failed.\nPlease check console output: ${env.BUILD_URL}"
        }
    }
}
```
---
# 🧰 5. Jenkins Setup Steps

On your Jenkins server (EC2 or local):
```
---
## 🔌 Install Plugins
- Amazon ECR  
- AWS CLI  
- Docker Pipeline  
- Git  
- Terraform Plugin
- Confiure your email to get email notifaction about pipeline sucess or failed.
---
## 🔐 Configure AWS Credentials
1. Go to **Jenkins → Manage Jenkins → Credentials**  
2. Add credentials of type **AWS Credentials**  
3. Name it: `aws-jenkins-creds`
```
---
## 🧑‍💻 Agent Requirements
Jenkins agent/server must have:
- Docker  
- AWS CLI  
- Terraform installed
   
---
## 🏗️ Create a Pipeline Job
1. Name: `ecs-website-deploy`  
2. Select: **“Pipeline script from SCM”**  
3. SCM: **Git** → paste your GitHub repository URL

---
## ▶️ Run the Pipeline
Jenkins will automatically:
- Build and push Docker image  
- Apply Terraform infrastructure  
- Deploy the application on ECS
---
# 🧩 1. Jenkins Server Setup (if not done)

On your Jenkins EC2 instance (**Ubuntu preferred**):

```bash
sudo apt update -y
sudo apt install -y docker.io unzip awscli
🏗️ Install Terraform
bash
Copy code
wget https://releases.hashicorp.com/terraform/1.9.5/terraform_1.9.5_linux_amd64.zip
unzip terraform_1.9.5_linux_amd64.zip
sudo mv terraform /usr/local/bin/
terraform -version
🐳 Add Jenkins to Docker Group
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
✅ Also ensure Docker and AWS CLI are installed and configured on your Jenkins server.
```
---
# ✅ Verify the Deployment

Once the pipeline finishes:

1. Go to **AWS ECS Console → Clusters → jenkins-ecs-cluster**  
2. Check the service → ensure the task is **running**  
3. Open the **Public IP** or **ALB DNS name** in your browser  

You’ll see:

> 🚀 Deployed via Jenkins on AWS ECS Fargate

# ⚠️ Notes on Deployment Issues & Fixes

During manual execution of deployment scripts using `terraform apply`, the following issues may occur:

## 🧩 Common Issues & Fixes

1. **Route53 – Custom Domain Required**  
   - The Route53 configuration requires a **custom domain name**.  
   - Update the Terraform variable or configuration file with your registered domain before running `terraform apply`.

2. **KMS Key Generation**  
   - A **KMS key** is needed for encrypting sensitive data or logs.  
   - Ensure you create and reference a KMS key in your Terraform setup.

# **ECS Configuration for Jenkins (ECR Section)**  
   - In the Jenkinsfile, make sure ECS details (cluster and service) are defined.  
   - Add the ECS configuration properly to allow Jenkins to deploy the image pushed to ECR.

## ✅ Recommendation

If you encounter any additional issues during execution or deployment, note them down and address them in your Terraform modules or Jenkins pipeline configuration.  
Always re-run the pipeline after fixing errors to verify successful deployment.

---

## 📸 12. Output Screenshots (Add Here)

# Dev Output Page ![Image](images/Dev_Output_Page.png) 

# Stagging Output Page ![Image](images/Stagging_Output_Page.png)

# Jenkins Pipelines Success Dev ![Image](images/dev_Pipelines_Success.png)

# Jenkins Pipelines Success Stagging ![Image](images/stagging_Pipelines_Success.png)

# Loadbalancers to access Dev and Stagging ![Image](images/Loadbalcers.png) 

# ECS Cluster ![Image](images/ESC_Cluster.png) 

# VPC Dev and Stagging ![Image](images/VPC_Dashboard.png)

# Aurora RDS ![Image](images/Aurora_Dashboard.png)

---
