# CloudMart - Multi-Cloud E-Commerce Application

## Overview

CloudMart is a **multi-cloud** e-commerce platform that integrates **AWS**, **Google Cloud**, **Microsoft Azure**, and **OpenAI** technologies. This platform provides a scalable, AI-powered solution for product recommendations, customer support, and sentiment analysis. The project leverages **Terraform** for infrastructure management, **Docker** for containerization, **Kubernetes** for deployment, **Lambda** functions for serverless processing, **BigQuery** for analytics, and **CI/CD pipelines** using **AWS CodePipeline**.

## Demo Video

[![Watch the video](assets/dmo.png)](https://player.vimeo.com/video/1055755843?title=0&byline=0&portrait=0&badge=0&autopause=0&player_id=0&app_id=58479)

## Technology Stack

- **Cloud Infrastructure**:
  - AWS EKS (Kubernetes)
  - AWS DynamoDB
  - AWS Lambda
  - Google Cloud BigQuery
- **AI Services**:
  - Amazon Bedrock with Claude 3 Sonnet
  - OpenAI GPT-4
  - Azure Text Analytics
- **Application Stack**:
  - Frontend: Node.js
  - Backend: Node.js
  - Containerization: Docker
  - Infrastructure as Code: Terraform

---

## Configuration

**1. AWS Configuration:**

- Create IAM Role for EC2
  1. Log in to the AWS Management Console.
  2. Navigate to the IAM dashboard.
  3. Click on "Roles" in the left sidebar, then click "Create role".
  4. Choose "AWS service" as the trusted entity type and "EC2" as the use case.
  5. Search for and attach the "AdministratorAccess" policy.
  6. Note: In a production environment, use a more restricted policy.
  7. Name the role "EC2Admin" and provide a description.
  8. Review and create the role.

#

- Launch EC2 Instance
  1. Go to the EC2 dashboard in the AWS Management Console.
  2. Click "Launch Instance".
  3. Choose an Amazon Linux 2 AMI.
  4. Select a t2.micro instance type.
  5. Configure instance details:
  6. Network: Default VPC
  7. Subnet: Any available
  8. Auto-assign Public IP: Enable
  9. IAM Role: Select "EC2Admin"
  10. Keep the default storage settings.
  11. Add a tag: Key="Name", Value="workstation".
  12. Create a security group allowing SSH access from EC2 Connect IP.
  13. Review and launch, selecting or creating a key pair.

#

- Connect to EC2 Instance and Install Terraform

  1. From the EC2 dashboard, select your "workstation" instance.
  2. Click "Connect" and use the "EC2 Instance Connect" method.
  3. In the browser-based SSH session, update system packages:

  ```
  sudo yum update -y
  ```

  4. Install yum-utils:

  ```
  sudo yum install -y yum-utils
  ```

  5. Add the HashiCorp repository:

  ```
  sudo yum-config-manager --add-repo https://rpm.releases.hashicorp.com/AmazonLinux/hashicorp.repo
  ```

  6. Install Terraform:

  ```
  sudo yum -y install terraform
  ```

  7. Verify installation:

  ```
  terraform version
  ```

#

**2. Google Cloud BigQuery Configuration:**

- Create a Google Cloud Project

  1. Go to the Google Cloud Console https://console.cloud.google.com/.
  2. Click on the project dropdown and select "New Project".
  3. Name the project "CloudMart" and create it.

- Enable BigQuery API
  1. In the Google Cloud Console, go to "APIs & Services" > "Dashboard".
  2. Click "+ ENABLE APIS AND SERVICES".
  3. Search for "BigQuery API" and enable it.
- Create BigQuery Dataset
  1. In the Google Cloud Console, go to BigQuery.
  2. In the Explorer pane, click on your project name.
  3. Click "CREATE DATASET".
  4. Set the Dataset ID to "cloudmart".
  5. Choose your data location and click "CREATE DATASET".
- Create BigQuery Table
  1. In the dataset you just created, click "CREATE TABLE".
  2. Set the Table name to "cloudmart-orders".
  3. Define the schema according to your order structure. Example schema:
  - id: STRING
  - items: JSON
  - userEmail: STRING
  - total: FLOAT
  - status: STRING
  - createdAt: TIMESTAMP
  4. Click "CREATE TABLE".
- Create Service Account and Key
  1. In the Google Cloud Console, go to "IAM & Admin" > "Service Accounts".
  2. Click "CREATE SERVICE ACCOUNT".
  3. Name it "cloudmart-bigquery-sa" and grant it the "BigQuery Data Editor" role.
  4. After creating the service account, click on it, go to the "Keys" tab, and click "ADD KEY" > "Create new key".
  5. Choose JSON as the key type and create the key.
  6. Save the downloaded JSON file as google_credentials.json.

#

**3. AWS Lambda Setup:**

#

- Configure Lambda Function for CloudMart

  1. Navigate to the Lambda function directory:

  ```
  cd challenge-day2/backend/src/lambda/addToBigQuery
  ```

  2. Install required dependencies:

  ```
  sudo yum install npm
  npm install
  ```

  3. Edit the google_credentials.json file in this directory and place the content of your key.

  4. Create a zip file of the entire directory:

  ```
  zip -r dynamodb_to_bigquery.zip .
  ```

  5. This zip file will be used when creating or updating the Lambda function.

  6. Return to the root directory of your project:

  ```
  cd ../../..
  ```

- Update Lambda Function Environment Variables\
   Add the google_credentials.json file to your .gitignore to ensure it is not committed to version control.

#

**4. Terraform Setup:**

- Create a new Terraform configuration

  1. Remove the main.tf file and create an empty one:

  ```
  rm main.tf
  nano main.tf
  ```

  2. Add the following Terraform code to the main.tf file to create the necessary AWS Lambda for BigQuery integration:

  ```
  provider "aws" {
  region = "us-east-1"
  }

  # Tables DynamoDB
  resource "aws_dynamodb_table" "cloudmart_products" {
  name           = "cloudmart-products"
  billing_mode   = "PAY_PER_REQUEST"
  hash_key       = "id"

  attribute {
      name = "id"
      type = "S"
  }
  }

  resource "aws_dynamodb_table" "cloudmart_orders" {
  name           = "cloudmart-orders"
  billing_mode   = "PAY_PER_REQUEST"
  hash_key       = "id"

  attribute {
      name = "id"
      type = "S"
  }

  stream_enabled   = true
  stream_view_type = "NEW_AND_OLD_IMAGES"
  }

  resource "aws_dynamodb_table" "cloudmart_tickets" {
  name           = "cloudmart-tickets"
  billing_mode   = "PAY_PER_REQUEST"
  hash_key       = "id"

  attribute {
      name = "id"
      type = "S"
  }
  }

  # IAM Role for Lambda function
  resource "aws_iam_role" "lambda_role" {
  name = "cloudmart_lambda_role"

  assume_role_policy = jsonencode({
      Version = "2012-10-17"
      Statement = [
      {
          Action = "sts:AssumeRole"
          Effect = "Allow"
          Principal = {
          Service = "lambda.amazonaws.com"
          }
      }
      ]
  })
  }

  # IAM Policy for Lambda function
  resource "aws_iam_role_policy" "lambda_policy" {
  name = "cloudmart_lambda_policy"
  role = aws_iam_role.lambda_role.id

  policy = jsonencode({
      Version = "2012-10-17"
      Statement = [
      {
          Effect = "Allow"
          Action = [
          "dynamodb:Scan",
          "dynamodb:GetRecords",
          "dynamodb:GetShardIterator",
          "dynamodb:DescribeStream",
          "dynamodb:ListStreams",
          "logs:CreateLogGroup",
          "logs:CreateLogStream",
          "logs:PutLogEvents"
          ]
          Resource = [
          aws_dynamodb_table.cloudmart_products.arn,
          aws_dynamodb_table.cloudmart_orders.arn,
          "${aws_dynamodb_table.cloudmart_orders.arn}/stream/*",
          aws_dynamodb_table.cloudmart_tickets.arn,
          "arn:aws:logs:*:*:*"
          ]
      }
      ]
  })
  }

  # Lambda function for listing products
  resource "aws_lambda_function" "list_products" {
  filename         = "list_products.zip"
  function_name    = "cloudmart-list-products"
  role             = aws_iam_role.lambda_role.arn
  handler          = "index.handler"
  runtime          = "nodejs20.x"
  source_code_hash = filebase64sha256("list_products.zip")

  environment {
      variables = {
      PRODUCTS_TABLE = aws_dynamodb_table.cloudmart_products.name
      }
  }
  }

  # Lambda permission for Bedrock
  resource "aws_lambda_permission" "allow_bedrock" {
  statement_id  = "AllowBedrockInvoke"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.list_products.function_name
  principal     = "bedrock.amazonaws.com"
  }

  # Output the ARN of the Lambda function
  output "list_products_function_arn" {
  value = aws_lambda_function.list_products.arn
  }

  # Lambda function for DynamoDB to BigQuery
  resource "aws_lambda_function" "dynamodb_to_bigquery" {
  filename         = "../challenge-day2/backend/src/lambda/addToBigQuery/dynamodb_to_bigquery.zip"
  function_name    = "cloudmart-dynamodb-to-bigquery"
  role             = aws_iam_role.lambda_role.arn
  handler          = "index.handler"
  runtime          = "nodejs20.x"
  source_code_hash = filebase64sha256("../challenge-day2/backend/src/lambda/addToBigQuery/dynamodb_to_bigquery.zip")

  environment {
      variables = {
      GOOGLE_CLOUD_PROJECT_ID        = "lustrous-bounty-436219-f1"
      BIGQUERY_DATASET_ID            = "cloudmart"
      BIGQUERY_TABLE_ID              = "cloudmart-orders"
      GOOGLE_APPLICATION_CREDENTIALS = "/var/task/google_credentials.json"
      }
  }
  }

  # Lambda event source mapping for DynamoDB stream
  resource "aws_lambda_event_source_mapping" "dynamodb_stream" {
  event_source_arn  = aws_dynamodb_table.cloudmart_orders.stream_arn
  function_name     = aws_lambda_function.dynamodb_to_bigquery.arn
  starting_position = "LATEST"
  }
  ```

  3. Save the file and exit the editor.
  4. Apply the configuration:

  ```
  terraform apply
  ```

  5. Type "yes" when prompted to create the resources.

**5. Configuring the Amazon Bedrock Agent**

- Model Access:

  1. In the Amazon Bedrock console, go to "Model access" in the navigation panel.
  2. Choose "Enable specific models".
  3. Select the Claude 3 Sonnet model.
  4. Wait until the model access status changes to "Access granted".

- Create the Agent:

  1.  In the Amazon Bedrock console, choose "Agents" under "Builder tools" in the navigation panel.
  2.  Click "Create agent".
  3.  Name the agent "cloudmart-product-recommendation-agent".
  4.  Select "Claude 3 Sonnet" as the base model.
  5.  Paste the following instructions in the "Instructions for the Agent" section:

          You are a product recommendations agent for CloudMart, an online e-commerce store. Your role is to assist customers in finding products that best suit their needs. Follow these instructions carefully:

          1. Begin each interaction by retrieving the full list of products from the API. This will inform you of the available products and their details.

          2. Your goal is to help users find suitable products based on their requirements. Ask questions to understand their needs and preferences if they're not clear from the user's initial input.

          3. Use the 'name' parameter to filter products when appropriate. Do not use or mention any other filter parameters that are not part of the API.

          4. Always base your product suggestions solely on the information returned by the API. Never recommend or mention products that are not in the API response.

          5. When suggesting products, provide the name, description, and price as returned by the API. Do not invent or modify any product details.

          6. If the user's request doesn't match any available products, politely inform them that we don't currently have such products and offer alternatives from the available list.

          7. Be conversational and friendly, but focus on helping the user find suitable products efficiently.

          8. Do not mention the API, database, or any technical aspects of how you retrieve the information. Present yourself as a knowledgeable sales assistant.

          9. If you're unsure about a product's availability or details, always check with the API rather than making assumptions.

          10. If the user asks about product features or comparisons, use only the information provided in the product descriptions from the API.

          11. Be prepared to assist with a wide range of product inquiries, as our e-commerce store may carry various types of items.

          12. If a user is looking for a specific type of product, use the 'name' parameter to search for relevant items, but be aware that this may not capture all categories or types of products.

          Remember, your primary goal is to help users find the best products for their needs from what's available in our store. Be helpful, informative, and always base your recommendations on the actual product data provided by the API.

- Configure the IAM Role:

  1. In the Bedrock Agent overview, locate the 'Permissions' section.
  2. Click on the IAM role link. This will take you to the IAM console with the correct role selected.
  3. In the IAM console, choose "Add permissions" and then "Create inline policy".
  4. In the JSON tab, paste the following policy:

  ```
  {
  "Version": "2012-10-17",
  "Statement": [
      {
      "Effect": "Allow",
      "Action": "lambda:InvokeFunction",
      "Resource": "arn:aws:lambda:*:*:function:cloudmart-list-products"
      },
      {
      "Effect": "Allow",
      "Action": "bedrock:InvokeModel",
      "Resource": "arn:aws:bedrock:*::foundation-model/anthropic.claude-3-sonnet-20240229-v1:0"
      }
      ]
  }
  ```

- Create an Action Group:

  1. In the "Action groups" section, create a new group called "Get-Product-Recommendations".
  2. Set the action group type as "Define with API schemas".
  3. Select the Lambda function "cloudmart-list-products" as the action group executor.
  4. In the "Action group schema" section, choose "Define via in-line schema editor".
  5. Paste the OpenAPI schema below into the schema editor:

  ```
  {
  "openapi": "3.0.0",
  "info": {
      "title": "Product Details API",
      "version": "1.0.0",
      "description": "This API retrieves product information. Filtering parameters are passed as query strings. If query strings are empty, it performs a full scan and retrieves the full product list."
  },
  "paths": {
      "/products": {
          "get": {
              "summary": "Retrieve product details",
              "description": "Retrieves a list of products based on the provided query string parameters. If no parameters are provided, it returns the full list of products.",
              "parameters": [
                  {
                      "name": "name",
                      "in": "query",
                      "description": "Retrieve details for a specific product by name",
                      "schema": {
                          "type": "string"
                      }
                  }
              ],
              "responses": {
                  "200": {
                      "description": "Successful response",
                      "content": {
                          "application/json": {
                              "schema": {
                                  "type": "array",
                                  "items": {
                                      "type": "object",
                                      "properties": {
                                          "name": {
                                              "type": "string"
                                          },
                                          "description": {
                                              "type": "string"
                                          },
                                          "price": {
                                              "type": "number"
                                          }
                                      }
                                  }
                              }
                          }
                      }
                  },
                  "500": {
                      "description": "Internal Server Error",
                      "content": {
                          "application/json": {
                              "schema": {
                                  "$ref": "#/components/schemas/ErrorResponse"
                              }
                          }
                      }
                  }
              }
          }
      }
  },
  "components": {
      "schemas": {
          "ErrorResponse": {
              "type": "object",
              "properties": {
                  "error": {
                      "type": "string",
                      "description": "Error message"
                  }
              },
              "required": [
                  "error"
              ]
          }
      }
  }
  }
  ```

- Review and Create the Agent:

  1. Review all agent configurations.
  2. Click "Prepare agent" to finalize the creation.

- Test the Agent:
  1. After the Bedrock Agent is created, use the "Test Agent" panel to have conversations with the chatbot.
  2. Verify if the agent is asking relevant questions about the recipient's gender, occasion, and desired category.
  3. Confirm if the agent is consulting the API and presenting appropriate product recommendations.

6. OpenAI Assistant Configuration:

- OpenAI Access:

  1. Access the OpenAI platform (https://platform.openai.com/). \
  2. Log in or create an account if you don't have one yet.

- Create the Assistant:

  1. Navigate to the "Assistants" section.
  2. Click on "Create New Assistant".
  3. Name the assistant "CloudMart Customer Support".
  4. Select the model gpt-4o.

- Configure the Assistant:

  In the "Instructions" section, paste the assistant's instructions:

  `  You are a customer support agent for CloudMart, an e-commerce platform. Your role is to assist customers with general inquiries, order issues, and provide helpful information about using the CloudMart platform. You don't have direct access to specific product or inventory information. Always be polite, patient, and focus on providing excellent customer service. If a customer asks about specific products or inventory, politely explain that you don't have access to that information and suggest they check the website or speak with a sales representative.
 `

- Save the Assistant:

  1. After creating the assistant, it will auto-save.
  2. Note down the Assistant ID, you'll need it for your environment variables.

- Generate API Key:
  1. Go to the API Keys section in your OpenAI account.
  2. Generate a new API key.
  3. Copy this key, you'll need it for your environment variables.

#

**6. Docker Configuration (Frontend & Backend):**

    1. Install Docker on EC2 (You can refer to previous sections for commands and setup.
    2. Create Docker image for both Frontend and Backend by following the steps mentioned for downloading and setting up the Dockerfiles in each directory.
    3. Deploy both Frontend and Backend on Kubernetes using EKS, configuring the YAML files and using commands for setting up the pods, services, etc.
    4. Push Docker images to ECR (Elastic Container Registry) for both the frontend and backend.
    5. Update Kubernetes Deployment with correct configurations and deploy the application.

#

**7. CI/CD Pipeline Configuration (AWS CodePipeline):** 1. Push application code to GitHub and link it to AWS CodePipeline. 2. Create the build project in AWS CodeBuild to build the Docker images. 3. Configure AWS CodeBuild for the Application Deployment to automatically deploy when there are changes in the repository. 4. Run the CI/CD Pipeline to test automatic deployment and ensure everything is deployed properly after each commit.

#

**8. Testing and Verify:** 1. Make changes on GitHub to verify the pipeline is triggered properly. 2. Monitor the Kubernetes Deployment to ensure that the updates are applied successfully. 3. Check the EC2 Instance, and other infrastructure like DynamoDB, and BigQuery to verify if everything is functioning correctly.

#

**9. Cleanup Resources:**

    1. Delete Kubernetes resources using commands like:

    ```
    kubectl delete service cloudmart-frontend-app-service
    kubectl delete deployment cloudmart-frontend-app
    kubectl delete service cloudmart-backend-app-service
    kubectl delete deployment cloudmart-backend-app
    eksctl delete cluster --name cloudmart --region us-east-1
    ```
    2. Remove other cloud resources like S3 buckets, Lambda functions, DynamoDB tables, and any temporary files.

#

**10. Check if everything is working or not**

    Check the pods for front-end:

    ```
    kubectl get pods
    ```

    Go to the address shown and use http//: at the start.
