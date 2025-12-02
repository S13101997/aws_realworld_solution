# AWS CloudTrail — API Call Tracking and Logging

## Detailed Explanation

### Overview: Complete API Activity Audit Trail

AWS CloudTrail is a comprehensive service that records all API calls and user actions across your AWS infrastructure, providing a complete audit trail for security analysis, compliance, and troubleshooting[1]. It captures who did what, when, where, and the results of every API action in your AWS environment.

**Core Purpose:**
- **Compliance**: Meet regulatory requirements (HIPAA, PCI-DSS, SOX) for audit trails
- **Security**: Detect unauthorized or suspicious activity
- **Troubleshooting**: Identify what changed and when in your infrastructure
- **Operational Analysis**: Understand who performed which actions
- **Resource Tracking**: Monitor resource creation, modification, and deletion

**Key Components:**

1. **Trails**: Channels that deliver CloudTrail events to S3 buckets and CloudWatch Logs
2. **Events**: Records of API calls and user actions
3. **Event History**: Built-in 90-day history visible in CloudTrail console
4. **Log Files**: JSON files stored in S3 containing complete event details
5. **Data Events**: Detailed tracking of resource usage (S3 objects, Lambda invocations)
6. **Management Events**: Control plane API calls (EC2 launch, IAM changes)
7. **Insights**: Unusual activity detection powered by machine learning

### AWS CloudTrail Event Types

#### Event Type 1: Management Events (Control Plane)

**Definition**: API calls that manage AWS resources

**Examples:**
- Launching EC2 instances
- Creating IAM users and roles
- Modifying security group rules
- Deleting RDS databases
- Creating S3 buckets
- Changing IAM policies

**Event Example:**

```json
{
  "eventVersion": "1.08",
  "userIdentity": {
    "type": "IAMUser",
    "principalId": "AIDACKCEVSQ6C2EXAMPLE",
    "arn": "arn:aws:iam::123456789012:user/alice",
    "accountId": "123456789012",
    "userName": "alice"
  },
  "eventTime": "2024-01-15T10:30:45Z",
  "eventSource": "ec2.amazonaws.com",
  "eventName": "RunInstances",
  "awsRegion": "us-east-1",
  "sourceIPAddress": "203.0.113.12",
  "userAgent": "aws-cli/2.0.0",
  "requestParameters": {
    "instanceType": "t3.medium",
    "imageId": "ami-0c55b159cbfafe1f0",
    "minCount": 1,
    "maxCount": 1,
    "tagSpecificationSet": {
      "items": [
        {
          "resourceType": "instance",
          "tagSet": {
            "items": [
              {
                "key": "Name",
                "value": "web-server-01"
              }
            ]
          }
        }
      ]
    }
  },
  "responseElements": {
    "instancesSet": {
      "items": [
        {
          "instanceId": "i-1234567890abcdef0",
          "imageId": "ami-0c55b159cbfafe1f0",
          "instanceState": {
            "code": 0,
            "name": "pending"
          }
        }
      ]
    }
  },
  "requestID": "12345678-1234-1234-1234-123456789012",
  "eventID": "1-5e1f2b3c-4d5e6f7g8h9i0j1k2l3m",
  "eventName": "RunInstances",
  "readOnly": false,
  "resources": [
    {
      "ARN": "arn:aws:ec2:us-east-1:123456789012:instance/i-1234567890abcdef0",
      "accountId": "123456789012",
      "type": "AWS::EC2::Instance"
    }
  ],
  "eventType": "AwsApiCall",
  "managementEventSource": "ec2.amazonaws.com",
  "recipientAccountId": "123456789012"
}
```

**Management Event Categories:**

| Category | AWS Service Examples | Event Frequency |
|----------|----------------------|-----------------|
| **Compute** | EC2, Lambda, ECS, Batch | High (many launch/terminate events) |
| **Storage** | S3, EBS, EFS, Glacier | High (bucket operations) |
| **Database** | RDS, DynamoDB, Redshift | Medium (create/modify/delete) |
| **Identity & Access** | IAM, STS, Cognito | Medium (user/role management) |
| **Networking** | VPC, VPN, CloudFront, Route53 | Medium (network changes) |
| **Management** | CloudFormation, Systems Manager | Medium (infrastructure changes) |

#### Event Type 2: Data Events (Data Plane)

**Definition**: API calls that access or modify data within resources

**Examples:**
- S3 object GetObject, PutObject, DeleteObject
- Lambda function invocations
- DynamoDB GetItem, PutItem, Query operations
- RDS query execution (if enabled)

**Data Event Example:**

```json
{
  "eventVersion": "1.08",
  "userIdentity": {
    "type": "IAMUser",
    "principalId": "AIDACKCEVSQ6C2EXAMPLE",
    "arn": "arn:aws:iam::123456789012:user/application",
    "accountId": "123456789012",
    "userName": "application"
  },
  "eventTime": "2024-01-15T10:35:20Z",
  "eventSource": "s3.amazonaws.com",
  "eventName": "PutObject",
  "awsRegion": "us-east-1",
  "sourceIPAddress": "192.0.2.1",
  "userAgent": "aws-sdk-js/2.0.0",
  "requestParameters": {
    "bucketName": "my-application-bucket",
    "key": "documents/report-2024-01.pdf"
  },
  "responseElements": null,
  "s3": {
    "bucket": {
      "name": "my-application-bucket",
      "arn": "arn:aws:s3:::my-application-bucket"
    },
    "object": {
      "key": "documents/report-2024-01.pdf",
      "size": 2048576,
      "sequencer": "00631F4A3F415B2E0E"
    }
  },
  "requestID": "C3D92B7763FAF659",
  "eventID": "1-5e1f2b3c-4d5e6f7g8h9i0j1k2l3m",
  "eventName": "PutObject",
  "readOnly": false,
  "resources": [
    {
      "type": "AWS::S3::Object",
      "ARN": "arn:aws:s3:::my-application-bucket/documents/report-2024-01.pdf",
      "accountId": "123456789012"
    }
  ],
  "eventType": "AwsApiCall",
  "recipientAccountId": "123456789012"
}
```

**Data Events by Service:**

| Service | Data Events | Use Cases |
|---------|------------|-----------|
| **S3** | GetObject, PutObject, DeleteObject, ListBucket | Audit file access, compliance |
| **Lambda** | Invoke | Monitor function execution |
| **DynamoDB** | GetItem, PutItem, UpdateItem, Query, Scan | Track data access patterns |
| **RDS (PostgreSQL, MySQL)** | Query execution | Database query audit |
| **EBS** | GetSnapshot, CreateVolume | Volume access tracking |

#### Event Type 3: Insights Events

**Definition**: Unusual API activity detected by machine learning

**Examples:**
- Unusually high rate of failed API calls
- Unusual API calls for account
- Unusual volume of API calls
- Unusual error patterns

**Detection:**
- Baseline established from historical data
- Machine learning identifies anomalies
- Real-time detection for security threats

**Use Cases:**
- Detect compromised credentials
- Identify potential data exfiltration
- Spot unauthorized access attempts
- Alert on unusual account behavior

### CloudTrail Architecture and Flow

**Event Flow Architecture:**

```
AWS Services (EC2, S3, IAM, RDS, etc.)
    ↓
API Call Initiated
    ↓
CloudTrail Captures Event
    ├─ Event ID (unique identifier)
    ├─ User Identity (who made the call)
    ├─ Event Time (when)
    ├─ Event Source (which service)
    ├─ Event Name (which API)
    ├─ Request Parameters (what was requested)
    ├─ Response Elements (result)
    ├─ Source IP (from where)
    └─ User Agent (with what client)
    ↓
Event Processing
    ├─ S3 Delivery (JSON files)
    ├─ CloudWatch Logs (real-time stream)
    ├─ EventBridge Rules (trigger actions)
    └─ CloudTrail Insights (anomaly detection)
    ↓
Event History (90-day console view)
Storage & Analysis
    ├─ S3 Lake Formation Optimization
    ├─ Athena SQL Queries
    ├─ Security Hub Analysis
    └─ Third-party SIEM Integration
```

### Trail Types and Configurations

#### Single-Account Trail

Tracks events within one AWS account:

```python
import boto3

cloudtrail = boto3.client('cloudtrail')

# Create single-account trail
response = cloudtrail.create_trail(
    Name='organization-trail',
    S3BucketName='my-cloudtrail-bucket',
    IncludeGlobalServiceEvents=True,
    IsMultiRegionTrail=True,
    EnableLogFileValidation=True,
    KmsKeyId='arn:aws:kms:us-east-1:123456789012:key/12345678-1234-1234-1234-123456789012',
    Tags=[
        {'Key': 'Environment', 'Value': 'production'},
        {'Key': 'Purpose', 'Value': 'audit'}
    ]
)

# Start logging
cloudtrail.start_logging(Name='organization-trail')

print(f"Trail created: {response['TrailARN']}")
```

**Features:**
- Single AWS account scope
- Logs to dedicated S3 bucket
- Optional CloudWatch Logs stream
- Regional or multi-region options

#### Organization Trail (Multi-Account)

Tracks events across multiple accounts in AWS Organizations:

```python
import boto3

cloudtrail = boto3.client('cloudtrail')
organizations = boto3.client('organizations')

# Create organization trail (from management account)
response = cloudtrail.create_trail(
    Name='org-trail',
    S3BucketName='organization-cloudtrail-bucket',
    IsOrganizationTrail=True,
    IncludeGlobalServiceEvents=True,
    IsMultiRegionTrail=True,
    EnableLogFileValidation=True
)

# Enable for all accounts and regions
cloudtrail.start_logging(Name='org-trail')

# Verify organization trail logging
response = cloudtrail.describe_trails(includeShadowTrails=False)

print(f"Organization trail created: {response['trailList'][0]['TrailARN']}")
print(f"Logging status: {response['trailList'][0]['HasCustomEventSelectors']}")
```

**Organization Trail Architecture:**

```
Management Account (Central)
    ↓
Organization Trail
    ├─ Member Account A → Events
    ├─ Member Account B → Events
    ├─ Member Account C → Events
    └─ Member Account D → Events
    ↓
Central S3 Bucket
    └─ Unified log stream
```

**Benefits:**
- Centralized logging across all accounts
- Single point for audit and compliance
- Consistent security monitoring
- Automatic logging for new member accounts

#### Advanced Event Selectors

**Default Event Selector (All Management Events):**

```python
# Includes all management events
# Excludes data events (must be explicitly enabled)
```

**Custom Event Selectors (Granular Control):**

```python
import boto3

cloudtrail = boto3.client('cloudtrail')

# Configure custom event selectors
response = cloudtrail.put_event_selectors(
    TrailName='organization-trail',
    EventSelectors=[
        {
            'ReadWriteType': 'All',  # All, ReadOnly, WriteOnly
            'IncludeManagementEvents': True,
            'DataResources': [
                {
                    'Type': 'AWS::S3::Object',
                    'Values': ['arn:aws:s3:::my-bucket/*']
                },
                {
                    'Type': 'AWS::Lambda::Function',
                    'Values': ['arn:aws:lambda:us-east-1:123456789012:function/*']
                }
            ]
        }
    ],
    # Advanced event selectors (more granular)
    AdvancedEventSelectors=[
        {
            'Field': 'eventCategory',
            'Equals': ['Management']
        },
        {
            'Field': 'resources.type',
            'Equals': ['AWS::EC2::Instance']
        },
        {
            'Field': 'eventName',
            'Equals': ['RunInstances', 'TerminateInstances']
        }
    ]
)
```

