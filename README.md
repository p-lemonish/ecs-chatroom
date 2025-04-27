# Chatroom Terraform Deployment

This repository contains Terraform configurations to deploy a full-stack chatroom application on AWS, including:

- **Go backend** on ECS Fargate behind an Application Load Balancer (ALB)
- **ECR** repository to store the backend Docker image
- **React frontend** hosted in a private S3 bucket and served via CloudFront with proper routing for API and WebSocket calls
- **CloudWatch Logs** for backend logging

## Prerequisites

- **AWS CLI** installed and configured with a profile that has permissions for ECS, ECR, S3, CloudFront, ALB, and CloudWatch
- **Terraform** v1.2.0 or newer
- **Docker** installed
- **Node.js** and **npm** (for building the React frontend)

## Quickstart

1. **Clone this repo**

2. **Deploy infrastructure**
   ```bash
   terraform init
   terraform apply 
   ```

3. **Build & push the backend Docker image**
   ```bash
   aws ecr get-login-password --region $REGION \
     | docker login --username AWS --password-stdin $REPO_URL

   docker build -t chatroom-backend:latest ../chatroom-go
   docker tag chatroom-backend:latest $REPO_URL:latest

   docker push $REPO_URL:latest
   ```

4. **Build & sync the frontend**
   ```bash
   cd ../chatroom-react
   npm install
   npm run build

   aws s3 sync dist s3://$BUCKET --region $REGION
   ```

