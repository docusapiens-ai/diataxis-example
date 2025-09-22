# AWS Infrastructure Reference

## Overview

MyComputer operates on AWS cloud infrastructure across multiple regions in Europe, providing high availability and scalability for all services.

## AWS Services Used

### Compute

#### Amazon ECS (Elastic Container Service)
- **Cluster**: `mycomputer-production`
- **Launch Type**: Fargate
- **Task CPU**: 512-2048 units
- **Task Memory**: 1024-4096 MB
- **Container Insights**: Enabled

#### AWS Fargate
- **Platform Version**: LATEST
- **Network Mode**: awsvpc
- **Spot Instances**: 60% capacity for non-critical workloads

### Networking

#### Amazon VPC
- **CIDR Block**: 10.0.0.0/16
- **Availability Zones**: 3 (eu-west-1a, eu-west-1b, eu-west-1c)
- **Subnets**:
  - Public: 10.0.1.0/24, 10.0.2.0/24, 10.0.3.0/24
  - Private: 10.0.10.0/24, 10.0.11.0/24, 10.0.12.0/24
  - Database: 10.0.20.0/24, 10.0.21.0/24, 10.0.22.0/24

#### Application Load Balancer
- **Name**: `mycomputer-alb`
- **Type**: Application
- **Scheme**: Internet-facing
- **IP Address Type**: IPv4
- **Security Group**: `sg-alb-public`

#### Amazon Route 53
- **Hosted Zone**: mycomputer.com
- **Record Types**:
  - A records for load balancers
  - CNAME records for services
  - MX records for email
- **Health Checks**: Every 30 seconds
- **Routing Policy**: Weighted (Primary: 90%, Secondary: 10%)

### Storage

#### Amazon RDS
- **Engine**: PostgreSQL 14.7
- **Instance Classes**:
  - Production: db.r6g.xlarge
  - Development: db.t3.medium
- **Multi-AZ**: Enabled
- **Backup Retention**: 30 days
- **Encryption**: AES-256

#### Amazon S3
- **Buckets**:
  - `mycomputer-assets`: Static assets
  - `mycomputer-backups`: Database backups
  - `mycomputer-logs`: Application logs
- **Versioning**: Enabled
- **Lifecycle Policies**:
  - Transition to IA after 30 days
  - Transition to Glacier after 90 days
  - Delete after 365 days

#### Amazon EFS
- **File System**: `fs-mycomputer`
- **Performance Mode**: General Purpose
- **Throughput Mode**: Bursting
- **Encryption**: At rest and in transit

### Security

#### AWS IAM
- **Roles**:
  - `ecsTaskExecutionRole`: ECS task execution
  - `ecsTaskRole`: Application permissions
  - `lambdaExecutionRole`: Lambda functions
  - `ec2InstanceRole`: EC2 instances

#### AWS Secrets Manager
- **Secrets**:
  - Database credentials
  - API keys
  - Stripe webhook secrets
  - JWT signing keys
- **Rotation**: Every 90 days

#### AWS WAF
- **Web ACL**: `mycomputer-waf`
- **Rules**:
  - Rate limiting: 2000 requests/5 minutes
  - SQL injection protection
  - XSS protection
  - Geographic restrictions

### Monitoring

#### Amazon CloudWatch
- **Log Groups**:
  - `/ecs/inventory`
  - `/ecs/customer-base`
  - `/ecs/payments`
  - `/ecs/locations`
- **Retention**: 30 days
- **Metric Namespaces**:
  - `AWS/ECS`
  - `AWS/RDS`
  - `AWS/ApplicationELB`
  - `MyComputer/Application`

#### AWS X-Ray
- **Service Map**: Enabled
- **Sampling Rate**: 10%
- **Trace Retention**: 30 days

### Container Registry

#### Amazon ECR
- **Repositories**:
  - `mycomputer/inventory`
  - `mycomputer/customer-base-api`
  - `mycomputer/customer-base-frontend`
  - `mycomputer/payments`
  - `mycomputer/locations`
- **Lifecycle Policy**: Keep last 10 images
- **Scan on Push**: Enabled

## Network Architecture

```
Internet
    |
    v
Route 53
    |
    v
CloudFront (CDN)
    |
    v
WAF
    |
    v
Application Load Balancer
    |
    +-------------------+-------------------+
    |                   |                   |
    v                   v                   v
Target Group 1      Target Group 2      Target Group 3
(Inventory)         (Customer Base)     (Payments)
    |                   |                   |
    v                   v                   v
ECS Tasks           ECS Tasks           ECS Tasks
    |                   |                   |
    v                   v                   v
RDS Instance        RDS Instance        RDS Instance
```

