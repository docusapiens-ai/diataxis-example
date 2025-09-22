# How to Set Up CloudWatch Monitoring

## Goal

This guide shows you how to configure comprehensive monitoring for MyComputer applications using AWS CloudWatch.

## Prerequisites

- Applications deployed to AWS ECS
- AWS CLI configured
- CloudWatch permissions in your IAM role

## Configure Application Metrics

### Step 1: Enable Container Insights

```bash
aws ecs put-account-setting-default \
  --name containerInsights \
  --value enabled
```

### Step 2: Create Custom Metrics

For each application, add custom metrics:

```javascript
// Example for Node.js applications
const AWS = require('aws-sdk');
const cloudwatch = new AWS.CloudWatch();

const putMetric = async (metricName, value, unit = 'Count') => {
  const params = {
    Namespace: 'MyComputer/Application',
    MetricData: [{
      MetricName: metricName,
      Value: value,
      Unit: unit,
      Timestamp: new Date()
    }]
  };
  
  await cloudwatch.putMetricData(params).promise();
};

// Usage
putMetric('OrdersProcessed', 1);
putMetric('ResponseTime', 250, 'Milliseconds');
```

## Set Up Dashboards

### Step 1: Create Dashboard

```bash
aws cloudwatch put-dashboard \
  --dashboard-name MyComputerOverview \
  --dashboard-body file://dashboard.json
```

### Step 2: Dashboard Configuration

Create `dashboard.json`:

```json
{
  "widgets": [
    {
      "type": "metric",
      "properties": {
        "metrics": [
          ["AWS/ECS", "CPUUtilization", {"stat": "Average"}],
          [".", "MemoryUtilization", {"stat": "Average"}],
          ["MyComputer/Application", "OrdersProcessed", {"stat": "Sum"}]
        ],
        "period": 300,
        "stat": "Average",
        "region": "eu-west-1",
        "title": "Application Performance"
      }
    }
  ]
}
```

## Configure Alarms

### Critical Alarms

```bash
# High CPU Usage
aws cloudwatch put-metric-alarm \
  --alarm-name high-cpu-inventory \
  --alarm-description "Inventory service CPU above 80%" \
  --metric-name CPUUtilization \
  --namespace AWS/ECS \
  --statistic Average \
  --period 300 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2

# API Response Time
aws cloudwatch put-metric-alarm \
  --alarm-name slow-api-response \
  --alarm-description "API response time above 1000ms" \
  --metric-name ResponseTime \
  --namespace MyComputer/Application \
  --statistic Average \
  --period 60 \
  --threshold 1000 \
  --comparison-operator GreaterThanThreshold
```

## Log Aggregation

### Step 1: Create Log Groups

```bash
aws logs create-log-group --log-group-name /ecs/inventory
aws logs create-log-group --log-group-name /ecs/customer-base
aws logs create-log-group --log-group-name /ecs/payments
aws logs create-log-group --log-group-name /ecs/locations
```

### Step 2: Set Retention

```bash
aws logs put-retention-policy \
  --log-group-name /ecs/inventory \
  --retention-in-days 30
```

### Step 3: Create Metric Filters

```bash
aws logs put-metric-filter \
  --log-group-name /ecs/inventory \
  --filter-name errors \
  --filter-pattern "[ERROR]" \
  --metric-transformations \
    metricName=ErrorCount,\
    metricNamespace=MyComputer/Logs,\
    metricValue=1
```

## Application Performance Monitoring

### Configure X-Ray Tracing

```javascript
// Add to your Node.js applications
const AWSXRay = require('aws-xray-sdk-core');
const AWS = AWSXRay.captureAWS(require('aws-sdk'));

// Middleware for Express
app.use(AWSXRay.express.openSegment('MyComputer'));
// ... your routes
app.use(AWSXRay.express.closeSegment());
```

### Enable Service Map

```bash
aws xray create-group \
  --group-name MyComputer \
  --filter-expression "service(\"inventory\") OR service(\"payments\")"
```

## Cost Monitoring

### Set Budget Alerts

```bash
aws budgets create-budget \
  --account-id 123456789012 \
  --budget file://budget.json \
  --notifications-with-subscribers file://notifications.json
```

## Synthetic Monitoring

### Create Canary Tests

```javascript
// CloudWatch Synthetics canary
const synthetics = require('Synthetics');
const log = require('SyntheticsLogger');

const checkApis = async function () {
  const endpoints = [
    'https://api.mycomputer.com/inventory/health',
    'https://api.mycomputer.com/customers/health',
    'https://api.mycomputer.com/payments/health',
    'https://api.mycomputer.com/locations/health'
  ];
  
  for (const endpoint of endpoints) {
    await synthetics.executeHttpStep(
      `Check ${endpoint}`,
      endpoint,
      {
        method: 'GET',
        headers: {'User-Agent': 'CloudWatch-Synthetics'}
      }
    );
  }
};

exports.handler = async () => {
  return await synthetics.executeStep('checkApis', checkApis);
};
```

## Notification Setup

### Configure SNS Topics

```bash
aws sns create-topic --name mycomputer-alerts
aws sns subscribe \
  --topic-arn arn:aws:sns:eu-west-1:123456789012:mycomputer-alerts \
  --protocol email \
  --notification-endpoint ops-team@mycomputer.com
```

### Connect Alarms to SNS

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name high-error-rate \
  --alarm-actions arn:aws:sns:eu-west-1:123456789012:mycomputer-alerts
```

## Best Practices

### Do's
- Set up alerts for both technical and business metrics
- Use composite alarms for complex conditions
- Implement log correlation across services
- Regular review of alarm thresholds

### Don'ts
- Don't create too many alarms (alarm fatigue)
- Don't ignore warning-level metrics
- Don't forget to test alarm notifications

## Troubleshooting

If metrics aren't appearing:
1. Check IAM permissions for CloudWatch
2. Verify the metric namespace and dimensions
3. Ensure the ECS task role has CloudWatch permissions

If alarms aren't triggering:
1. Verify the threshold values
2. Check the evaluation period
3. Test with manual metric data

## Related Guides

- [Configure Auto-scaling](./configure-scaling.md)
- [Disaster Recovery](./disaster-recovery.md)
- [AWS Infrastructure Reference](../reference/aws-infrastructure.md)
