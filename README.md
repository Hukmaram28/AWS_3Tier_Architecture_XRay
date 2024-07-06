# 3-Tier Web Architecture, CI/CD Automation, and X-ray Tracing in AWS

## Overview

This project provides a high level walkthrough of creating a 3-tier web architecture in aws cloud, automating the CI/CD pipeline for web and app server builds and deployments to ECS Fargate, and demonstrating observability by instrumenting AWS X-ray. We will use CloudFormation YAML templates to create the infrastructure. The web tier features a React.js app, while the backend is a Node.js Express application with a MySQL database.

The web and API repositories are hosted separately in private repositories. However, for demonstration purposes, I will provide the Dockerfile and buildspec.yml files in the `api` and `web` folders without the full source code.

## Architecture Diagram

![Architecture Diagram](/images/3TA.png)

## Step 0: Network Configuration

We will create a Virtual Private Cloud (VPC) with 2 public subnets, 4 private subnets, one Internet Gateway, 2 NAT Gateways, and corresponding route tables for public and private subnets. Additionally, we will configure 4 security groups: one for the Application Load Balancer (ALB), one for the web server (allowing incoming traffic from the ALB security group), one for the app server (allowing inbound traffic from the ALB), and one for the RDS MySQL database. The YAML file for this configuration is located in the `Iac` folder under the name `Network.yml`.

## Step 1: Database Setup

We will create a Multi-AZ RDS MySQL DB instance using a DB snapshot which I created from the production database. For the database subnet group, we will use the two private db subnets created in the previous stack. The DB instance DNS name, DB port, and database name will be stored in the System Manager Parameter Store, while the database master username and password will be stored in Secrets Manager. These variables will be injected into the ECS containers at runtime. The YAML file for this configuration is located in the `Iac` folder under the name `rds.yml`.

## Step 2: ECS Cluster and App Server Setup

We will create an ECS cluster and the following IAM roles:
- An IAM Execution Role for ECS (to pull images from ECR, retrieve secrets from Secrets Manager or SSM Parameter Store)
- An IAM Task Role for containers to access other AWS services (e.g., to write to the X-ray daemon)

Additionally, we will create an auto-scaling role, auto-scaling target, and bind it to an auto-scaling policy using a target tracking scaling policy based on average CPU utilization of the ECS service. We will also set up a load balancer target group, a load balancer listener rule, and a Multi-AZ load balancer. A listener rule with the path `/api/*` will forward traffic to the app server ECS container ports.

We will create a CloudWatch log group to collect ECS logs and attach this to the ecs task definition.
Next, we will create a task definition for the app server and an ECS service to deploy the app container in ECS Fargate, using the two private subnets created earlier. Make sure you set proper environment variables and secrets in the task deinition, These environment variables will be injected to the tasks at the runtime. 

Make sure to push the app server image to ECR prior executing this stack and provide the image uri in the stack parameter.
The YAML file for this configuration is located in the `Iac` folder under the name `api_fargate.yml`.

## Step 3: Web Application Setup

Similar to Step 2, we will create an ECS Fargate task definition and a service to deploy the web application into the 2 public subnets. We will add a listener rule with the wildcard path `*` to the ALB to forward traffic to the web application. This rule will have a lower priority than the previous rule created for the app server with the path pattern `/api/*`, ensuring that API calls are forwarded to the app servers while the remaining traffic is directed to the web application. The YAML file for this configuration is located in the `Iac` folder under the name `web_fargate.yml`.

## Step 4: CI/CD Pipeline Setup

We can either push the app and web application source code repositories to GitHub or AWS CodeCommit. We will add a `buildspec.yml` file to both repositories with appropriate build phases. 

- **Pre-build phase**: Log in to AWS ECR.
- **Build phase**: Build the Docker image.
- **Post-build phase**: Push the image to the ECR repository and store an `imagedefinitions.json` file with the image details. This file will be used by CodePipeline to deploy the latest build image to the target ECS cluster.

After creating the `buildspec.yml` file, we need to create build project for each web and app servers in AWS CodeBuild with the required environment variables such as `ACCOUNT_ID` and `ECR_REPO_NAME`.

Next, we will create two CodePipelines with Source, Build, and Deploy stages. In the Deploy phase, select ECS as the target, which we created as part of the infrastructure.

You can also write a CloudFormation template for the above CodePipeline pipelines to create the pipeline with just a few clicks using CloudFormation.


## Step-5: Observability using X-Ray