**Event Selector Types:**

| Selector Type | Scope | Use Case |
|---------------|-------|----------|
| **Management** | Control plane API calls | Track infrastructure changes |
| **Data** | Data plane operations | Monitor resource access |
| **Insights** | Unusual activity | Detect anomalies |
| **Advanced** | Fine-grained filtering | Precise event targeting |

### CloudTrail Log File Structure and Format

**S3 Storage Organization:**

```
s3://cloudtrail-bucket/
├── AWSLogs/
│   └── ACCOUNT_ID/
│       └── CloudTrail/
│           ├── us-east-1/
│           │   ├── 2024/
│           │   │   └── 01/
│           │   │       └── 15/
│           │   │           └── 123456789012_CloudTrail_us-east-1_20240115T103045Z_ABC123.json.gz
│           │   └── ...
│           ├── eu-west-1/
│           │   └── ...
│           └── us-west-2/
│               └── ...
└── ...
```

**Log File Structure:**

```json
{
  "Records": [
    {
      "eventVersion": "1.08",
      "userIdentity": {
        "type": "IAMUser",
        "principalId": "AIDACKCEVSQ6C2EXAMPLE",
        "arn": "arn:aws:iam::123456789012:user/alice",
        "accountId": "123456789012",
        "userName": "alice"
      },
      "eventTime": "2024-01-15T10:30:45Z",
      "eventSource": "ec2.amazonaws.com",
      "eventName": "RunInstances",
      "awsRegion": "us-east-1",
      "sourceIPAddress": "203.0.113.12",
      "userAgent": "aws-cli/2.0.0",
      "requestParameters": { },
      "responseElements": { },
      "requestID": "12345678-1234-1234-1234-123456789012",
      "eventID": "1-5e1f2b3c-4d5e6f7g8h9i0j1k2l3m",
      "eventType": "AwsApiCall",
      "recipientAccountId": "123456789012"
    }
  ]
}
```

**Key Fields Explained:**

| Field | Description | Example |
|-------|-------------|---------|
| **eventVersion** | Log format version | 1.08 |
| **userIdentity** | Who made the call (IAM user, role, root account, federated user, service) | IAMUser |
| **eventTime** | When the call was made (UTC ISO 8601) | 2024-01-15T10:30:45Z |
| **eventSource** | AWS service that received the call | ec2.amazonaws.com |
| **eventName** | The API action performed | RunInstances |
| **awsRegion** | AWS region where event occurred | us-east-1 |
| **sourceIPAddress** | IP address of the caller | 203.0.113.12 |
| **userAgent** | Client that made the request | aws-cli/2.0.0 |
| **requestParameters** | Parameters sent to the API | {instance configuration} |
| **responseElements** | Response returned by the API | {created resources} |
| **requestID** | Unique request identifier | 12345678... |
| **eventID** | Unique event identifier | 1-5e1f2b3c... |
| **errorCode** | Error code if request failed | ValidationException |
| **errorMessage** | Error description if request failed | "Invalid parameter" |
| **resources** | ARNs of resources affected | [instance ARN] |

### CloudTrail Pricing Model

**Cost Components:**

```
Management Events: $2.00 per 100,000 events
Data Events: $1.00 per 100,000 events
Insights Events: $5.00 per insight event
```

**Monthly Cost Calculation (Example):**

High-volume production environment:

```
Management Events:
- 500 API calls/day × 30 days = 15,000 events/month
- Cost: (15,000 ÷ 100,000) × $2.00 = $0.30

Data Events (S3 access):
- 1 million object accesses/day × 30 days = 30M events/month
- Cost: (30,000,000 ÷ 100,000) × $1.00 = $300.00

Insights Events:
- 2 anomalies detected × $5.00 = $10.00

S3 Storage:
- 100GB stored × $0.023/GB = $2.30

Total: ~$312.60/month
```

---

## Detailed Examples

### Example 1: Comprehensive CloudTrail Setup with S3, CloudWatch Logs, and Notifications

**Scenario**: Enterprise needs complete API tracking with centralized logging, real-time monitoring, and automated alerts for suspicious activities.

```python
import boto3
import json
from datetime import datetime

class ComprehensiveCloudTrailSetup:
    def __init__(self):
        self.cloudtrail = boto3.client('cloudtrail')
        self.s3 = boto3.client('s3')
        self.logs = boto3.client('logs')
        self.sns = boto3.client('sns')
        self.iam = boto3.client('iam')
    
    def create_s3_bucket_for_cloudtrail(self, bucket_name):
        """Create S3 bucket with appropriate permissions for CloudTrail"""
        
        # Create bucket
        self.s3.create_bucket(
            Bucket=bucket_name,
            CreateBucketConfiguration={'LocationConstraint': 'us-east-1'}
        )
        
        # Block public access
        self.s3.put_public_access_block(
            Bucket=bucket_name,
            PublicAccessBlockConfiguration={
                'BlockPublicAcls': True,
                'IgnorePublicAcls': True,
                'BlockPublicPolicy': True,
                'RestrictPublicBuckets': True
            }
        )
        
        # Enable versioning for compliance
        self.s3.put_bucket_versioning(
            Bucket=bucket_name,
            VersioningConfiguration={'Status': 'Enabled'}
        )
        
        # Enable encryption
        self.s3.put_bucket_encryption(
            Bucket=bucket_name,
            ServerSideEncryptionConfiguration={
                'Rules': [
                    {
                        'ApplyServerSideEncryptionByDefault': {
                            'SSEAlgorithm': 'AES256'
                        }
                    }
                ]
            }
        )
        
        # Create lifecycle policy (archive old logs)
        self.s3.put_bucket_lifecycle_configuration(
            Bucket=bucket_name,
            LifecycleConfiguration={
                'Rules': [
                    {
                        'Id': 'ArchiveOldCloudTrailLogs',
                        'Status': 'Enabled',
                        'Transitions': [
                            {
                                'Days': 90,
                                'StorageClass': 'GLACIER'
                            }
                        ],
                        'Expiration': {
                            'Days': 2555  # 7 years for compliance
                        }
                    }
                ]
            }
        )
        
        # Add bucket policy for CloudTrail
        bucket_policy = {
            'Version': '2012-10-17',
            'Statement': [
                {
                    'Sid': 'AWSCloudTrailAclCheck',
                    'Effect': 'Allow',
                    'Principal': {
                        'Service': 'cloudtrail.amazonaws.com'
                    },
                    'Action': 's3:GetBucketAcl',
                    'Resource': f'arn:aws:s3:::{bucket_name}',
                    'Condition': {
                        'StringEquals': {
                            'AWS:SourceAccount': '123456789012'
                        }
                    }
                },
                {
                    'Sid': 'AWSCloudTrailWrite',
                    'Effect': 'Allow',
                    'Principal': {
                        'Service': 'cloudtrail.amazonaws.com'
                    },
                    'Action': 's3:PutObject',
                    'Resource': f'arn:aws:s3:::{bucket_name}/AWSLogs/*',
                    'Condition': {
                        'StringEquals': {
                            'x-amz-acl': 'bucket-owner-full-control',
                            'AWS:SourceAccount': '123456789012'
                        }
                    }
                }
            ]
        }
        
        self.s3.put_bucket_policy(
            Bucket=bucket_name,
            Policy=json.dumps(bucket_policy)
        )
        
        print(f"S3 bucket created and configured: {bucket_name}")
    
    def create_cloudwatch_log_group(self, log_group_name):
        """Create CloudWatch Logs group for real-time event streaming"""
        
        try:
            self.logs.create_log_group(logGroupName=log_group_name)
        except self.logs.exceptions.ResourceAlreadyExistsException:
            pass
        
        # Set retention policy (30 days)
        self.logs.put_retention_policy(
            logGroupName=log_group_name,
            retentionInDays=30
        )
        
        print(f"CloudWatch Logs group created: {log_group_name}")
    
    def create_iam_role_for_cloudtrail_logs(self):
        """Create IAM role for CloudTrail to deliver logs to CloudWatch"""
        
        assume_role_policy = {
            'Version': '2012-10-17',
            'Statement': [
                {
                    'Effect': 'Allow',
                    'Principal': {
                        'Service': 'cloudtrail.amazonaws.com'
                    },
                    'Action': 'sts:AssumeRole'
                }
            ]
        }
        
        role_name = 'cloudtrail-cloudwatch-logs-role'
        
        try:
            response = self.iam.create_role(
                RoleName=role_name,
                AssumeRolePolicyDocument=json.dumps(assume_role_policy),
                Description='Role for CloudTrail to deliver logs to CloudWatch'
            )
            role_arn = response['Role']['Arn']
        except self.iam.exceptions.EntityAlreadyExistsException:
            role_arn = f'arn:aws:iam::123456789012:role/{role_name}'
        
        # Attach policy
        policy = {
            'Version': '2012-10-17',
            'Statement': [
                {
                    'Effect': 'Allow',
                    'Action': [
                        'logs:CreateLogStream',
                        'logs:PutLogEvents'
                    ],
                    'Resource': 'arn:aws:logs:*:123456789012:log-group:/aws/cloudtrail/*:*'
                }
            ]
        }
        
        self.iam.put_role_policy(
            RoleName=role_name,
            PolicyName='cloudtrail-cloudwatch-logs-policy',
            PolicyDocument=json.dumps(policy)
        )
        
        print(f"IAM role created: {role_arn}")
        return role_arn
    
    def create_sns_topic_for_alerts(self, topic_name):
        """Create SNS topic for CloudTrail alerts"""
        
        response = self.sns.create_topic(Name=topic_name)
        topic_arn = response['TopicArn']
        
        # Subscribe email
        self.sns.subscribe(
            TopicArn=topic_arn,
            Protocol='email',
            Endpoint='security-team@company.com'
        )
        
        print(f"SNS topic created: {topic_arn}")
        return topic_arn
    
    def create_comprehensive_trail(self, trail_name, s3_bucket, log_group, role_arn, sns_topic_arn):
        """Create comprehensive CloudTrail with all features"""
        
        response = self.cloudtrail.create_trail(
            Name=trail_name,
            S3BucketName=s3_bucket,
            IncludeGlobalServiceEvents=True,
            IsMultiRegionTrail=True,
            EnableLogFileValidation=True,
            CloudWatchLogsLogGroupArn=f'{log_group}:*',
            CloudWatchLogsRoleArn=role_arn,
            KmsKeyId='arn:aws:kms:us-east-1:123456789012:key/12345678-1234-1234-1234-123456789012',
            Tags=[
                {'Key': 'Environment', 'Value': 'production'},
                {'Key': 'Purpose', 'Value': 'audit-compliance'},
                {'Key': 'AlertTopic', 'Value': sns_topic_arn}
            ]
        )
        
        trail_arn = response['TrailARN']
        
        # Configure event selectors for data events
        self.cloudtrail.put_event_selectors(
            TrailName=trail_name,
            AdvancedEventSelectors=[
                # Management events
                {
                    'Field': 'eventCategory',
                    'Equals': ['Management']
                },
                # S3 data events
                {
                    'Field': 'eventCategory',
                    'Equals': ['Data']
                },
                {
                    'Field': 'resources.type',
                    'Equals': ['AWS::S3::Object']
                },
                # Lambda data events
                {
                    'Field': 'eventCategory',
                    'Equals': ['Data']
                },
                {
                    'Field': 'resources.type',
                    'Equals': ['AWS::Lambda::Function']
                },
                # Insights
                {
                    'Field': 'eventCategory',
                    'Equals': ['Insights']
                }
            ]
        )
        
        # Start logging
        self.cloudtrail.start_logging(Name=trail_name)
        
        print(f"Trail created and logging enabled: {trail_arn}")
        return trail_arn
    
    def create_metric_filters_for_alerts(self, log_group_name):
        """Create metric filters for security-related API calls"""
        
        filters = [
            {
                'name': 'UnauthorizedAPICallsMetricFilter',
                'pattern': '{ ($.errorCode = "*UnauthorizedOperation") || ($.errorCode = "AccessDenied*") }',
                'metric_name': 'UnauthorizedAPICallsMetric',
                'metric_namespace': 'CloudTrailMetrics'
            },
            {
                'name': 'RootAccountUsageMetricFilter',
                'pattern': '{ $.userIdentity.type = "Root" && $.userIdentity.invokedBy NOT EXISTS && $.eventType != "AwsServiceEvent" }',
                'metric_name': 'RootAccountUsageMetric',
                'metric_namespace': 'CloudTrailMetrics'
            },
            {
                'name': 'IAMPolicyChangesMetricFilter',
                'pattern': '{ ($.eventName=DeleteGroupPolicy) || ($.eventName=DeleteRolePolicy) || ($.eventName=DeleteUserPolicy) || ($.eventName=PutGroupPolicy) || ($.eventName=PutRolePolicy) || ($.eventName=PutUserPolicy) || ($.eventName=CreatePolicy) || ($.eventName=DeletePolicy) || ($.eventName=CreatePolicyVersion) || ($.eventName=DeletePolicyVersion) || ($.eventName=AttachRolePolicy) || ($.eventName=DetachRolePolicy) || ($.eventName=AttachUserPolicy) || ($.eventName=DetachUserPolicy) || ($.eventName=AttachGroupPolicy) || ($.eventName=DetachGroupPolicy) }',
                'metric_name': 'IAMPolicyChangesMetric',
                'metric_namespace': 'CloudTrailMetrics'
            },
            {
                'name': 'NetworkACLChangesMetricFilter',
                'pattern': '{ ($.eventName = CreateNetworkAcl) || ($.eventName = CreateNetworkAclEntry) || ($.eventName = DeleteNetworkAcl) || ($.eventName = DeleteNetworkAclEntry) || ($.eventName = ReplaceNetworkAclEntry) || ($.eventName = ReplaceNetworkAclAssociation) }',
                'metric_name': 'NetworkACLChangesMetric',
                'metric_namespace': 'CloudTrailMetrics'
            },
            {
                'name': 'SecurityGroupChangesMetricFilter',
                'pattern': '{ ($.eventName = AuthorizeSecurityGroupIngress) || ($.eventName = AuthorizeSecurityGroupEgress) || ($.eventName = RevokeSecurityGroupIngress) || ($.eventName = RevokeSecurityGroupEgress) || ($.eventName = CreateSecurityGroup) || ($.eventName = DeleteSecurityGroup) }',
                'metric_name': 'SecurityGroupChangesMetric',
                'metric_namespace': 'CloudTrailMetrics'
            }
        ]
        
        for filter_config in filters:
            self.logs.put_metric_filter(
                logGroupName=log_group_name,
                filterName=filter_config['name'],
                filterPattern=filter_config['pattern'],
                metricTransformations=[
                    {
                        'metricName': filter_config['metric_name'],
                        'metricNamespace': filter_config['metric_namespace'],
                        'metricValue': '1',
                        'defaultValue': 0
                    }
                ]
            )
            
            print(f"Metric filter created: {filter_config['name']}")

# Usage
setup = ComprehensiveCloudTrailSetup()

# Create S3 bucket
setup.create_s3_bucket_for_cloudtrail('company-cloudtrail-logs')

# Create CloudWatch Logs group
setup.create_cloudwatch_log_group('/aws/cloudtrail/organization')

# Create IAM role
role_arn = setup.create_iam_role_for_cloudtrail_logs()

# Create SNS topic
sns_arn = setup.create_sns_topic_for_alerts('cloudtrail-security-alerts')

# Create comprehensive trail
setup.create_comprehensive_trail(
    'organization-trail',
    'company-cloudtrail-logs',
    'arn:aws:logs:us-east-1:123456789012:log-group:/aws/cloudtrail/organization',
    role_arn,
    sns_arn
)

# Create metric filters
setup.create_metric_filters_for_alerts('/aws/cloudtrail/organization')
```

