# GenAI Intelligent Document Processing (GenAIIDP) - Complete Setup Guide

## Table of Contents

1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [AWS Deployment (Production)](#aws-deployment-production)
   - [Quick Deploy using CloudFormation](#quick-deploy-using-cloudformation)
   - [Deploy from Source Code](#deploy-from-source-code)
4. [Local Development Setup](#local-development-setup)
   - [Setting Up Development Environment](#setting-up-development-environment)
   - [Running Lambda Functions Locally](#running-lambda-functions-locally)
   - [Running the Web UI Locally](#running-the-web-ui-locally)
5. [Using the Python SDK Locally](#using-the-python-sdk-locally)
   - [Jupyter Notebook Setup](#jupyter-notebook-setup)
   - [Running Document Processing Pipeline](#running-document-processing-pipeline)
6. [Configuration and Customization](#configuration-and-customization)
7. [Testing and Validation](#testing-and-validation)
8. [Troubleshooting](#troubleshooting)
9. [Best Practices](#best-practices)

## Overview

GenAIIDP can be deployed in two primary ways:
1. **AWS Deployment**: Full production deployment on AWS infrastructure
2. **Local Development**: Run components locally for development and testing

## Prerequisites

### System Requirements

- **Operating System**: Linux, macOS, or Windows with WSL2
- **Python**: 3.11 or later
- **Docker**: Required for local Lambda testing
- **Node.js**: 18.x or later (for UI development)
- **Memory**: Minimum 8GB RAM recommended
- **Storage**: At least 10GB free space

### AWS Requirements

- **AWS Account**: With appropriate permissions
- **AWS CLI**: Configured with credentials
- **Bedrock Model Access**: Request access to required models:
  - Amazon Nova models (all)
  - Anthropic Claude 3.x and 4.x models
  - Amazon Titan Text Embeddings V2

### Required Software Installation

```bash
# Install AWS CLI
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Install AWS SAM CLI
wget https://github.com/aws/aws-sam-cli/releases/latest/download/aws-sam-cli-linux-x86_64.zip
unzip aws-sam-cli-linux-x86_64.zip -d sam-installation
sudo ./sam-installation/install

# Install Python 3.11+
sudo apt update
sudo apt install python3.11 python3.11-venv python3-pip

# Install Node.js 18.x
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt-get install -y nodejs

# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker $USER
```

## AWS Deployment (Production)

### Quick Deploy using CloudFormation

This is the fastest way to deploy GenAIIDP to AWS.

#### Step 1: Request Bedrock Model Access

1. Log into AWS Console
2. Navigate to Amazon Bedrock
3. Go to Model access
4. Request access to:
   - Amazon Nova (all models)
   - Anthropic Claude 3.x and 4.x
   - Amazon Titan Text Embeddings V2

#### Step 2: Deploy Stack

**Option A: One-Click Deploy**

For US East (N. Virginia):
[![Launch Stack](https://cdn.rawgit.com/buildkite/cloudformation-launch-stack-button-svg/master/launch-stack.svg)](https://us-east-1.console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/create/review?templateURL=https://s3.us-east-1.amazonaws.com/aws-ml-blog-us-east-1/artifacts/genai-idp/idp-main.yaml&stackName=IDP)

For US West (Oregon):
[![Launch Stack](https://cdn.rawgit.com/buildkite/cloudformation-launch-stack-button-svg/master/launch-stack.svg)](https://us-west-2.console.aws.amazon.com/cloudformation/home?region=us-west-2#/stacks/create/review?templateURL=https://s3.us-west-2.amazonaws.com/aws-ml-blog-us-west-2/artifacts/genai-idp/idp-main.yaml&stackName=IDP)

**Option B: Manual Deploy**

```bash
# Deploy using AWS CLI
aws cloudformation create-stack \
  --stack-name GenAIIDP \
  --template-url https://s3.us-east-1.amazonaws.com/aws-ml-blog-us-east-1/artifacts/genai-idp/idp-main.yaml \
  --parameters \
    ParameterKey=AdminEmail,ParameterValue=your-email@example.com \
    ParameterKey=IDPPattern,ParameterValue="Pattern2 - Packet processing with Textract and Bedrock" \
  --capabilities CAPABILITY_NAMED_IAM CAPABILITY_AUTO_EXPAND \
  --region us-east-1
```

#### Step 3: Access the Application

1. Check email for temporary password (first-time deployment)
2. Go to CloudFormation Console → Your Stack → Outputs tab
3. Open `ApplicationWebURL` link
4. Login with admin email and temporary password
5. Set new permanent password

### Deploy from Source Code

For customization or development, build and deploy from source.

#### Step 1: Clone Repository

```bash
# Clone the repository
git clone <repository-url> genai-idp
cd genai-idp
```

#### Step 2: Build and Publish

```bash
# Make publish script executable
chmod +x publish.sh

# Run publish script
# Usage: ./publish.sh <bucket-prefix> <stack-prefix> <region> [public]
./publish.sh my-idp-bucket idp us-east-1

# Example output:
# Template URL: https://s3.us-east-1.amazonaws.com/my-idp-bucket-us-east-1/idp/idp-main.yaml
# 1-Click Launch URL: https://console.aws.amazon.com/cloudformation/...
```

#### Step 3: Deploy Stack

Use the 1-Click Launch URL from the publish output or deploy via CLI:

```bash
aws cloudformation deploy \
  --template-url <template-url-from-publish> \
  --stack-name GenAIIDP-Dev \
  --parameter-overrides \
    AdminEmail=your-email@example.com \
    IDPPattern="Pattern2 - Packet processing with Textract and Bedrock" \
  --capabilities CAPABILITY_NAMED_IAM CAPABILITY_AUTO_EXPAND
```

## Local Development Setup

### Setting Up Development Environment

#### Step 1: Create Python Virtual Environment

```bash
cd genai-idp
python3.11 -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate
```

#### Step 2: Install IDP Common Package

```bash
# Install the core library with all features
cd lib/idp_common_pkg
pip install -e ".[all]"

# Or install specific components
pip install -e ".[ocr,classification,extraction]"

# Install development dependencies
pip install pytest ruff jupyter ipykernel
```

#### Step 3: Configure AWS Credentials

```bash
# Configure AWS CLI
aws configure

# Or set environment variables
export AWS_ACCESS_KEY_ID=your-access-key
export AWS_SECRET_ACCESS_KEY=your-secret-key
export AWS_DEFAULT_REGION=us-east-1
```

### Running Lambda Functions Locally

#### Step 1: Build Lambda Functions

```bash
# Navigate to pattern directory
cd patterns/pattern-2

# Build all functions
sam build
```

#### Step 2: Create Test Environment

Create `testing/env.json`:
```json
{
  "OCRFunction": {
    "OUTPUT_BUCKET": "test-output-bucket",
    "AWS_DEFAULT_REGION": "us-east-1"
  },
  "ClassificationFunction": {
    "OUTPUT_BUCKET": "test-output-bucket",
    "CONFIGURATION_BUCKET": "test-config-bucket",
    "AWS_DEFAULT_REGION": "us-east-1"
  }
}
```

#### Step 3: Test Individual Functions

```bash
# Test OCR function
sam local invoke OCRFunction \
  -e testing/OCRFunction-event.json \
  --env-vars testing/env.json

# Test Classification function
sam local invoke ClassificationFunction \
  -e testing/ClassificationFunctionEvent.json \
  --env-vars testing/env.json

# Test Extraction function
sam local invoke ExtractionFunction \
  -e testing/ExtractionFunction-event.json \
  --env-vars testing/env.json
```

### Running the Web UI Locally

#### Step 1: Get Configuration from Deployed Stack

If you have a deployed stack, get the environment configuration:

```bash
# Get the WebUITestEnvFile output from CloudFormation
aws cloudformation describe-stacks \
  --stack-name GenAIIDP \
  --query 'Stacks[0].Outputs[?OutputKey==`WebUITestEnvFile`].OutputValue' \
  --output text > src/ui/.env
```

#### Step 2: Set Up UI Development Environment

```bash
cd src/ui

# Install dependencies
npm install

# Create .env file if not copied from stack
cat > .env << EOF
REACT_APP_USER_POOL_ID=your-user-pool-id
REACT_APP_USER_POOL_CLIENT_ID=your-client-id
REACT_APP_IDENTITY_POOL_ID=your-identity-pool-id
REACT_APP_APPSYNC_GRAPHQL_URL=your-graphql-url
REACT_APP_AWS_REGION=us-east-1
REACT_APP_SETTINGS_PARAMETER=/idp/settings
EOF
```

#### Step 3: Start Development Server

```bash
# Start the UI development server
npm start

# The UI will be available at http://localhost:3000
```

## Using the Python SDK Locally

### Jupyter Notebook Setup

#### Step 1: Install Jupyter

```bash
pip install jupyter ipykernel
python -m ipykernel install --user --name=genai-idp --display-name="GenAI IDP"
```

#### Step 2: Start Jupyter

```bash
cd notebooks/examples
jupyter notebook
```

### Running Document Processing Pipeline

#### Step 1: Setup and Configuration

Create a configuration file `config/main.yaml`:

```yaml
# AWS Configuration
aws:
  region: us-east-1
  s3:
    bucket: your-test-bucket
    prefix: idp-testing

# Processing Configuration
processing:
  pattern: pattern2
  ocr:
    backend: textract  # or 'bedrock'
    features:
      - TABLES
      - FORMS
  
  classification:
    method: multimodalPageLevelClassification
    model: "anthropic.claude-3-5-sonnet-20241022-v2:0"
  
  extraction:
    model: "anthropic.claude-3-5-sonnet-20241022-v2:0"
    temperature: 0
```

#### Step 2: Run Processing Pipeline

```python
# Example notebook code
from pathlib import Path
from idp_common.models import Document
from idp_common.ocr.service import OcrService
from idp_common.classification.service import ClassificationService
from idp_common.extraction.service import ExtractionService

# Initialize document
doc_path = Path("../../samples/rvl_cdip_package.pdf")
document = Document(pages=[], input_key=str(doc_path))

# Step 1: OCR
ocr_service = OcrService()
document = ocr_service.process_document(document, str(doc_path))

# Step 2: Classification
classification_service = ClassificationService(
    backend='bedrock',
    model_id='anthropic.claude-3-5-sonnet-20241022-v2:0'
)
document = classification_service.process_document(document)

# Step 3: Extraction
extraction_service = ExtractionService(
    model_id='anthropic.claude-3-5-sonnet-20241022-v2:0'
)
document = extraction_service.process_document(document)

# View results
print(document.to_json())
```

#### Step 3: Use Example Notebooks

The repository includes comprehensive example notebooks:

```bash
# Navigate to examples
cd notebooks/examples

# Run the setup notebook first
jupyter notebook step0_setup.ipynb

# Then run subsequent steps:
# - step1_ocr.ipynb
# - step2_classification.ipynb
# - step3_extraction.ipynb
# - step4_assessment.ipynb
# - step5_summarization.ipynb
# - step6_evaluation.ipynb
```

## Configuration and Customization

### Document Classes Configuration

Create custom document classes in `config/classes.yaml`:

```yaml
classes:
  - name: invoice
    description: "Commercial invoice document"
    attributes:
      - name: invoice_number
        description: "Unique invoice identifier"
      - name: invoice_date
        description: "Date of invoice"
      - name: total_amount
        description: "Total amount due"
      
  - name: receipt
    description: "Purchase receipt"
    attributes:
      - name: merchant_name
        description: "Name of the merchant"
      - name: transaction_date
        description: "Date of purchase"
      - name: total
        description: "Total amount paid"
```

### Model Configuration

Configure AI models in respective configuration files:

```yaml
# config/extraction.yaml
extraction:
  model: "anthropic.claude-3-5-sonnet-20241022-v2:0"
  temperature: 0
  system_prompt: |
    You are an expert at extracting structured data from documents.
    Extract only the requested attributes. Be precise and accurate.
  task_prompt: |
    Extract the following attributes from this {DOCUMENT_CLASS} document:
    {ATTRIBUTE_NAMES_AND_DESCRIPTIONS}
    
    Output as JSON.
```

## Testing and Validation

### Unit Tests

```bash
# Run all tests
make test

# Run specific test module
pytest lib/idp_common_pkg/tests/unit/test_ocr_service.py -v

# Run with coverage
pytest --cov=idp_common --cov-report=html
```

### Integration Tests

```bash
# Test with real AWS services
export AWS_PROFILE=test-profile
pytest lib/idp_common_pkg/tests/integration/ -v
```

### Load Testing

```bash
# Test with steady load
python scripts/simulate_load.py \
  -s source-bucket \
  -k samples/test.pdf \
  -d input-bucket \
  -r 100 \
  -t 5

# Test with variable load
python scripts/simulate_dynamic_load.py \
  -s source-bucket \
  -k samples/test.pdf \
  -d input-bucket \
  -f load_schedule.csv
```

## Troubleshooting

### Common Issues and Solutions

#### 1. AWS Credentials Issues

```bash
# Verify credentials
aws sts get-caller-identity

# Check credential configuration
aws configure list
```

#### 2. Bedrock Model Access

```bash
# List available models
aws bedrock list-foundation-models --region us-east-1

# If models not available, request access in AWS Console
```

#### 3. Docker Issues (Local Lambda Testing)

```bash
# Verify Docker is running
docker ps

# Reset Docker if needed
sudo systemctl restart docker
```

#### 4. Python Package Issues

```bash
# Clean install
pip uninstall idp-common
cd lib/idp_common_pkg
pip install -e ".[all]" --force-reinstall
```

#### 5. UI Development Issues

```bash
# Clear npm cache
npm cache clean --force
rm -rf node_modules package-lock.json
npm install
```

### Debug Mode

Enable detailed logging:

```python
import logging
logging.basicConfig(level=logging.DEBUG)

# For specific modules
logging.getLogger('idp_common.ocr').setLevel(logging.DEBUG)
```

## Best Practices

### Development Workflow

1. **Branch Strategy**
   ```bash
   git checkout -b feature/your-feature
   # Make changes
   make lint
   make test
   git commit -m "feat: add new feature"
   ```

2. **Code Quality**
   ```bash
   # Run linting and formatting
   make lint
   
   # Run tests before commit
   make test
   ```

3. **Environment Management**
   - Use separate AWS accounts for dev/test/prod
   - Use different S3 buckets for each environment
   - Tag resources appropriately

### Security Best Practices

1. **Credentials Management**
   - Never commit AWS credentials
   - Use IAM roles when possible
   - Rotate access keys regularly

2. **Resource Access**
   - Follow least privilege principle
   - Use resource-based policies
   - Enable CloudTrail logging

3. **Data Protection**
   - Encrypt S3 buckets
   - Use HTTPS for all communications
   - Implement data retention policies

### Performance Optimization

1. **Local Development**
   - Use caching for Bedrock responses
   - Process documents in batches
   - Optimize image sizes before processing

2. **AWS Deployment**
   - Monitor CloudWatch metrics
   - Adjust Lambda memory/timeout
   - Use appropriate concurrency limits

### Cost Management

1. **Development**
   - Use smaller models for testing
   - Process sample documents first
   - Clean up test resources

2. **Production**
   - Monitor costs with AWS Cost Explorer
   - Set up billing alerts
   - Optimize model selection for document types

## Next Steps

1. **Explore Example Notebooks**: Start with the step-by-step examples in `notebooks/examples/`
2. **Customize Configuration**: Modify document classes and attributes for your use case
3. **Test with Your Documents**: Upload sample documents and validate results
4. **Monitor Performance**: Use CloudWatch dashboards to track processing metrics
5. **Scale Gradually**: Start with small batches and increase volume gradually

For additional help:
- Check the [Troubleshooting Guide](docs/troubleshooting.md)
- Review [Architecture Documentation](docs/architecture.md)
- Consult [API Documentation](docs/api-reference.md)