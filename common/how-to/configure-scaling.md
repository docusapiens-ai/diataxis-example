# How to Configure Auto-scaling

## Goal

Configure automatic scaling for MyComputer applications to handle variable load efficiently while optimizing costs.

## Prerequisites

- Applications deployed on AWS ECS
- CloudWatch metrics configured
- Application Autoscaling IAM permissions

## Configure ECS Service Auto-scaling

### Step 1: Register Scalable Target

For each service, register it as a scalable target:

```bash
aws application-autoscaling register-scalable-target \
  --service-namespace ecs \
  --resource-id service/mycomputer-production/inventory-service \
  --scalable-dimension ecs:service:DesiredCount \
  --min-capacity 2 \
  --max-capacity 20
```

### Step 2: Create Scaling Policies

#### Target Tracking Scaling (Recommended)

```bash
# CPU-based scaling
aws application-autoscaling put-scaling-policy \
  --service-namespace ecs \
  --resource-id service/mycomputer-production/inventory-service \
  --scalable-dimension ecs:service:DesiredCount \
  --policy-name cpu-target-tracking \
  --policy-type TargetTrackingScaling \
  --target-tracking-scaling-policy-configuration '{
    "TargetValue": 70.0,
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "ECSServiceAverageCPUUtilization"
    },
    "ScaleOutCooldown": 60,
    "ScaleInCooldown": 180
  }'
```

#### Memory-based Scaling

```bash
aws application-autoscaling put-scaling-policy \
  --service-namespace ecs \
  --resource-id service/mycomputer-production/inventory-service \
  --scalable-dimension ecs:service:DesiredCount \
  --policy-name memory-target-tracking \
  --policy-type TargetTrackingScaling \
  --target-tracking-scaling-policy-configuration '{
    "TargetValue": 75.0,
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "ECSServiceAverageMemoryUtilization"
    }
  }'
```

### Step 3: Configure Custom Metrics Scaling

For business metrics like request count:

```json
{
  "TargetValue": 1000.0,
  "CustomizedMetricSpecification": {
    "MetricName": "RequestCount",
    "Namespace": "MyComputer/Application",
    "Dimensions": [
      {
        "Name": "ServiceName",
        "Value": "inventory"
      }
    ],
    "Statistic": "Average",
    "Unit": "Count"
  }
}
```

## Configure Step Scaling

For more granular control:

```bash
aws application-autoscaling put-scaling-policy \
  --service-namespace ecs \
  --resource-id service/mycomputer-production/payments-service \
  --scalable-dimension ecs:service:DesiredCount \
  --policy-name step-scaling-policy \
  --policy-type StepScaling \
  --step-scaling-policy-configuration '{
    "AdjustmentType": "ChangeInCapacity",
    "Cooldown": 60,
    "MetricAggregationType": "Average",
    "StepAdjustments": [
      {
        "MetricIntervalLowerBound": 0,
        "MetricIntervalUpperBound": 10,
        "ScalingAdjustment": 1
      },
      {
        "MetricIntervalLowerBound": 10,
        "MetricIntervalUpperBound": 20,
        "ScalingAdjustment": 2
      },
      {
        "MetricIntervalLowerBound": 20,
        "ScalingAdjustment": 3
      }
    ]
  }'
```

## Configure Scheduled Scaling

For predictable traffic patterns:

```bash
# Scale up for business hours
aws application-autoscaling put-scheduled-action \
  --service-namespace ecs \
  --resource-id service/mycomputer-production/customer-base-service \
  --scalable-dimension ecs:service:DesiredCount \
  --scheduled-action-name scale-up-business-hours \
  --schedule "cron(0 8 ? * MON-FRI *)" \
  --scalable-target-action MinCapacity=5,MaxCapacity=30

# Scale down for nights
aws application-autoscaling put-scheduled-action \
  --service-namespace ecs \
  --resource-id service/mycomputer-production/customer-base-service \
  --scalable-dimension ecs:service:DesiredCount \
  --scheduled-action-name scale-down-night \
  --schedule "cron(0 20 ? * * *)" \
  --scalable-target-action MinCapacity=2,MaxCapacity=10
```

## Database Scaling

### RDS Auto-scaling