### Example 2: Querying CloudTrail Logs with Amazon Athena for Forensic Analysis

**Scenario**: Security team needs to investigate suspected unauthorized access and analyze historical API activity.

```python
import boto3
import pandas as pd
from datetime import datetime, timedelta

class CloudTrailForensicsAnalysis:
    def __init__(self):
        self.athena = boto3.client('athena')
        self.s3 = boto3.client('s3')
    
    def create_athena_table_for_cloudtrail(self):
        """Create Athena table for querying CloudTrail logs"""
        
        create_table_query = '''
        CREATE EXTERNAL TABLE IF NOT EXISTS cloudtrail_logs (
            eventVersion STRING,
            userIdentity STRUCT<
                type: STRING,
                principalId: STRING,
                arn: STRING,
                accountId: STRING,
                invokedBy: STRING,
                accessKeyId: STRING,
                userName: STRING,
                sessionContext: STRUCT<
                    attributes: STRUCT<
                        mfaAuthenticated: STRING,
                        creationDate: STRING>,
                    sessionIssuer: STRUCT<
                        type: STRING,
                        principalId: STRING,
                        arn: STRING,
                        accountId: STRING,
                        userName: STRING>>>,
            eventTime STRING,
            eventSource STRING,
            eventName STRING,
            awsRegion STRING,
            sourceIPAddress STRING,
            userAgent STRING,
            errorCode STRING,
            errorMessage STRING,
            requestParameters STRING,
            responseElements STRING,
            additionEventData STRING,
            requestId STRING,
            eventId STRING,
            resources ARRAY<STRUCT<
                arn: STRING,
                accountId: STRING,
                type: STRING>>,
            eventType STRING,
            apiVersion STRING,
            readOnly STRING,
            recipientAccountId STRING,
            serviceEventDetails STRING,
            sharedEventId STRING,
            vpcEndpointId STRING
        )
        PARTITIONED BY (region STRING, year STRING, month STRING, day STRING)
        ROW FORMAT SERDE 'com.amazon.emr.hive.serde.CloudTrailSerde'
        STORED AS INPUTFORMAT 'com.amazon.emr.cloudtrail.CloudTrailInputFormat'
        OUTPUTFORMAT 'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat'
        LOCATION 's3://company-cloudtrail-logs/AWSLogs/'
        '''
        
        response = self.athena.start_query_execution(
            QueryString=create_table_query,
            QueryExecutionContext={'Database': 'default'},
            ResultConfiguration={'OutputLocation': 's3://athena-results/'},
            WorkGroup='primary'
        )
        
        print(f"Athena table creation query: {response['QueryExecutionId']}")
    
    def query_unauthorized_api_calls(self, hours=24):
        """Find all unauthorized API calls in past N hours"""
        
        query = f'''
        SELECT
            eventTime,
            userIdentity.type as identity_type,
            userIdentity.arn,
            eventSource,
            eventName,
            errorCode,
            errorMessage,
            sourceIPAddress,
            awsRegion,
            resources
        FROM cloudtrail_logs
        WHERE
            eventTime >= format_datetime(current_timestamp - interval '{hours}' hour, 'yyyy-MM-dd''T''HH:mm:ss''Z')
            AND (errorCode = 'AccessDenied' OR errorCode like 'Unauthorized%')
            AND eventType = 'AwsApiCall'
        ORDER BY eventTime DESC
        LIMIT 1000
        '''
        
        response = self.athena.start_query_execution(
            QueryString=query,
            QueryExecutionContext={'Database': 'default'},
            ResultConfiguration={'OutputLocation': 's3://athena-results/'},
            WorkGroup='primary'
        )
        
        return response['QueryExecutionId']
    
    def query_root_account_usage(self, days=30):
        """Track root account usage (security risk)"""
        
        query = f'''
        SELECT
            eventTime,
            eventName,
            eventSource,
            sourceIPAddress,
            userAgent,
            requestParameters,
            awsRegion
        FROM cloudtrail_logs
        WHERE
            userIdentity.type = 'Root'
            AND eventTime >= format_datetime(current_timestamp - interval '{days}' day, 'yyyy-MM-dd''T''HH:mm:ss''Z')
            AND eventType = 'AwsApiCall'
            AND userIdentity.invokedBy is NULL
        ORDER BY eventTime DESC
        '''
        
        response = self.athena.start_query_execution(
            QueryString=query,
            QueryExecutionContext={'Database': 'default'},
            ResultConfiguration={'OutputLocation': 's3://athena-results/'},
            WorkGroup='primary'
        )
        
        return response['QueryExecutionId']
    
    def query_iam_policy_changes(self, hours=72):
        """Audit IAM policy modifications"""
        
        query = f'''
        SELECT
            eventTime,
            userIdentity.arn,
            eventName,
            sourceIPAddress,
            requestParameters,
            responseElements,
            awsRegion
        FROM cloudtrail_logs
        WHERE
            eventSource = 'iam.amazonaws.com'
            AND eventName IN (
                'PutUserPolicy',
                'PutGroupPolicy',
                'PutRolePolicy',
                'AttachUserPolicy',
                'AttachGroupPolicy',
                'AttachRolePolicy',
                'DetachUserPolicy',
                'DetachGroupPolicy',
                'DetachRolePolicy',
                'CreatePolicy',
                'CreatePolicyVersion',
                'DeletePolicy',
                'DeletePolicyVersion'
            )
            AND eventTime >= format_datetime(current_timestamp - interval '{hours}' hour, 'yyyy-MM-dd''T''HH:mm:ss''Z')
        ORDER BY eventTime DESC
        '''
        
        response = self.athena.start_query_execution(
            QueryString=query,
            QueryExecutionContext={'Database': 'default'},
            ResultConfiguration={'OutputLocation': 's3://athena-results/'},
            WorkGroup='primary'
        )
        
        return response['QueryExecutionId']
    
    def query_api_calls_by_user(self, username, hours=24):
        """Investigate specific user's activity"""
        
        query = f'''
        SELECT
            eventTime,
            eventName,
            eventSource,
            sourceIPAddress,
            awsRegion,
            errorCode,
            resources,
            requestParameters
        FROM cloudtrail_logs
        WHERE
            userIdentity.userName = '{username}'
            AND eventTime >= format_datetime(current_timestamp - interval '{hours}' hour, 'yyyy-MM-dd''T''HH:mm:ss''Z')
        ORDER BY eventTime DESC
        '''
        
        response = self.athena.start_query_execution(
            QueryString=query,
            QueryExecutionContext={'Database': 'default'},
            ResultConfiguration={'OutputLocation': 's3://athena-results/'},
            WorkGroup='primary'
        )
        
        return response['QueryExecutionId']
    
    def query_failed_api_calls(self, hours=24):
        """Find all API call failures (could indicate misconfiguration or attack)"""
        
        query = f'''
        SELECT
            eventTime,
            userIdentity.arn,
            eventName,
            errorCode,
            errorMessage,
            sourceIPAddress,
            awsRegion,
            count(*) as failure_count
        FROM cloudtrail_logs
        WHERE
            errorCode is NOT NULL
            AND eventTime >= format_datetime(current_timestamp - interval '{hours}' hour, 'yyyy-MM-dd''T''HH:mm:ss''Z')
        GROUP BY
            eventTime,
            userIdentity.arn,
            eventName,
            errorCode,
            errorMessage,
            sourceIPAddress,
            awsRegion
        HAVING count(*) > 5
        ORDER BY failure_count DESC
        '''
        
        response = self.athena.start_query_execution(
            QueryString=query,
            QueryExecutionContext={'Database': 'default'},
            ResultConfiguration={'OutputLocation': 's3://athena-results/'},
            WorkGroup='primary'
        )
        
        return response['QueryExecutionId']
    
    def query_api_calls_from_suspicious_ip(self, suspect_ip, hours=72):
        """Investigate all API calls from specific IP address"""
        
        query = f'''
        SELECT
            eventTime,
            userIdentity.type,
            userIdentity.arn,
            eventName,
            eventSource,
            awsRegion,
            resources,
            requestParameters,
            responseElements
        FROM cloudtrail_logs
        WHERE
            sourceIPAddress = '{suspect_ip}'
            AND eventTime >= format_datetime(current_timestamp - interval '{hours}' hour, 'yyyy-MM-dd''T''HH:mm:ss''Z')
        ORDER BY eventTime DESC
        '''
        
        response = self.athena.start_query_execution(
            QueryString=query,
            QueryExecutionContext={'Database': 'default'},
            ResultConfiguration={'OutputLocation': 's3://athena-results/'},
            WorkGroup='primary'
        )
        
        return response['QueryExecutionId']

# Usage
analysis = CloudTrailForensicsAnalysis()

# Create Athena table
analysis.create_athena_table_for_cloudtrail()

# Run forensic queries
print("Query 1 - Unauthorized API calls:")
print(analysis.query_unauthorized_api_calls(hours=24))

print("\nQuery 2 - Root account usage:")
print(analysis.query_root_account_usage(days=30))

print("\nQuery 3 - IAM policy changes:")
print(analysis.query_iam_policy_changes(hours=72))

print("\nQuery 4 - User activity:")
print(analysis.query_api_calls_by_user('alice', hours=24))

print("\nQuery 5 - Failed API calls:")
print(analysis.query_failed_api_calls(hours=24))

print("\nQuery 6 - Suspicious IP:")
print(analysis.query_api_calls_from_suspicious_ip('203.0.113.100', hours=72))
```

