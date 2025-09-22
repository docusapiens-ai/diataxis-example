# How to Implement Disaster Recovery Procedures

## Goal

Establish and execute disaster recovery procedures to ensure business continuity for MyComputer services in case of failures.

## Prerequisites

- AWS multi-region setup
- Backup policies configured
- Access to AWS Backup and RDS services
- Disaster Recovery team contacts

## Backup Strategy

### Database Backups

#### Configure Automated Backups

```bash
# RDS automated backups
aws rds modify-db-instance \
  --db-instance-identifier mycomputer-inventory-db \
  --backup-retention-period 30 \
  --preferred-backup-window "03:00-04:00" \
  --apply-immediately

# Enable point-in-time recovery
aws rds modify-db-instance \
  --db-instance-identifier mycomputer-inventory-db \
  --enable-iam-database-authentication \
  --apply-immediately
```

#### Create Manual Snapshots

```bash
# Create manual snapshot before major changes
aws rds create-db-snapshot \
  --db-instance-identifier mycomputer-inventory-db \
  --db-snapshot-identifier inventory-snapshot-$(date +%Y%m%d-%H%M%S)

# Copy snapshot to another region
aws rds copy-db-snapshot \
  --source-db-snapshot-identifier arn:aws:rds:eu-west-1:123456789012:snapshot:inventory-snapshot \
  --target-db-snapshot-identifier inventory-dr-snapshot \
  --region eu-central-1
```

### Application Data Backups

#### S3 Cross-Region Replication

```bash
# Enable versioning
aws s3api put-bucket-versioning \
  --bucket mycomputer-assets \
  --versioning-configuration Status=Enabled

# Configure replication
aws s3api put-bucket-replication \
  --bucket mycomputer-assets \
  --replication-configuration file://replication.json
```

`replication.json`:
```json
{
  "Role": "arn:aws:iam::123456789012:role/replication-role",
  "Rules": [
    {
      "ID": "ReplicateAll",
      "Priority": 1,
      "Status": "Enabled",
      "Filter": {},
      "Destination": {
        "Bucket": "arn:aws:s3:::mycomputer-assets-dr",
        "ReplicationTime": {
          "Status": "Enabled",
          "Time": {
            "Minutes": 15
          }
        },
        "StorageClass": "STANDARD_IA"
      }
    }
  ]
}
```

## Recovery Procedures

### Database Recovery

#### Point-in-Time Recovery

```bash
# Restore to specific point in time
aws rds restore-db-instance-to-point-in-time \
  --source-db-instance-identifier mycomputer-inventory-db \
  --target-db-instance-identifier inventory-db-recovered \
  --restore-time 2024-01-15T03:30:00.000Z
```

#### Restore from Snapshot

```bash
# Restore from snapshot
aws rds restore-db-instance-from-db-snapshot \
  --db-instance-identifier inventory-db-recovered \
  --db-snapshot-identifier inventory-snapshot-20240115
```

### Application Recovery

#### ECS Service Recovery

```bash
# Update service with new task definition
aws ecs update-service \
  --cluster mycomputer-production \
  --service inventory-service \
  --task-definition inventory-service:stable \
  --force-new-deployment

# Scale up immediately
aws ecs update-service \
  --cluster mycomputer-production \
  --service inventory-service \
  --desired-count 5
```

## Failover Procedures

### Multi-Region Failover

#### Step 1: Health Check

```bash
# Check primary region health
./scripts/health-check.sh eu-west-1

# If unhealthy, proceed with failover
```

#### Step 2: DNS Failover

```bash
# Update Route53 weighted routing
aws route53 change-resource-record-sets \
  --hosted-zone-id Z123456789 \
  --change-batch '{
    "Changes": [{
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "api.mycomputer.com",
        "Type": "A",
        "SetIdentifier": "Primary",
        "Weight": 0,
        "AliasTarget": {
          "HostedZoneId": "Z215JYRZR8TBV5",
          "DNSName": "primary-alb.eu-west-1.elb.amazonaws.com",
          "EvaluateTargetHealth": true
        }
      }
    },
    {
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "api.mycomputer.com",
        "Type": "A",
        "SetIdentifier": "Secondary",
        "Weight": 100,
        "AliasTarget": {
          "HostedZoneId": "Z3F0SRJ5LGBH90",
          "DNSName": "secondary-alb.eu-central-1.elb.amazonaws.com",
          "EvaluateTargetHealth": true
        }
      }
    }]
  }'
```

#### Step 3: Verify Failover

```bash
# Test endpoints
curl -I https://api.mycomputer.com/health
dig api.mycomputer.com

# Monitor CloudWatch metrics
aws cloudwatch get-metric-statistics \
  --namespace AWS/Route53 \
  --metric-name HealthCheckStatus \
  --dimensions Name=HealthCheckId,Value=abc123 \
  --start-time 2024-01-15T00:00:00Z \
  --end-time 2024-01-15T01:00:00Z \
  --period 300 \
  --statistics Average
```

