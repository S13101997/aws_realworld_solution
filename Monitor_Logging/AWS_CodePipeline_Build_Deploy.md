# AWS CodePipeline, CodeBuild, and CodeDeploy: Complete Guide

## Table of Contents
1. [Detailed Explanations](#detailed-explanations)
2. [Comprehensive Examples](#comprehensive-examples)
3. [Top 5 Interview Questions](#top-5-interview-questions)

---

## Detailed Explanations

### 1. AWS CodePipeline: Orchestration and Automation

#### Overview: The Pipeline Conductor

AWS CodePipeline is a fully managed continuous delivery service that automates the entire software release process. It orchestrates the workflow from source code through build, test, and deployment stages.

**Core Purpose:**
- **Workflow Automation**: Automate the complete software release process
- **Stage Management**: Define clear, sequential stages (Source → Build → Test → Deploy)
- **Action Execution**: Execute actions within each stage (CodeBuild, CodeDeploy, Lambda, etc.)
- **State Tracking**: Monitor pipeline execution and provide real-time feedback
- **Artifact Management**: Pass build artifacts between stages via S3
- **Integration Hub**: Connect with various AWS services and third-party tools

#### Architecture Components

```
Source Stage
    ↓ (Trigger on code changes)
Build Stage (CodeBuild)
    ↓ (Produces artifacts)
Test Stage (CodeBuild, Testing Services)
    ↓ (Validates artifacts)
Deploy Stage (CodeDeploy, CloudFormation, ECS, Lambda)
    ↓
Production Environment
```

#### Key Concepts

**Pipelines**
- Definition: Complete workflow from source to production
- Contains: Stages (minimum 2: Source + at least one action stage)
- Triggers: On-push, manual, scheduled
- Artifacts: S3-stored binaries passed between stages

**Stages**
- Logical grouping of actions
- Sequential execution (Stage 1 → Stage 2 → Stage 3)
- Cannot proceed to next stage until current stage succeeds
- Examples: Source, Build, Test, Production Deployment

**Actions**
- Individual tasks within a stage
- Can run in parallel within same stage
- Configured with specific providers (CodeBuild, CodeDeploy, Lambda, etc.)
- Requires input artifacts and produces output artifacts

**Transitions**
- How pipeline moves between stages
- Automatic: If stage succeeds, move to next
- Manual Approval: Requires human intervention (approval actions)
- Conditional: Based on outcomes or rules

#### Event-Driven Execution

**Automatic Triggers:**
- Code commit to repository (CodeCommit, GitHub, BitBucket, CodeStar)
- S3 object creation in source bucket
- Pipeline execution API call
- CloudWatch Events/EventBridge rules

**Manual Triggers:**
- Console execution start
- Pipeline API calls
- CodePipeline release change button

#### Artifact Flow

```
Source Artifact (Source Stage)
    ↓
Build Artifact (CodeBuild)
    ↓ S3 Bucket (Intermediate Storage)
    ↓
Test Artifact (Test Stage)
    ↓
Deployment Artifact (CodeDeploy)
    ↓
Running Application
```

**Artifact Management:**
- All artifacts stored in S3 (configured during pipeline creation)
- Default: Artifacts automatically cleaned after 30 days
- Can customize retention policies
- Artifacts include build outputs, test results, deployment packages

---

### 2. AWS CodeBuild: Build and Test Automation

#### Overview: Build Automation Engine

AWS CodeBuild is a fully managed continuous integration service that compiles source code, runs tests, and produces deployment-ready software packages without managing build infrastructure.

**Core Purpose:**
- **Build Automation**: Compile and package source code
- **Test Execution**: Run unit, integration, and security tests
- **Environment Management**: Provide pre-configured and custom build environments
- **Artifact Generation**: Create deployment packages (Docker images, JARs, ZIPs, etc.)
- **CI Integration**: Trigger builds on code changes automatically
- **Scaling**: Automatically scale build capacity based on demand

#### Build Process Flow

```
Source Code (CodeCommit, GitHub, S3, BitBucket)
    ↓
CodeBuild Project Configuration
    ↓ (Uses buildspec.yml)
Install Phase (Dependencies, tools)
    ↓
Pre-Build Phase (Authentication, setup)
    ↓
Build Phase (Compile, package)
    ↓
Post-Build Phase (Test, push to registry)
    ↓
Build Artifacts (S3, Docker Hub, ECR, etc.)
```

#### Build Specification (buildspec.yml)

The buildspec.yml file defines the build process in phases:

```yaml
version: 0.2

phases:
  install:
    # Install dependencies and tools
    commands:
      - echo "Installing dependencies..."
      - npm install
      - python -m pip install --upgrade pip
  
  pre_build:
    # Pre-build setup (authentication, environment setup)
    commands:
      - echo "Pre-build phase"
      - echo "Logging in to Docker Hub..."
      - echo $DOCKERHUB_PASSWORD | docker login -u $DOCKERHUB_USERNAME --password-stdin
  
  build:
    # Main build commands
    commands:
      - echo "Building application..."
      - npm run build
      - echo "Running tests..."
      - npm test
      - echo "Building Docker image..."
      - docker build -t myapp:latest .
  
  post_build:
    # Post-build actions (push to registry, generate reports)
    commands:
      - echo "Pushing Docker image..."
      - docker tag myapp:latest 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:latest
      - docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:latest
      - printf '[{"name":"myapp","imageUri":"%s"}]' 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:latest > imagedefinitions.json

artifacts:
  # Output artifacts stored in S3
  files:
    - imagedefinitions.json
    - build/**/*
    - dist/**/*
  name: BuildArtifact

cache:
  # Cache for faster builds
  paths:
    - '/root/.npm/**/*'
    - 'node_modules/**/*'
    - '.gradle/**/*'

env:
  # Environment variables (plain text and parameter store)
  variables:
    DOCKER_BUILDKIT: 1
    AWS_DEFAULT_REGION: us-east-1
  parameter-store:
    DOCKERHUB_USERNAME: /codebuild/dockerhub/username
    DOCKERHUB_PASSWORD: /codebuild/dockerhub/password
  secrets-manager:
    SLACK_WEBHOOK: slack/webhook:url

reports:
  # Test reports and coverage
  unittest:
    files:
      - 'test-results.xml'
    file-format: 'JUNITXML'
  coverage:
    files:
      - 'coverage/coverage.xml'
    file-format: 'COBERTURAXML'
```

#### Supported Build Environments

**Managed Images (Pre-configured):**
- Amazon Linux 2
- Ubuntu 20.04, 22.04
- Windows Server 2019, 2022

**Runtime Support:**
- Java (Maven, Gradle)
- Python (pip, pipenv, poetry)
- Node.js (npm, yarn)
- Ruby (bundler)
- Go
- .NET/C#
- PHP
- Docker

**Custom Build Environments:**
- Use custom Docker images
- Install custom tools and runtimes
- Pre-install frequently used dependencies
- Store in ECR for private usage

#### Build Cache

Purpose: Reduce build time by caching dependencies

**Cache Types:**
- S3 Cache: Store in S3 bucket
- Local Cache: Store in CodeBuild environment (faster)

**Example Cache Configuration:**
```yaml
cache:
  paths:
    - 'node_modules/**/*'
    - '.gradle/**/*'
    - '/root/.m2/**/*'
```

Benefits:
- 10-50% faster builds
- Reduced bandwidth usage
- Lower build costs

---

### 3. AWS CodeDeploy: Application Deployment Automation

#### Overview: Deployment Orchestration

AWS CodeDeploy is a fully managed deployment service that automates application deployment to various compute services including EC2 instances, Lambda functions, on-premises servers, and ECS.

**Core Purpose:**
- **Deployment Automation**: Automate software deployment process
- **Deployment Strategies**: Support multiple deployment patterns (in-place, blue-green, canary)
- **Health Monitoring**: Track deployment status and instance health
- **Rollback Capability**: Automatic or manual rollback on failure
- **Traffic Management**: Gradual traffic shifting strategies
- **Multi-Environment Support**: Deploy across different compute services

#### Deployment Process Flow

```
Deployment Configuration (CodeDeploy)
    ↓
Application Revision (Source code + appspec.yaml)
    ↓ (Downloaded to target instances)
CodeDeploy Agent (Runs on EC2/on-premises)
    ↓
Application Stop (BeforeBlockTraffic phase)
    ↓
Install Application (ApplicationStart phase)
    ↓
Start Application Services (ApplicationStart phase)
    ↓
Validation Tests (ValidateService phase)
    ↓
Deployment Complete
```

#### Application Specification (appspec.yaml)

The appspec.yaml file defines how to deploy the application:

```yaml
version: 0.0

# OS Type: windows or linux
os: linux

# Define source files and their destination
files:
  - source: /
    destination: /opt/myapp

# Permissions for deployed files
permissions:
  - object: /opt/myapp
    pattern: "**"
    owner: ec2-user
    group: ec2-user
    mode: 755
    type:
      - directory
  - object: /opt/myapp
    pattern: "**"
    owner: ec2-user
    group: ec2-user
    mode: 644
    type:
      - file

# Lifecycle event hooks
hooks:
  ApplicationStop:
    - location: scripts/stop_application.sh
      timeout: 300
      runas: root
  
  BeforeInstall:
    - location: scripts/install_dependencies.sh
      timeout: 300
      runas: root
  
  ApplicationStart:
    - location: scripts/start_application.sh
      timeout: 300
      runas: root
  
  ValidateService:
    - location: scripts/validate_service.sh
      timeout: 300
      runas: root

# Environment variables passed to hooks
Resources:
  - TargetInstances:
      TagFilters:
        - Key: Environment
          Value: Production
          Type: KEY_AND_VALUE
```

#### Deployment Strategies

**1. In-Place Deployment (Rolling Update)**
- Update instances while application is running
- Instances taken offline one at a time
- Application remains available (reduced capacity)
- Fastest rollback (rollback to previous version)
- Risk: Temporary capacity reduction

```
Instance 1: v1 → v2
Instance 2: v1 → v2
Instance 3: v1 → v2

↓
All instances running v2 (with downtime during deployment)
```

**2. Blue-Green Deployment**
- Maintain two identical environments (Blue and Green)
- Deploy to inactive environment (Green)
- Test thoroughly before switching traffic
- Instant rollback by switching back to Blue
- No downtime, higher resource cost

```
Blue Environment (Active)       Green Environment (Standby)
v1 running                      ← Deploy v2
v1 running                      v2 testing
v1 running                      v2 ready
↓
Traffic switch
↓
v1 standby                      v2 running
v1 standby                      v2 running
v1 standby                      v2 running
```

**3. Canary Deployment**
- Gradually shift traffic to new version
- Small percentage (e.g., 10%) initially
- Monitor metrics and errors
- Gradual increase if healthy
- Quick rollback if issues detected

```
Time 0:    90% v1, 10% v2
Time 5m:   75% v1, 25% v2
Time 10m:  50% v1, 50% v2
Time 15m:  25% v1, 75% v2
Time 20m:  0% v1, 100% v2
```

**4. Linear Deployment**
- Shift traffic in equal increments
- Deploy to fixed percentage every interval
- Less risk than canary, faster than in-place
- Good balance between speed and risk

```
Time 0:    75% v1, 25% v2
Time 10m:  50% v1, 50% v2
Time 20m:  25% v1, 75% v2
Time 30m:  0% v1, 100% v2
```

#### CodeDeploy Components

**CodeDeploy Agent**
- Installed on EC2/on-premises servers
- Monitors for deployment commands
- Downloads and executes appspec.yaml hooks
- Reports deployment status
- Automatically installed on EC2 using SSM

**Deployment Groups**
- Collection of instances targeted for deployment
- Define by EC2 tags, Auto Scaling groups, or both
- Configure health checks and rollback behavior
- Associate triggers and notifications

**Application**
- Logical grouping of deployment groups
- Container for versioning and history
- Each application can have multiple environments

**Revisions**
- Specific version of application
- Contains source code and appspec.yaml
- Stored in S3 or GitHub
- Each revision has unique identifier

---

## Comprehensive Examples

### Example 1: Complete CI/CD Pipeline for Node.js Application

**Scenario**: Deploy Node.js Express application with Docker to EC2

#### Step 1: Create CodePipeline

```python
import boto3
import json

class CI_CDPipeline:
    def __init__(self):
        self.codepipeline = boto3.client('codepipeline')
        self.codebuild = boto3.client('codebuild')
        self.codedeploy = boto3.client('codedeploy')
        self.iam = boto3.client('iam')
        self.s3 = boto3.client('s3')
    
    def create_iam_role_for_pipeline(self):
        """Create IAM role for CodePipeline"""
        
        assume_role_policy = {
            'Version': '2012-10-17',
            'Statement': [{
                'Effect': 'Allow',
                'Principal': {'Service': 'codepipeline.amazonaws.com'},
                'Action': 'sts:AssumeRole'
            }]
        }
        
        response = self.iam.create_role(
            RoleName='codepipeline-role',
            AssumeRolePolicyDocument=json.dumps(assume_role_policy)
        )
        
        # Attach policy
        pipeline_policy = {
            'Version': '2012-10-17',
            'Statement': [
                {
                    'Effect': 'Allow',
                    'Action': [
                        's3:GetObject',
                        's3:PutObject',
                        's3:GetObjectVersion'
                    ],
                    'Resource': 'arn:aws:s3:::my-pipeline-bucket/*'
                },
                {
                    'Effect': 'Allow',
                    'Action': [
                        'codebuild:BatchGetBuilds',
                        'codebuild:BatchGetProjects',
                        'codebuild:StartBuild'
                    ],
                    'Resource': '*'
                },
                {
                    'Effect': 'Allow',
                    'Action': [
                        'codedeploy:CreateDeployment',
                        'codedeploy:GetApplication',
                        'codedeploy:GetApplicationRevision',
                        'codedeploy:GetDeployment',
                        'codedeploy:GetDeploymentConfig'
                    ],
                    'Resource': '*'
                }
            ]
        }
        
        self.iam.put_role_policy(
            RoleName='codepipeline-role',
            PolicyName='codepipeline-policy',
            PolicyDocument=json.dumps(pipeline_policy)
        )
        
        return response['Role']['Arn']
    
    def create_codebuild_project(self):
        """Create CodeBuild project for building Docker image"""
        
        buildspec = """
version: 0.2

phases:
  install:
    commands:
      - echo "Installing dependencies..."
      - npm install
  
  pre_build:
    commands:
      - echo "Pre-build phase..."
      - echo "Logging in to ECR..."
      - aws ecr get-login-password --region $AWS_DEFAULT_REGION | docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com
      - REPOSITORY_URI=$AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/nodejs-app
      - COMMIT_HASH=$(echo $CODEBUILD_RESOLVED_SOURCE_VERSION | cut -c 1-7)
      - IMAGE_TAG=${COMMIT_HASH:=latest}
  
  build:
    commands:
      - echo "Building application..."
      - npm run build
      - echo "Running tests..."
      - npm test
      - echo "Building Docker image on `date`"
      - docker build -t $REPOSITORY_URI:latest .
      - docker tag $REPOSITORY_URI:latest $REPOSITORY_URI:$IMAGE_TAG
  
  post_build:
    commands:
      - echo "Pushing Docker image to ECR..."
      - docker push $REPOSITORY_URI:latest
      - docker push $REPOSITORY_URI:$IMAGE_TAG
      - echo "Writing image definitions file..."
      - printf '[{"name":"nodejs-app","imageUri":"%s"}]' $REPOSITORY_URI:$IMAGE_TAG > imagedefinitions.json

artifacts:
  files:
    - imagedefinitions.json
  name: BuildArtifact

cache:
  paths:
    - '/root/.npm/**/*'
    - 'node_modules/**/*'
"""
        
        response = self.codebuild.create_project(
            name='nodejs-build-project',
            source={
                'type': 'GITHUB',
                'location': 'https://github.com/user/nodejs-app.git'
            },
            artifacts={
                'type': 'CODEPIPELINE'
            },
            environment={
                'type': 'LINUX_CONTAINER',
                'computeType': 'BUILD_GENERAL1_SMALL',
                'image': 'aws/codebuild/amazonlinux2-x86_64-standard:5.0',
                'environmentVariables': [
                    {
                        'name': 'AWS_DEFAULT_REGION',
                        'value': 'us-east-1',
                        'type': 'PLAINTEXT'
                    },
                    {
                        'name': 'AWS_ACCOUNT_ID',
                        'value': '123456789012',
                        'type': 'PLAINTEXT'
                    },
                    {
                        'name': 'IMAGE_REPO_NAME',
                        'value': 'nodejs-app',
                        'type': 'PLAINTEXT'
                    }
                ],
                'privilegedMode': True
            },
            serviceRole='arn:aws:iam::123456789012:role/codebuild-role',
            logsConfig={
                'cloudWatchLogs': {
                    'status': 'ENABLED',
                    'groupName': '/aws/codebuild/nodejs-app'
                }
            }
        )
        
        return response['project']['arn']
    
    def create_codedeploy_application(self):
        """Create CodeDeploy application"""
        
        response = self.codedeploy.create_app(
            applicationName='nodejs-app'
        )
        
        return response
    
    def create_codedeploy_deployment_group(self):
        """Create CodeDeploy deployment group"""
        
        response = self.codedeploy.create_deployment_group(
            applicationName='nodejs-app',
            deploymentGroupName='production',
            deploymentConfigName='CodeDeployDefault.OneAtATime',
            ec2TagFilters=[
                {
                    'Key': 'Environment',
                    'Value': 'Production',
                    'Type': 'KEY_AND_VALUE'
                }
            ],
            deploymentStyle={
                'deploymentType': 'IN_PLACE',
                'deploymentOption': 'WITH_TRAFFIC_CONTROL'
            },
            autoRollbackConfiguration={
                'enabled': True,
                'events': ['DEPLOYMENT_FAILURE', 'DEPLOYMENT_STOP_ON_ALARM']
            }
        )
        
        return response
    
    def create_pipeline(self, role_arn):
        """Create CodePipeline"""
        
        pipeline_config = {
            'name': 'nodejs-app-pipeline',
            'roleArn': role_arn,
            'artifactStore': {
                'type': 'S3',
                'location': 'my-pipeline-bucket'
            },
            'stages': [
                {
                    'name': 'Source',
                    'actions': [{
                        'name': 'SourceAction',
                        'actionTypeId': {
                            'category': 'Source',
                            'owner': 'ThirdParty',
                            'provider': 'GitHub',
                            'version': '1'
                        },
                        'configuration': {
                            'Owner': 'user',
                            'Repo': 'nodejs-app',
                            'Branch': 'main',
                            'OAuthToken': 'your-github-token'
                        },
                        'outputArtifacts': [{
                            'name': 'SourceOutput'
                        }]
                    }]
                },
                {
                    'name': 'Build',
                    'actions': [{
                        'name': 'BuildAction',
                        'actionTypeId': {
                            'category': 'Build',
                            'owner': 'AWS',
                            'provider': 'CodeBuild',
                            'version': '1'
                        },
                        'configuration': {
                            'ProjectName': 'nodejs-build-project'
                        },
                        'inputArtifacts': [{
                            'name': 'SourceOutput'
                        }],
                        'outputArtifacts': [{
                            'name': 'BuildOutput'
                        }]
                    }]
                },
                {
                    'name': 'Deploy',
                    'actions': [{
                        'name': 'DeployAction',
                        'actionTypeId': {
                            'category': 'Deploy',
                            'owner': 'AWS',
                            'provider': 'CodeDeploy',
                            'version': '1'
                        },
                        'configuration': {
                            'ApplicationName': 'nodejs-app',
                            'DeploymentGroupName': 'production'
                        },
                        'inputArtifacts': [{
                            'name': 'BuildOutput'
                        }]
                    }]
                }
            ]
        }
        
        response = self.codepipeline.create_pipeline(pipeline=pipeline_config)
        return response
```

#### Step 2: buildspec.yml (Node.js Application)

```yaml
version: 0.2

phases:
  install:
    commands:
      - echo "Installing dependencies..."
      - npm install
      - npm install -g jest
  
  pre_build:
    commands:
      - echo "Pre-build phase - Setting up Docker..."
      - aws ecr get-login-password --region $AWS_DEFAULT_REGION | docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com
      - REPOSITORY_URI=$AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/nodejs-app
      - COMMIT_HASH=$(echo $CODEBUILD_RESOLVED_SOURCE_VERSION | cut -c 1-7)
      - IMAGE_TAG=${COMMIT_HASH:=latest}
  
  build:
    commands:
      - echo "Building application..."
      - npm run build
      - echo "Running unit tests..."
      - npm test -- --coverage
      - echo "Building Docker image on `date`"
      - docker build -t $REPOSITORY_URI:latest .
      - docker tag $REPOSITORY_URI:latest $REPOSITORY_URI:$IMAGE_TAG
  
  post_build:
    commands:
      - echo "Logging in to Amazon ECR..."
      - docker push $REPOSITORY_URI:latest
      - docker push $REPOSITORY_URI:$IMAGE_TAG
      - echo "Writing image definitions file..."
      - printf '[{"name":"nodejs-app","imageUri":"%s"}]' $REPOSITORY_URI:$IMAGE_TAG > imagedefinitions.json

artifacts:
  files:
    - imagedefinitions.json
    - package.json
    - package-lock.json
  name: BuildArtifact

cache:
  paths:
    - '/root/.npm/**/*'
    - 'node_modules/**/*'

reports:
  coverage:
    files:
      - 'coverage/cobertura-coverage.xml'
    file-format: 'COBERTURAXML'
```

#### Step 3: appspec.yaml (Deployment Configuration)

```yaml
version: 0.0

os: linux

files:
  - source: /
    destination: /opt/nodejs-app

permissions:
  - object: /opt/nodejs-app
    pattern: "**"
    owner: ec2-user
    group: ec2-user
    type:
      - directory
  - object: /opt/nodejs-app
    pattern: "**"
    owner: ec2-user
    group: ec2-user
    type:
      - file

hooks:
  ApplicationStop:
    - location: scripts/stop_app.sh
      timeout: 300
      runas: root
  
  BeforeInstall:
    - location: scripts/install_dependencies.sh
      timeout: 300
      runas: root
  
  ApplicationStart:
    - location: scripts/start_app.sh
      timeout: 300
      runas: root
  
  ValidateService:
    - location: scripts/validate_service.sh
      timeout: 300
      runas: root
```

#### Step 4: Deployment Scripts

**scripts/stop_app.sh**
```bash
#!/bin/bash
echo "Stopping Node.js application..."
pkill -f "node.*app.js" || true
```

**scripts/install_dependencies.sh**
```bash
#!/bin/bash
echo "Installing Node.js dependencies..."
cd /opt/nodejs-app
npm install --production
```

**scripts/start_app.sh**
```bash
#!/bin/bash
echo "Starting Node.js application..."
cd /opt/nodejs-app
nohup npm start > /tmp/app.log 2>&1 &
```

**scripts/validate_service.sh**
```bash
#!/bin/bash
echo "Validating service..."
sleep 5
curl -f http://localhost:3000/health || exit 1
```

---

### Example 2: Blue-Green Deployment with Load Balancer

**Scenario**: Deploy application with zero downtime using blue-green strategy

```python
class BlueGreenDeployment:
    def __init__(self):
        self.codedeploy = boto3.client('codedeploy')
        self.elbv2 = boto3.client('elbv2')
    
    def create_blue_green_deployment_group(self):
        """Create deployment group with blue-green strategy"""
        
        response = self.codedeploy.create_deployment_group(
            applicationName='nodejs-app',
            deploymentGroupName='blue-green-production',
            deploymentConfigName='CodeDeployDefault.AllAtOnce',
            deploymentStyle={
                'deploymentType': 'BLUE_GREEN',
                'deploymentOption': 'WITH_TRAFFIC_CONTROL'
            },
            loadBalancerInfo={
                'targetGroupInfoList': [{
                    'name': 'production-tg'
                }]
            },
            autoRollbackConfiguration={
                'enabled': True,
                'events': [
                    'DEPLOYMENT_FAILURE',
                    'DEPLOYMENT_STOP_ON_ALARM'
                ]
            },
            alarmConfiguration={
                'enabled': True,
                'alarms': [{
                    'name': 'high-error-rate'
                }]
            }
        )
        
        return response
    
    def deploy_blue_green(self, application_name, deployment_group):
        """Execute blue-green deployment"""
        
        deployment = self.codedeploy.create_deployment(
            applicationName=application_name,
            deploymentGroupName=deployment_group,
            revision={
                'revisionType': 'S3',
                's3Location': {
                    'bucket': 'my-pipeline-bucket',
                    'key': 'nodejs-app-revision.zip',
                    'bundleType': 'zip'
                }
            },
            deploymentConfigName='CodeDeployDefault.AllAtOnce',
            description='Blue-green deployment with traffic control',
            autoRollbackConfiguration={
                'enabled': True,
                'events': ['DEPLOYMENT_FAILURE']
            }
        )
        
        return deployment['deploymentId']
```

---

### Example 3: Multi-Stage Pipeline with Manual Approval

**Scenario**: Pipeline with approval gate before production deployment

```python
def create_multi_stage_pipeline_with_approval(self, role_arn):
    """Create pipeline with manual approval stage"""
    
    pipeline = {
        'name': 'multi-stage-pipeline',
        'roleArn': role_arn,
        'artifactStore': {
            'type': 'S3',
            'location': 'my-pipeline-bucket'
        },
        'stages': [
            {
                'name': 'Source',
                'actions': [{
                    'name': 'SourceAction',
                    'actionTypeId': {
                        'category': 'Source',
                        'owner': 'ThirdParty',
                        'provider': 'GitHub',
                        'version': '1'
                    },
                    'configuration': {
                        'Owner': 'user',
                        'Repo': 'app-repo',
                        'Branch': 'main'
                    },
                    'outputArtifacts': [{'name': 'SourceOutput'}]
                }]
            },
            {
                'name': 'Build',
                'actions': [{
                    'name': 'BuildAction',
                    'actionTypeId': {
                        'category': 'Build',
                        'owner': 'AWS',
                        'provider': 'CodeBuild',
                        'version': '1'
                    },
                    'configuration': {
                        'ProjectName': 'build-project'
                    },
                    'inputArtifacts': [{'name': 'SourceOutput'}],
                    'outputArtifacts': [{'name': 'BuildOutput'}]
                }]
            },
            {
                'name': 'DeployToStaging',
                'actions': [{
                    'name': 'DeployStaging',
                    'actionTypeId': {
                        'category': 'Deploy',
                        'owner': 'AWS',
                        'provider': 'CodeDeploy',
                        'version': '1'
                    },
                    'configuration': {
                        'ApplicationName': 'my-app',
                        'DeploymentGroupName': 'staging'
                    },
                    'inputArtifacts': [{'name': 'BuildOutput'}]
                }]
            },
            {
                'name': 'ApprovalForProduction',
                'actions': [{
                    'name': 'ManualApproval',
                    'actionTypeId': {
                        'category': 'Approval',
                        'owner': 'AWS',
                        'provider': 'Manual',
                        'version': '1'
                    },
                    'configuration': {
                        'CustomData': 'Please review staging deployment before approving production deployment',
                        'NotificationArn': 'arn:aws:sns:us-east-1:123456789012:approval-topic'
                    }
                }]
            },
            {
                'name': 'DeployToProduction',
                'actions': [{
                    'name': 'DeployProduction',
                    'actionTypeId': {
                        'category': 'Deploy',
                        'owner': 'AWS',
                        'provider': 'CodeDeploy',
                        'version': '1'
                    },
                    'configuration': {
                        'ApplicationName': 'my-app',
                        'DeploymentGroupName': 'production'
                    },
                    'inputArtifacts': [{'name': 'BuildOutput'}],
                    'roleArn': 'arn:aws:iam::123456789012:role/codepipeline-deploy-role'
                }]
            }
        ]
    }
    
    response = self.codepipeline.create_pipeline(pipeline=pipeline)
    return response
```

---

## Top 5 Interview Questions

### Question 1: Explain the Difference Between CodeBuild, CodeDeploy, and CodePipeline

**Scenario**: "Walk through the differences between these three services and explain where each fits in a CI/CD workflow."

**Answer Structure:**

**CodeBuild - Build & Test Automation**
- **Purpose**: Compile source code, run tests, produce deployment artifacts
- **Triggered by**: Code commits, manual triggers, scheduled events
- **Output**: Build artifacts (Docker images, JAR files, ZIPs)
- **Cost**: Based on build minutes consumed
- **Infrastructure**: Fully managed, no server management
- **Example**: "npm install && npm build && npm test"

**CodeDeploy - Application Deployment**
- **Purpose**: Deploy applications to EC2, Lambda, on-premises servers
- **Triggered by**: CodePipeline, manual API calls, on-schedule
- **Deployment Strategies**: In-place, Blue-Green, Canary, Linear
- **Cost**: Based on instances deployed
- **Agent-based**: Requires CodeDeploy agent on target instances
- **Example**: "Stop app → Install new version → Start app → Validate"

**CodePipeline - Workflow Orchestration**
- **Purpose**: Orchestrate entire CI/CD workflow from source to production
- **Contains**: Stages (Source, Build, Test, Deploy)
- **Actions**: Specific tasks within stages (CodeBuild, CodeDeploy, Lambda, etc.)
- **Cost**: Based on active pipelines, not execution count
- **Artifact Management**: Passes artifacts between stages via S3
- **Example**: GitHub Push → CodeBuild → CodeDeploy → EC2

**Workflow Comparison:**

```
Scenario: Deploy Node.js app to EC2

1. Developer pushes code to GitHub
   ↓ (CodePipeline triggered)
2. CodeBuild: Build Docker image, run tests
   ↓ (Artifact: Docker image in ECR)
3. CodeDeploy: Pull image, stop old container, start new container
   ↓ (All orchestrated by CodePipeline)
4. Application running with new code
```

**CodeBuild Workflow (Detailed):**
```
Source Code
    ↓
CodeBuild Environment (Linux container)
    ↓ (buildspec.yml defines phases)
    ├─ Install: npm install
    ├─ Pre-Build: Docker login
    ├─ Build: npm test, docker build
    └─ Post-Build: docker push to ECR
    ↓
Output: Docker image in ECR
```

**CodeDeploy Workflow (Detailed):**
```
Deployment Target
    ↓
CodeDeploy Agent (running on EC2)
    ↓ (appspec.yaml defines hooks)
    ├─ ApplicationStop: Stop old service
    ├─ BeforeInstall: Install dependencies
    ├─ ApplicationStart: Start new service
    └─ ValidateService: Health checks
    ↓
Application Running
```

**When Each Shines:**

- **Use CodeBuild when**: Need to compile, test, or package code
- **Use CodeDeploy when**: Need to deploy to multiple instances, support multiple deployment strategies
- **Use CodePipeline when**: Need to orchestrate multi-step release process

**Real-World Example:**

```
GitHub Push (main branch)
    ↓ CodePipeline Source Stage (Triggered)
    ↓
CodeBuild Stage
├─ npm install
├─ npm run lint
├─ npm test
├─ docker build -t myapp:latest
└─ docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:latest
    ↓ Output: imagedefinitions.json
    ↓
CodeDeploy Stage (ECS Cluster)
├─ Pull Docker image from ECR
├─ Update ECS task definition
├─ Rolling update of ECS service
└─ Validate service health
    ↓
Application Updated in Production
```

---

### Question 2: Design End-to-End CI/CD Pipeline with Multiple Environments

**Scenario**: "Design a complete CI/CD pipeline with dev, staging, and production environments. Include build stages, testing, approval gates, and rollback strategy."

**Answer Structure:**

**Pipeline Architecture:**

```
Source (GitHub)
    ↓
Build Stage (CodeBuild)
├─ Compile code
├─ Run unit tests
├─ Build Docker image
└─ Push to ECR
    ↓
Test Stage (CodeBuild)
├─ Integration tests
├─ Security scanning
└─ Code analysis
    ↓
Deploy to Dev (CodeDeploy)
├─ Automatic deployment
├─ Smoke tests
└─ Developer testing
    ↓
Manual Approval 1
    ↓
Deploy to Staging (CodeDeploy)
├─ Blue-green deployment
├─ Load testing
├─ Performance testing
└─ UAT
    ↓
Manual Approval 2
    ↓
Deploy to Production (CodeDeploy)
├─ Canary deployment (5% traffic)
├─ Monitor metrics (5 minutes)
├─ Gradual rollout (25%, 50%, 100%)
└─ Automatic rollback on failure
```

**Implementation:**

```python
def create_multi_environment_pipeline(self):
    """Create production-ready pipeline with multiple environments"""
    
    pipeline = {
        'name': 'production-pipeline',
        'roleArn': 'arn:aws:iam::123456789012:role/codepipeline-role',
        'artifactStore': {
            'type': 'S3',
            'location': 'prod-pipeline-artifacts',
            'encryptionKey': {
                'id': 'arn:aws:kms:us-east-1:123456789012:key/xxxxx',
                'type': 'KMS'
            }
        },
        'stages': [
            # Stage 1: Source
            {
                'name': 'Source',
                'actions': [{
                    'name': 'GitHub',
                    'actionTypeId': {
                        'category': 'Source',
                        'owner': 'ThirdParty',
                        'provider': 'GitHub',
                        'version': '1'
                    },
                    'configuration': {
                        'Owner': 'company',
                        'Repo': 'app-repo',
                        'Branch': 'main'
                    },
                    'outputArtifacts': [{'name': 'SourceOutput'}]
                }]
            },
            
            # Stage 2: Build
            {
                'name': 'Build',
                'actions': [{
                    'name': 'Build',
                    'actionTypeId': {
                        'category': 'Build',
                        'owner': 'AWS',
                        'provider': 'CodeBuild',
                        'version': '1'
                    },
                    'configuration': {
                        'ProjectName': 'build-project'
                    },
                    'inputArtifacts': [{'name': 'SourceOutput'}],
                    'outputArtifacts': [{'name': 'BuildOutput'}]
                }]
            },
            
            # Stage 3: Unit Tests
            {
                'name': 'UnitTests',
                'actions': [{
                    'name': 'UnitTests',
                    'actionTypeId': {
                        'category': 'Build',
                        'owner': 'AWS',
                        'provider': 'CodeBuild',
                        'version': '1'
                    },
                    'configuration': {
                        'ProjectName': 'unit-tests'
                    },
                    'inputArtifacts': [{'name': 'BuildOutput'}]
                }]
            },
            
            # Stage 4: Deploy to Dev
            {
                'name': 'DeployToDev',
                'actions': [{
                    'name': 'DeployDev',
                    'actionTypeId': {
                        'category': 'Deploy',
                        'owner': 'AWS',
                        'provider': 'CodeDeploy',
                        'version': '1'
                    },
                    'configuration': {
                        'ApplicationName': 'my-app',
                        'DeploymentGroupName': 'dev'
                    },
                    'inputArtifacts': [{'name': 'BuildOutput'}]
                }]
            },
            
            # Stage 5: Integration Tests
            {
                'name': 'IntegrationTests',
                'actions': [{
                    'name': 'IntegrationTests',
                    'actionTypeId': {
                        'category': 'Build',
                        'owner': 'AWS',
                        'provider': 'CodeBuild',
                        'version': '1'
                    },
                    'configuration': {
                        'ProjectName': 'integration-tests'
                    },
                    'inputArtifacts': [{'name': 'BuildOutput'}]
                }]
            },
            
            # Stage 6: Approval for Staging
            {
                'name': 'ApprovalForStaging',
                'actions': [{
                    'name': 'ApprovalForStaging',
                    'actionTypeId': {
                        'category': 'Approval',
                        'owner': 'AWS',
                        'provider': 'Manual',
                        'version': '1'
                    },
                    'configuration': {
                        'CustomData': 'Approve to proceed to staging deployment',
                        'NotificationArn': 'arn:aws:sns:us-east-1:123456789012:approvals'
                    }
                }]
            },
            
            # Stage 7: Deploy to Staging (Blue-Green)
            {
                'name': 'DeployToStaging',
                'actions': [{
                    'name': 'DeployStaging',
                    'actionTypeId': {
                        'category': 'Deploy',
                        'owner': 'AWS',
                        'provider': 'CodeDeploy',
                        'version': '1'
                    },
                    'configuration': {
                        'ApplicationName': 'my-app',
                        'DeploymentGroupName': 'staging'
                    },
                    'inputArtifacts': [{'name': 'BuildOutput'}]
                }]
            },
            
            # Stage 8: Performance Tests
            {
                'name': 'PerformanceTests',
                'actions': [{
                    'name': 'PerformanceTests',
                    'actionTypeId': {
                        'category': 'Build',
                        'owner': 'AWS',
                        'provider': 'CodeBuild',
                        'version': '1'
                    },
                    'configuration': {
                        'ProjectName': 'performance-tests'
                    },
                    'inputArtifacts': [{'name': 'BuildOutput'}]
                }]
            },
            
            # Stage 9: Approval for Production
            {
                'name': 'ApprovalForProduction',
                'actions': [{
                    'name': 'ApprovalForProduction',
                    'actionTypeId': {
                        'category': 'Approval',
                        'owner': 'AWS',
                        'provider': 'Manual',
                        'version': '1'
                    },
                    'configuration': {
                        'CustomData': 'Approve to proceed to PRODUCTION deployment',
                        'NotificationArn': 'arn:aws:sns:us-east-1:123456789012:production-approvals'
                    }
                }]
            },
            
            # Stage 10: Deploy to Production (Canary)
            {
                'name': 'DeployToProduction',
                'actions': [{
                    'name': 'DeployProduction',
                    'actionTypeId': {
                        'category': 'Deploy',
                        'owner': 'AWS',
                        'provider': 'CodeDeploy',
                        'version': '1'
                    },
                    'configuration': {
                        'ApplicationName': 'my-app',
                        'DeploymentGroupName': 'production'
                    },
                    'inputArtifacts': [{'name': 'BuildOutput'}],
                    'roleArn': 'arn:aws:iam::123456789012:role/production-deploy'
                }]
            }
        ]
    }
    
    return pipeline
```

**Deployment Group Configurations:**

```python
# Dev Deployment Group (In-Place, immediate)
{
    'applicationName': 'my-app',
    'deploymentGroupName': 'dev',
    'deploymentStyle': {
        'deploymentType': 'IN_PLACE',
        'deploymentOption': 'WITHOUT_TRAFFIC_CONTROL'
    },
    'deploymentConfigName': 'CodeDeployDefault.AllAtOnce',
    'autoRollbackConfiguration': {
        'enabled': True,
        'events': ['DEPLOYMENT_FAILURE']
    }
}

# Staging Deployment Group (Blue-Green, zero-downtime)
{
    'applicationName': 'my-app',
    'deploymentGroupName': 'staging',
    'deploymentStyle': {
        'deploymentType': 'BLUE_GREEN',
        'deploymentOption': 'WITH_TRAFFIC_CONTROL'
    },
    'loadBalancerInfo': {
        'targetGroupInfoList': [{'name': 'staging-tg'}]
    },
    'autoRollbackConfiguration': {
        'enabled': True,
        'events': ['DEPLOYMENT_FAILURE']
    }
}

# Production Deployment Group (Canary, gradual rollout)
{
    'applicationName': 'my-app',
    'deploymentGroupName': 'production',
    'deploymentStyle': {
        'deploymentType': 'IN_PLACE',
        'deploymentOption': 'WITH_TRAFFIC_CONTROL'
    },
    'deploymentConfigName': 'CodeDeployDefault.Canary10Percent5Minutes',
    'loadBalancerInfo': {
        'targetGroupInfoList': [{'name': 'production-tg'}]
    },
    'alarmConfiguration': {
        'enabled': True,
        'alarms': [
            {'name': 'HighErrorRate'},
            {'name': 'HighLatency'},
            {'name': 'HighCPU'}
        ]
    },
    'autoRollbackConfiguration': {
        'enabled': True,
        'events': [
            'DEPLOYMENT_FAILURE',
            'DEPLOYMENT_STOP_ON_ALARM'
        ]
    }
}
```

**Rollback Strategy:**

```
Production Deployment
├─ Canary (5% traffic for 5 min)
│  ├─ Metrics OK → Proceed
│  └─ Alarm triggered → Auto-rollback
├─ 25% traffic for 5 min
│  ├─ Metrics OK → Proceed
│  └─ Alarm triggered → Auto-rollback
├─ 50% traffic for 5 min
│  ├─ Metrics OK → Proceed
│  └─ Alarm triggered → Auto-rollback
└─ 100% traffic (Complete)
   └─ Monitors continuously
      └─ Alarm → Manual rollback option
```

---

### Question 3: Optimize CodeBuild for Faster Builds and Cost Reduction

**Scenario**: "Builds are taking 15 minutes and costing $500/month. Design a strategy to reduce build time to 7 minutes and costs to $300/month."

**Answer Structure:**

**Cost & Performance Analysis:**

Current State:
```
Build Time: 15 minutes
Builds/day: 10
Total builds/month: 300
Cost: $500/month (~$1.67 per build)

Optimization Goal:
Build Time: 7 minutes
Cost: $300/month
Efficiency: 50% faster, 40% cheaper
```

**Optimization Strategies:**

**1. Cache Dependencies (Biggest Impact)**

```yaml
version: 0.2

cache:
  paths:
    - '/root/.npm/**/*'
    - 'node_modules/**/*'
    - '.gradle/**/*'
    - '/root/.m2/**/*'

phases:
  install:
    commands:
      - echo "Installing dependencies from cache..."
      - npm ci  # Faster than npm install with lock file
```

Impact: 40-60% reduction in install phase

**2. Use Build Instance Optimization**

```python
# Instead of:
'computeType': 'BUILD_GENERAL1_SMALL'  # 3GB RAM, 2 vCPU

# Use:
'computeType': 'BUILD_GENERAL1_MEDIUM'  # 7GB RAM, 4 vCPU
# Faster builds, but costs more → NET savings with parallelization
```

**3. Parallel Test Execution**

```yaml
build:
  commands:
    - echo "Running tests in parallel..."
    - npm test -- --maxWorkers=4  # Parallel test execution
    - npm run lint  # Parallel linting
    - npm run security-scan  # Parallel security scan
```

Impact: 30-50% reduction in test phase

**4. Use ECR Cache**

```yaml
pre_build:
  commands:
    - aws ecr get-login-password | docker login --username AWS --password-stdin $ECR_REPO_URI
    - REPOSITORY_URI=$AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/myapp
    - docker pull $REPOSITORY_URI:latest || true  # Cache layer pulling

build:
  commands:
    - docker build --cache-from $REPOSITORY_URI:latest -t $REPOSITORY_URI:latest .
```

Impact: 20-40% reduction in Docker build phase

**5. Reduce Build Artifacts**

```yaml
artifacts:
  files:
    - 'dist/**/*'           # Only dist folder
    - 'imagedefinitions.json'
  exclude-paths:
    - 'node_modules/**/*'
    - 'coverage/**/*'
    - '**/*.test.js'
```

Impact: Faster artifact upload (5-10% improvement)

**6. Build Matrix for Parallel Stages**

```python
def create_optimized_build_project(self):
    """Create build project with optimization"""
    
    return {
        'name': 'optimized-build',
        'source': {
            'type': 'GITHUB',
            'location': 'https://github.com/user/repo.git'
        },
        'artifacts': {'type': 'CODEPIPELINE'},
        'environment': {
            'type': 'LINUX_CONTAINER',
            'computeType': 'BUILD_GENERAL1_MEDIUM',  # Optimization 1
            'image': 'aws/codebuild/amazonlinux2-x86_64-standard:5.0',
            'environmentVariables': [
                {
                    'name': 'CODEBUILD_REPORT_GROUP_NOTIFICATION_ENABLED',
                    'value': 'true'
                },
                {
                    'name': 'CODEBUILD_BUILD_NUM_PARALLEL',
                    'value': '4'  # Parallelization
                }
            ],
            'privilegedMode': True,
            'imagePullCredentialsType': 'CODEBUILD'
        },
        'cache': {
            'type': 'S3',
            'location': 'my-build-cache/codebuild-cache'  # Optimization 2
        },
        'logsConfig': {
            'cloudWatchLogs': {'status': 'ENABLED', 'groupName': '/aws/codebuild/optimized'}
        }
    }
```

**Cost-Benefit Analysis:**

```
Current Build (BUILD_GENERAL1_SMALL, 15 min):
├─ Install: 5 min
├─ Build: 5 min
├─ Test: 3 min
└─ Publish: 2 min
Cost: 15 min × $0.005/min = $0.075/build

Optimized (BUILD_GENERAL1_MEDIUM, 7 min with cache):
├─ Install: 1 min (cached)
├─ Build: 3 min (ECR cache)
├─ Test: 2 min (parallel)
└─ Publish: 1 min
Cost: 7 min × $0.01/min = $0.07/build

Monthly Impact (300 builds):
├─ Time saved: 300 × 8 min = 40 hours/month
├─ New cost: 300 × $0.07 = $21/month (compute)
│  + 500GB cache storage = $25/month
│  Total: ~$46/month
├─ Previous cost: $500/month
└─ Savings: $454/month (91% reduction!)
```

---

### Question 4: Implement Blue-Green Deployment with Automatic Rollback

**Scenario**: "Design and implement a blue-green deployment strategy with automatic rollback capability, CloudWatch monitoring, and health checks."

**Answer Structure:**

**Blue-Green Architecture:**

```
Before Deployment:
└─ Blue Environment (Production)
   ├─ EC2 Instances (v1.0)
   ├─ RDS Database (shared)
   └─ Load Balancer → Blue (100% traffic)

Green Environment (Standby):
   ├─ EC2 Instances (empty)
   └─ Ready for deployment

↓ Deploy to Green

After Deployment (Pre-Switch):
├─ Blue Environment (v1.0, 100% traffic)
└─ Green Environment (v2.0, testing)
   └─ Health checks running
      ├─ Application responding
      ├─ Database connectivity OK
      ├─ Performance metrics normal
      └─ No errors detected

↓ Traffic Switch

After Traffic Switch:
├─ Blue Environment (v1.0, 0% traffic, standby)
└─ Green Environment (v2.0, 100% traffic)
   └─ Continuous monitoring
      ├─ Error rate < 1%
      ├─ Latency < 200ms
      ├─ CPU < 70%
      └─ Success!

↓ (If failure) Auto-Rollback

Rollback:
├─ Blue Environment (v1.0, 100% traffic)
└─ Green Environment (v2.0, 0% traffic, for investigation)
```

**Implementation:**

```python
class BlueGreenDeploymentManager:
    def __init__(self):
        self.codedeploy = boto3.client('codedeploy')
        self.elbv2 = boto3.client('elbv2')
        self.cloudwatch = boto3.client('cloudwatch')
        self.sns = boto3.client('sns')
    
    def create_blue_green_deployment_config(self):
        """Configure blue-green deployment group with monitoring"""
        
        deployment_group = {
            'applicationName': 'my-app',
            'deploymentGroupName': 'production-blue-green',
            'deploymentStyle': {
                'deploymentType': 'BLUE_GREEN',
                'deploymentOption': 'WITH_TRAFFIC_CONTROL'
            },
            'loadBalancerInfo': {
                'targetGroupInfoList': [
                    {'name': 'production-tg'}
                ]
            },
            'blueGreenDeploymentConfiguration': {
                'terminateBlueInstancesOnDeploymentSuccess': {
                    'action': 'KEEP_ALIVE',  # Keep for quick rollback
                    'terminationWaitTimeInMinutes': 5
                },
                'deploymentReadyOption': {
                    'actionOnTimeout': 'CONTINUE_DEPLOYMENT'
                },
                'greenFleetProvisioningOption': {
                    'action': 'COPY_AUTO_SCALING_GROUP'
                }
            },
            'deploymentConfigName': 'CodeDeployDefault.AllAtOnceBlueGreen',
            'autoRollbackConfiguration': {
                'enabled': True,
                'events': [
                    'DEPLOYMENT_FAILURE',
                    'DEPLOYMENT_STOP_ON_ALARM'
                ]
            },
            'alarmConfiguration': {
                'enabled': True,
                'alarms': [
                    {'name': 'HighErrorRate'},
                    {'name': 'HighLatency'},
                    {'name': 'HighCPU'}
                ]
            },
            'triggerConfigurations': [
                {
                    'triggerEvents': ['DeploymentSuccess', 'DeploymentFailure'],
                    'triggerTargetArn': 'arn:aws:sns:us-east-1:123456789012:deployment-notifications'
                }
            ]
        }
        
        return deployment_group
    
    def create_deployment_with_rollback(self, app_name, deployment_group):
        """Create deployment with automatic rollback"""
        
        deployment = self.codedeploy.create_deployment(
            applicationName=app_name,
            deploymentGroupName=deployment_group,
            revision={
                'revisionType': 'S3',
                's3Location': {
                    'bucket': 'my-deploy-artifacts',
                    'key': 'app-v2.0.zip',
                    'bundleType': 'zip'
                }
            },
            deploymentConfigName='CodeDeployDefault.AllAtOnceBlueGreen',
            description='Blue-green deployment with automatic rollback',
            autoRollbackConfiguration={
                'enabled': True,
                'events': [
                    'DEPLOYMENT_FAILURE',
                    'DEPLOYMENT_STOP_ON_ALARM'
                ]
            }
        )
        
        deployment_id = deployment['deploymentId']
        self._monitor_deployment(deployment_id, app_name, deployment_group)
        
        return deployment_id
    
    def _monitor_deployment(self, deployment_id, app_name, deployment_group):
        """Monitor deployment and trigger rollback if needed"""
        
        # Create CloudWatch alarms
        self._create_monitoring_alarms(app_name)
        
        # Poll deployment status
        import time
        max_attempts = 120  # 2 hours
        attempt = 0
        
        while attempt < max_attempts:
            response = self.codedeploy.get_deployment(deploymentId=deployment_id)
            status = response['deploymentInfo']['status']
            
            print(f"Deployment Status: {status}")
            
            if status == 'Succeeded':
                print("Deployment successful!")
                self._notify('Deployment Successful', 
                            f"Blue-green deployment {deployment_id} completed successfully")
                break
            
            elif status == 'Failed':
                print("Deployment failed! Initiating rollback...")
                self._trigger_rollback(app_name, deployment_group)
                self._notify('Deployment Failed',
                            f"Blue-green deployment {deployment_id} failed. Rollback initiated.")
                break
            
            elif status == 'Stopped':
                print("Deployment stopped!")
                break
            
            time.sleep(30)
            attempt += 1
    
    def _create_monitoring_alarms(self, app_name):
        """Create CloudWatch alarms for deployment monitoring"""
        
        alarms = [
            {
                'AlarmName': f'{app_name}-HighErrorRate',
                'ComparisonOperator': 'GreaterThanThreshold',
                'EvaluationPeriods': 2,
                'MetricName': 'HTTPError5XX',
                'Namespace': 'AWS/ApplicationELB',
                'Period': 300,
                'Statistic': 'Sum',
                'Threshold': 10,
                'ActionsEnabled': True,
                'AlarmActions': ['arn:aws:sns:us-east-1:123456789012:alerts']
            },
            {
                'AlarmName': f'{app_name}-HighLatency',
                'ComparisonOperator': 'GreaterThanThreshold',
                'EvaluationPeriods': 2,
                'MetricName': 'TargetResponseTime',
                'Namespace': 'AWS/ApplicationELB',
                'Period': 300,
                'Statistic': 'Average',
                'Threshold': 1,  # 1 second
                'ActionsEnabled': True,
                'AlarmActions': ['arn:aws:sns:us-east-1:123456789012:alerts']
            }
        ]
        
        for alarm in alarms:
            try:
                self.cloudwatch.put_metric_alarm(**alarm)
            except Exception as e:
                print(f"Error creating alarm: {e}")
    
    def _trigger_rollback(self, app_name, deployment_group):
        """Manually trigger rollback"""
        
        # Get previous successful revision
        response = self.codedeploy.list_deployments(
            applicationName=app_name,
            deploymentGroupName=deployment_group,
            includeOnlyStatuses=['Succeeded'],
            sortBy='createTime',
            sortOrder='descending'
        )
        
        if response['deployments']:
            previous_deployment_id = response['deployments'][0]
            
            # Get revision of previous deployment
            prev_deployment = self.codedeploy.get_deployment(
                deploymentId=previous_deployment_id
            )
            
            previous_revision = prev_deployment['deploymentInfo']['revision']
            
            # Deploy previous version
            rollback = self.codedeploy.create_deployment(
                applicationName=app_name,
                deploymentGroupName=deployment_group,
                revision=previous_revision,
                description=f'Automatic rollback from failed deployment'
            )
            
            print(f"Rollback deployment initiated: {rollback['deploymentId']}")
    
    def _notify(self, subject, message):
        """Send notification"""
        
        self.sns.publish(
            TopicArn='arn:aws:sns:us-east-1:123456789012:deployment-notifications',
            Subject=subject,
            Message=message
        )

# Usage
manager = BlueGreenDeploymentManager()
deployment_group_config = manager.create_blue_green_deployment_config()
deployment_id = manager.create_deployment_with_rollback('my-app', 'production-blue-green')
```

---

### Question 5: Troubleshoot Pipeline Failures and Implement Best Practices

**Scenario**: "Pipeline is failing at CodeBuild stage with cryptic error messages. Walk through how you would troubleshoot, identify the root cause, and implement best practices to prevent future failures."

**Answer Structure:**

**Troubleshooting Framework:**

```
Pipeline Failure
    ↓
Check Pipeline Logs
├─ CodePipeline console (stage status)
├─ CloudWatch Logs (service logs)
└─ S3 artifacts (previous successful build)

Identify Failure Point
├─ Source stage (code retrieval)
├─ Build stage (compilation/tests)
├─ Deploy stage (deployment)
└─ Test stage (validation)

Analyze Root Cause
├─ Permission issues (IAM roles)
├─ Configuration issues (buildspec.yml)
├─ Environment issues (missing dependencies)
├─ Code issues (compilation errors)
└─ Infrastructure issues (service limits)

Fix Issue
├─ Update buildspec.yml
├─ Modify IAM permissions
├─ Update environment
└─ Fix code

Test & Validate
├─ Manual build/deploy
├─ Monitor logs
└─ Verify health checks

Implement Prevention
├─ Code reviews
├─ Automated testing
├─ Pre-commit hooks
└─ Documentation
```

**Common CodeBuild Failures & Solutions:**

```
Error 1: "No SDK found"
├─ Cause: Missing runtime installation
├─ Solution: Add to install phase
└─ buildspec.yml:
    phases:
      install:
        commands:
          - npm install -g npm@latest
          - node --version
          - npm --version

Error 2: "Dockerfile not found"
├─ Cause: Wrong working directory or path
├─ Solution: Use full path or cd first
└─ buildspec.yml:
    build:
      commands:
        - cd ./application
        - docker build -t myapp .

Error 3: "Permission denied - Docker socket"
├─ Cause: privilegedMode not set
├─ Solution: Enable in environment
└─ CodeBuild project:
    environment:
      privilegedMode: True

Error 4: "Failed to resolve dependencies"
├─ Cause: Cache expired or corrupted
├─ Solution: Clear cache or update
└─ buildspec.yml:
    cache:
      paths:
        - 'node_modules/**/*'

Error 5: "Artifact not found"
├─ Cause: Output not in artifacts section
├─ Solution: Add to artifacts
└─ buildspec.yml:
    artifacts:
      files:
        - 'dist/**/*'
        - 'package.json'
```

**Best Practices Implementation:**

```python
class BestPracticesImplementation:
    def __init__(self):
        self.codebuild = boto3.client('codebuild')
        self.codepipeline = boto3.client('codepipeline')
        self.logs = boto3.client('logs')
    
    def implement_comprehensive_logging(self):
        """Enable detailed logging for troubleshooting"""
        
        buildspec = """
version: 0.2

env:
  variables:
    DEBUG: "true"
    VERBOSE: "true"

phases:
  pre_build:
    commands:
      - echo "====== Environment Info ======"
      - echo "AWS Region: $AWS_DEFAULT_REGION"
      - echo "AWS Account: $AWS_ACCOUNT_ID"
      - echo "CodeBuild Project: $CODEBUILD_PROJECT_NAME"
      - echo "Build ID: $CODEBUILD_BUILD_ID"
      - echo "Source Version: $CODEBUILD_RESOLVED_SOURCE_VERSION"
      - echo "Node version: $(node --version)"
      - echo "NPM version: $(npm --version)"
      - echo "Docker version: $(docker --version)"
      - echo "=============================="
      - echo "Starting build at $(date)"
  
  build:
    commands:
      - set -e  # Exit on first error
      - echo "Installing dependencies..."
      - npm ci || npm install
      - echo "Building..."
      - npm run build 2>&1 | tee build.log  # Capture output
      - echo "Running tests..."
      - npm test 2>&1 | tee test.log
      - echo "Build completed successfully"
  
  post_build:
    on-failure:
      - echo "Build failed! Collecting debug info..."
      - echo "Node version: $(node --version)"
      - echo "NPM cache:"
      - npm cache verify
      - echo "Disk space:"
      - df -h

artifacts:
  files:
    - build/**/*
    - package.json
    - '*.log'  # Include logs for troubleshooting
  name: BuildArtifact
"""
        
        return buildspec
    
    def implement_pipeline_monitoring(self):
        """Create CloudWatch dashboard for pipeline monitoring"""
        
        dashboard_body = {
            'widgets': [
                {
                    'type': 'metric',
                    'properties': {
                        'metrics': [
                            ['AWS/CodeBuild', 'SuccessfulBuilds', {'stat': 'Sum'}],
                            ['.', 'FailedBuilds', {'stat': 'Sum'}],
                            ['.', 'Duration', {'stat': 'Average'}]
                        ],
                        'period': 300,
                        'stat': 'Sum',
                        'region': 'us-east-1',
                        'title': 'CodeBuild Metrics'
                    }
                },
                {
                    'type': 'log',
                    'properties': {
                        'query': '''
fields @timestamp, @message, @duration
| filter @message like /ERROR/
| stats count() as error_count by bin(5m)
''',
                        'region': 'us-east-1',
                        'title': 'Build Errors'
                    }
                }
            ]
        }
        
        return dashboard_body
    
    def implement_error_handling(self):
        """Add comprehensive error handling"""
        
        buildspec = """
version: 0.2

phases:
  build:
    commands:
      - |
        run_command() {
          local cmd="$1"
          local description="$2"
          echo "Running: $description"
          if ! eval "$cmd"; then
            echo "ERROR: $description failed with exit code $?"
            exit 1
          fi
          echo "SUCCESS: $description completed"
        }
      - run_command "npm ci" "Install dependencies"
      - run_command "npm run lint" "Run linter"
      - run_command "npm test" "Run tests"
      - run_command "npm run build" "Build application"

on-failure:
  commands:
    - echo "Build failed at phase: $CODEBUILD_BUILD_SUCCEEDING"
    - echo "Error details:"
    - tail -100 $CODEBUILD_LOG  # Show last 100 lines of build log
    - echo "Uploading debug logs..."
    - aws s3 cp build.log s3://my-debug-bucket/builds/$CODEBUILD_BUILD_ID/build.log
    - echo "Debug information available at: s3://my-debug-bucket/builds/$CODEBUILD_BUILD_ID/"
"""
        
        return buildspec
    
    def implement_notifications(self):
        """Setup comprehensive notifications"""
        
        # CodePipeline notifications
        pipeline_notifications = {
            'Detail': {
                'pipeline': ['all'],
                'state': ['FAILED'],
                'execution': ['Pipeline execution failed']
            },
            'DetailType': 'CodePipeline Pipeline Execution State Change',
            'Source': 'aws.codepipeline'
        }
        
        # CodeBuild notifications
        build_notifications = {
            'Detail': {
                'build-status': ['FAILED'],
                'project-name': ['my-build-project']
            },
            'DetailType': 'CodeBuild Build State Change',
            'Source': 'aws.codebuild'
        }
        
        return pipeline_notifications, build_notifications
```

**Prevention Checklist:**

```
Code Quality:
☐ Unit tests (>80% coverage)
☐ Code linting (ESLint, SonarQube)
☐ Security scanning (SonarQube, Snyk)
☐ Code review requirement
☐ Pre-commit hooks

Build Configuration:
☐ Clear buildspec.yml with comments
☐ Version pinning for dependencies
☐ Caching enabled
☐ Logging enabled (INFO level minimum)
☐ Error handling in scripts

Pipeline Configuration:
☐ Artifact retention policy
☐ Manual approval gates
☐ Monitoring/alarms enabled
☐ SNS notifications configured
☐ CloudWatch Logs enabled

Deployment Configuration:
☐ Health checks defined
☐ Rollback strategy configured
☐ Monitoring alarms set
☐ Automated tests in pipeline
☐ Gradual rollout strategy

Documentation:
☐ Runbook for common failures
☐ Deployment procedures documented
☐ Alert handling procedures
☐ Team training completed
☐ On-call rotation defined
```

---

## Summary

This comprehensive guide covers:

1. **Detailed Explanations**: Architecture, components, and workflows
2. **Practical Examples**: Real-world implementations with Python code
3. **Interview Questions**: Top 5 questions with detailed answers
4. **Best Practices**: Optimization, monitoring, and troubleshooting
5. **Production-Ready**: Secure, scalable, and resilient patterns

Use this guide for interview preparation, pipeline design, and troubleshooting production issues.