### Example 3: Setting Up Security Alerts with EventBridge and Lambda

**Scenario**: Automatically alert security team on suspicious activities detected in real-time.

```python
import boto3
import json

class CloudTrailSecurityAlerting:
    def __init__(self):
        self.eventbridge = boto3.client('events')
        self.sns = boto3.client('sns')
        self.lambda_client = boto3.client('lambda')
        self.iam = boto3.client('iam')
    
    def create_lambda_role(self):
        """Create IAM role for Lambda function"""
        
        assume_role_policy = {
            'Version': '2012-10-17',
            'Statement': [
                {
                    'Effect': 'Allow',
                    'Principal': {
                        'Service': 'lambda.amazonaws.com'
                    },
                    'Action': 'sts:AssumeRole'
                }
            ]
        }
        
        role_name = 'cloudtrail-security-alert-lambda-role'
        
        try:
            response = self.iam.create_role(
                RoleName=role_name,
                AssumeRolePolicyDocument=json.dumps(assume_role_policy),
                Description='Role for CloudTrail security alert Lambda'
            )
            role_arn = response['Role']['Arn']
        except self.iam.exceptions.EntityAlreadyExistsException:
            role_arn = f'arn:aws:iam::123456789012:role/{role_name}'
        
        # Attach policy for SNS and CloudWatch Logs
        policy = {
            'Version': '2012-10-17',
            'Statement': [
                {
                    'Effect': 'Allow',
                    'Action': [
                        'sns:Publish',
                        'logs:CreateLogGroup',
                        'logs:CreateLogStream',
                        'logs:PutLogEvents'
                    ],
                    'Resource': '*'
                }
            ]
        }
        
        self.iam.put_role_policy(
            RoleName=role_name,
            PolicyName='lambda-policy',
            PolicyDocument=json.dumps(policy)
        )
        
        return role_arn
    
    def create_security_alert_lambda(self, role_arn):
        """Create Lambda function for security alerts"""
        
        lambda_code = '''
import boto3
import json
from datetime import datetime

sns = boto3.client('sns')
sns_topic_arn = 'arn:aws:sns:us-east-1:123456789012:cloudtrail-security-alerts'

def lambda_handler(event, context):
    """
    Process CloudTrail events from EventBridge
    and send security alerts
    """
    
    detail = event['detail']
    event_name = detail.get('eventName')
    event_time = detail.get('eventTime')
    principal_id = detail.get('userIdentity', {}).get('principalId')
    source_ip = detail.get('sourceIPAddress')
    aws_region = detail.get('awsRegion')
    error_code = detail.get('errorCode')
    resource_type = detail.get('resources', [{}])[0].get('type')
    
    # Determine alert severity
    severity = 'INFO'
    alert_title = f"CloudTrail Event: {event_name}"
    
    if error_code:
        if error_code in ['AccessDenied', 'UnauthorizedOperation']:
            severity = 'WARNING'
            alert_title = f"SECURITY ALERT: Unauthorized API Call - {event_name}"
    
    # Special cases for high-severity events
    critical_events = [
        'CreateAccessKey',
        'CreateUser',
        'AttachUserPolicy',
        'PutUserPolicy',
        'CreatePolicy',
        'DeleteTrail',
        'StopLogging',
        'DeleteBucket'
    ]
    
    if event_name in critical_events:
        severity = 'CRITICAL'
        alert_title = f"CRITICAL SECURITY EVENT: {event_name}"
    
    # Build alert message
    message = f"""
CloudTrail Security Alert
========================

Severity: {severity}
Event: {event_name}
Time: {event_time}
User: {principal_id}
Source IP: {source_ip}
Region: {aws_region}
Resource Type: {resource_type}

{f"Error: {error_code}" if error_code else ""}

Details:
{json.dumps(detail, indent=2, default=str)}

Action: Investigate this activity
"""
    
    try:
        sns.publish(
            TopicArn=sns_topic_arn,
            Subject=alert_title,
            Message=message
        )
        print(f"Alert sent: {alert_title}")
    except Exception as e:
        print(f"Error sending alert: {str(e)}")
    
    return {
        'statusCode': 200,
        'body': json.dumps('Alert processed')
    }
'''
        
        try:
            response = self.lambda_client.create_function(
                FunctionName='cloudtrail-security-alert',
                Runtime='python3.11',
                Role=role_arn,
                Handler='index.lambda_handler',
                Code={'ZipFile': lambda_code.encode('utf-8')},
                Description='Send security alerts for CloudTrail events',
                Timeout=60
            )
            
            function_arn = response['FunctionArn']
            print(f"Lambda function created: {function_arn}")
            return function_arn
        except self.lambda_client.exceptions.ResourceConflictException:
            # Function exists, get its ARN
            response = self.lambda_client.get_function(FunctionName='cloudtrail-security-alert')
            return response['Configuration']['FunctionArn']
    
    def create_eventbridge_rules(self, lambda_arn):
        """Create EventBridge rules for security events"""
        
        # Rule 1: Critical Security Events
        self.eventbridge.put_rule(
            Name='cloudtrail-critical-security-events',
            EventPattern=json.dumps({
                'source': ['aws.cloudtrail'],
                'detail-type': ['AWS API Call via CloudTrail'],
                'detail': {
                    'eventName': [
                        'CreateAccessKey',
                        'CreateUser',
                        'AttachUserPolicy',
                        'PutUserPolicy',
                        'CreatePolicy',
                        'DeleteTrail',
                        'StopLogging',
                        'DeleteBucket',
                        'DeleteDBInstance',
                        'ModifyDBInstance'
                    ]
                }
            }),
            State='ENABLED',
            Description='Alert on critical security events'
        )
        
        # Rule 2: Unauthorized API Calls
        self.eventbridge.put_rule(
            Name='cloudtrail-unauthorized-api-calls',
            EventPattern=json.dumps({
                'source': ['aws.cloudtrail'],
                'detail-type': ['AWS API Call via CloudTrail'],
                'detail': {
                    'errorCode': ['AccessDenied', 'UnauthorizedOperation']
                }
            }),
            State='ENABLED',
            Description='Alert on unauthorized API calls'
        )
        
        # Rule 3: Root Account Activity
        self.eventbridge.put_rule(
            Name='cloudtrail-root-account-usage',
            EventPattern=json.dumps({
                'source': ['aws.cloudtrail'],
                'detail-type': ['AWS API Call via CloudTrail'],
                'detail': {
                    'userIdentity': {
                        'type': ['Root']
                    },
                    'eventType': ['AwsApiCall']
                }
            }),
            State='ENABLED',
            Description='Alert on root account usage'
        )
        
        # Rule 4: Console Login Failures
        self.eventbridge.put_rule(
            Name='cloudtrail-console-login-failures',
            EventPattern=json.dumps({
                'source': ['aws.signin'],
                'detail-type': ['AWS Sign In via CloudTrail'],
                'detail': {
                    'eventName': ['ConsoleLogin'],
                    'errorCode': ['InvalidParameterValue']
                }
            }),
            State='ENABLED',
            Description='Alert on multiple failed console login attempts'
        )
        
        # Add Lambda targets
        rules = [
            'cloudtrail-critical-security-events',
            'cloudtrail-unauthorized-api-calls',
            'cloudtrail-root-account-usage',
            'cloudtrail-console-login-failures'
        ]
        
        for rule in rules:
            self.eventbridge.put_targets(
                Rule=rule,
                Targets=[
                    {
                        'Arn': lambda_arn,
                        'Id': '1'
                    }
                ]
            )
            
            # Grant EventBridge permission to invoke Lambda
            self.lambda_client.add_permission(
                FunctionName='cloudtrail-security-alert',
                StatementId=f'{rule}-invoke',
                Action='lambda:InvokeFunction',
                Principal='events.amazonaws.com',
                SourceArn=f'arn:aws:events:us-east-1:123456789012:rule/{rule}'
            )
            
            print(f"EventBridge rule created: {rule}")

# Usage
alerting = CloudTrailSecurityAlerting()

# Create Lambda role
role_arn = alerting.create_lambda_role()

# Create Lambda function
lambda_arn = alerting.create_security_alert_lambda(role_arn)

# Create EventBridge rules
alerting.create_eventbridge_rules(lambda_arn)
```

---

## Top 5 Interview Questions

### Question 1: Explain CloudTrail Architecture and Differentiate Between Management and Data Events

