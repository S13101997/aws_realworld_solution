# Complete CloudFormation Blue-Green Deployment Implementation

## Overview

Blue-green deployments with CloudFormation use **AWS CodeDeploy + ECS** to maintain two identical environments, swapping ALB traffic after green validation. This ensures zero-downtime deployments for production applications with gradually shifting traffic patterns.

## Full CloudFormation Template

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'Blue-Green ECS Deployment with CloudFormation'

Parameters:
  ImageURI:
    Type: String
    Default: '123456789012.dkr.ecr.us-east-1.amazonaws.com/app-renderer:latest'
    Description: 'Docker image URI for the application'
  ClusterName:
    Type: String
    Default: 'Application-ECS-Cluster'
    Description: 'ECS cluster name'
  EnvironmentName:
    Type: String
    Default: 'production'
    AllowedValues: ['development', 'staging', 'production']

Resources:
  # VPC & Networking
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: '10.0.0.0/16'
      EnableDnsHostnames: true
      EnableDnsSupport: true

  PublicSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: '10.0.1.0/24'
      AvailabilityZone: !Select [0, !GetAZs '']
      MapPublicIpOnLaunch: true

  PublicSubnet2:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: '10.0.2.0/24'
      AvailabilityZone: !Select [1, !GetAZs '']
      MapPublicIpOnLaunch: true

  InternetGateway:
    Type: AWS::EC2::InternetGateway

  AttachGateway:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      VpcId: !Ref VPC
      InternetGatewayId: !Ref InternetGateway

  PublicRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC

  PublicRoute:
    Type: AWS::EC2::Route
    Properties:
      RouteTableId: !Ref PublicRouteTable
      DestinationCidrBlock: '0.0.0.0/0'
      GatewayId: !Ref InternetGateway

  SubnetRouteTableAssociation1:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PublicSubnet1
      RouteTableId: !Ref PublicRouteTable

  SubnetRouteTableAssociation2:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PublicSubnet2
      RouteTableId: !Ref PublicRouteTable

  # Security Groups
  ECSSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: 'Security group for ECS tasks'
      VpcId: !Ref VPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 8080
          ToPort: 8080
          CidrIp: '0.0.0.0/0'
        - IpProtocol: tcp
          FromPort: 443
          ToPort: 443
          CidrIp: '0.0.0.0/0'

  ALBSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: 'Security group for ALB'
      VpcId: !Ref VPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: '0.0.0.0/0'
        - IpProtocol: tcp
          FromPort: 443
          ToPort: 443
          CidrIp: '0.0.0.0/0'

  # ECS Cluster
  ECSCluster:
    Type: AWS::ECS::Cluster
    Properties:
      ClusterName: !Ref ClusterName
      CapacityProviders:
        - FARGATE
        - FARGATE_SPOT
      DefaultCapacityProviderStrategy:
        - CapacityProvider: FARGATE_SPOT
          Weight: 1
          Base: 0

  # Application Load Balancer
  ALB:
    Type: AWS::ElasticLoadBalancingV2::LoadBalancer
    Properties:
      Name: app-alb
      Subnets:
        - !Ref PublicSubnet1
        - !Ref PublicSubnet2
      SecurityGroups:
        - !Ref ALBSecurityGroup
      Scheme: internet-facing
      Type: application

  # Blue Target Group (Live)
  BlueTargetGroup:
    Type: AWS::ElasticLoadBalancingV2::TargetGroup
    Properties:
      Name: 'app-blue-tg'
      Port: 8080
      Protocol: HTTP
      VpcId: !Ref VPC
      TargetType: ip
      HealthCheckPath: '/healthz'
      HealthCheckProtocol: HTTP
      HealthCheckIntervalSeconds: 30
      HealthCheckTimeoutSeconds: 5
      HealthyThresholdCount: 2
      UnhealthyThresholdCount: 3
      Matcher:
        HttpCode: '200'

  # Green Target Group (New)
  GreenTargetGroup:
    Type: AWS::ElasticLoadBalancingV2::TargetGroup
    Properties:
      Name: 'app-green-tg'
      Port: 8080
      Protocol: HTTP
      VpcId: !Ref VPC
      TargetType: ip
      HealthCheckPath: '/healthz'
      HealthCheckProtocol: HTTP
      HealthCheckIntervalSeconds: 30
      HealthCheckTimeoutSeconds: 5
      HealthyThresholdCount: 2
      UnhealthyThresholdCount: 3
      Matcher:
        HttpCode: '200'

  # ALB Listener (starts pointing to Blue)
  Listener:
    Type: AWS::ElasticLoadBalancingV2::Listener
    Properties:
      LoadBalancerArn: !Ref ALB
      Port: 80
      Protocol: HTTP
      DefaultActions:
        - Type: forward
          TargetGroupArn: !Ref BlueTargetGroup

  # CloudWatch Log Group
  LogGroup:
    Type: AWS::Logs::LogGroup
    Properties:
      LogGroupName: !Sub '/ecs/app-${EnvironmentName}'
      RetentionInDays: 30

  # IAM Roles
  ECSExecutionRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: ecs-tasks.amazonaws.com
            Action: 'sts:AssumeRole'
      ManagedPolicyArns:
        - 'arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy'
      Policies:
        - PolicyName: ECRAccess
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action:
                  - 'ecr:GetAuthorizationToken'
                  - 'ecr:BatchGetImage'
                  - 'ecr:GetDownloadUrlForLayer'
                Resource: '*'
              - Effect: Allow
                Action:
                  - 'logs:CreateLogStream'
                  - 'logs:PutLogEvents'
                Resource: !GetAtt LogGroup.Arn

  ECSTaskRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: ecs-tasks.amazonaws.com
            Action: 'sts:AssumeRole'
      Policies:
        - PolicyName: S3Access
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action:
                  - 's3:GetObject'
                  - 's3:PutObject'
                Resource: '*'

  CodeDeployRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: codedeploy.amazonaws.com
            Action: 'sts:AssumeRole'
      ManagedPolicyArns:
        - 'arn:aws:iam::aws:policy/service-role/AWSCodeDeployRoleForECS'

  # Task Definitions
  BlueTaskDefinition:
    Type: AWS::ECS::TaskDefinition
    Properties:
      Family: 'app-blue'
      NetworkMode: awsvpc
      RequiresCompatibilities:
        - FARGATE
      Cpu: '1024'
      Memory: '2048'
      ExecutionRoleArn: !GetAtt ECSExecutionRole.Arn
      TaskRoleArn: !GetAtt ECSTaskRole.Arn
      ContainerDefinitions:
        - Name: app
          Image: !Ref ImageURI
          PortMappings:
            - ContainerPort: 8080
              Protocol: tcp
          LogConfiguration:
            LogDriver: awslogs
            Options:
              awslogs-group: !Ref LogGroup
              awslogs-region: !Ref 'AWS::Region'
              awslogs-stream-prefix: ecs
          Environment:
            - Name: ENVIRONMENT
              Value: !Ref EnvironmentName
          HealthCheck:
            Command:
              - CMD-SHELL
              - 'curl -f http://localhost:8080/healthz || exit 1'
            Interval: 30
            Timeout: 5
            Retries: 2
            StartPeriod: 60

  GreenTaskDefinition:
    Type: AWS::ECS::TaskDefinition
    Properties:
      Family: 'app-green'
      NetworkMode: awsvpc
      RequiresCompatibilities:
        - FARGATE
      Cpu: '1024'
      Memory: '2048'
      ExecutionRoleArn: !GetAtt ECSExecutionRole.Arn
      TaskRoleArn: !GetAtt ECSTaskRole.Arn
      ContainerDefinitions:
        - Name: app
          Image: !Ref ImageURI
          PortMappings:
            - ContainerPort: 8080
              Protocol: tcp
          LogConfiguration:
            LogDriver: awslogs
            Options:
              awslogs-group: !Ref LogGroup
              awslogs-region: !Ref 'AWS::Region'
              awslogs-stream-prefix: ecs
          Environment:
            - Name: ENVIRONMENT
              Value: !Ref EnvironmentName
          HealthCheck:
            Command:
              - CMD-SHELL
              - 'curl -f http://localhost:8080/healthz || exit 1'
            Interval: 30
            Timeout: 5
            Retries: 2
            StartPeriod: 60

  # Blue ECS Service
  BlueService:
    Type: AWS::ECS::Service
    DependsOn: Listener
    Properties:
      Cluster: !Ref ECSCluster
      ServiceName: 'app-blue'
      TaskDefinition: !Ref BlueTaskDefinition
      DesiredCount: 3
      LaunchType: FARGATE
      LoadBalancers:
        - ContainerName: app
          ContainerPort: 8080
          TargetGroupArn: !Ref BlueTargetGroup
      NetworkConfiguration:
        AwsvpcConfiguration:
          Subnets:
            - !Ref PublicSubnet1
            - !Ref PublicSubnet2
          SecurityGroups:
            - !Ref ECSSecurityGroup
          AssignPublicIp: ENABLED
      DeploymentConfiguration:
        MaximumPercent: 200
        MinimumHealthyPercent: 100
      EnableECSManagedTags: true

  # Green ECS Service (initially 0 tasks)
  GreenService:
    Type: AWS::ECS::Service
    DependsOn: Listener
    Properties:
      Cluster: !Ref ECSCluster
      ServiceName: 'app-green'
      TaskDefinition: !Ref GreenTaskDefinition
      DesiredCount: 0
      LaunchType: FARGATE
      LoadBalancers:
        - ContainerName: app
          ContainerPort: 8080
          TargetGroupArn: !Ref GreenTargetGroup
      NetworkConfiguration:
        AwsvpcConfiguration:
          Subnets:
            - !Ref PublicSubnet1
            - !Ref PublicSubnet2
          SecurityGroups:
            - !Ref ECSSecurityGroup
          AssignPublicIp: ENABLED
      DeploymentConfiguration:
        MaximumPercent: 200
        MinimumHealthyPercent: 100
      EnableECSManagedTags: true

  # CodeDeploy Application
  CodeDeployApp:
    Type: AWS::CodeDeploy::Application
    Properties:
      ApplicationName: app-codedeploy
      ComputePlatform: ECS

  # CodeDeploy Deployment Group with Blue-Green Configuration
  DeploymentGroup:
    Type: AWS::CodeDeploy::DeploymentGroup
    Properties:
      ApplicationName: !Ref CodeDeployApp
      DeploymentGroupName: 'app-blue-green-dg'
      ServiceRoleArn: !GetAtt CodeDeployRole.Arn
      DeploymentConfigName: 'CodeDeployDefault.ECSLinear10PercentEvery1Minutes'
      DeploymentStyle:
        DeploymentType: BLUE_GREEN
        DeploymentOption: WITH_TRAFFIC_CONTROL
      BlueGreenDeploymentConfiguration:
        TerminateBlueInstancesOnDeployment:
          Action: TERMINATE
          TerminationWaitTimeInMinutes: 5
        DeploymentReadyOption:
          ActionOnTimeout: CONTINUE_DEPLOYMENT
        GreenFleetProvisioningOption:
          Action: COPY_AUTO_SCALING_GROUP
      LoadBalancerInfo:
        TargetGroupPairInfoList:
          - TargetGroup:
              Name: !GetAtt BlueTargetGroup.TargetGroupName
            TrafficRoute:
              ListenerArns:
                - !Ref Listener
      TriggerConfigurations: []
      AlarmConfiguration:
        Enabled: true
        Alarms:
          - Name: !Ref DeploymentErrorAlarm

  # CloudWatch Alarms for Deployment Monitoring
  DeploymentErrorAlarm:
    Type: AWS::CloudWatch::Alarm
    Properties:
      AlarmName: 'app-deployment-errors'
      ComparisonOperator: GreaterThanThreshold
      EvaluationPeriods: 2
      MetricName: HTTPCode_Target_5XX
      Namespace: AWS/ApplicationELB
      Period: 60
      Statistic: Sum
      Threshold: 10
      Dimensions:
        - Name: LoadBalancer
          Value: !GetAtt ALB.LoadBalancerFullName
      TreatMissingData: notBreaching

  TargetResponseTimeAlarm:
    Type: AWS::CloudWatch::Alarm
    Properties:
      AlarmName: 'app-target-response-time'
      ComparisonOperator: GreaterThanThreshold
      EvaluationPeriods: 3
      MetricName: TargetResponseTime
      Namespace: AWS/ApplicationELB
      Period: 60
      Statistic: Average
      Threshold: 1.0
      Dimensions:
        - Name: LoadBalancer
          Value: !GetAtt ALB.LoadBalancerFullName
      TreatMissingData: notBreaching

