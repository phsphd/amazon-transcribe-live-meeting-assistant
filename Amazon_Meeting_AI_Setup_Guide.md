# Amazon Meeting AI (LMA) - Enterprise Setup & Deployment Guide

## Table of Contents
- [Prerequisites](#prerequisites)
- [AWS Deployment Options](#aws-deployment-options)
- [Enterprise Deployment](#enterprise-deployment)
- [Local Development Setup](#local-development-setup)
- [Production Configuration](#production-configuration)
- [Security Hardening](#security-hardening)
- [High Availability Setup](#high-availability-setup)
- [Monitoring & Operations](#monitoring-operations)
- [Troubleshooting](#troubleshooting)
- [Migration Guide](#migration-guide)

## Prerequisites

### System Requirements

#### For AWS Deployment
- **AWS Account**: Enterprise account with appropriate IAM permissions
- **AWS Services Required**:
  - Amazon Bedrock (Claude 3.x, Titan Embeddings enabled)
  - Amazon Transcribe (concurrent stream quota: 25+)
  - VPC with private subnets (for enterprise deployment)
  - AWS Organizations (for multi-account setup)
- **Compliance Requirements**:
  - SOC2 compliant AWS account setup
  - CloudTrail enabled
  - AWS Config enabled
  - GuardDuty active

#### For Development
- **Operating System**: Linux, macOS, or Windows with WSL2
- **Hardware**: 16GB RAM, 50GB free disk space (recommended)
- **Software Requirements**:
  ```bash
  Node.js: v18.x (required, use nvm)
  Python: 3.8+ with pip3 and virtualenv
  Docker: 20.10+ (must be running)
  AWS CLI: 2.x
  AWS SAM CLI: 1.118.0+
  Terraform: 1.5+ (for enterprise deployments)
  kubectl: 1.28+ (for EKS deployments)
  ```

### Installing Prerequisites

#### Complete Installation Script
```bash
#!/bin/bash
# Save as install-prerequisites.sh

# Update system
sudo apt-get update && sudo apt-get upgrade -y

# Install basic tools
sudo apt-get install -y curl wget git zip unzip make jq

# Install Node.js 18.x via nvm
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
nvm install 18
nvm use 18
nvm alias default 18

# Install Python and dependencies
sudo apt-get install -y python3 python3-pip python3-venv
pip3 install --upgrade pip
pip3 install virtualenv

# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker $USER
sudo systemctl enable docker
sudo systemctl start docker

# Install AWS CLI v2
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
rm -rf aws awscliv2.zip

# Install SAM CLI
wget https://github.com/aws/aws-sam-cli/releases/latest/download/aws-sam-cli-linux-x86_64.zip
unzip aws-sam-cli-linux-x86_64.zip -d sam-installation
sudo ./sam-installation/install
rm -rf sam-installation aws-sam-cli-linux-x86_64.zip

# Install Terraform (optional, for enterprise)
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install terraform

# Verify installations
echo "Verifying installations..."
node --version
python3 --version
docker --version
aws --version
sam --version
terraform --version

echo "Installation complete! Please log out and back in for Docker permissions to take effect."
```

### AWS Account Preparation

#### 1. Enable Required Services
```bash
# Enable Bedrock models
aws bedrock put-model-invocation-logging-configuration \
  --logging-config '{"cloudWatchConfig":{"logGroupName":"/aws/bedrock/modelinvocations","roleArn":"arn:aws:iam::ACCOUNT:role/BedrockLoggingRole"}}'

# Request model access
aws bedrock create-model-access-request \
  --model-id anthropic.claude-3-haiku-20240307-v1:0
aws bedrock create-model-access-request \
  --model-id anthropic.claude-3-sonnet-20240229-v1:0
aws bedrock create-model-access-request \
  --model-id amazon.titan-embed-text-v2:0

# Increase Transcribe limits
aws service-quotas request-service-quota-increase \
  --service-code transcribe \
  --quota-code L-0F21857C \
  --desired-value 100
```

#### 2. Create Deployment IAM Role
```bash
# Create deployment policy
cat > lma-deployment-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "cloudformation:*",
        "s3:*",
        "iam:*",
        "lambda:*",
        "dynamodb:*",
        "kinesis:*",
        "cognito-idp:*",
        "appsync:*",
        "ecs:*",
        "ec2:*",
        "elasticloadbalancing:*",
        "cloudfront:*",
        "route53:*",
        "events:*",
        "logs:*",
        "sns:*",
        "sqs:*",
        "bedrock:*",
        "transcribe:*",
        "translate:*"
      ],
      "Resource": "*"
    }
  ]
}
EOF

# Create role
aws iam create-role --role-name LMADeploymentRole \
  --assume-role-policy-document file://trust-policy.json
aws iam put-role-policy --role-name LMADeploymentRole \
  --policy-name LMADeploymentPolicy \
  --policy-document file://lma-deployment-policy.json
```

## AWS Deployment Options

### Option 1: Quick Deploy (Development/Testing)

Use pre-built CloudFormation templates for fastest deployment:

```bash
# Deploy using AWS CLI
aws cloudformation create-stack \
  --stack-name LMA \
  --template-url https://s3.us-east-1.amazonaws.com/aws-ml-blog-us-east-1/artifacts/lma/lma-main.yaml \
  --parameters \
    ParameterKey=AdminEmail,ParameterValue=admin@company.com \
    ParameterKey=AllowedSignUpEmailDomain,ParameterValue=company.com \
    ParameterKey=MeetingAssistService,ParameterValue="BEDROCK_KNOWLEDGE_BASE (Create)" \
  --capabilities CAPABILITY_IAM CAPABILITY_NAMED_IAM CAPABILITY_AUTO_EXPAND
```

### Option 2: Custom Build & Deploy

#### 1. Clone and Prepare Repository
```bash
git clone https://github.com/aws-samples/amazon-transcribe-live-meeting-assistant.git
cd amazon-transcribe-live-meeting-assistant

# Checkout specific version for stability
git checkout v0.2.11
```

#### 2. Configure Build Environment
```bash
# Create build configuration
cat > build-config.sh << EOF
export ARTIFACT_BUCKET="my-lma-artifacts-$(date +%s)"
export ARTIFACT_PREFIX="lma/v0.2.11"
export AWS_REGION="us-east-1"
export STACK_NAME="LMA"
EOF

source build-config.sh
```

#### 3. Build and Publish
```bash
# Run build process
./publish.sh $ARTIFACT_BUCKET $ARTIFACT_PREFIX $AWS_REGION

# Deploy the stack
aws cloudformation create-stack \
  --stack-name $STACK_NAME \
  --template-url $(cat cfn-output-url.txt) \
  --parameters file://parameters.json \
  --capabilities CAPABILITY_IAM CAPABILITY_NAMED_IAM CAPABILITY_AUTO_EXPAND
```

### Option 3: Enterprise Deployment (Production)

#### 1. Multi-Account Architecture
```yaml
# organizations-setup.yaml
OrganizationStructure:
  ManagementAccount:
    - SecurityAccount:
        - LogArchive
        - Audit
    - ProductionAccount:
        - LMA-Prod
        - Shared-Services
    - DevelopmentAccount:
        - LMA-Dev
        - LMA-Test
```

#### 2. Network Architecture
```yaml
# vpc-architecture.yaml
VPCConfiguration:
  CIDR: 10.0.0.0/16
  PrivateSubnets:
    - 10.0.1.0/24  # AZ1 - Application
    - 10.0.2.0/24  # AZ2 - Application
    - 10.0.3.0/24  # AZ1 - Database
    - 10.0.4.0/24  # AZ2 - Database
  PublicSubnets:
    - 10.0.101.0/24  # AZ1 - ALB/NAT
    - 10.0.102.0/24  # AZ2 - ALB/NAT
  TransitGateway:
    EnableDNSSupport: true
    DefaultRouteTable: false
```

#### 3. Terraform Deployment
```hcl
# main.tf
module "lma_enterprise" {
  source = "./modules/lma-enterprise"
  
  environment = "production"
  region      = var.aws_region
  
  vpc_config = {
    vpc_id             = module.vpc.vpc_id
    private_subnet_ids = module.vpc.private_subnet_ids
    public_subnet_ids  = module.vpc.public_subnet_ids
  }
  
  security_config = {
    enable_guardduty       = true
    enable_security_hub    = true
    enable_config          = true
    enable_cloudtrail      = true
    enable_vpc_flow_logs   = true
  }
  
  high_availability = {
    multi_az               = true
    enable_auto_scaling    = true
    min_capacity          = 2
    max_capacity          = 10
    enable_read_replicas  = true
  }
  
  compliance = {
    enable_encryption_at_rest = true
    enable_backup            = true
    retention_days           = 90
    enable_pitr             = true
  }
}
```

## Enterprise Deployment

### 1. Infrastructure as Code Setup

#### CloudFormation Parameter File
```json
{
  "Parameters": {
    "AdminEmail": "admin@enterprise.com",
    "AllowedSignUpEmailDomain": "enterprise.com",
    "MeetingAssistService": "BEDROCK_KNOWLEDGE_BASE (Create)",
    "MeetingAssistQnABotOpenSearchNodeCount": "4",
    "UseExistingVPC": "true",
    "VPC": "vpc-xxxxxxxxxxxxx",
    "PublicSubnet1": "subnet-xxxxxxxxxxxxx",
    "PublicSubnet2": "subnet-xxxxxxxxxxxxx", 
    "PrivateSubnet1": "subnet-xxxxxxxxxxxxx",
    "PrivateSubnet2": "subnet-xxxxxxxxxxxxx",
    "EnableAppSyncApiCache": "true",
    "AppSyncApiCacheInstanceType": "LARGE",
    "CloudFrontPriceClass": "PriceClass_All",
    "CloudFrontAllowedGeos": "US,CA,GB,DE,FR,JP",
    "MeetingRecordExpirationInDays": "90",
    "TranscriptionExpirationInDays": "30",
    "CloudWatchLogsExpirationInDays": "90",
    "EnableAudioRecording": "true",
    "IsContentRedactionEnabled": "true",
    "TranscribePiiEntityTypes": "ALL",
    "BedrockGuardrailId": "xxxxxxxxxxxxx",
    "BedrockGuardrailVersion": "1"
  }
}
```

#### 2. Automated Deployment Script
```bash
#!/bin/bash
# deploy-enterprise.sh

set -e

# Configuration
ENVIRONMENT=${1:-production}
CONFIG_FILE="config/${ENVIRONMENT}.json"
STACK_NAME="LMA-${ENVIRONMENT}"

# Validate prerequisites
echo "Validating prerequisites..."
aws sts get-caller-identity || { echo "AWS CLI not configured"; exit 1; }

# Create S3 bucket for artifacts if not exists
ARTIFACT_BUCKET="lma-artifacts-$(aws sts get-caller-identity --query Account --output text)"
aws s3 mb s3://${ARTIFACT_BUCKET} --region ${AWS_REGION} 2>/dev/null || true

# Enable versioning
aws s3api put-bucket-versioning \
  --bucket ${ARTIFACT_BUCKET} \
  --versioning-configuration Status=Enabled

# Build and publish
echo "Building LMA artifacts..."
./publish.sh ${ARTIFACT_BUCKET} lma/${ENVIRONMENT} ${AWS_REGION}

# Deploy stack
echo "Deploying CloudFormation stack..."
aws cloudformation deploy \
  --template-url https://s3.${AWS_REGION}.amazonaws.com/${ARTIFACT_BUCKET}/lma/${ENVIRONMENT}/lma-main.yaml \
  --stack-name ${STACK_NAME} \
  --parameter-overrides file://${CONFIG_FILE} \
  --capabilities CAPABILITY_IAM CAPABILITY_NAMED_IAM CAPABILITY_AUTO_EXPAND \
  --tags Environment=${ENVIRONMENT} Project=LMA

# Wait for completion
aws cloudformation wait stack-create-complete --stack-name ${STACK_NAME}

# Output results
aws cloudformation describe-stacks \
  --stack-name ${STACK_NAME} \
  --query 'Stacks[0].Outputs'
```

### 3. High Availability Configuration

#### Multi-AZ Fargate Service
```yaml
# ecs-service-ha.yaml
FargateService:
  Type: AWS::ECS::Service
  Properties:
    Cluster: !Ref ECSCluster
    DesiredCount: 3
    LaunchType: FARGATE
    TaskDefinition: !Ref TaskDefinition
    NetworkConfiguration:
      AwsvpcConfiguration:
        Subnets:
          - !Ref PrivateSubnet1
          - !Ref PrivateSubnet2
          - !Ref PrivateSubnet3
        SecurityGroups:
          - !Ref FargateSecurityGroup
    LoadBalancers:
      - ContainerName: websocket-server
        ContainerPort: 8080
        TargetGroupArn: !Ref TargetGroup
    HealthCheckGracePeriodSeconds: 60
    DeploymentConfiguration:
      MinimumHealthyPercent: 100
      MaximumPercent: 200
      DeploymentCircuitBreaker:
        Enable: true
        Rollback: true
```

#### Auto Scaling Configuration
```yaml
AutoScalingTarget:
  Type: AWS::ApplicationAutoScaling::ScalableTarget
  Properties:
    MaxCapacity: 10
    MinCapacity: 3
    ResourceId: !Sub service/${ClusterName}/${ServiceName}
    RoleARN: !Sub arn:aws:iam::${AWS::AccountId}:role/aws-service-role/ecs.application-autoscaling.amazonaws.com/AWSServiceRoleForApplicationAutoScaling_ECSService
    ScalableDimension: ecs:service:DesiredCount
    ServiceNamespace: ecs

AutoScalingPolicy:
  Type: AWS::ApplicationAutoScaling::ScalingPolicy
  Properties:
    PolicyName: LMATargetTrackingScalingPolicy
    PolicyType: TargetTrackingScaling
    ScalingTargetId: !Ref AutoScalingTarget
    TargetTrackingScalingPolicyConfiguration:
      PredefinedMetricSpecification:
        PredefinedMetricType: ECSServiceAverageCPUUtilization
      TargetValue: 70.0
      ScaleInCooldown: 300
      ScaleOutCooldown: 60
```

## Production Configuration

### 1. Environment Variables

#### Application Configuration
```bash
# .env.production
# Core Settings
NODE_ENV=production
AWS_REGION=us-east-1
LOG_LEVEL=info

# Security
ENABLE_HTTPS=true
SECURE_COOKIES=true
SESSION_SECRET=$(openssl rand -base64 32)
JWT_SECRET=$(openssl rand -base64 32)
CORS_ORIGIN=https://meetings.company.com

# Performance
ENABLE_CACHE=true
CACHE_TTL=3600
CONNECTION_POOL_SIZE=20
MAX_CONCURRENT_TRANSCRIPTIONS=100

# Monitoring
ENABLE_XRAY=true
ENABLE_METRICS=true
METRICS_NAMESPACE=LMA/Production

# Feature Flags
ENABLE_VIDEO_PROCESSING=false
ENABLE_REAL_TIME_ANALYTICS=true
ENABLE_ADVANCED_SEARCH=true
```

### 2. Database Optimization

#### DynamoDB Configuration
```yaml
CallsTable:
  Type: AWS::DynamoDB::Table
  Properties:
    BillingMode: PAY_PER_REQUEST
    PointInTimeRecoverySpecification:
      PointInTimeRecoveryEnabled: true
    SSESpecification:
      SSEEnabled: true
      SSEType: KMS
      KMSMasterKeyId: !Ref DynamoDBKMSKey
    StreamSpecification:
      StreamViewType: NEW_AND_OLD_IMAGES
    GlobalSecondaryIndexes:
      - IndexName: CallOwnerIndex
        Keys:
          - AttributeName: CallOwner
            KeyType: HASH
          - AttributeName: CreatedAt
            KeyType: RANGE
        Projection:
          ProjectionType: ALL
      - IndexName: CallStatusIndex
        Keys:
          - AttributeName: Status
            KeyType: HASH
          - AttributeName: UpdatedAt
            KeyType: RANGE
        Projection:
          ProjectionType: KEYS_ONLY
    Tags:
      - Key: BackupPolicy
        Value: Daily
```

### 3. Caching Strategy

#### Redis Configuration
```yaml
RedisCluster:
  Type: AWS::ElastiCache::CacheCluster
  Properties:
    CacheNodeType: cache.r6g.large
    Engine: redis
    NumCacheNodes: 3
    AZMode: cross-az
    CacheSubnetGroupName: !Ref CacheSubnetGroup
    VpcSecurityGroupIds:
      - !Ref RedisSecurityGroup
    SnapshotRetentionLimit: 7
    SnapshotWindow: 03:00-05:00
    PreferredMaintenanceWindow: sun:05:00-sun:06:00
    NotificationTopicArn: !Ref OpsAlertTopic
```

## Security Hardening

### 1. Network Security

#### Security Groups
```yaml
# Least privilege security groups
WebSocketSecurityGroup:
  Type: AWS::EC2::SecurityGroup
  Properties:
    GroupDescription: WebSocket server security group
    VpcId: !Ref VPC
    SecurityGroupIngress:
      - IpProtocol: tcp
        FromPort: 443
        ToPort: 443
        SourceSecurityGroupId: !Ref ALBSecurityGroup
    SecurityGroupEgress:
      - IpProtocol: tcp
        FromPort: 443
        ToPort: 443
        DestinationPrefixListId: !Ref S3PrefixList
      - IpProtocol: tcp
        FromPort: 443
        ToPort: 443
        DestinationSecurityGroupId: !Ref VPCEndpointSecurityGroup
```

#### VPC Endpoints
```bash
# Create VPC endpoints for AWS services
for service in s3 dynamodb transcribe bedrock-runtime; do
  aws ec2 create-vpc-endpoint \
    --vpc-id vpc-xxxxx \
    --service-name com.amazonaws.${AWS_REGION}.${service} \
    --route-table-ids rtb-xxxxx \
    --subnet-ids subnet-xxxxx subnet-yyyyy
done
```

### 2. Data Encryption

#### KMS Configuration
```yaml
LMAKMSKey:
  Type: AWS::KMS::Key
  Properties:
    Description: LMA master encryption key
    KeyPolicy:
      Version: '2012-10-17'
      Statement:
        - Sid: Enable IAM User Permissions
          Effect: Allow
          Principal:
            AWS: !Sub 'arn:aws:iam::${AWS::AccountId}:root'
          Action: 'kms:*'
          Resource: '*'
        - Sid: Allow services
          Effect: Allow
          Principal:
            Service:
              - s3.amazonaws.com
              - dynamodb.amazonaws.com
              - logs.amazonaws.com
          Action:
            - 'kms:Decrypt'
            - 'kms:GenerateDataKey'
          Resource: '*'
```

### 3. Secrets Management

```bash
# Store sensitive configuration in Secrets Manager
aws secretsmanager create-secret \
  --name /lma/production/config \
  --description "LMA production configuration" \
  --secret-string '{
    "jwt_secret": "'$(openssl rand -base64 32)'",
    "session_secret": "'$(openssl rand -base64 32)'",
    "api_keys": {
      "internal": "'$(uuidgen)'",
      "external": "'$(uuidgen)'"
    }
  }'

# Lambda function to retrieve secrets
import boto3
import json

def get_secret(secret_name):
    client = boto3.client('secretsmanager')
    response = client.get_secret_value(SecretId=secret_name)
    return json.loads(response['SecretString'])
```

## High Availability Setup

### 1. Multi-Region Architecture

```yaml
# Cross-region replication
S3ReplicationRole:
  Type: AWS::IAM::Role
  Properties:
    AssumeRolePolicyDocument:
      Statement:
        - Effect: Allow
          Principal:
            Service: s3.amazonaws.com
          Action: sts:AssumeRole
    Policies:
      - PolicyName: S3ReplicationPolicy
        PolicyDocument:
          Statement:
            - Effect: Allow
              Action:
                - s3:GetReplicationConfiguration
                - s3:ListBucket
              Resource: !GetAtt RecordingsBucket.Arn
            - Effect: Allow
              Action:
                - s3:GetObjectVersionForReplication
                - s3:GetObjectVersionAcl
              Resource: !Sub '${RecordingsBucket.Arn}/*'
            - Effect: Allow
              Action:
                - s3:ReplicateObject
                - s3:ReplicateDelete
              Resource: !Sub 'arn:aws:s3:::${ReplicaBucket}/*'
```

### 2. Database Replication

```python
# DynamoDB global table setup
import boto3

def setup_global_table():
    dynamodb = boto3.client('dynamodb')
    
    # Create replica in secondary region
    response = dynamodb.create_table_replica(
        TableName='LMA-Calls',
        ReplicaRegion='us-west-2',
        ReplicaTags=[
            {
                'Key': 'Environment',
                'Value': 'Production'
            }
        ],
        TableClass='STANDARD',
        GlobalSecondaryIndexes=[
            {
                'IndexName': 'CallOwnerIndex',
                'ProvisionedThroughputOverride': {
                    'ReadCapacityUnits': 10,
                    'WriteCapacityUnits': 10
                }
            }
        ]
    )
    
    return response
```

### 3. Disaster Recovery

#### Backup Strategy
```yaml
BackupPlan:
  Type: AWS::Backup::BackupPlan
  Properties:
    BackupPlanTags:
      Environment: Production
      Application: LMA
    BackupPlan:
      BackupPlanName: LMA-Production-Backup
      Rules:
        - RuleName: DailyBackup
          TargetBackupVault: !Ref BackupVault
          ScheduleExpression: cron(0 5 ? * * *)
          StartWindowMinutes: 60
          CompletionWindowMinutes: 120
          Lifecycle:
            DeleteAfterDays: 30
            MoveToColdStorageAfterDays: 7
          RecoveryPointTags:
            Type: Daily
        - RuleName: WeeklyBackup  
          TargetBackupVault: !Ref BackupVault
          ScheduleExpression: cron(0 5 ? * SUN *)
          StartWindowMinutes: 60
          CompletionWindowMinutes: 180
          Lifecycle:
            DeleteAfterDays: 90
            MoveToColdStorageAfterDays: 30
          RecoveryPointTags:
            Type: Weekly
```

## Monitoring & Operations

### 1. CloudWatch Dashboard

```json
{
  "widgets": [
    {
      "type": "metric",
      "properties": {
        "metrics": [
          ["LMA", "WebSocketConnections", {"stat": "Sum"}],
          [".", "ActiveTranscriptions", {"stat": "Average"}],
          [".", "TranscriptionErrors", {"stat": "Sum"}],
          [".", "APILatency", {"stat": "Average"}],
          [".", "APIErrors", {"stat": "Sum"}]
        ],
        "period": 300,
        "stat": "Average",
        "region": "us-east-1",
        "title": "LMA Health Metrics"
      }
    },
    {
      "type": "metric",
      "properties": {
        "metrics": [
          ["AWS/Lambda", "Duration", {"FunctionName": "CallEventProcessor"}],
          [".", "Errors", {"FunctionName": "CallEventProcessor"}],
          [".", "ConcurrentExecutions", {"FunctionName": "CallEventProcessor"}]
        ],
        "period": 60,
        "stat": "Average",
        "region": "us-east-1",
        "title": "Lambda Performance"
      }
    }
  ]
}
```

### 2. Alerting Configuration

```yaml
HighErrorRateAlarm:
  Type: AWS::CloudWatch::Alarm
  Properties:
    AlarmName: LMA-HighErrorRate
    AlarmDescription: High error rate in LMA
    MetricName: APIErrors
    Namespace: LMA
    Statistic: Sum
    Period: 300
    EvaluationPeriods: 2
    Threshold: 100
    ComparisonOperator: GreaterThanThreshold
    AlarmActions:
      - !Ref AlertTopic
    TreatMissingData: notBreaching

WebSocketConnectionAlarm:
  Type: AWS::CloudWatch::Alarm
  Properties:
    AlarmName: LMA-WebSocketConnectionDrop
    AlarmDescription: Significant drop in WebSocket connections
    MetricName: WebSocketConnections
    Namespace: LMA
    Statistic: Average
    Period: 300
    EvaluationPeriods: 2
    Threshold: 10
    ComparisonOperator: LessThanThreshold
    AlarmActions:
      - !Ref AlertTopic
```

### 3. Log Aggregation

```python
# CloudWatch Insights query for error analysis
def analyze_errors():
    query = """
    fields @timestamp, @message, error.type, error.message
    | filter @message like /ERROR/
    | stats count() by error.type
    | sort count() desc
    | limit 20
    """
    
    client = boto3.client('logs')
    response = client.start_query(
        logGroupName='/aws/lambda/CallEventProcessor',
        startTime=int((datetime.now() - timedelta(hours=1)).timestamp()),
        endTime=int(datetime.now().timestamp()),
        queryString=query
    )
    
    return response
```

## Troubleshooting

### Common Issues and Solutions

#### 1. WebSocket Connection Failures
```bash
# Check WebSocket server health
aws ecs describe-services \
  --cluster LMA-Cluster \
  --services websocket-service \
  --query 'services[0].deployments'

# View recent logs
aws logs tail /ecs/lma-websocket --follow

# Test WebSocket connection
wscat -c wss://your-websocket-endpoint/ws \
  -H "Authorization: Bearer YOUR_TOKEN"
```

#### 2. Transcription Accuracy Issues
```python
# Update custom vocabulary
import boto3

transcribe = boto3.client('transcribe')

vocabulary_items = [
    "LMA",
    "Bedrock",
    "Claude",
    "Anthropic"
]

response = transcribe.create_vocabulary(
    VocabularyName='LMA-Custom-Vocabulary',
    LanguageCode='en-US',
    Phrases=vocabulary_items
)
```

#### 3. Performance Optimization
```bash
# Analyze Lambda performance
aws lambda get-function-concurrency \
  --function-name CallEventProcessor

# Update memory and timeout
aws lambda update-function-configuration \
  --function-name CallEventProcessor \
  --memory-size 1024 \
  --timeout 300
```

### Debug Mode

```bash
# Enable debug logging
aws ssm put-parameter \
  --name /lma/production/debug \
  --value "true" \
  --type String \
  --overwrite

# View debug logs
aws logs tail /aws/lambda/CallEventProcessor \
  --filter-pattern "[DEBUG]" \
  --follow
```

## Migration Guide

### Migrating from SaaS Solutions

#### 1. Data Export
```python
# Export meeting data from existing system
import requests
import json

def export_meetings_from_saas():
    meetings = []
    page = 1
    
    while True:
        response = requests.get(
            f"https://api.saasprovider.com/meetings?page={page}",
            headers={"Authorization": f"Bearer {API_KEY}"}
        )
        
        data = response.json()
        meetings.extend(data['meetings'])
        
        if not data['has_next']:
            break
            
        page += 1
    
    with open('meetings_export.json', 'w') as f:
        json.dump(meetings, f)
    
    return len(meetings)
```

#### 2. Data Import
```python
# Import meetings into LMA
import boto3
import json
from datetime import datetime

def import_meetings_to_lma():
    dynamodb = boto3.resource('dynamodb')
    table = dynamodb.Table('LMA-Calls')
    
    with open('meetings_export.json', 'r') as f:
        meetings = json.load(f)
    
    with table.batch_writer() as batch:
        for meeting in meetings:
            batch.put_item(Item={
                'CallId': meeting['id'],
                'CallOwner': meeting['owner_email'],
                'MeetingTopic': meeting['title'],
                'CreatedAt': meeting['start_time'],
                'UpdatedAt': datetime.now().isoformat(),
                'Status': 'ENDED',
                'Summary': meeting.get('summary', ''),
                'Participants': meeting.get('participants', [])
            })
    
    print(f"Imported {len(meetings)} meetings")
```

### Version Upgrade Process

```bash
#!/bin/bash
# upgrade-lma.sh

# Backup current configuration
aws cloudformation describe-stacks \
  --stack-name LMA \
  --query 'Stacks[0].Parameters' > backup-params.json

# Create change set
aws cloudformation create-change-set \
  --stack-name LMA \
  --change-set-name upgrade-$(date +%Y%m%d) \
  --template-url https://s3.amazonaws.com/lma-artifacts/latest/lma-main.yaml \
  --parameters file://backup-params.json \
  --capabilities CAPABILITY_IAM CAPABILITY_NAMED_IAM

# Review changes
aws cloudformation describe-change-set \
  --stack-name LMA \
  --change-set-name upgrade-$(date +%Y%m%d)

# Execute upgrade
read -p "Proceed with upgrade? (y/n) " -n 1 -r
if [[ $REPLY =~ ^[Yy]$ ]]; then
  aws cloudformation execute-change-set \
    --stack-name LMA \
    --change-set-name upgrade-$(date +%Y%m%d)
fi
```

## Best Practices

### 1. Security Best Practices
- Enable MFA for all admin users
- Use temporary credentials where possible
- Implement least privilege access
- Regular security audits
- Enable GuardDuty and Security Hub
- Use VPC endpoints for AWS services

### 2. Operational Best Practices
- Implement proper tagging strategy
- Use Infrastructure as Code
- Automate deployments with CI/CD
- Regular backup testing
- Document runbooks
- Implement proper monitoring

### 3. Cost Optimization
- Use Reserved Instances for stable workloads
- Implement auto-scaling
- Enable S3 lifecycle policies
- Use spot instances for batch processing
- Regular cost analysis
- Right-size resources

## Support Resources

### Documentation
- [Main README](README.md)
- [Developer Guide](README_DEVELOPERS.md)
- [API Documentation](API_DOCS.md)
- [Architecture Guide](ARCHITECTURE.md)

### Community
- GitHub Issues: https://github.com/aws-samples/amazon-transcribe-live-meeting-assistant/issues
- AWS Forums: https://forums.aws.amazon.com
- Stack Overflow: Tag with 'aws-lma'

### Professional Support
- AWS Professional Services
- AWS Partner Network
- Enterprise Support Plans

## Conclusion

This guide provides comprehensive instructions for deploying LMA in various environments. For production deployments, focus on security, high availability, and monitoring. Regular updates and maintenance ensure optimal performance and security.

For the latest updates and features, check the [CHANGELOG](CHANGELOG.md) and subscribe to release notifications on GitHub.