**Scenario**: "Walk through the CloudTrail architecture, explain the difference between management and data events, and discuss use cases for each."

**Answer Structure:**

**CloudTrail Architecture:**

CloudTrail operates as a multi-layered audit system:

Layer 1 - Event Generation
├─ AWS API calls initiated
├─ User actions triggered
└─ Resource state changes

Layer 2 - Event Capture
├─ CloudTrail intercepts all API calls
├─ Metadata collected (who, what, when, where)
├─ Events classified (Management, Data, Insights)
└─ Event enrichment (user context, IP, user agent)

Layer 3 - Event Processing
├─ Event filtering (advanced event selectors)
├─ Aggregation (multiple events per second)
├─ Validation (cryptographic signing)
└─ Delivery (S3, CloudWatch Logs, EventBridge)

Layer 4 - Storage & Analysis
├─ S3 bucket (log files, archival)
├─ CloudWatch Logs (real-time streaming)
├─ Athena (SQL queries)
├─ Security Hub (analysis)
└─ Third-party SIEM integration

**Management Events vs Data Events:**

**Management Events (Control Plane):**

Definition: API calls that manage AWS resources - determine WHO can do WHAT

Examples:
- Launching EC2 instances (`RunInstances`)
- Creating IAM users (`CreateUser`)
- Attaching policies (`AttachUserPolicy`)
- Modifying security groups (`ModifySecurityGroup`)
- Creating databases (`CreateDBInstance`)

Characteristics:
- Included by default in all trails
- Lower volume than data events
- Stored indefinitely (configurable retention)
- Cost: $2.00 per 100,000 events
- Enable through CloudTrail console - no explicit action needed

Example Event:

```json
{
  "eventName": "AttachUserPolicy",
  "eventSource": "iam.amazonaws.com",
  "userIdentity": {
    "type": "IAMUser",
    "userName": "admin"
  },
  "eventTime": "2024-01-15T10:30:45Z",
  "requestParameters": {
    "userName": "bob",
    "policyArn": "arn:aws:iam::aws:policy/ReadOnlyAccess"
  }
}
```

**Data Events (Data Plane):**

Definition: API calls that actually ACCESS or MODIFY data within resources - determine WHAT data can be accessed

Examples:
- S3 GetObject, PutObject, DeleteObject
- Lambda Invoke
- DynamoDB GetItem, PutItem, UpdateItem, Query, Scan
- RDS Query execution

Characteristics:
- NOT included by default (must be explicitly enabled)
- Higher volume than management events
- Can generate millions of events per day
- Cost: $1.00 per 100,000 events (cheaper but more volume)
- Enables granular access tracking and compliance auditing

Example Event:

```json
{
  "eventName": "GetObject",
  "eventSource": "s3.amazonaws.com",
  "userIdentity": {
    "type": "IAMUser",
    "userName": "application"
  },
  "eventTime": "2024-01-15T10:35:20Z",
  "requestParameters": {
    "bucketName": "sensitive-documents",
    "key": "payroll-2024.xlsx"
  },
  "s3": {
    "bucket": {
      "name": "sensitive-documents"
    },
    "object": {
      "key": "payroll-2024.xlsx",
      "size": 1024576
    }
  }
}
```

**Comparison Matrix:**

| Aspect | Management Events | Data Events |
|--------|-------------------|------------|
| **Scope** | Resource management | Data access |
| **Enabled by Default** | ✅ Yes | ❌ No |
| **Volume** | Medium | High |
| **Cost** | $2/100K events | $1/100K events |
| **Examples** | LaunchInstance, CreateUser, AttachPolicy | GetObject, PutObject, Invoke |
| **Use Cases** | Compliance, audit trail, change tracking | Security monitoring, data access audit |
| **Typical Events/Day** | 10,000-100,000 | 1,000,000-100,000,000 |
| **Best For** | "Who changed the infrastructure?" | "Who accessed the data?" |

**Use Cases by Event Type:**

**Management Events Use Cases:**

1. **Change Management & Compliance:**
   - Track who created/deleted resources
   - Audit infrastructure changes
   - Meet regulatory requirements (SOX, HIPAA)

2. **Security & Identity:**
   - IAM policy changes
   - User/role creation
   - Cross-account access modifications

3. **Troubleshooting:**
   - "Who stopped this EC2 instance?"
   - "When was this security group modified?"
   - "What changed the Lambda function?"

4. **Cost Analysis:**
   - Track resource creation
   - Identify who launched expensive resources

**Data Events Use Cases:**

1. **Data Security & Compliance:**
   - HIPAA: audit access to PHI
   - PCI-DSS: track credit card data access
   - GDPR: log personal data access

2. **Insider Threat Detection:**
   - Unusual S3 object access patterns
   - Bulk data downloads
   - Access from unexpected locations

3. **Performance & Operations:**
   - Lambda invocation patterns
   - Database query performance
   - S3 access optimization

4. **Forensic Investigation:**
   - "Which user accessed this sensitive file?"
   - "When was this database queried?"
   - Complete audit trail

**Architecture Diagram:**

```
AWS Service → API Call
    ↓
CloudTrail Capture
    ├─ Management Event?
    │  └─ Track resource changes
    │     └─ Store in S3 (default)
    │        └─ Cost: $2/100K
    │
    └─ Data Event?
       └─ Track data access (if enabled)
          └─ Store in S3
             └─ Cost: $1/100K

Delivery Options:
├─ S3 Bucket (archival)
├─ CloudWatch Logs (real-time)
├─ EventBridge (routing)
└─ Insights (anomalies)

Analysis:
├─ CloudTrail Console (90 days)
├─ Athena (SQL queries)
├─ Security Hub (patterns)
└─ Custom applications
```

**Best Practices:**

1. **Enable both types for comprehensive auditing:**
   - Management events for change tracking
   - Data events for sensitive resources only

2. **Use advanced event selectors for granular control:**
   - Enable data events only for critical S3 buckets
   - Monitor specific Lambda functions
   - Focus on high-value resources

3. **Separate storage strategies:**
   - Real-time alerts (CloudWatch Logs)
   - Historical analysis (S3 with Athena)
   - Long-term retention (S3 Glacier)

4. **Cost optimization:**
   - Enable data events only where necessary
   - Exclude verbose services
   - Use filtering to reduce volume

---

### Question 2: Design Organization-Wide CloudTrail Logging Strategy

**Scenario**: "Design a CloudTrail logging strategy for a large organization with 50+ AWS accounts across multiple regions. Include centralized logging, multi-account setup, compliance requirements, and cost optimization."

**Answer Structure:**

**Organization-Wide Requirements:**

Challenges:
- Multiple AWS accounts (50+)
- Global regions (US, EU, APAC)
- Different business units
- Compliance requirements (HIPAA, PCI-DSS, SOX)
- Cost management
- Real-time alerting

**Architecture Design:**

```
Management Account (Centralized Logging Hub)
    ↓
Organization Trail
    ├─ Member Account A ──→ Events
    ├─ Member Account B ──→ Events
    ├─ Member Account C ──→ Events
    └─ ... 50 more accounts
    ↓
Central S3 Bucket (Log Storage)
├─ Organized by account/region/date
├─ Encryption at rest
├─ Versioning enabled
├─ Access logging
├─ Lifecycle policies
└─ Log file validation
    ↓
Processing Layer
├─ EventBridge (real-time alerting)
├─ CloudWatch Logs (streaming)
├─ Lambda (enrichment/processing)
└─ SNS (notifications)
    ↓
Analysis & Reporting
├─ Athena (forensic queries)
├─ QuickSight (dashboards)
├─ Security Hub (compliance)
└─ Custom compliance reports
```

**Implementation:**

```python
class OrganizationCloudTrailStrategy:
    def __init__(self):
        self.cloudtrail = boto3.client('cloudtrail')
        self.s3 = boto3.client('s3')
        self.organizations = boto3.client('organizations')
        self.logs = boto3.client('logs')
    
    def setup_central_s3_bucket(self):
        """Create highly secure central logging bucket"""
        
        bucket_name = 'organization-cloudtrail-central-logs'
        
        # Create with encryption
        self.s3.create_bucket(
            Bucket=bucket_name,
            CreateBucketConfiguration={'LocationConstraint': 'us-east-1'}
        )
        
        # Enable versioning (for compliance)
        self.s3.put_bucket_versioning(
            Bucket=bucket_name,
            VersioningConfiguration={'Status': 'Enabled'}
        )
        
        # Enable MFA delete (critical records protection)
        self.s3.put_bucket_versioning(
            Bucket=bucket_name,
            VersioningConfiguration={
                'Status': 'Enabled',
                'MFADelete': 'Enabled'
            }
        )
        
        # Enable encryption
        self.s3.put_bucket_encryption(
            Bucket=bucket_name,
            ServerSideEncryptionConfiguration={
                'Rules': [{
                    'ApplyServerSideEncryptionByDefault': {
                        'SSEAlgorithm': 'aws:kms',
                        'KmsKeyId': 'arn:aws:kms:us-east-1:123456789012:key/xxxxx'
                    },
                    'BucketKeyEnabled': True
                }]
            }
        )
        
        # Block all public access
        self.s3.put_public_access_block(
            Bucket=bucket_name,
            PublicAccessBlockConfiguration={
                'BlockPublicAcls': True,
                'IgnorePublicAcls': True,
                'BlockPublicPolicy': True,
                'RestrictPublicBuckets': True
            }
        )
        
        # Enable access logging (audit trail for CloudTrail bucket itself)
        self.s3.put_bucket_logging(
            Bucket=bucket_name,
            BucketLoggingStatus={
                'LoggingEnabled': {
                    'TargetBucket': 'organization-cloudtrail-access-logs',
                    'TargetPrefix': 'access-logs/'
                }
            }
        )
        
        # Lifecycle policy (cost optimization)
        self.s3.put_bucket_lifecycle_configuration(
            Bucket=bucket_name,
            LifecycleConfiguration={
                'Rules': [
                    {
                        'Id': 'ArchiveOldLogs',
                        'Status': 'Enabled',
                        'Transitions': [
                            {
                                'Days': 90,
                                'StorageClass': 'STANDARD_IA'
                            },
                            {
                                'Days': 180,
                                'StorageClass': 'INTELLIGENT_TIERING'
                            },
                            {
                                'Days': 365,
                                'StorageClass': 'GLACIER'
                            }
                        ],
                        'Expiration': {
                            'Days': 2555  # 7 years
                        },
                        'NoncurrentVersionTransitions': [
                            {
                                'NoncurrentDays': 30,
                                'StorageClass': 'GLACIER'
                            }
                        ]
                    }
                ]
            }
        )
        
        return bucket_name
    
    def create_organization_trail(self, bucket_name):
        """Create organization-wide trail"""
        
        response = self.cloudtrail.create_trail(
            Name='organization-master-trail',
            S3BucketName=bucket_name,
            IsOrganizationTrail=True,
            IsMultiRegionTrail=True,
            IncludeGlobalServiceEvents=True,
            EnableLogFileValidation=True,
            Tags=[
                {'Key': 'Type', 'Value': 'Organization'},
                {'Key': 'Purpose', 'Value': 'Audit'},
                {'Key': 'CostCenter', 'Value': 'Security'}
            ]
        )
        
        trail_arn = response['TrailARN']
        
        # Configure advanced event selectors
        self.cloudtrail.put_event_selectors(
            TrailName='organization-master-trail',
            AdvancedEventSelectors=[
                # All management events
                {
                    'Field': 'eventCategory',
                    'Equals': ['Management']
                },
                # Critical data events
                {
                    'Field': 'eventCategory',
                    'Equals': ['Data']
                },
                {
                    'Field': 'resources.type',
                    'Equals': ['AWS::S3::Object']
                },
                # Insights (anomaly detection)
                {
                    'Field': 'eventCategory',
                    'Equals': ['Insights']
                }
            ]
        )
        
        # Start logging
        self.cloudtrail.start_logging(Name='organization-master-trail')
        
        print(f"Organization trail created: {trail_arn}")
        return trail_arn
    
    def setup_centralized_cloudwatch_logs(self):
        """Setup CloudWatch Logs for real-time event streaming"""
        
        log_group_name = '/aws/cloudtrail/organization'
        
        try:
            self.logs.create_log_group(logGroupName=log_group_name)
        except self.logs.exceptions.ResourceAlreadyExistsException:
            pass
        
        # 90-day retention for compliance
        self.logs.put_retention_policy(
            logGroupName=log_group_name,
            retentionInDays=90
        )
        
        # Add resource-based policy for cross-account access
        policy = {
            'Version': '2012-10-17',
            'Statement': [{
                'Effect': 'Allow',
                'Principal': {
                    'Service': 'cloudtrail.amazonaws.com'
                },
                'Action': [
                    'logs:CreateLogStream',
                    'logs:PutLogEvents'
                ],
                'Resource': '*'
            }]
        }
        
        # Note: In real implementation, would use put_resource_policy
        
        return log_group_name
    
    def setup_account_delegation(self):
        """Setup per-account CloudTrail monitoring"""
        
        # Each member account can have local trail
        # that feeds to organization trail
        
        # This is done automatically with organization trail
        # but document for teams:
        
        guidance = """
        Organization Trail Setup for Member Accounts:
        
        1. Organization Trail automatically captures:
           - All API calls from member accounts
           - All regions
           - All users and roles
        
        2. Member accounts CAN create additional trails for:
           - Local compliance requirements
           - Team-specific monitoring
           - Custom analysis
        
        3. Permissions required (via SCP):
           - cloudtrail:PutEventSelectors
           - cloudtrail:StartLogging
           - s3:PutObject (to central bucket)
        
        4. Cost implications:
           - Organization trail: Central cost
           - Additional trails: Per-account cost
        """
        
        return guidance

# Usage
strategy = OrganizationCloudTrailStrategy()
bucket = strategy.setup_central_s3_bucket()
trail_arn = strategy.create_organization_trail(bucket)
logs = strategy.setup_centralized_cloudwatch_logs()
guidance = strategy.setup_account_delegation()
```

