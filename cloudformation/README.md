# RDS Dev Database Restoration Automation

## Overview

This CloudFormation template automates the restoration of a development RDS PostgreSQL database from production snapshots. It creates a scheduled Lambda function that runs every Sunday at 12:00 AM UTC to restore the dev database from the latest production snapshot.

## Features

- **Automated Snapshot Selection**: Automatically finds and uses the latest snapshot from the production database
- **Scheduled Execution**: Runs every Sunday at 12:00 AM UTC via EventBridge
- **Smart Database Management**: Deletes existing dev database before restoration (if it exists)
- **Network Configuration Preservation**: Maintains the same VPC, security groups, and subnet configuration as production
- **Instance Type Flexibility**: Configurable dev instance type (default: `db.t4g.large`)
- **Comprehensive Logging**: CloudWatch logs for monitoring and troubleshooting
- **Error Alerting**: CloudWatch alarm for Lambda failures
- **Tag-based Tracking**: Automatic tagging for cost tracking and resource management

## Architecture

```
┌─────────────────┐
│   EventBridge   │  (Cron: Sunday 12 AM UTC)
│      Rule       │
└────────┬────────┘
         │ Triggers
         ▼
┌─────────────────┐
│     Lambda      │  (Python 3.11)
│    Function     │
└────────┬────────┘
         │
         ├──► Get latest snapshot from prod DB
         │
         ├──► Check if dev DB exists
         │
         ├──► Delete dev DB (if exists)
         │
         └──► Restore from snapshot with dev config
```

## Prerequisites

Before deploying this stack, ensure the following resources exist:

1. **Production Database**: The source RDS PostgreSQL database (`prod-orbitshift-m6g-2xl`) must exist
2. **VPC Configuration**: The production database's VPC, security groups, and DB subnet group must be properly configured
3. **IAM Permissions**: You need permissions to create IAM roles, Lambda functions, and EventBridge rules
4. **RDS Snapshots**: At least one snapshot (automated or manual) of the production database must exist

## Deployment Instructions

### Option 1: AWS Console

1. Navigate to AWS CloudFormation console
2. Click **Create Stack** → **With new resources**
3. Choose **Upload a template file** and select `rds-dev-restore-automation.yaml`
4. Click **Next**
5. Fill in the stack details:
   - **Stack name**: e.g., `rds-dev-restore-automation`
   - **Parameters** (optional - defaults are pre-configured):
     - `SourceDBIdentifier`: Production database identifier (default: `prod-orbitshift-m6g-2xl`)
     - `TargetDBIdentifier`: Dev database identifier (default: `dev-orbitshift-postgres`)
     - `TargetDBInstanceClass`: Dev instance type (default: `db.t4g.large`)
     - `ScheduleExpression`: Cron schedule (default: `cron(0 0 ? * SUN *)`)
     - `LambdaTimeout`: Lambda timeout in seconds (default: 900 = 15 minutes)
6. Click **Next**
7. Configure stack options (tags, permissions, etc.) as needed
8. Click **Next**
9. Review and check **I acknowledge that AWS CloudFormation might create IAM resources**
10. Click **Create stack**

### Option 2: AWS CLI

```bash
aws cloudformation create-stack \
  --stack-name rds-dev-restore-automation \
  --template-body file://rds-dev-restore-automation.yaml \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameters \
    ParameterKey=SourceDBIdentifier,ParameterValue=prod-orbitshift-m6g-2xl \
    ParameterKey=TargetDBIdentifier,ParameterValue=dev-orbitshift-postgres \
    ParameterKey=TargetDBInstanceClass,ParameterValue=db.t4g.large \
  --region us-east-1
```

### Option 3: AWS CLI with Parameter File

Create a `parameters.json` file:

```json
[
  {
    "ParameterKey": "SourceDBIdentifier",
    "ParameterValue": "prod-orbitshift-m6g-2xl"
  },
  {
    "ParameterKey": "TargetDBIdentifier",
    "ParameterValue": "dev-orbitshift-postgres"
  },
  {
    "ParameterKey": "TargetDBInstanceClass",
    "ParameterValue": "db.t4g.large"
  }
]
```

