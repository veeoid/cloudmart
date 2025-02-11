# CloudMart - Multi-Cloud E-Commerce Application

## Overview

CloudMart is a **multi-cloud** e-commerce platform that integrates **AWS**, **Google Cloud**, **Microsoft Azure**, and **OpenAI** technologies. This platform provides a scalable, AI-powered solution for product recommendations, customer support, and sentiment analysis. The project leverages **Terraform** for infrastructure management, **Docker** for containerization, **Kubernetes** for deployment, **Lambda** functions for serverless processing, **BigQuery** for analytics, and **CI/CD pipelines** using **AWS CodePipeline**.

## Demo Video

[![Watch the video](assets/dmo.png)](https://player.vimeo.com/video/1055755843?title=0&byline=0&portrait=0&badge=0&autopause=0&player_id=0&app_id=58479)

## Prerequisites

Ensure you have the following installed before proceeding:

- **Terraform**: For managing infrastructure as code.
- **Docker**: For containerizing both frontend and backend.
- **kubectl**: For interacting with Kubernetes clusters.
- **AWS CLI**: For managing AWS resources.
- **Google Cloud SDK**: For managing Google Cloud services.
- **Azure CLI**: For managing Azure resources.
- **Node.js**: For backend development.

## Project Structure

```plaintext
/CloudMart
├── /challenge-day2
│   ├── /backend
│   ├── /frontend
│   ├── /src
│   ├── Dockerfile
│   └── cloudmart-backend.yaml
├── /terraform-project
│   └── main.tf
└── /lambda
    ├── list_products.zip
    └── addToBigQuery
```

## Step-by-Step Guide

### **1. Setup and Configuration**

#### **1.1 Backup Existing Code**

Before making any changes, create backups of your existing project.

```bash
cp -R challenge-day2/ challenge-day2_bkp
cp -R terraform-project/ terraform-project_bkp
```

#### **1.2 Clean Existing Application Files**

Clean the existing application files, retaining only the `Dockerfile` and `.yaml` files.

##### Backend:

```bash
cd challenge-day2/backend
rm -rf $(find . -mindepth 1 -maxdepth 1 -not \( -name ".*" -o -name Dockerfile -o -name "*.yaml" \))
```

##### Frontend:

```bash
cd challenge-day2/frontend
rm -rf $(find . -mindepth 1 -maxdepth 1 -not \( -name ".*" -o -name Dockerfile -o -name "*.yaml" \))
```

#### **1.3 Download Updated Source Code**

##### Backend:

```bash
cd challenge-day2/backend
wget https://tcb-public-events.s3.amazonaws.com/mdac/resources/final/cloudmart-backend-final.zip
unzip cloudmart-backend-final.zip
```

##### Frontend:

```bash
cd challenge-day2/frontend
wget https://tcb-public-events.s3.amazonaws.com/mdac/resources/final/cloudmart-frontend-final.zip
unzip cloudmart-frontend-final.zip
git add -A
git commit -m "final code"
git push
```

### **2. Google Cloud BigQuery Setup**

CloudMart integrates **Google Cloud BigQuery** to manage product order data. Follow these steps:

#### **2.1 Create Google Cloud Project**

1. Go to [Google Cloud Console](https://console.cloud.google.com/), create a new project named **CloudMart**.

#### **2.2 Enable BigQuery API**

1. Enable the **BigQuery API** in the **Google Cloud Console**.

#### **2.3 Create a BigQuery Dataset**

1. Create a dataset in **BigQuery** named `cloudmart` and create a table named `cloudmart-orders` with the following schema:
   - `id` (STRING)
   - `items` (JSON)
   - `userEmail` (STRING)
   - `total` (FLOAT)
   - `status` (STRING)
   - `createdAt` (TIMESTAMP)

#### **2.4 Create Service Account and Key**

1. Create a service account `cloudmart-bigquery-sa` with **BigQuery Data Editor** role.
2. Download the **JSON key** and save it as `google_credentials.json`.

#### **2.5 Lambda Function Setup for BigQuery**

1. Navigate to `challenge-day2/backend/src/lambda/addToBigQuery`.
2. Install dependencies and create a zip file:
   ```bash
   sudo yum install npm
   npm install
   zip -r dynamodb_to_bigquery.zip .
   ```

#### **2.6 Update Lambda Function Environment Variables**

- Ensure the **`google_credentials.json`** file is included, but **do not commit it to version control**, and add it to your `.gitignore` file.

### **3. AWS Setup**

CloudMart leverages **AWS Lambda**, **DynamoDB**, and **S3** services, with **Terraform** used to set up the infrastructure.

#### **3.1 Lambda Setup for BigQuery**

1. In **AWS Lambda**, create a function using the **dynamodb_to_bigquery.zip** file created earlier.
2. Update the **Lambda function** environment to point to the Google Cloud credentials and BigQuery dataset.

#### **3.2 Terraform Setup for AWS Resources**

1. Remove the existing `main.tf` and create a new one:

   ```bash
   rm main.tf
   nano main.tf
   ```

2. Add Terraform configuration for **DynamoDB**, **IAM Role**, **Lambda function**:

```hcl
provider "aws" {
  region = "us-east-1"
}

resource "aws_dynamodb_table" "cloudmart_orders" {
  name           = "cloudmart-orders"
  billing_mode   = "PAY_PER_REQUEST"
  hash_key       = "id"
  stream_enabled = true
  stream_view_type = "NEW_AND_OLD_IMAGES"
}

resource "aws_lambda_function" "dynamodb_to_bigquery" {
  filename         = "../challenge-day2/backend/src/lambda/addToBigQuery/dynamodb_to_bigquery.zip"
  function_name    = "cloudmart-dynamodb-to-bigquery"
  role             = aws_iam_role.lambda_role.arn
  handler          = "index.handler"
  runtime          = "nodejs20.x"
  source_code_hash = filebase64sha256("../challenge-day2/backend/src/lambda/addToBigQuery/dynamodb_to_bigquery.zip")
}
```

#### **3.3 Apply Terraform**

Run the following command to apply the Terraform configuration:

```bash
terraform apply
```

### **4. Azure Text Analytics Setup**

CloudMart uses **Azure Text Analytics** for sentiment analysis.

#### **4.1 Create an Azure Account**

1. Go to the [Azure portal](https://portal.azure.com/) and sign up for an account.

#### **4.2 Create Text Analytics Resource**

1. In the **Azure portal**, search for and create a **Text Analytics** resource named `cloudmart-text-analytics`.

#### **4.3 Get Endpoint and Key**

1. After the resource is created, go to the **Keys and Endpoint** section and copy the endpoint and one of the keys.

### **5. Backend Configuration and Deployment**

#### **5.1 Update Backend Deployment File**

1. Open the `cloudmart-backend.yaml` file and add the **Azure** and **OpenAI** environment variables:

```yaml
- name: AZURE_ENDPOINT
  value: "xxxx"
- name: AZURE_API_KEY
  value: "xxxx"
```

#### **5.2 Build and Deploy the Backend**

1. Build the Docker image and deploy the updated backend service using Kubernetes:

```bash
kubectl apply -f cloudmart-backend.yaml
```

### **6. Test the AI Assistant**

1. **Amazon Bedrock**: Test the **CloudMart product recommendation agent**.
2. **OpenAI**: Test the **CloudMart Customer Support Assistant** to handle customer inquiries.

### **7. AWS CodePipeline Integration**

#### **7.1 Create GitHub Repository**

1. Create a GitHub repository called `cloudmart`.
2. Push the updated **frontend** and **backend** code to GitHub using:

```bash
git add -A
git commit -m "app sent to repo"
git push
```

#### **7.2 Configure AWS CodePipeline**

1. **Create Pipeline**:
   - Go to **AWS CodePipeline** and create a new pipeline named `cloudmart-cicd-pipeline`.
   - Set **GitHub repository** as the source.
   - Add **cloudmartBuild** as the build stage using **AWS CodeBuild**.
   - Add **cloudmartDeploy** for the deployment stage.

#### **7.3 Configure AWS CodeBuild for Docker Build**

1. Create a new build project **cloudmartBuild** and link it to the **cloudmart-application** GitHub repository.
2. Use **Amazon Linux 2** as the build image and enable Docker builds.
3. Use the following **`buildspec.yml`** for Docker image build and push:

```yaml
version: 0.2
phases:
  install:
    runtime-versions:
      docker: 20
  pre_build:
    commands:
      - aws ecr-public get-login-password --region us-east-1 | docker login --username AWS --password-stdin public.ecr.aws/l4c0j8h9
  build:
    commands:
      - docker build -t $REPOSITORY_URI:latest .
      - docker tag $REPOSITORY_URI:latest $REPOSITORY_URI:$CODEBUILD_RESOLVED_SOURCE_VERSION
  post_build:
    commands:
      - docker push $REPOSITORY_URI:latest
      - docker push $REPOSITORY_URI:$CODEBUILD_RESOLVED_SOURCE_VERSION
      - printf '[{"name":"cloudmart-app","imageUri":"%s"}]' $REPOSITORY_URI:$imageTag > imagedefinitions.json
```

#### **7.4 AWS CodeBuild for Deployment**

1. Create another **cloudmartDeploy** project for deployment using the `cloudmart-frontend.yaml` file.
2. Use the following **`buildspec.yml`** for the deployment phase:

```yaml
version: 0.2
phases:
  install:
    runtime-versions:
      docker: 20
  post_build:
    commands:
      - aws eks update-kubeconfig --region us-east-1 --name cloudmart
      - kubectl apply -f cloudmart-frontend.yaml
```

#### **7.5 Test CodePipeline**

1. **Make Changes**: Modify application code, push to GitHub.
2. **Monitor CodePipeline**: Observe the pipeline execution through build and deployment.
3. **Verify**: Ensure the deployment is successful by checking the Kubernetes pods and services.