Outputs:
  LoadBalancerDNS:
    Description: 'DNS name of the load balancer'
    Value: !GetAtt ALB.DNSName
    Export:
      Name: !Sub '${AWS::StackName}-LoadBalancerDNS'

  BlueTargetGroupArn:
    Description: 'ARN of the Blue target group'
    Value: !Ref BlueTargetGroup
    Export:
      Name: !Sub '${AWS::StackName}-BlueTargetGroup'

  GreenTargetGroupArn:
    Description: 'ARN of the Green target group'
    Value: !Ref GreenTargetGroup
    Export:
      Name: !Sub '${AWS::StackName}-GreenTargetGroup'

  CodeDeployAppName:
    Description: 'CodeDeploy application name'
    Value: !Ref CodeDeployApp
    Export:
      Name: !Sub '${AWS::StackName}-CodeDeployApp'

  CodeDeployDeploymentGroup:
    Description: 'CodeDeploy deployment group name'
    Value: !Ref DeploymentGroup
    Export:
      Name: !Sub '${AWS::StackName}-DeploymentGroup'
```

## Step-by-Step Deployment Process

### Phase 1: Initial Stack Deployment

```bash
# 1. Deploy the initial CloudFormation stack with Blue environment
aws cloudformation deploy \
  --template-file blue-green.yaml \
  --stack-name app-blue-green \
  --capabilities CAPABILITY_IAM \
  --parameter-overrides \
    ImageURI=123456789012.dkr.ecr.us-east-1.amazonaws.com/app:v1.0 \
    ClusterName=production-cluster \
    EnvironmentName=production \
  --region us-east-1