Then deploy:

```bash
aws cloudformation create-stack \
  --stack-name rds-dev-restore-automation \
  --template-body file://rds-dev-restore-automation.yaml \
  --parameters file://parameters.json \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-east-1
```

## How It Works

### Workflow

1. **Trigger**: EventBridge rule triggers Lambda function every Sunday at 12:00 AM UTC
2. **Configuration Retrieval**: Lambda fetches VPC, security group, and subnet configuration from production database
3. **Snapshot Discovery**: Lambda queries for the latest available snapshot (automated or manual) from the production database
4. **Database Cleanup**: If dev database exists, Lambda deletes it and waits for deletion to complete
5. **Restoration**: Lambda restores the dev database from the latest snapshot with:
   - Specified instance class (`db.t4g.large` by default)
   - Same network configuration as production
   - Automatic tagging for tracking
6. **Logging**: All operations are logged to CloudWatch Logs for audit and troubleshooting

### Lambda Function Logic

The Lambda function performs these steps:

```python
1. Get production database configuration (VPC, security groups, subnet group)
2. Find latest snapshot (checks both automated and manual snapshots)
3. Check if target dev database exists
4. If exists, delete it and wait for deletion (uses RDS waiter)
5. Restore database from snapshot with dev configuration
6. Apply tags for tracking (Environment, RestoredFrom, RestoredAt)
```

### Snapshot Selection Priority

- Checks both **automated** and **manual** snapshots
- Filters only **available** snapshots (ignoring creating, copying, etc.)
- Selects the **most recent** snapshot by creation time
- Raises error if no snapshots are found

## Manual Testing

After deployment, you can manually test the Lambda function:

### Using AWS Console

1. Navigate to Lambda console
2. Find the function (named like `rds-dev-restore-automation-rds-restore`)
3. Click **Test** tab
4. Create a new test event with empty JSON: `{}`
5. Click **Test** to invoke the function
6. Monitor CloudWatch Logs for execution details

### Using AWS CLI

```bash
# Get the Lambda function name from stack outputs
FUNCTION_NAME=$(aws cloudformation describe-stacks \
  --stack-name rds-dev-restore-automation \
  --query 'Stacks[0].Outputs[?OutputKey==`LambdaFunctionName`].OutputValue' \
  --output text)

# Invoke the function
aws lambda invoke \
  --function-name $FUNCTION_NAME \
  --payload '{}' \
  response.json

# View the response
cat response.json
```

## Monitoring and Troubleshooting

### CloudWatch Logs

Lambda execution logs are available in CloudWatch Logs:

- **Log Group**: `/aws/lambda/<stack-name>-rds-restore`
- **Retention**: 30 days

To view logs:

```bash
# Get log group name
LOG_GROUP=$(aws cloudformation describe-stacks \
  --stack-name rds-dev-restore-automation \
  --query 'Stacks[0].Outputs[?OutputKey==`LogGroupName`].OutputValue' \
  --output text)

# View recent logs
aws logs tail $LOG_GROUP --follow
```

### CloudWatch Alarms

A CloudWatch alarm monitors Lambda function errors:

- **Alarm Name**: `<stack-name>-lambda-errors`
- **Condition**: Triggers when Lambda has ≥1 error in 5 minutes
- **Action**: None by default (can be configured to send SNS notifications)

### Common Issues and Solutions

#### Issue: "No snapshots found"

**Cause**: No available snapshots exist for the production database

**Solution**: 
- Verify production database identifier is correct
- Check if snapshots exist: `aws rds describe-db-snapshots --db-instance-identifier prod-orbitshift-m6g-2xl`
- Ensure at least one snapshot has `Status: available`

#### Issue: Lambda timeout

**Cause**: Database deletion takes longer than Lambda timeout

**Solution**:
- Increase `LambdaTimeout` parameter (max 900 seconds = 15 minutes)
- Update stack with higher timeout value

