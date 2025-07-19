# Live Meeting Assistant (LMA) - Complete Setup Guide

This guide covers installation and setup for both local development and AWS deployment of the Live Meeting Assistant application.

## Table of Contents
- [Prerequisites](#prerequisites)
- [AWS Deployment](#aws-deployment)
- [Local Development Setup](#local-development-setup)
- [Post-Deployment Configuration](#post-deployment-configuration)
- [Troubleshooting](#troubleshooting)

## Prerequisites

### System Requirements

#### For Development (Local/Build Server)
- **Operating System**: Linux, macOS, or Windows with WSL
- **Hardware**: Minimum 8GB RAM, 20GB free disk space
- **Node.js**: v18.x (required)
- **Python**: 3.8+ with pip3 and virtualenv
- **Docker**: Latest version (must be running)
- **AWS CLI**: Latest version
- **AWS SAM CLI**: v1.118.0 or higher
- **Additional tools**: git, zip, npm, bash

#### For AWS Deployment
- **AWS Account**: With appropriate IAM permissions
- **AWS Services Access**:
  - Amazon Bedrock models enabled (Claude 3.x, Titan Text Embeddings V2)
  - Service quotas for concurrent Transcribe streams (default: 25)
- **Budget**: ~$110/month for base infrastructure + usage costs

### Installing Prerequisites

#### 1. Install Node.js 18.x
```bash
# Using Node Version Manager (nvm)
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash
source ~/.bashrc
nvm install 18
nvm use 18
```

#### 2. Install Python Dependencies
```bash
# Install Python 3 and pip
sudo apt-get update
sudo apt-get install python3 python3-pip

# Install virtualenv
pip3 install virtualenv
```

#### 3. Install Docker
```bash
# For Ubuntu/Debian
sudo apt-get install docker.io
sudo systemctl start docker
sudo usermod -aG docker $USER
# Log out and back in for group changes to take effect
```

#### 4. Install AWS CLI
```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
aws configure  # Enter your AWS credentials
```

#### 5. Install SAM CLI
```bash
# Download and install SAM CLI
wget https://github.com/aws/aws-sam-cli/releases/latest/download/aws-sam-cli-linux-x86_64.zip
unzip aws-sam-cli-linux-x86_64.zip -d sam-installation
sudo ./sam-installation/install
```

#### 6. Install Additional Tools
```bash
sudo apt-get install zip git make
```

## AWS Deployment

### Option 1: Quick Deploy Using Pre-built Templates

The easiest way to deploy LMA is using the pre-built CloudFormation templates:

1. **Enable Bedrock Models** (Required before deployment):
   - Log into AWS Console
   - Navigate to Amazon Bedrock
   - Go to Model access
   - Request access to:
     - Anthropic Claude 3.x models (Haiku, Sonnet)
     - Amazon Titan Text Embeddings V2

2. **Deploy the Stack**:
   - Choose your region and click the appropriate Launch Stack button:

   | Region | Launch Link |
   |--------|-------------|
   | US East (N. Virginia) | [Launch Stack](https://us-east-1.console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/create/review?templateURL=https://s3.us-east-1.amazonaws.com/aws-ml-blog-us-east-1/artifacts/lma/lma-main.yaml&stackName=LMA) |
   | US West (Oregon) | [Launch Stack](https://us-west-2.console.aws.amazon.com/cloudformation/home?region=us-west-2#/stacks/create/review?templateURL=https://s3.us-west-2.amazonaws.com/aws-ml-blog-us-west-2/artifacts/lma/lma-main.yaml&stackName=LMA) |

3. **Configure Stack Parameters**:
   ```
   Stack name: LMA
   Admin Email: your-email@example.com
   Authorized Account Email Domain: example.com
   Meeting Assist Service: Choose one:
     - BEDROCK_LLM (simplest, no knowledge base)
     - BEDROCK_KNOWLEDGE_BASE (Create) (recommended)
     - Q_BUSINESS (Use Existing) (if you have Q Business)
   
   # Leave other parameters as defaults unless you have specific requirements
   ```

4. **Deploy**: 
   - Check acknowledgment boxes
   - Click "Create stack"
   - Wait 35-40 minutes for deployment

### Option 2: Build and Deploy from Source

#### 1. Clone the Repository
```bash
git clone https://github.com/aws-samples/amazon-transcribe-live-meeting-assistant.git
cd amazon-transcribe-live-meeting-assistant
```

#### 2. Build and Publish to Your S3 Bucket
```bash
# Create deployment artifacts
./publish.sh <your-bucket-name> <prefix> <region>
# Example:
./publish.sh my-lma-artifacts lma us-east-1
```

This will:
- Build all components
- Create S3 bucket if needed
- Upload artifacts
- Display CloudFormation template URLs

#### 3. Deploy Using Your Template
```bash
# The publish script will output a CF Launch URL like:
# https://us-east-1.console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/create/review?templateURL=...

# Click the URL and follow the same parameter configuration as Option 1
```

## Local Development Setup

### 1. Deploy LMA to AWS First
You need a deployed LMA stack to get the necessary configuration values for local development.

### 2. Set Up the Web UI

#### Get Configuration Values
1. Go to AWS CloudFormation Console
2. Find your LMA stack
3. Navigate to the nested AISTACK
4. Go to Outputs tab
5. Find `LocalUITestingEnv` - it contains all needed values

#### Configure Local Environment
```bash
cd lma-ai-stack/source/ui/

# Create .env file with values from LocalUITestingEnv
cat > .env << EOF
REACT_APP_USER_POOL_ID=us-west-2_XXXXXXXXX
REACT_APP_USER_POOL_CLIENT_ID=XXXXXXXXXXXXXXXXXXXXXXXXXX
REACT_APP_IDENTITY_POOL_ID=us-west-2:XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX
REACT_APP_APPSYNC_GRAPHQL_URL=https://XXXXXXXXXX.appsync-api.us-west-2.amazonaws.com/graphql
REACT_APP_AWS_REGION=us-west-2
REACT_APP_SETTINGS_PARAMETER=LMA-LMASettingsParameter-XXXXXXXXXXXX
REACT_APP_ENABLE_LEX_AGENT_ASSIST=true
EOF

# Install dependencies
npm install

# Start development server
npm run start
```

The UI will be available at http://localhost:3000

### 3. Set Up the WebSocket Server (Optional)

```bash
cd lma-websocket-transcriber-stack/source/app/

# Install dependencies
npm run setup

# Build check
npm run buildcheck

# For actual testing, you'll need to configure environment variables
# pointing to your AWS resources (KDS, S3, etc.)
```

### 4. Set Up the Browser Extension

```bash
cd lma-browser-extension-stack/

# Install dependencies
npm install

# Build the extension
npm run build

# The built extension will be in the 'build' directory
# Load it as an unpacked extension in Chrome
```

## Post-Deployment Configuration

### 1. Set Admin Password
1. Check your email for "Welcome to Live Meeting Assistant!"
2. Log in with temporary password
3. Set a permanent password (8+ chars, upper/lower/numbers/special)

### 2. Install Browser Extension
1. In LMA UI, click "Download Chrome Extension"
2. Extract the downloaded `lma-chrome-extension.zip`
3. Open Chrome and navigate to `chrome://extensions`
4. Enable Developer mode
5. Click "Load unpacked" and select the extracted folder
6. Pin the extension to your toolbar

### 3. Configure Knowledge Base (if using)
If you selected BEDROCK_KNOWLEDGE_BASE (Create):
1. Add documents to the S3 bucket specified during setup
2. Wait for automatic sync (or trigger manually in Bedrock console)

### 4. Test the Installation
1. Open a supported meeting platform in Chrome
2. Click the LMA extension icon
3. Log in with your credentials
4. Start a meeting and click "Start Listening"
5. Open in LMA to see live transcription

## Configuration Files

### Environment Variables for Local Development

#### UI Environment (.env)
```bash
REACT_APP_USER_POOL_ID=<from-stack-output>
REACT_APP_USER_POOL_CLIENT_ID=<from-stack-output>
REACT_APP_IDENTITY_POOL_ID=<from-stack-output>
REACT_APP_APPSYNC_GRAPHQL_URL=<from-stack-output>
REACT_APP_AWS_REGION=<your-region>
REACT_APP_SETTINGS_PARAMETER=<from-stack-output>
REACT_APP_ENABLE_LEX_AGENT_ASSIST=true
```

#### WebSocket Server Environment
```bash
AWS_REGION=<your-region>
RECORDINGS_BUCKET_NAME=<your-bucket>
RECORDING_FILE_PREFIX=lma-audio-recordings/
CPU_HEALTH_THRESHOLD=50
LOCAL_TEMP_DIR=/tmp/
WS_LOG_LEVEL=debug
SHOULD_RECORD_CALL=true
```

## Common Deployment Scenarios

### 1. Development Environment
```bash
# Use minimal resources
Meeting Assist Service: BEDROCK_LLM
Meeting Assist QnABot OpenSearch Node Count: 1
Enable AppSync Api Cache: false
```

### 2. Production Environment
```bash
# Use robust configuration
Meeting Assist Service: BEDROCK_KNOWLEDGE_BASE (Create)
Meeting Assist QnABot OpenSearch Node Count: 4
Enable AppSync Api Cache: true
AppSync Api Cache Instance Type: LARGE
```

### 3. Multi-User Organization
```bash
Admin Email: admin@company.com
Authorized Account Email Domain: company.com
# This allows all @company.com users to create accounts
```

## Troubleshooting

### Common Issues

#### 1. CloudFormation Deployment Fails
- Check the Events tab for the first CREATE_FAILED message
- If it's in a nested stack, navigate to that stack
- Common causes:
  - Bedrock models not enabled
  - Service quota limits exceeded
  - IAM permission issues

#### 2. Transcription Not Working
- Verify Transcribe service limits (25 concurrent streams default)
- Check WebSocket server logs in CloudWatch
- Ensure browser extension has microphone permissions

#### 3. Meeting Assistant Not Responding
- Verify Bedrock model access is enabled
- Check Lambda function logs for errors
- Ensure knowledge base is synced (if using)

#### 4. Local Development Issues
- Verify all environment variables are set correctly
- Check AWS credentials are configured
- Ensure you're using Node.js v18.x

### Debug Commands

```bash
# Check stack status
aws cloudformation describe-stacks --stack-name LMA

# View Lambda logs
aws logs tail /aws/lambda/AISTACK-CallEventProcessor --follow

# Check ECS task logs
aws logs tail /ecs/LMA-WEBSOCKET --follow
```

## Security Considerations

1. **Network Security**:
   - Use VPC endpoints for AWS services
   - Enable VPC Flow Logs
   - Restrict security groups to necessary ports

2. **Data Security**:
   - Enable S3 bucket encryption
   - Use KMS for sensitive data
   - Enable CloudTrail logging

3. **Access Control**:
   - Use least privilege IAM policies
   - Enable MFA for admin accounts
   - Regularly rotate credentials

## Cost Optimization

1. **Reduce Base Costs**:
   - Use 1 OpenSearch node for dev/test
   - Disable AppSync cache when not needed
   - Set shorter retention periods

2. **Monitor Usage**:
   - Set up billing alerts
   - Use Cost Explorer to track spending
   - Review CloudWatch metrics regularly

## Next Steps

1. **Customize the Solution**:
   - Add custom vocabulary for your domain
   - Create custom summary prompts
   - Integrate with your knowledge base

2. **Extend Functionality**:
   - Add webhook integrations
   - Implement custom Lambda hooks
   - Create domain-specific features

3. **Production Readiness**:
   - Set up monitoring and alerting
   - Implement backup strategies
   - Create runbooks for operations

For more information, refer to the [main README](README.md) and [Developer README](README_DEVELOPERS.md).