# 2. Verify stack creation
aws cloudformation describe-stacks \
  --stack-name app-blue-green \
  --query 'Stacks[0].StackStatus' \
  --region us-east-1

# Output: CREATE_COMPLETE
```

### Phase 2: Building and Pushing New Image

```bash
# 1. Build the new application image (v2.0)
docker build -t app:v2.0 .

# 2. Authenticate with ECR
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin 123456789012.dkr.ecr.us-east-1.amazonaws.com

# 3. Tag image for ECR
docker tag app:v2.0 \
  123456789012.dkr.ecr.us-east-1.amazonaws.com/app:v2.0

# 4. Push to ECR
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/app:v2.0

# 5. Verify image in ECR
aws ecr list-images \
  --repository-name app \
  --region us-east-1 \
  --query 'imageIds[*].imageTag'
```

### Phase 3: Triggering Deployment

```bash
# 1. Create appspec.json for CodeDeploy
cat > appspec.yaml << 'EOF'
version: 0.0
Resources:
  - TargetService:
      Type: AWS::ECS::Service
      Properties:
        TaskDefinition: "arn:aws:ecs:us-east-1:123456789012:task-definition/app-green:1"
        LoadBalancerInfo:
          ContainerName: "app"
          ContainerPort: 8080
        PlatformVersion: "LATEST"
        NetworkConfiguration:
          AwsvpcConfiguration:
            Subnets:
              - "subnet-xxxxx"
              - "subnet-yyyyy"
            SecurityGroups:
              - "sg-zzzzz"
            AssignPublicIp: "ENABLED"