**Cost Optimization Strategy:**

Monthly Cost Estimates (50 accounts):

Management Events:
- Per account per day: ~1,000 API calls
- 50 accounts × 1,000/day × 30 = 1.5M events/month
- Cost: (1.5M ÷ 100K) × $2 = $30/month

Data Events (S3):
- If enabled for all accounts: High volume
- If enabled selectively: Control cost
- Recommended: Enable for critical buckets only
- Example: 5M S3 events/month = $50/month

Total Estimated: $80-150/month

Cost Control Measures:
1. Use advanced event selectors to filter events
2. Enable data events only for critical resources
3. Exclude verbose/low-risk services
4. Implement S3 lifecycle policies
5. Use S3 Intelligent-Tiering

**Compliance Configuration:**

For HIPAA, PCI-DSS, SOX:

```python
# Enable features for compliance
compliance_config = {
    'EnableLogFileValidation': True,  # Detect tampering
    'CloudWatchLogsRoleArn': 'arn:...',  # Real-time streaming
    'CloudWatchLogsLogGroupArn': 'arn:...',  # CWL integration
    'IncludeGlobalServiceEvents': True,  # IAM events
    'IsMultiRegionTrail': True,  # All regions
    'Tags': {
        'Compliance': 'HIPAA,PCI-DSS,SOX',
        'RetentionYears': '7',
        'MFADeleteRequired': 'true'
    }
}
```

---

### Question 3: Design Forensic Investigation Workflow Using CloudTrail

**Scenario**: "A security incident occurred 2 days ago. Walk through how you would use CloudTrail to investigate what happened, identify the attacker, and find all affected resources."

**Answer Structure:**

**Incident Investigation Workflow:**

Phase 1 - Initial Discovery (First Hour)
├─ Identify attack surface
├─ Determine timeline
├─ Get preliminary scope
└─ Activate incident response

Phase 2 - Evidence Collection (Hours 1-4)
├─ Collect all relevant CloudTrail events
├─ Document findings
├─ Preserve logs (immutable copy)
└─ Identify timeline of events

Phase 3 - Analysis (Hours 4-8)
├─ Analyze attacker identity
├─ Track all actions taken
├─ Identify compromised credentials
├─ Map affected resources
└─ Determine scope of breach

Phase 4 - Remediation (Hours 8+)
├─ Revoke credentials
├─ Reset MFA
├─ Apply patches
├─ Monitor for re-entry
└─ Prepare incident report

**Detailed Investigation Steps:**

```python
import boto3
from datetime import datetime, timedelta
import json

class CloudTrailForensicInvestigation:
    def __init__(self):
        self.cloudtrail = boto3.client('cloudtrail')
        self.athena = boto3.client('athena')
        self.s3 = boto3.client('s3')
    
    # Step 1: Identify Incident Scope
    def step1_identify_scope(self, incident_start_time, incident_end_time):
        """Determine what happened and when"""
        
        query = f'''
        SELECT
            COUNT(*) as total_events,
            COUNT(DISTINCT userIdentity.arn) as unique_users,
            COUNT(DISTINCT sourceIPAddress) as unique_ips,
            COUNT(DISTINCT eventSource) as unique_services,
            COUNT(DISTINCT eventName) as unique_actions
        FROM cloudtrail_logs
        WHERE eventTime BETWEEN '{incident_start_time}' AND '{incident_end_time}'
        '''
        
        print("Step 1: Incident Scope Analysis")
        print("Query: Find total events, users, IPs, services involved")
        print(query)
        
        return {
            'query': query,
            'purpose': 'Understand scale and scope of incident'
        }
    
    # Step 2: Identify Attacker/Compromised User
    def step2_identify_attacker(self, incident_start_time, incident_end_time):
        """Find which user/credentials were compromised"""
        
        query = f'''
        SELECT
            userIdentity.arn,
            userIdentity.type,
            COUNT(*) as event_count,
            COUNT(DISTINCT sourceIPAddress) as unique_ips,
            COUNT(DISTINCT eventName) as unique_actions,
            MIN(eventTime) as first_event,
            MAX(eventTime) as last_event
        FROM cloudtrail_logs
        WHERE
            eventTime BETWEEN '{incident_start_time}' AND '{incident_end_time}'
            AND eventType = 'AwsApiCall'
        GROUP BY
            userIdentity.arn,
            userIdentity.type
        ORDER BY event_count DESC
        '''
        
        print("\nStep 2: Identify Attacker/Compromised Credentials")
        print("Query: Find users with unusual activity")
        print(query)
        
        return {
            'query': query,
            'purpose': 'Identify compromised user or attacker identity'
        }
    
    # Step 3: Timeline of Events
    def step3_timeline_analysis(self, username, incident_start_time, incident_end_time):
        """Get detailed timeline of all actions"""
        
        query = f'''
        SELECT
            eventTime,
            eventName,
            eventSource,
            sourceIPAddress,
            requestParameters,
            responseElements,
            errorCode,
            resources
        FROM cloudtrail_logs
        WHERE
            userIdentity.arn LIKE '%{username}%'
            AND eventTime BETWEEN '{incident_start_time}' AND '{incident_end_time}'
        ORDER BY eventTime ASC
        '''
        
        print("\nStep 3: Timeline of Events")
        print("Query: Chronological list of all actions by attacker")
        print(query)
        
        return {
            'query': query,
            'purpose': 'Understand sequence of attacker actions'
        }
    
    # Step 4: Identify Affected Resources
    def step4_affected_resources(self, incident_start_time, incident_end_time):
        """Find all resources that were accessed/modified"""
        
        query = f'''
        SELECT
            resources[0].arn,
            resources[0].type,
            eventName,
            COUNT(*) as access_count,
            userIdentity.arn,
            sourceIPAddress
        FROM cloudtrail_logs
        WHERE
            eventTime BETWEEN '{incident_start_time}' AND '{incident_end_time}'
            AND resources IS NOT NULL
            AND resources[0].arn IS NOT NULL
        GROUP BY
            resources[0].arn,
            resources[0].type,
            eventName,
            userIdentity.arn,
            sourceIPAddress
        ORDER BY access_count DESC
        '''
        
        print("\nStep 4: Affected Resources")
        print("Query: All resources accessed or modified")
        print(query)
        
        return {
            'query': query,
            'purpose': 'Identify compromised resources'
        }
    
    # Step 5: Identify Lateral Movement
    def step5_lateral_movement(self, username, incident_start_time, incident_end_time):
        """Find if attacker accessed other accounts/services"""
        
        query = f'''
        SELECT
            eventTime,
            eventName,
            requestParameters,
            responseElements,
            eventSource,
            sourceIPAddress,
            awsRegion
        FROM cloudtrail_logs
        WHERE
            userIdentity.arn LIKE '%{username}%'
            AND eventTime BETWEEN '{incident_start_time}' AND '{incident_end_time}'
            AND eventSource IN ('sts.amazonaws.com', 'iam.amazonaws.com', 'organizations.amazonaws.com')
        ORDER BY eventTime ASC
        '''
        
        print("\nStep 5: Lateral Movement Analysis")
        print("Query: Find privilege escalation or cross-account access")
        print(query)
        
        return {
            'query': query,
            'purpose': 'Detect lateral movement or privilege escalation'
        }
    
    # Step 6: Identify Data Access
    def step6_data_exfiltration(self, incident_start_time, incident_end_time):
        """Find if sensitive data was accessed or stolen"""
        
        query = f'''
        SELECT
            eventTime,
            eventName,
            userIdentity.arn,
            sourceIPAddress,
            s3.object.key,
            s3.object.size,
            resources
        FROM cloudtrail_logs
        WHERE
            eventTime BETWEEN '{incident_start_time}' AND '{incident_end_time}'
            AND (
                (eventSource = 's3.amazonaws.com' AND eventName IN ('GetObject', 'ListBucket', 'GetObjectVersion'))
                OR (eventSource = 'rds.amazonaws.com' AND eventName LIKE 'Describe%')
                OR (eventSource = 'dynamodb.amazonaws.com' AND eventName IN ('Scan', 'Query', 'GetItem'))
            )
        ORDER BY eventTime DESC
        '''
        
        print("\nStep 6: Data Access/Exfiltration")
        print("Query: Find suspicious data access patterns")
        print(query)
        
        return {
            'query': query,
            'purpose': 'Identify data theft or unauthorized access'
        }
    
    # Step 7: Create Immutable Evidence Copy
    def step7_preserve_evidence(self, incident_id):
        """Create immutable copy of evidence for investigation"""
        
        evidence_bucket = f'incident-{incident_id}-evidence'
        
        # Create bucket
        self.s3.create_bucket(
            Bucket=evidence_bucket,
            CreateBucketConfiguration={'LocationConstraint': 'us-east-1'}
        )
        
        # Enable Object Lock for immutability
        self.s3.put_object_lock_legal_hold(
            Bucket=evidence_bucket,
            Key='evidence/',
            LegalHold={'Status': 'ON'}
        )
        
        # Copy relevant CloudTrail logs
        # (In production, would copy actual S3 objects)
        
        print(f"\nStep 7: Evidence Preservation")
        print(f"Created immutable evidence bucket: {evidence_bucket}")
        
        return evidence_bucket
    
    def generate_incident_report(self, investigation_results):
        """Generate forensic investigation report"""
        
        report = f"""
╔═════════════════════════════════════════════════════════════════╗
║              CLOUDTRAIL FORENSIC INVESTIGATION REPORT           ║
╚═════════════════════════════════════════════════════════════════╝

INCIDENT SUMMARY
================
- Investigation Date: {datetime.now().isoformat()}
- Incident ID: INC-2024-001
- Status: ONGOING INVESTIGATION

KEY FINDINGS
============
1. COMPROMISED CREDENTIALS
   - User ARN: arn:aws:iam::123456789012:user/compromised-user
   - First suspicious event: 2024-01-13 14:23:45 UTC
   - Total suspicious events: 1,250+

2. ATTACKER PROFILE
   - Source IPs: 
     * 203.0.113.10 (primary)
     * 203.0.113.11 (secondary)
   - Access method: Console login
   - MFA status: DISABLED
   - First action: Change password

3. TIMELINE OF EVENTS
   - 14:23 - Console login from suspicious IP
   - 14:25 - Disabled MFA on account
   - 14:30 - Created new IAM user "devops-backup"
   - 14:35 - Attached AdminAccess policy to new user
   - 14:40 - Created access key for new user
   - 15:00 - S3 bucket discovery (ListBucket)
   - 15:15 - Download of sensitive documents
   - 16:00 - Attempted lateral movement to other accounts

4. AFFECTED RESOURCES
   - S3 Buckets: 3 (sensitive-data, customer-records, billing)
   - RDS Instances: 1 (production-db)
   - IAM Users: 2 created (devops-backup, admin-temp)
   - EC2 Instances: 1 modified (security group changes)

5. DATA EXFILTRATION
   - Files accessed: 450+ objects
   - Total size: 2.3 GB
   - Sensitive: Customer PII, Financial records
   - Compromised records: ~50,000 customers

IMMEDIATE ACTIONS
=================
✓ Revoke compromised user access keys
✓ Reset MFA on all accounts
✓ Terminate malicious IAM users
✓ Reset RDS passwords
✓ Restrict S3 bucket access
✓ Enable CloudWatch Logs analysis

ONGOING MONITORING
=================
- Monitor newly created IAM users
- Alert on unusual S3 access patterns
- Track all console logins from suspicious IPs
- Monitor for similar attack patterns

RECOMMENDATIONS
================
1. Enable MFA enforcement for all users
2. Implement IP whitelisting for sensitive resources
3. Enable CloudTrail data events for all S3 buckets
4. Deploy SIEM for real-time threat detection
5. Implement strict access controls
        """
        
        return report

# Usage
investigation = CloudTrailForensicInvestigation()

# Run investigation steps
incident_start = datetime.now() - timedelta(days=2)
incident_end = datetime.now()

step1 = investigation.step1_identify_scope(
    incident_start.isoformat(),
    incident_end.isoformat()
)

step2 = investigation.step2_identify_attacker(
    incident_start.isoformat(),
    incident_end.isoformat()
)

step3 = investigation.step3_timeline_analysis(
    'compromised-user',
    incident_start.isoformat(),
    incident_end.isoformat()
)

step4 = investigation.step4_affected_resources(
    incident_start.isoformat(),
    incident_end.isoformat()
)

step5 = investigation.step5_lateral_movement(
    'compromised-user',
    incident_start.isoformat(),
    incident_end.isoformat()
)

step6 = investigation.step6_data_exfiltration(
    incident_start.isoformat(),
    incident_end.isoformat()
)

evidence_bucket = investigation.step7_preserve_evidence('2024-001')

report = investigation.generate_incident_report({})
print(report)
```

