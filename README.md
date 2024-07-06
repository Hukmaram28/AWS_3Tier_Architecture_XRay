# 3-Tier Web Architecture, CI/CD Automation, and X-ray Tracing in AWS

## Overview

This project provides a hands-on walkthrough of creating a 3-tier web architecture in aws cloud, automating the CI/CD pipeline for web and app server builds and deployments to ECS Fargate, and demonstrating observability by instrumenting AWS X-ray. We will use CloudFormation YAML templates to create the infrastructure. The web tier features a React.js app, while the backend is a Node.js Express application with a MySQL database.

The web and API repositories are hosted separately in private repositories. However, for demonstration purposes, I will provide the Dockerfile and buildspec.yml files in the `api` and `web` folders.

## Architecture Diagrams

![Architecture Diagram](/images/3TA.png)

## Step 0: Network Configuration

We will create a Virtual Private Cloud (VPC) with 2 public subnets, 4 private subnets, one Internet Gateway, 2 NAT Gateways, and corresponding route tables for public and private subnets. Additionally, we will configure 4 security groups: one for the Application Load Balancer (ALB), one for the web server (allowing incoming traffic from the ALB security group), one for the app server (allowing inbound traffic from the ALB), and one for the RDS MySQL database. The YAML file for this configuration is located in the `Iac` folder under the name `Network.yml`.

## Step 1: Database Setup

We will create a Multi-AZ RDS MySQL DB instance using a DB snapshot from the production database. For the database subnets, we will use the two private subnets created in the previous stack. The DB instance DNS, DB port, and database name will be stored in the System Manager Parameter Store, while the database master username and password will be stored in Secrets Manager. These variables will be injected into the ECS containers at runtime. The YAML file for this configuration is located in the `Iac` folder under the name `rds.yml`.

## Step 2: ECS Cluster and App Server Setup

We will create an ECS cluster and the following IAM roles:
- An IAM Execution Role for ECS (to retrieve secrets from Secrets Manager or SSM Parameter Store)
- An IAM Task Role for containers to access other AWS services (e.g., to write to the X-ray daemon)

Additionally, we will create an auto-scaling role, auto-scaling target, and bind it to an auto-scaling policy using a target tracking scaling policy based on average CPU utilization by the ECS service. We will also set up a load balancer target group, a load balancer listener rule, and a Multi-AZ load balancer. A listener rule with the path `/api/*` will forward traffic to the app server ECS container ports.

Next, we will create a task definition for the app server and an ECS service to deploy the app container in ECS Fargate, using the two private subnets created earlier. The YAML file for this configuration is located in the `Iac` folder under the name `api_fargate.yml`.

## Step 3: Web Application Setup

Similar to Step 2, we will create an ECS Fargate task definition and a service to deploy the web application into the 2 public subnets. We will add a listener rule with the wildcard path `*` to the ALB to forward traffic to the web application. This rule will have a lower priority than the previous rule created for the app server with the path pattern `/api/*`, ensuring that API calls are forwarded to the app servers while the remaining traffic is directed to the web application. The YAML file for this configuration is located in the `Iac` folder under the name `web_fargate.yml`.