```bash
# Enable storage autoscaling
aws rds modify-db-instance \
  --db-instance-identifier mycomputer-inventory-db \
  --allocated-storage 100 \
  --max-allocated-storage 1000 \
  --apply-immediately

# Aurora Serverless v2 scaling
aws rds modify-db-cluster \
  --db-cluster-identifier mycomputer-locations-cluster \
  --serverless-v2-scaling-configuration MinCapacity=0.5,MaxCapacity=4
```

## Application-level Scaling Configuration

### Configure Health Checks

```javascript
// Health check endpoint for proper scaling decisions
app.get('/health', async (req, res) => {
  const checks = {
    database: await checkDatabase(),
    memory: process.memoryUsage(),
    uptime: process.uptime()
  };
  
  const healthy = checks.database && 
    checks.memory.heapUsed < checks.memory.heapTotal * 0.9;
  
  res.status(healthy ? 200 : 503).json({
    status: healthy ? 'healthy' : 'unhealthy',
    checks
  });
});
```

### Implement Graceful Shutdown

```javascript
// Graceful shutdown for scaling down
process.on('SIGTERM', async () => {
  console.log('SIGTERM received, starting graceful shutdown');
  
  // Stop accepting new requests
  server.close(() => {
    console.log('HTTP server closed');
  });
  
  // Wait for ongoing requests to complete
  await waitForRequestsToComplete();
  
  // Close database connections
  await closeConnections();
  
  process.exit(0);
});
```

## Load Testing for Scaling Validation

### Using Artillery

```yaml
# load-test.yml
config:
  target: "https://api.mycomputer.com"
  phases:
    - duration: 60
      arrivalRate: 10
      name: "Warm up"
    - duration: 300
      arrivalRate: 100
      name: "Sustained load"
    - duration: 120
      arrivalRate: 200
      name: "Peak load"

scenarios:
  - name: "API Test"
    flow:
      - get:
          url: "/inventory/items"
      - think: 5
      - post:
          url: "/inventory/items"
          json:
            name: "Test Item"
            quantity: 10
```

Run the test:
```bash
artillery run load-test.yml
```

## Monitoring Scaling Events

### Create CloudWatch Dashboard

```json
{
  "widgets": [
    {
      "type": "metric",
      "properties": {
        "metrics": [
          ["AWS/ECS", "ServiceDesiredCount", {"stat": "Average"}],
          [".", "ServiceRunningCount", {"stat": "Average"}],
          ["AWS/ApplicationELB", "TargetResponseTime", {"stat": "Average"}],
          [".", "RequestCount", {"stat": "Sum"}]
        ],
        "view": "timeSeries",
        "region": "eu-west-1",
        "title": "Scaling Metrics"
      }
    }
  ]
}
```

## Cost Optimization

### Implement Predictive Scaling

```bash
aws autoscaling put-scaling-policy \
  --auto-scaling-group-name mycomputer-asg \
  --policy-name predictive-scaling \
  --policy-type PredictiveScaling \
  --predictive-scaling-configuration '{
    "MetricSpecifications": [{
      "TargetValue": 70,
      "PredefinedMetricPairSpecification": {
        "PredefinedMetricType": "ASGCPUUtilization"
      }
    }]
  }'
```

### Use Spot Instances

```json
{
  "capacityProviders": ["FARGATE", "FARGATE_SPOT"],
  "defaultCapacityProviderStrategy": [
    {
      "capacityProvider": "FARGATE",
      "weight": 1,
      "base": 2
    },
    {
      "capacityProvider": "FARGATE_SPOT",
      "weight": 4
    }
  ]
}
```

## Troubleshooting

### Scaling Not Triggering

1. Check CloudWatch metrics are being published
2. Verify IAM permissions for Application Autoscaling
3. Review scaling policy thresholds
4. Check cooldown periods

### Rapid Scaling (Flapping)

1. Increase cooldown periods
2. Adjust target values
3. Implement request buffering
4. Review health check configuration

## Best Practices

- Start with conservative scaling policies
- Use multiple metrics for scaling decisions
- Implement circuit breakers to prevent cascade failures
- Test scaling policies under load
- Monitor costs and optimize capacity providers

## Related Documentation

- [Setup Monitoring](./setup-monitoring.md)
- [Disaster Recovery](./disaster-recovery.md)
- [AWS Infrastructure Reference](../reference/aws-infrastructure.md)