## Data Recovery Procedures

### Corrupted Data Recovery

```bash
# Identify corruption time
aws rds describe-db-log-files \
  --db-instance-identifier mycomputer-inventory-db \
  --filename-contains error

# Restore to point before corruption
aws rds restore-db-instance-to-point-in-time \
  --source-db-instance-identifier mycomputer-inventory-db \
  --target-db-instance-identifier inventory-db-clean \
  --restore-time 2024-01-14T23:00:00.000Z

# Verify data integrity
psql -h inventory-db-clean.region.rds.amazonaws.com \
  -U admin -d inventory \
  -c "SELECT COUNT(*) FROM inventory_items;"
```

### Lost Transaction Recovery

```sql
-- Connect to recovered database
psql -h inventory-db-recovered.region.rds.amazonaws.com -U admin -d inventory

-- Check audit logs for lost transactions
SELECT * FROM audit_log 
WHERE created_at > '2024-01-15 03:00:00'
ORDER BY created_at;

-- Replay critical transactions
BEGIN;
INSERT INTO inventory_transactions 
SELECT * FROM backup.inventory_transactions 
WHERE transaction_date > '2024-01-15 03:00:00';
COMMIT;
```

## Disaster Recovery Testing

### Monthly DR Drill

```bash
#!/bin/bash
# dr-drill.sh

echo "Starting DR Drill at $(date)"

# 1. Create test environment
aws cloudformation create-stack \
  --stack-name dr-test-$(date +%Y%m%d) \
  --template-body file://dr-test-environment.yaml

# 2. Restore databases
aws rds restore-db-instance-from-db-snapshot \
  --db-instance-identifier test-inventory-db \
  --db-snapshot-identifier latest-inventory-snapshot

# 3. Deploy applications
aws ecs create-service \
  --cluster dr-test \
  --service-name inventory-test \
  --task-definition inventory-service:latest

# 4. Run validation tests
npm run test:dr

# 5. Generate report
./generate-dr-report.sh > dr-report-$(date +%Y%m%d).txt

# 6. Clean up
aws cloudformation delete-stack --stack-name dr-test-$(date +%Y%m%d)
```

## Recovery Time Objectives (RTO)

| Service | RTO | RPO | Priority |
|---------|-----|-----|----------|
| Customer Base | 15 min | 5 min | Critical |
| Payments | 15 min | 0 min | Critical |
| Inventory | 30 min | 15 min | High |
| Locations | 1 hour | 30 min | Medium |

## Incident Response Runbook

### Severity Levels

- **P1 (Critical)**: Complete service outage
- **P2 (High)**: Partial service degradation
- **P3 (Medium)**: Non-critical feature unavailable
- **P4 (Low)**: Minor issues

### Response Procedures

#### P1 Incident Response

1. **Detection** (0-5 minutes)
   ```bash
   # Automated alert triggers
   # PagerDuty notification sent
   ```

2. **Assessment** (5-15 minutes)
   ```bash
   # Check service health
   ./scripts/assess-incident.sh
   
   # Determine impact scope
   aws cloudwatch get-metric-data \
     --metric-data-queries file://impact-queries.json
   ```

3. **Communication** (15-20 minutes)
   - Update status page
   - Notify stakeholders
   - Create incident channel

4. **Recovery** (20+ minutes)
   - Execute failover if needed
   - Scale resources
   - Apply emergency patches

5. **Verification**
   ```bash
   # Run smoke tests
   npm run test:smoke
   
   # Monitor metrics
   ./scripts/monitor-recovery.sh
   ```

## Backup Validation

### Weekly Backup Tests

```bash
#!/bin/bash
# validate-backups.sh

# Test database restore
aws rds restore-db-instance-from-db-snapshot \
  --db-instance-identifier validation-test \
  --db-snapshot-identifier $(aws rds describe-db-snapshots \
    --query 'DBSnapshots[0].DBSnapshotIdentifier' \
    --output text)

# Wait for restoration
aws rds wait db-instance-available \
  --db-instance-identifier validation-test

# Validate data
psql -h validation-test.region.rds.amazonaws.com \
  -U admin -d inventory \
  -c "SELECT COUNT(*) FROM inventory_items;" > validation-result.txt

# Clean up
aws rds delete-db-instance \
  --db-instance-identifier validation-test \
  --skip-final-snapshot
```

## Documentation and Contacts

### Emergency Contacts

- On-call Engineer: Via PagerDuty
- Infrastructure Lead: +44 7XXX XXXXXX
- Database Admin: +44 7XXX XXXXXX
- Security Team: security@mycomputer.com

### Key Documentation

- [AWS Infrastructure Reference](../reference/aws-infrastructure.md)
- [Monitoring Setup](./setup-monitoring.md)
- [Scaling Configuration](./configure-scaling.md)

## Post-Incident Review

After any disaster recovery event:

1. Document timeline of events
2. Identify root cause
3. Calculate actual RTO/RPO
4. Update procedures based on lessons learned
5. Schedule follow-up DR drill