---

### Question 4: Optimize CloudTrail Costs While Maintaining Compliance

**Scenario**: "Your CloudTrail costs are $500/month and need to reduce by 40% without compromising compliance or security. Design a cost optimization strategy."

**Answer Structure:**

**Cost Analysis Breakdown:**

Current Costs:

```
Management Events:        $200 (50M events/month)
Data Events (S3):         $150 (150M objects/month)
Data Events (Other):       $50 (50M events/month)
Insights:                  $30 (6 anomalies/month)
S3 Storage:               $60 (200GB stored)
Athena Queries:           $10 (cost of queries)
Total:                   $500/month
```

**Optimization Strategy:**

```python
class CloudTrailCostOptimization:
    def __init__(self):
        self.cloudtrail = boto3.client('cloudtrail')
        self.s3 = boto3.client('s3')
    
    # Strategy 1: Selective Data Event Logging
    def strategy1_selective_data_events(self):
        """
        Reduce data events from $200 to $50/month
        Strategy: Enable data events ONLY for critical resources
        """
        
        optimization = {
            'before': '$200/month (all S3 buckets + all services)',
            'after': '$50/month (critical S3 buckets only)',
            'savings': '$150/month (75% reduction)',
            'approach': 'Use advanced event selectors'
        }
        
        # Configure to track ONLY critical S3 buckets
        config = {
            'AdvancedEventSelectors': [
                {
                    'Field': 'eventCategory',
                    'Equals': ['Data']
                },
                {
                    'Field': 'resources.type',
                    'Equals': ['AWS::S3::Object']
                },
                {
                    'Field': 'resources.arn',
                    'StartsWith': [
                        'arn:aws:s3:::sensitive-data-bucket/*',
                        'arn:aws:s3:::customer-records-bucket/*',
                        'arn:aws:s3:::pii-backup-bucket/*'
                    ]
                }
            ]
        }
        
        return optimization, config
    
    # Strategy 2: Disable Unnecessary Management Events
    def strategy2_filter_management_events(self):
        """
        Reduce management events from $200 to $140/month
        Strategy: Exclude low-risk, high-volume services
        """
        
        optimization = {
            'before': '$200/month (all management events)',
            'after': '$140/month (excluding low-risk services)',
            'savings': '$60/month (30% reduction)',
            'excluded_services': [
                'monitoring.amazonaws.com',  # CloudWatch
                'cloudformation.amazonaws.com',  # Low-risk
                'ec2-instance-connect.amazonaws.com'  # Auto-scaling
            ]
        }
        
        # Exclude specific event sources
        config = {
            'AdvancedEventSelectors': [
                {
                    'Field': 'eventCategory',
                    'Equals': ['Management']
                },
                {
                    'Field': 'eventSource',
                    'NotEquals': [
                        'monitoring.amazonaws.com',
                        'ec2-instance-connect.amazonaws.com'
                    ]
                }
            ]
        }
        
        return optimization, config
    
    # Strategy 3: S3 Storage Optimization
    def strategy3_storage_optimization(self):
        """
        Reduce S3 storage from $60 to $20/month
        Strategy: Aggressive archival and cleanup
        """
        
        optimization = {
            'before': '$60/month (200GB mixed storage)',
            'after': '$20/month (50GB hot + 150GB cold)',
            'savings': '$40/month (67% reduction)',
            'actions': [
                'Archive logs after 30 days to Glacier',
                'Delete logs after 1 year (retention policy)',
                'Use S3 Intelligent-Tiering'
            ]
        }
        
        # Implement aggressive lifecycle policy
        config = {
            'LifecycleConfiguration': {
                'Rules': [
                    {
                        'Id': 'ArchiveAfter30Days',
                        'Status': 'Enabled',
                        'Transitions': [
                            {'Days': 30, 'StorageClass': 'STANDARD_IA'},
                            {'Days': 60, 'StorageClass': 'GLACIER'},
                            {'Days': 180, 'StorageClass': 'DEEP_ARCHIVE'}
                        ],
                        'Expiration': {'Days': 365}
                    }
                ]
            }
        }
        
        return optimization, config
    
    # Strategy 4: Disable Insights (if not critical)
    def strategy4_insights_optimization(self):
        """
        Reduce Insights from $30 to $10/month
        Strategy: Disable for non-critical accounts or enable selective
        """
        
        optimization = {
            'before': '$30/month (6 anomalies/month)',
            'after': '$10/month (2 anomalies/month)',
            'savings': '$20/month (67% reduction)',
            'approach': 'Disable Insights on non-critical accounts'
        }
        
        return optimization
    
    def generate_optimization_plan(self):
        """Generate complete cost optimization roadmap"""
        
        plan = """
CLOUDTRAIL COST OPTIMIZATION PLAN
==================================

Current Monthly Cost: $500

OPTIMIZATION STRATEGY
=====================

Phase 1 - Quick Wins (Week 1)
├─ Selective S3 data events          -$150/month
├─ Filter low-risk management events  -$60/month
└─ Subtotal savings:                 -$210/month (42%)

Phase 2 - Storage Optimization (Week 2)
├─ Implement aggressive archival      -$40/month
├─ Reduce retention period            -$10/month
└─ Subtotal savings:                  -$50/month (10%)

Phase 3 - Analytics Optimization (Week 3)
├─ Disable non-critical Insights      -$20/month
└─ Optimize Athena queries            -$5/month

TOTAL OPTIMIZATION
==================
Total Savings:  -$285/month
New Cost:       $215/month (57% reduction)
ROI:            Immediate (Week 1)

RECOMMENDATIONS PRIORITY
=========================
Priority 1 - MUST DO (Quick wins)
- Selective data events (-$150/month)
- Filter management events (-$60/month)
- Implement S3 lifecycle (-$40/month)

Priority 2 - SHOULD DO (Compliance trade-offs)
- Reduce retention period
- Disable non-critical Insights
- Use S3 Intelligent-Tiering

Priority 3 - COULD DO (Advanced optimization)
- Implement custom event parsing
- Use EventBridge filtering
- Implement sampling for high-volume events

COMPLIANCE CONSIDERATIONS
==========================
✓ HIPAA: 7-year retention (configure lifecycle accordingly)
✓ PCI-DSS: 1-year minimum retention
✓ SOX: 7-year retention required
✓ GDPR: Data retention must respect privacy laws

ROLLBACK PLAN
=============
If optimization causes issues:
1. Re-enable data events (restore full visibility)
2. Increase retention period (restore historical data)
3. Re-enable Insights (restore anomaly detection)
All changes can be rolled back in <30 minutes
        """
        
        return plan

# Usage
optimizer = CloudTrailCostOptimization()

# Get optimization strategies
strat1, config1 = optimizer.strategy1_selective_data_events()
strat2, config2 = optimizer.strategy2_filter_management_events()
strat3, config3 = optimizer.strategy3_storage_optimization()
strat4 = optimizer.strategy4_insights_optimization()

# Generate plan
plan = optimizer.generate_optimization_plan()
print(plan)
```

---

### Question 5: CloudTrail and Compliance Requirements

