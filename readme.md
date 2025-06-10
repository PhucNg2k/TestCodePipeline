# AWS CloudFormation CI/CD Pipeline

This repository contains an Infrastructure as Code (IaC) solution using AWS CloudFormation with a complete CI/CD pipeline. The pipeline automatically validates and deploys infrastructure changes when templates are updated.

## Architecture

### Infrastructure Components (Nested Stack)
- VPC with public and private subnets
- Network components (Internet Gateway, NAT Gateway, Route Tables)
- Security Groups for EC2 instances
- EC2 instances in public and private subnets

### CI/CD Pipeline Components
- CodeCommit: Source control
- CodeBuild: Template validation (cfn-lint + taskcat)
- CloudFormation: Infrastructure deployment

## Prerequisites

1. AWS CLI installed and configured
2. An AWS account with appropriate permissions
3. Git installed locally
4. An EC2 key pair in your AWS account

## Setup Instructions

### 1. Create CodeCommit Repository

```bash
# Create the repository
aws codecommit create-repository --repository-name infrastructure-repo --repository-description "Infrastructure as Code Repository"

# Note down the repository URL from the output
```

### 2. Configure Git Credentials for CodeCommit

```bash
# Generate HTTPS Git credentials in AWS Console:
# IAM -> Users -> Your User -> Security Credentials -> HTTPS Git credentials for AWS CodeCommit
```

### 3. Clone and Set Up Repository

```bash
# Clone the repository
git clone <CODECOMMIT_REPO_URL>
cd infrastructure-repo

# Copy your infrastructure files to the repository
mkdir Task2
cp -r /path/to/your/templates/* Task2/

# Initial commit
git add .
git commit -m "Initial commit with infrastructure templates"
git push
```

### 4. Deploy the Pipeline

```bash
# Get your current IP address for the AllowedIP parameter
curl https://checkip.amazonaws.com

# Deploy the pipeline stack
aws cloudformation create-stack \
  --stack-name infrastructure-pipeline \
  --template-body file://Task2/pipeline-stack.yaml \
  --parameters \
    ParameterKey=AllowedIP,ParameterValue=$(curl -s https://checkip.amazonaws.com)/32 \
    ParameterKey=KeyName,ParameterValue=your-key-pair-name \
    ParameterKey=RepositoryName,ParameterValue=infrastructure-repo \
    ParameterKey=BranchName,ParameterValue=main \
  --capabilities CAPABILITY_IAM

# Monitor the stack creation
aws cloudformation describe-stacks --stack-name infrastructure-pipeline
```

## Pipeline Workflow

1. **Source Stage**:
   - Monitors changes in the `Task2` folder of the CodeCommit repository
   - Triggers pipeline when changes are pushed

2. **Validation Stage**:
   - Runs `cfn-lint` on all templates
   - Executes `taskcat` tests
   - Fails the pipeline if validation errors are found

3. **Deploy Stage**:
   - Deploys the nested stack using CloudFormation
   - Creates/updates all infrastructure components

## Making Changes

1. Clone the repository if you haven't already:
```bash
git clone <CODECOMMIT_REPO_URL>
cd infrastructure-repo
```

2. Make changes to templates in the `Task2` folder:
   - `root-stack.yaml`: Main template
   - `modules/*.yaml`: Individual component templates

3. Commit and push changes:
```bash
git add .
git commit -m "Description of changes"
git push
```

4. The pipeline will automatically:
   - Detect the changes
   - Validate the templates
   - Deploy the updates if validation passes

## Monitoring

### Pipeline Status
```bash
# Get pipeline execution status
aws codepipeline get-pipeline-state --name infrastructure-pipeline

# Get CloudFormation stack status
aws cloudformation describe-stacks --stack-name infrastructure-stack
```

### Logs and Troubleshooting
- Pipeline execution details: AWS Console -> CodePipeline
- Build logs: AWS Console -> CodeBuild
- Deployment logs: AWS Console -> CloudFormation

## Cleanup

To delete all resources:

1. Delete the infrastructure stack:
```bash
aws cloudformation delete-stack --stack-name infrastructure-stack
```

2. Delete the pipeline stack:
```bash
aws cloudformation delete-stack --stack-name infrastructure-pipeline
```

3. Delete the CodeCommit repository:
```bash
aws codecommit delete-repository --repository-name infrastructure-repo
```

## Security Notes

- The pipeline uses IAM roles with necessary permissions
- EC2 instances are secured with security groups
- SSH access is restricted to your IP address
- Private instances are only accessible through the public instance

## Template Structure

```
Task2/
├── pipeline-stack.yaml     # CI/CD pipeline definition
├── root-stack.yaml        # Main infrastructure template
├── .taskcat.yml          # Taskcat configuration
└── modules/
    ├── vpc-module.yaml
    ├── network-module.yaml
    ├── security-module.yaml
    └── compute-module.yaml
``` 