# AWS ECS Deployment Tutorial

## Introduction

This tutorial will guide you through deploying MyComputer applications to AWS ECS (Elastic Container Service). By the end, you'll have all four applications running in a highly available, scalable production environment.

## What You'll Deploy

We will deploy:
- Inventory Management API
- Customer Base (Frontend & Backend)
- Payments Processing Service
- Locations Database Service

## Prerequisites

Before starting, ensure you have:
- AWS CLI configured with appropriate credentials
- Docker images pushed to Amazon ECR
- Terraform installed (v1.5+)
- Basic understanding of AWS services

## Step 1: Prepare Your AWS Environment

First, create the necessary AWS resources:

```bash
aws configure
# Enter your AWS Access Key ID
# Enter your AWS Secret Access Key
# Default region: eu-west-1
# Default output format: json
```

Verify your configuration:
```bash
aws sts get-caller-identity
```

You should see your AWS account details.

## Step 2: Create ECR Repositories

Create repositories for each application:

```bash
aws ecr create-repository --repository-name mycomputer/inventory
aws ecr create-repository --repository-name mycomputer/customer-base-api
aws ecr create-repository --repository-name mycomputer/customer-base-frontend
aws ecr create-repository --repository-name mycomputer/payments
aws ecr create-repository --repository-name mycomputer/locations
```

Note the repository URIs returned - you'll need these.

## Step 3: Build and Push Docker Images

For each application, build and push the Docker image:

```bash
# Login to ECR
aws ecr get-login-password --region eu-west-1 | docker login --username AWS --password-stdin [YOUR_ACCOUNT_ID].dkr.ecr.eu-west-1.amazonaws.com

# Build and push Inventory
cd inventory
docker build -t mycomputer/inventory .
docker tag mycomputer/inventory:latest [YOUR_ACCOUNT_ID].dkr.ecr.eu-west-1.amazonaws.com/mycomputer/inventory:latest
docker push [YOUR_ACCOUNT_ID].dkr.ecr.eu-west-1.amazonaws.com/mycomputer/inventory:latest
```

Repeat for each application. You'll see upload progress for each layer.

## Step 4: Create ECS Cluster

```bash
aws ecs create-cluster --cluster-name mycomputer-production
```

You should see: "cluster created successfully"

## Step 5: Create Task Definitions

Create a task definition for the Inventory service:

```bash
cat > inventory-task-def.json << EOF
{
  "family": "inventory-service",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "512",
  "memory": "1024",
  "containerDefinitions": [
    {
      "name": "inventory",
      "image": "[YOUR_ACCOUNT_ID].dkr.ecr.eu-west-1.amazonaws.com/mycomputer/inventory:latest",
      "portMappings": [
        {
          "containerPort": 3001,
          "protocol": "tcp"
        }
      ],
      "environment": [
        {"name": "NODE_ENV", "value": "production"},
        {"name": "PORT", "value": "3001"}
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/inventory",
          "awslogs-region": "eu-west-1",
          "awslogs-stream-prefix": "ecs"
        }
      }
    }
  ]
}
EOF

aws ecs register-task-definition --cli-input-json file://inventory-task-def.json
```

## Step 6: Create Application Load Balancer

```bash
# Create ALB
aws elbv2 create-load-balancer \
  --name mycomputer-alb \
  --subnets subnet-xxx subnet-yyy \
  --security-groups sg-xxx \
  --scheme internet-facing \
  --type application
```

Note the ALB ARN returned.

## Step 7: Create Target Groups

```bash
aws elbv2 create-target-group \
  --name inventory-targets \
  --protocol HTTP \
  --port 3001 \
  --vpc-id vpc-xxx \
  --target-type ip \
  --health-check-path /health
```

## Step 8: Create ECS Services

```bash
aws ecs create-service \
  --cluster mycomputer-production \
  --service-name inventory-service \
  --task-definition inventory-service:1 \
  --desired-count 2 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[subnet-xxx,subnet-yyy],securityGroups=[sg-xxx],assignPublicIp=ENABLED}" \
  --load-balancers targetGroupArn=arn:aws:elasticloadbalancing:...,containerName=inventory,containerPort=3001
```

You'll see the service being created. ECS will automatically start 2 tasks.

## Step 9: Configure Auto Scaling

```bash
aws application-autoscaling register-scalable-target \
  --service-namespace ecs \
  --resource-id service/mycomputer-production/inventory-service \
  --scalable-dimension ecs:service:DesiredCount \
  --min-capacity 2 \
  --max-capacity 10

aws application-autoscaling put-scaling-policy \
  --service-namespace ecs \
  --resource-id service/mycomputer-production/inventory-service \
  --scalable-dimension ecs:service:DesiredCount \
  --policy-name cpu-scaling \
  --policy-type TargetTrackingScaling \
  --target-tracking-scaling-policy-configuration file://scaling-policy.json
```

## Step 10: Verify Deployment

Check service status:
```bash
aws ecs describe-services \
  --cluster mycomputer-production \
  --services inventory-service
```

Look for "runningCount": 2 and "status": "ACTIVE"

Test the endpoint:
```bash
curl http://mycomputer-alb-xxx.eu-west-1.elb.amazonaws.com/health
```

## Step 11: Deploy Remaining Services

Repeat steps 5-10 for:
- Customer Base API and Frontend
- Payments Service
- Locations Service

Each service follows the same pattern but with different ports and configurations.

## Step 12: Configure Service Discovery

```bash
aws servicediscovery create-private-dns-namespace \
  --name mycomputer.local \
  --vpc vpc-xxx

aws servicediscovery create-service \
  --name inventory \
  --namespace-id ns-xxx \
  --dns-config "DnsRecords=[{Type=A,TTL=60}]"
```

This allows services to communicate using internal DNS names.

## What You've Accomplished

You have successfully:
- ✅ Created an ECS cluster in AWS
- ✅ Deployed containerized applications
- ✅ Configured load balancing
- ✅ Set up auto-scaling
- ✅ Enabled service discovery

## Production Checklist

Before going live:
- [ ] Configure CloudWatch alarms
- [ ] Set up backup strategies
- [ ] Implement secrets management
- [ ] Configure WAF rules
- [ ] Set up CI/CD pipelines

## Next Steps

- [How to Configure Monitoring](../how-to/setup-monitoring.md)
- [Disaster Recovery Procedures](../how-to/disaster-recovery.md)
- [Scalability Patterns](../explanation/scalability-patterns.md)
