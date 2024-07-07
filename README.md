# 3-Tier Web Architecture, CI/CD Automation, and X-ray Tracing in AWS

## Overview

This project provides a high level walkthrough of creating a 3-tier web architecture in aws cloud for a distributed microservices architechture, Setting up a CI/CD pipeline for web and app server builds and deployments to ECS Fargate, and demonstrating observability by instrumenting AWS X-ray. We will be use CloudFormation YAML templates to create the infrastructure. 

The web tier features a React.js web application (https://www.crypteye.io/), while the backend is a TypeScript TypeORM based Node.js express application with a MySQL database.
The web and API repositories are hosted separately in private repositories. However, for demonstration purposes, I will provide the Dockerfile and buildspec.yml files in the `api` and `web` folders without providing the full source code.

## Architecture Diagram

![Architecture Diagram](/images/3TA.png)

## Step 0: Network Configuration

We will create a Virtual Private Cloud (VPC) with:

- 2 public subnets in two availability zones (for the web tier and Multi-AZ load balancer)
- 4 private subnets, 2 in each availability zone (2 subnets for the app tier and 2 for the database tier)
- 1 Internet Gateway (to allow internet access in public subnets)
- 2 NAT Gateways (to allow internet access in private subnets)
- Corresponding route tables for public and private subnets

Additionally, we will configure 4 security groups:

- One for the Application Load Balancer (ALB)
- One for the web server. This security group will have an inbound rule to allow traffic coming from the ALB's security group
- One for the app server. This security group will also have an inbound rule to allow traffic from the ALB
- One for the RDS MySQL database

The YAML file for this configuration is located in the `Iac` folder under the name `network.yml`.

![VPC Diagram](/images/vpc_arch.png)

![VPC Stack](/images/vpc_stack.png)

## Step 1: Database Setup

We will create a Multi-AZ RDS MySQL DB instance using a DB snapshot, which I previously created from the production database. We will need the ARN of the snapshot. For the database subnet group, we will use the two private DB subnets created in the previous stack. 
The DB instance DNS name, DB port, and database name will be stored in the System Manager Parameter Store, while the database master username and password will be stored in Secrets Manager. These variables will be injected into the ECS containers at runtime.

The YAML file for this configuration is located in the `Iac` folder under the name `rds.yml`.

![RDS Stack](/images/rds_stack.png)

## Step 2: ECS Cluster and App Server Setup

We will create an ECS cluster and the following IAM roles:

- An IAM Execution Role for ECS (to pull images from ECR, retrieve secrets from Secrets Manager, and from SSM Parameter Store)
- An IAM Task Role for containers to access other AWS services (e.g., to write to the X-ray daemon)

Additionally, we will:

- Create an auto-scaling role
- Create an auto-scaling target for the API ECS service
- Attach an auto-scaling policy to the target, which is a target tracking scaling policy based on average CPU utilization of the ECS service

We will also set up:

- A load balancer target group
- A load balancer listener rule
- A Multi-AZ load balancer
- A listener rule with the path `/api/*` to forward traffic to the app server ECS container ports

We will:

- Create a CloudWatch log group to collect ECS logs and attach this to the ECS task definition
- Create a task definition for the app server containers
- Create an ECS service to deploy the app container in ECS Fargate, using the two private subnets created in the network stack

Ensure you set proper environment variables and secrets in the task definition using SSM Parameter Store and Secrets Manager. We provide secret values such as database username and password in `Secrets:` block and non-secret values such as load balancer url in `Environment:` block.
These environment variables will be injected into the containers at runtime. 

Before executing this stack, make sure to push the app server docker image to aws ECR and provide the image URI in the stack parameter.

The YAML file for this configuration is located in the `Iac` folder under the name `api_fargate.yml`.

![App Stack](/images/app_stack.png)
![Api Service](/images/api_service.png)

## Step 3: Web Application Setup

This step is similar to what we did in the previous stack. We will:

- Create an ECS Fargate task definition for the web application
- Create an ECS service to deploy the web application into the 2 public subnets
- Add a listener rule with the wildcard path `*` to the ALB to forward traffic to the web application

Note: The listener rule will have a lower priority than the rule created for the app server with the path pattern `/api/*`, ensuring that API calls are forwarded to the app servers while the remaining traffic is directed to the web application.

The YAML file for this configuration is located in the `Iac` folder under the name `web_fargate.yml`.


Congratulations! We have successfully deployed:
- An RDS MySQL database
- A Typescript, TypeORM nodejs app server
- A react.js web application

These components have been deployed into a 3-tier architecture in AWS.
We can now access the web application by visiting the load balancer's DNS name in the browser.

If you encounter any problems accessing API endpoints or the database, make sure you have properly configured the environment variables in the app and web task definitions. You can make use of cloudWatch log groups which we setup in the task definition to look for application container logs.

![Web Stack](/images/web_stack.png)
![CloudWatch Logs](/images/cloudwatch_logs.png)
![Web Service](/images/web_service.png)

## Step 4: CI/CD Pipeline Setup

To set up the CI/CD pipeline, we need to publish our source code. We can either push the app and web application source code repositories to GitHub or AWS CodeCommit. Then, we can set up various branches from the `main` branch, such as `dev`, `it`, and `prod`. For our pipeline, we will use the `dev` branch, so we will check out to the `dev` branch.

We will add a `buildspec.yml` file to both repositories with appropriate build phases:

- **Pre-build phase**: Log in to AWS ECR.
- **Build phase**: Build the Docker image.
- **Post-build phase**: Push the image to the ECR repository and store an `imagedefinitions.json` file with the image name and imageUri. This file will be used by CodePipeline to deploy the latest build image to the target ECS cluster.

After creating the `buildspec.yml` file, we need to create build projects for both the web and app servers in AWS CodeBuild. Use the source code from CodeCommit or GitHub, then provide the required environment variables for the build, such as `ACCOUNT_ID` and `ECR_REPO_NAME`. Trigger a build to check if it works and verify that the Docker images are pushed to the ECR repositories.

Next, we will create CodePipelines (one for each of the web and app repositories) with Source, Build, and Deploy stages. In the Deploy phase, select ECS as the target, which we created as part of the infrastructure, and provide container names that match those used during infrastructure creation.

Once the setup is complete, the source code will be built and deployed to ECS as soon as changes are pushed to the repositories.

You can also write a CloudFormation template for the above CodePipeline pipelines to create the pipeline with just a few clicks using CloudFormation.

![CodeCommit](/images/codeCommit.png)
![CodeBuild](/images/codeBuild.png)
![CodePipeline](/images/codePipeline.png)

## Step 5: Observability using X-Ray

To enable X-Ray tracing and reporting, we first need to instrument our source code with X-Ray. AWS provides different X-Ray SDKs for different languages. Please refer to the documentation [here](https://docs.aws.amazon.com/xray/latest/devguide/aws-xray.html).

Our backend is a TypeScript-TypeORM application with a MySQL database. We need to install the `aws-xray-sdk` npm package and instrument the code accordingly. Sample examples for nodejs provided by AWS can be found [here](https://github.com/aws-samples/aws-xray-sdk-node-sample/blob/master/index.js). Although we can trace our backend database calls, there is no official support provided by the X-Ray SDK for TypeORM thus far. Therefore, I was unable to instrument DB calls, but this can be done using other open-source observability tools like OpenTelemetry.

Our front end is a React.js single-page application, which means the page metadata is not dynamically updated when a user navigates to different pages. To solve this issue and provide better SEO, I used server-side rendering to update web metadata dynamically, allowing me to instrument the X-Ray code on the web server side as well. This can be seen in the X-Ray tracing map.

Once the code is instrumented, we need to update our web and API CloudFormation templates to include a sidecar container along with essential containers for the X-Ray daemon to collect tracing data and report to the X-Ray API. By default the X-Ray daemon uses port 2000 UDP for the collection of traces from x-ray-sdk.

To do this, make the following changes in `api_fargate.yml` and `web_fargate.yml` template files:
- Add the X-Ray daemon container to the task definition. We can use the official image provided by AWS.
- Add a log group to store X-Ray container logs in CloudWatch.
- Update the container task role and add the following `ManagedPolicyArn` to allow the daemon container to report tracing data to the X-Ray API: "arn:aws:iam::aws:policy/AWSXRayDaemonWriteAccess"

Once these changes are made, update the web and API stacks using cloudformation and deploy the latest source code with X-Ray instrumentation using codePipeline. You will be able to see the tracing data in AWS X-Ray and visualize the data using the X-Ray tracing map.

![X-Ray Trace Map](/images/x-ray-trace-map.png)
![X-Ray traces records](/images/x-ray-trace-record.png)
![X-Ray traces](/images/x-ray-traces.png)