## Security Groups

### ALB Security Group (`sg-alb-public`)
| Type | Protocol | Port | Source |
|------|----------|------|--------|
| Inbound | HTTPS | 443 | 0.0.0.0/0 |
| Inbound | HTTP | 80 | 0.0.0.0/0 |
| Outbound | All | All | 0.0.0.0/0 |

### ECS Tasks Security Group (`sg-ecs-tasks`)
| Type | Protocol | Port | Source |
|------|----------|------|--------|
| Inbound | TCP | 3000-3010 | sg-alb-public |
| Outbound | All | All | 0.0.0.0/0 |

### RDS Security Group (`sg-rds`)
| Type | Protocol | Port | Source |
|------|----------|------|--------|
| Inbound | PostgreSQL | 5432 | sg-ecs-tasks |
| Outbound | All | All | 0.0.0.0/0 |

## Environment Variables

### Common Variables
```bash
NODE_ENV=production
AWS_REGION=eu-west-1
LOG_LEVEL=info
```

### Service-Specific Variables

#### Inventory Service
```bash
PORT=3001
DB_HOST=inventory-db.cluster-xxx.eu-west-1.rds.amazonaws.com
DB_NAME=inventory
DB_PORT=5432
REDIS_URL=redis://inventory-cache.xxx.cache.amazonaws.com:6379
```

#### Customer Base Service
```bash
PORT=3002
DB_HOST=customers-db.cluster-xxx.eu-west-1.rds.amazonaws.com
DB_NAME=customers
JWT_SECRET_ARN=arn:aws:secretsmanager:eu-west-1:xxx:secret:jwt-key
SESSION_TIMEOUT=3600
```

#### Payments Service
```bash
PORT=3003
DB_HOST=payments-db.cluster-xxx.eu-west-1.rds.amazonaws.com
DB_NAME=payments
STRIPE_SECRET_KEY_ARN=arn:aws:secretsmanager:eu-west-1:xxx:secret:stripe
WEBHOOK_ENDPOINT_SECRET_ARN=arn:aws:secretsmanager:eu-west-1:xxx:secret:webhook
```

#### Locations Service
```bash
PORT=3004
DB_HOST=locations-db.cluster-xxx.eu-west-1.rds.amazonaws.com
DB_NAME=locations
POSTGIS_ENABLED=true
MAX_SEARCH_RADIUS=100
```

## Tagging Strategy

All resources follow this tagging convention:

```json
{
  "Environment": "production|staging|development",
  "Application": "inventory|customer-base|payments|locations",
  "Team": "platform|backend|frontend",
  "CostCenter": "engineering|operations",
  "ManagedBy": "terraform|manual",
  "Owner": "team-email@mycomputer.com"
}
```

## Cost Optimization

### Reserved Instances
- RDS: 3-year term, all upfront
- EC2: 1-year term, partial upfront

### Savings Plans
- Compute Savings Plan: $10,000/month commitment
- EC2 Instance Savings Plan: For baseline capacity

### Spot Instances
- Used for: Development environments, batch processing
- Spot percentage: 60% for non-critical workloads

## Compliance

### Data Protection
- Encryption at rest: AES-256
- Encryption in transit: TLS 1.2+
- Key Management: AWS KMS

### Certifications
- ISO 27001
- SOC 2 Type II
- GDPR Compliant
- PCI DSS Level 1

## Disaster Recovery

### Backup Strategy
- **RTO**: 15 minutes (critical), 1 hour (non-critical)
- **RPO**: 5 minutes (critical), 30 minutes (non-critical)
- **Backup Regions**: eu-west-1 (primary), eu-central-1 (secondary)

### Multi-Region Setup
- Active-Passive configuration
- Database replication lag: < 1 second
- Automated failover via Route 53 health checks

## Performance Specifications

### Service Level Objectives (SLOs)
- API Response Time: p99 < 500ms
- Availability: 99.95%
- Error Rate: < 0.1%

### Capacity Planning
- Peak Load: 10,000 requests/second
- Database Connections: 200 per service
- Memory Utilization Target: 70%
- CPU Utilization Target: 60%

## Maintenance Windows

- **RDS Maintenance**: Sunday 03:00-04:00 UTC
- **ECS Deployments**: Rolling updates, zero downtime
- **Infrastructure Updates**: Tuesday/Thursday 02:00-04:00 UTC