Hooks:
  - BeforeInstall: "PreDeploymentHook"
  - AfterInstall: "PostDeploymentHook"
EOF

# 2. Trigger CodeDeploy deployment
DEPLOYMENT_ID=$(aws deploy create-deployment \
  --application-name app-codedeploy \
  --deployment-group-name app-blue-green-dg \
  --revision revisionType=S3,s3Location=s3://my-bucket/appspec.yaml \
  --description "Deploy v2.0 - Blue to Green" \
  --deployment-config-name CodeDeployDefault.ECSLinear10PercentEvery1Minutes \
  --query 'deploymentId' \
  --output text \
  --region us-east-1)

echo "Deployment ID: $DEPLOYMENT_ID"

# 3. Monitor deployment progress
aws deploy get-deployment \
  --deployment-id $DEPLOYMENT_ID \
  --query 'deploymentInfo.status' \
  --region us-east-1
```

### Phase 4: Monitoring and Validation

```bash
# 1. Watch real-time traffic shift
aws cloudwatch get-metric-statistics \
  --namespace AWS/ApplicationELB \
  --metric-name TargetConnectionCount \
  --dimensions Name=TargetGroup,Value=targetgroup/app-blue-tg/xxxxx \
  --start-time $(date -u -d '5 minutes ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 60 \
  --statistics Sum \
  --region us-east-1

# 2. Check error rates on Green before full switch
aws cloudwatch get-metric-statistics \
  --namespace AWS/ApplicationELB \
  --metric-name HTTPCode_Target_5XX \
  --dimensions Name=TargetGroup,Value=targetgroup/app-green-tg/yyyyy \
  --start-time $(date -u -d '10 minutes ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 60 \
  --statistics Sum \
  --region us-east-1

# 3. View deployment events
aws deploy list-deployment-targets \
  --deployment-id $DEPLOYMENT_ID \
  --region us-east-1

# 4. Get detailed deployment status
aws deploy get-deployment \
  --deployment-id $DEPLOYMENT_ID \
  --query 'deploymentInfo.[status,creator,startTime,completeTime]' \
  --region us-east-1
```

## Traffic Shifting Timeline

The `CodeDeployDefault.ECSLinear10PercentEvery1Minutes` configuration shifts traffic as follows:

| Time | Blue Traffic | Green Traffic | Phase |
|------|--------------|---------------|-------|
| 0 min | 100% | 0% | Start - Blue fully live |
| 1 min | 90% | 10% | Initial canary |
| 2 min | 80% | 20% | Monitoring for errors |
| 3 min | 70% | 30% | Continued validation |
| 4 min | 60% | 40% | Halfway point |
| 5 min | 50% | 50% | Halfway validation |
| 6 min | 0% | 100% | Full switch to Green |

**At each step**, CloudWatch alarms monitor:
- HTTP 5xx errors (threshold: >10 per minute)
- Response time (threshold: >1.0 second)
- Target connection count

## Automatic Rollback Scenario

If errors spike during traffic shift:

```bash
# Deployment automatically pauses and can rollback
aws deploy stop-deployment \
  --deployment-id $DEPLOYMENT_ID \
  --auto-rollback-enabled \
  --region us-east-1

# Manual rollback if needed
aws deploy create-deployment \
  --application-name app-codedeploy \
  --deployment-group-name app-blue-green-dg \
  --revision revisionType=S3,s3Location=s3://my-bucket/appspec.yaml \
  --description "Rollback to Blue" \
  --deployment-config-name CodeDeployDefault.AllAtOnce \
  --region us-east-1
```

## Real-World Implementation Examples

### Example 1: REST API Service

- **Blue**: v1.0 with 3 Fargate tasks
- **Green**: v2.0 with database query optimization
- **Traffic Shift**: 10% increments over 6 minutes
- **Validation**: 0.1% error threshold, <500ms latency
- **Result**: Zero downtime during peak 50k req/min

### Example 2: Data Processing Pipeline

- **Blue**: Batch processing with older SDK
- **Green**: Optimized SDK with parallel tasks
- **Traffic Shift**: Gradual increase with 30-second windows
- **Monitoring**: Job completion rates, processing latency
- **Rollback**: Triggered if job failure rate >1%

### Example 3: WebSocket/Real-Time Service

- **Blue**: Active connections maintained
- **Green**: New connection handling protocol
- **Strategy**: Drain Blue connections during shift (5-minute grace period)
- **Validation**: Reconnection success rate >99.9%
- **Duration**: 10-minute total deployment window

## Cost Implications

| Resource | Cost Impact | Notes |
|----------|-------------|-------|
| Running 2 ECS services | 2x compute | Temporary during deployment |
| ALB rules | Minimal | Same ALB for both groups |
| CloudWatch metrics | +$0.10/alarm | Additional monitoring alarms |
| Data transfer | Minimal | Internal ALB traffic |

**Total cost of blue-green during deployment:** ~2x compute for 6-10 minutes

## Best Practices

1. **Health Check Configuration**: Set `StartPeriod` to 60+ seconds for warm-up
2. **Gradual Traffic Shift**: Use Linear instead of AllAtOnce for validation
3. **Alarm Thresholds**: Conservative values (5-10% above normal)
4. **Automatic Rollback**: Always enable for critical services
5. **Deployment Windows**: Schedule during low-traffic periods when possible
6. **Testing**: Run full integration tests in Green before traffic shift
7. **Documentation**: Track deployment history with CodeDeploy tags

## Troubleshooting

### Deployment Stuck in "In Progress"

```bash
# Check if Green tasks are healthy
aws ecs describe-services \
  --cluster production-cluster \
  --services app-green \
  --query 'services[0].[deployments,events]' \
  --region us-east-1

# Check task definition
aws ecs describe-task-definition \
  --task-definition app-green \
  --query 'taskDefinition.containerDefinitions[0].image' \
  --region us-east-1
```

### High Error Rate During Shift

```bash
# View logs from Green tasks
aws logs tail /ecs/app-production --follow --region us-east-1

# Stop deployment and manually investigate
aws deploy stop-deployment \
  --deployment-id $DEPLOYMENT_ID \
  --auto-rollback-enabled \
  --region us-east-1
```

### ALB Not Switching Traffic

```bash
# Verify listener rule points to correct target group
aws elbv2 describe-listeners \
  --load-balancer-arn arn:aws:elasticloadbalancing:... \
  --query 'Listeners[*].[DefaultActions,Port]' \
  --region us-east-1

# Update listener target if needed
aws elbv2 modify-listener \
  --listener-arn arn:aws:elasticloadbalancing:... \
  --default-actions Type=forward,TargetGroupArn=arn:aws:... \
  --region us-east-1
```

## Conclusion

CloudFormation blue-green deployments with CodeDeploy provide:

✓ **Zero-downtime deployments** with automated traffic shifting  
✓ **Gradual validation** via configurable traffic percentages  
✓ **Automatic rollback** on error detection  
✓ **Cost efficiency** (temporary 2x compute during deployment)  
✓ **Infrastructure as Code** for reproducible deployments  
✓ **Full audit trail** via CloudFormation and CodeDeploy events  

This approach scales from small applications to enterprise systems handling millions of requests per minute.