#### Issue: Permission denied errors

**Cause**: IAM role lacks necessary permissions

**Solution**:
- Review CloudWatch logs for specific permission errors
- Verify IAM role has all required RDS and EC2 permissions
- Check resource-based policies on RDS instances

#### Issue: VPC configuration mismatch

**Cause**: Target database can't be created in production's VPC configuration

**Solution**:
- Verify DB subnet group exists and is valid
- Check security groups allow necessary traffic
- Ensure subnet group has subnets in multiple availability zones

## Important Notes

### Data Refresh Cycle

- **Schedule**: Database refreshes every Sunday at 12:00 AM UTC
- **Duration**: Restoration typically takes 10-30 minutes depending on database size
- **Data Age**: Dev database will contain production data from the latest snapshot (usually within 24 hours)

### Database Credentials

- **Same as Production**: Restored database uses the same master username and password as the production snapshot
- **Security**: Ensure dev environment has appropriate access controls
- **Rotation**: If production credentials are rotated, dev database credentials are updated on next restore

### Cost Considerations

- **RDS Instance**: Running `db.t4g.large` instance costs approximately $0.073/hour (~$53/month)
- **Storage**: Charged for allocated storage (same size as production snapshot)
- **Lambda**: Minimal cost (free tier covers most usage)
- **Snapshots**: Automated backups of dev database can be disabled to save costs
- **Recommendation**: Consider stopping dev database when not in use (manual action required)

### Network and Security

- **Network Isolation**: Dev database uses same VPC and security groups as production
- **Access Control**: Review security group rules to ensure appropriate access restrictions
- **Encryption**: Encryption settings are inherited from the snapshot

### Manual Trigger Option

To manually trigger a database refresh at any time:

```bash
# Get Lambda function name
FUNCTION_NAME=$(aws cloudformation describe-stacks \
  --stack-name rds-dev-restore-automation \
  --query 'Stacks[0].Outputs[?OutputKey==`LambdaFunctionName`].OutputValue' \
  --output text)

# Invoke Lambda function
aws lambda invoke \
  --function-name $FUNCTION_NAME \
  --payload '{}' \
  response.json

echo "Check response.json for results"
```

### Customization Options

#### Change Schedule

Update the `ScheduleExpression` parameter:

- **Daily at 2 AM UTC**: `cron(0 2 * * ? *)`
- **Every 6 hours**: `rate(6 hours)`
- **First day of month**: `cron(0 0 1 * ? *)`

Reference: [EventBridge Schedule Expressions](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-create-rule-schedule.html)

#### Change Instance Type

Update the `TargetDBInstanceClass` parameter to any supported instance type:

```bash
aws cloudformation update-stack \
  --stack-name rds-dev-restore-automation \
  --use-previous-template \
  --parameters \
    ParameterKey=TargetDBInstanceClass,ParameterValue=db.t4g.medium
  --capabilities CAPABILITY_NAMED_IAM
```

## Cleanup

To remove all resources created by this stack:

```bash
# Delete the CloudFormation stack
aws cloudformation delete-stack --stack-name rds-dev-restore-automation

# Wait for deletion to complete
aws cloudformation wait stack-delete-complete --stack-name rds-dev-restore-automation

# Manually delete the dev database if needed
aws rds delete-db-instance \
  --db-instance-identifier dev-orbitshift-postgres \
  --skip-final-snapshot
```

**Note**: The dev database is NOT automatically deleted when the stack is removed. You must delete it manually if desired.

## Support and Contribution

For issues, questions, or contributions, please refer to the repository's main documentation.

## License

This template is provided as-is for use in your AWS environment. Ensure compliance with your organization's policies and AWS best practices.

## Additional Resources

- [AWS RDS Documentation](https://docs.aws.amazon.com/rds/)
- [AWS Lambda Documentation](https://docs.aws.amazon.com/lambda/)
- [AWS EventBridge Documentation](https://docs.aws.amazon.com/eventbridge/)
- [RDS Snapshot Best Practices](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_WorkingWithAutomatedBackups.html)