**Scenario**: "Your organization needs to meet HIPAA, PCI-DSS, and SOX compliance requirements. Design a CloudTrail strategy that satisfies all requirements while being cost-effective."

**Answer Structure:**

**Compliance Requirements Comparison:**

| Requirement | HIPAA | PCI-DSS | SOX |
|-------------|-------|---------|-----|
| **Audit Trail Retention** | 6 years minimum | 1 year minimum | 7 years |
| **Real-time Monitoring** | Required | Required | Required |
| **Log Integrity** | Yes (immutability) | Yes | Yes (no tampering) |
| **Encryption** | At rest & transit | At rest & transit | At rest & transit |
| **Access Control Logging** | PHI access | Payment card access | Financial system access |
| **User Identity Tracking** | Required | Required | Required |
| **Failure Alerts** | Required | Required | Required |

**Compliance-Focused CloudTrail Configuration:**

```python
class ComplianceCloudTrailStrategy:
    def __init__(self):
        self.cloudtrail = boto3.client('cloudtrail')
        self.s3 = boto3.client('s3')
        self.logs = boto3.client('logs')
        self.kms = boto3.client('kms')
    
    def setup_hipaa_compliant_trail(self):
        """
        HIPAA Requirements:
        - Track who accesses PHI
        - Immutable audit logs (6+ years)
        - Encryption at rest
        - Access control logging
        """
        
        # 1. Create KMS key for encryption
        kms_response = self.kms.create_key(
            Description='CloudTrail HIPAA compliance key',
            Origin='AWS_KMS',
            KeyUsage='ENCRYPT_DECRYPT',
            MultiRegion=False,
            Tags=[
                {'TagKey': 'Compliance', 'TagValue': 'HIPAA'},
                {'TagKey': 'Purpose', 'TagValue': 'Audit-Trail'}
            ]
        )
        kms_key_id = kms_response['KeyMetadata']['KeyId']
        
        # 2. Create S3 bucket with compliance settings
        bucket_name = 'hipaa-audit-logs'
        self.s3.create_bucket(
            Bucket=bucket_name,
            CreateBucketConfiguration={'LocationConstraint': 'us-east-1'}
        )
        
        # Enable Object Lock for immutability
        # Note: Must be done at creation, shown for reference
        object_lock_config = {
            'ObjectLockEnabled': 'Enabled',
            'Rule': {
                'DefaultRetention': {
                    'Mode': 'COMPLIANCE',  # Cannot be overridden
                    'Years': 6  # 6-year retention
                }
            }
        }
        
        # 3. Enable log file validation
        trail_response = self.cloudtrail.create_trail(
            Name='hipaa-compliance-trail',
            S3BucketName=bucket_name,
            IncludeGlobalServiceEvents=True,
            IsMultiRegionTrail=True,
            EnableLogFileValidation=True,  # CRITICAL for HIPAA
            KmsKeyId=f'arn:aws:kms:us-east-1:123456789012:key/{kms_key_id}',
            CloudWatchLogsLogGroupArn='arn:aws:logs:us-east-1:123456789012:log-group:/hipaa/audit:*',
            CloudWatchLogsRoleArn='arn:aws:iam::123456789012:role/cloudtrail-cwl-role'
        )
        
        # 4. Configure data events for PHI access
        self.cloudtrail.put_event_selectors(
            TrailName='hipaa-compliance-trail',
            AdvancedEventSelectors=[
                # Management events (all)
                {'Field': 'eventCategory', 'Equals': ['Management']},
                # Data events - S3 PHI buckets
                {
                    'Field': 'eventCategory',
                    'Equals': ['Data']
                },
                {
                    'Field': 'resources.type',
                    'Equals': ['AWS::S3::Object']
                },
                {
                    'Field': 'resources.arn',
                    'StartsWith': ['arn:aws:s3:::phi-bucket/*']
                },
                # Data events - RDS database access
                {
                    'Field': 'eventCategory',
                    'Equals': ['Data']
                },
                {
                    'Field': 'resources.type',
                    'Equals': ['AWS::RDS::DBInstance']
                }
            ]
        )
        
        # 5. Lifecycle policy for 6-year retention
        self.s3.put_bucket_lifecycle_configuration(
            Bucket=bucket_name,
            LifecycleConfiguration={
                'Rules': [{
                    'Id': 'HIPAASixYearRetention',
                    'Status': 'Enabled',
                    'Transitions': [
                        {'Days': 90, 'StorageClass': 'STANDARD_IA'},
                        {'Days': 180, 'StorageClass': 'GLACIER'},
                        {'Days': 365, 'StorageClass': 'DEEP_ARCHIVE'}
                    ],
                    'Expiration': {'Days': 2190}  # 6 years
                }]
            }
        )
        
        print("HIPAA-compliant CloudTrail configured")
        return trail_response['TrailARN']
    
    def setup_pci_dss_compliant_trail(self):
        """
        PCI-DSS Requirements:
        - Monitor cardholder data access
        - Real-time alerting
        - 1-year minimum retention
        - Encrypted storage
        """
        
        bucket_name = 'pci-dss-audit-logs'
        
        # PCI-DSS specific configuration
        trail_response = self.cloudtrail.create_trail(
            Name='pci-dss-compliance-trail',
            S3BucketName=bucket_name,
            IncludeGlobalServiceEvents=True,
            IsMultiRegionTrail=True,
            EnableLogFileValidation=True,
            KmsKeyId='arn:aws:kms:us-east-1:123456789012:key/pci-key'
        )
        
        # Configure for payment card data access
        self.cloudtrail.put_event_selectors(
            TrailName='pci-dss-compliance-trail',
            AdvancedEventSelectors=[
                # IAM changes (access to payment systems)
                {
                    'Field': 'eventSource',
                    'Equals': ['iam.amazonaws.com']
                },
                # Database access
                {
                    'Field': 'eventSource',
                    'Equals': ['rds.amazonaws.com']
                },
                # Payment-related S3 buckets
                {
                    'Field': 'eventCategory',
                    'Equals': ['Data']
                },
                {
                    'Field': 'resources.arn',
                    'StartsWith': ['arn:aws:s3:::payment-data/*']
                }
            ]
        )
        
        # PCI-DSS requires 1-year minimum
        self.s3.put_bucket_lifecycle_configuration(
            Bucket=bucket_name,
            LifecycleConfiguration={
                'Rules': [{
                    'Id': 'PCIDSSOneYearRetention',
                    'Status': 'Enabled',
                    'Expiration': {'Days': 365}  # 1 year
                }]
            }
        )
        
        print("PCI-DSS-compliant CloudTrail configured")
        return trail_response['TrailARN']
    
    def setup_sox_compliant_trail(self):
        """
        SOX (Sarbanes-Oxley) Requirements:
        - Financial system activity logging
        - 7-year retention
        - Immutable logs
        - Detailed access tracking
        """
        
        bucket_name = 'sox-audit-logs'
        
        # SOX specific configuration
        trail_response = self.cloudtrail.create_trail(
            Name='sox-compliance-trail',
            S3BucketName=bucket_name,
            IncludeGlobalServiceEvents=True,
            IsMultiRegionTrail=True,
            EnableLogFileValidation=True,  # Immutability
            KmsKeyId='arn:aws:kms:us-east-1:123456789012:key/sox-key'
        )
        
        # Configure for financial system access
        self.cloudtrail.put_event_selectors(
            TrailName='sox-compliance-trail',
            AdvancedEventSelectors=[
                # All management events
                {'Field': 'eventCategory', 'Equals': ['Management']},
                # Specific financial system events
                {
                    'Field': 'eventSource',
                    'Equals': [
                        'rds.amazonaws.com',
                        'dynamodb.amazonaws.com',
                        'kms.amazonaws.com'
                    ]
                },
                # Financial database access
                {
                    'Field': 'eventCategory',
                    'Equals': ['Data']
                },
                {
                    'Field': 'resources.arn',
                    'StartsWith': ['arn:aws:rds:*:*:db:financial-*']
                }
            ]
        )
        
        # SOX requires 7-year retention
        self.s3.put_bucket_lifecycle_configuration(
            Bucket=bucket_name,
            LifecycleConfiguration={
                'Rules': [{
                    'Id': 'SOXSevenYearRetention',
                    'Status': 'Enabled',
                    'Transitions': [
                        {'Days': 180, 'StorageClass': 'GLACIER'},
                        {'Days': 730, 'StorageClass': 'DEEP_ARCHIVE'}
                    ],
                    'Expiration': {'Days': 2555}  # 7 years
                }]
            }
        )
        
        print("SOX-compliant CloudTrail configured")
        return trail_response['TrailARN']
    
    def create_compliance_dashboard(self):
        """Create CloudWatch dashboard for compliance monitoring"""
        
        dashboard_body = {
            'widgets': [
                {
                    'type': 'metric',
                    'properties': {
                        'metrics': [
                            ['AWS/CloudTrail', 'CallCount', {'stat': 'Sum'}],
                            ['CloudTrailMetrics', 'UnauthorizedAPICallsMetric'],
                            ['CloudTrailMetrics', 'RootAccountUsageMetric']
                        ],
                        'period': 300,
                        'stat': 'Sum',
                        'region': 'us-east-1',
                        'title': 'Compliance Metrics'
                    }
                }
            ]
        }
        
        return dashboard_body

# Usage
compliance = ComplianceCloudTrailStrategy()

hipaa_arn = compliance.setup_hipaa_compliant_trail()
print(f"HIPAA Trail: {hipaa_arn}")

pci_arn = compliance.setup_pci_dss_compliant_trail()
print(f"PCI-DSS Trail: {pci_arn}")

sox_arn = compliance.setup_sox_compliant_trail()
print(f"SOX Trail: {sox_arn}")

dashboard = compliance.create_compliance_dashboard()
print("Compliance dashboard created")
```

---

## References

[1] AWS CloudTrail User Guide - AWS Documentation. https://docs.aws.amazon.com/awscloudtrail/latest/userguide/

[2] CloudTrail Event Reference - AWS Documentation. https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-event-reference.html

[3] CloudTrail Log File Examples - AWS Documentation. https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-log-file-examples.html

[4] Querying CloudTrail Logs with Amazon Athena - AWS Documentation. https://docs.aws.amazon.com/athena/latest/ug/querying-cloudtrail-logs.html

[5] CloudTrail Compliance with Regulations - AWS Compliance. https://aws.amazon.com/compliance/

[6] CloudTrail Security Best Practices - AWS Security Blog. https://aws.amazon.com/blogs/security/

[7] CloudTrail Organization Trails - AWS Documentation. https://docs.aws.amazon.com/awscloudtrail/latest/userguide/creating-an-organizational-trail-prepare.html

[8] CloudTrail Insights - AWS Documentation. https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-insights.html

[9] CloudTrail Data Events - AWS Documentation. https://docs.aws.amazon.com/awscloudtrail/latest/userguide/logging-data-events-with-cloudtrail.html

[10] CloudTrail Log File Validation - AWS Documentation. https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-log-file-examples.html

[11] AWS Security Best Practices - AWS Whitepaper. https://d1.awsstatic.com/whitepapers/aws-security-best-practices.pdf

[12] Forensic Investigation with CloudTrail - AWS Security Blog. https://aws.amazon.com/blogs/security/how-to-use-cloudtrail/
