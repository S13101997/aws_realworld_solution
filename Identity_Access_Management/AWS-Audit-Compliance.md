# AWS CloudTrail & AWS Config — Audit and Compliance Tracking

## Detailed Explanation

### Overview: Audit and Compliance in AWS

AWS provides two complementary services for auditing and compliance tracking:

1. **AWS CloudTrail**: Audit logging service that records all API calls and activities in your AWS account
2. **AWS Config**: Configuration assessment and compliance tracking service that monitors resource configurations

Both services provide the foundation for operational auditing, governance, and compliance but focus on different aspects of AWS activity.

### AWS CloudTrail

#### What is AWS CloudTrail?

AWS CloudTrail is a service that enables operational and risk auditing, governance, and compliance of your AWS account. It records API calls and related events made by users, roles, or AWS services in your account[1].

**Key Characteristics:**
- **Comprehensive Logging**: Captures all API calls to any AWS service
- **Event History**: Available by default; shows recent events (90 days)
- **Trail Creation**: Optional long-term storage to S3 buckets for archival
- **Multi-Region**: Single trail can log events from all regions
- **Event Types**: Management events, data events, insight events
- **Cost**: $2.00 per 100,000 management events (trail); $1.00 per 100,000 data events
- **Retention**: 90 days in event history; unlimited in S3 (depends on retention policy)
- **Max Event Size**: 256 KB

**CloudTrail Event Flow**:

```
User/Application → AWS Service → CloudTrail → S3 Bucket
                                   ↓
                            Event History
                            (90 days)
                                   ↓
                         CloudTrail Lake (Long-term)
```

#### CloudTrail Event Types

**1. Management Events** (Control Plane Operations):
- Configuration changes to resources
- API calls that modify AWS environment
- Authentication/authorization operations
- Resource creation, deletion, modification
- Examples: RunInstances, PutBucketPolicy, CreateSecurityGroup

**2. Data Events** (Data Plane Operations):
- Operations on data within resources
- Read/write operations at object level
- Resource usage tracking
- Examples: GetObject, PutObject, DeleteObject, Invoke (Lambda)

**3. Insights Events**:
- Unusual API activity detection
- Anomaly-based alerts
- High API volume detection
- Failed authorization patterns

**4. Non-API Events**:
- Console login events
- AWS Management Console sign-in failures
- Cross-account assume role

#### CloudTrail Architecture

```
┌─────────────────────────────────────────┐
│ AWS Account Activity                    │
├─────────────────────────────────────────┤
│ • Console logins                        │
│ • API calls                             │
│ • Service operations                    │
│ • Resource changes                      │
└──────────────────┬──────────────────────┘
                   │
         ┌─────────▼──────────┐
         │   CloudTrail       │
         │  Log Processing    │
         └─────────┬──────────┘
                   │
        ┌──────────┴──────────┬──────────────┐
        │                     │              │
    ┌───▼────┐         ┌─────▼────┐   ┌────▼────┐
    │Event   │         │CloudTrail │   │CloudWatch
    │History │         │Lake       │   │Logs
    │(90d)   │         │(Long-term)│   │
    └────────┘         └───────────┘   └─────────┘
        │
        └────────────────────┐
                             │
                        ┌────▼─────┐
                        │ S3 Bucket │
                        │(Archive)  │
                        └───────────┘
```

### AWS Config

#### What is AWS Config?

AWS Config is a service that enables you to assess, audit, and evaluate the configurations of your AWS resources. It continuously monitors and records how your AWS resources are configured and how they change over time[2].

**Key Characteristics:**
- **Configuration Monitoring**: Tracks state of AWS resources
- **Compliance Evaluation**: Checks compliance against defined rules
- **Configuration History**: Maintains timeline of configuration changes
- **Resource Relationships**: Maps resource dependencies
- **Notifications**: Alerts on configuration changes via SNS/Lambda
- **Cost**: Based on configuration items recorded and rules evaluated
- **Retention**: Configurable retention period (default 7 years)
- **Config Rules**: Managed and custom compliance rules

**AWS Config Event Flow**:

```
AWS Resource Change → CloudTrail API Call → AWS Config Detection
                                                    ↓
                                        Config Rule Evaluation
                                                    ↓
                         ┌──────────────┬──────────┴──────────┐
                         │              │                     │
                    Compliant      Non-Compliant        Unknown
                         │              │                     │
                         └──────────────┴──────────┬──────────┘
                                                  ↓
                                            Notification
                                        (SNS/EventBridge/Lambda)
                                                  ↓
                                          Remediation (Optional)
```

#### Config Rules

Config Rules continuously evaluate your AWS resources against desired configurations.

**Managed Rules** (AWS-provided):
- Pre-built rules for common compliance requirements
- Examples: s3-bucket-server-side-encryption-enabled, iam-password-policy-check
- No custom logic needed
- Regular updates by AWS

**Custom Rules**:
- Lambda-based or CloudFormation Guard-based
- Evaluate resources based on custom business logic
- Triggered by configuration changes or on schedule
- Maximum 50 custom rules per account

**Rule Compliance Status**:
- **COMPLIANT**: Resource meets rule requirement
- **NON_COMPLIANT**: Resource violates rule requirement
- **NOT_APPLICABLE**: Rule doesn't apply to resource type
- **INSUFFICIENT_DATA**: Not enough data to evaluate

#### Conformance Packs

Collections of Config Rules grouped around compliance framework.

**Features**:
- Pre-built for industry standards (CIS, PCI DSS, HIPAA, NIST)
- Template-based deployment
- Consistency across organization
- Compliance scoring
- Multi-account, multi-region support

---

### Feature Comparison Matrix

| Feature | CloudTrail | Config |
|---------|:----------:|:------:|
| **Purpose** | API activity logging | Configuration assessment |
| **What It Tracks** | Who did what, when, where | What is the current state |
| **Event Granularity** | Individual API calls | Resource configuration snapshots |
| **Real-time Monitoring** | ✅ (minutes) | ✅ (minutes) |
| **Historical Data** | ✅ (90 days in console, unlimited in S3) | ✅ (configurable, up to 7 years) |
| **Compliance Rules** | ❌ | ✅ (managed & custom) |
| **Configuration History** | ❌ | ✅ (detailed timeline) |
| **Relationships Tracking** | ❌ | ✅ (resource dependencies) |
| **Anomaly Detection** | ✅ (CloudTrail Insights) | ❌ |
| **Automation** | ❌ | ✅ (remediation actions) |
| **Cost Model** | Per event volume | Per configuration item |
| **Primary Use Case** | Security auditing, incident response | Compliance tracking, configuration audit |
| **Best For** | "Who made this change?" | "Is this configuration compliant?" |

---

### Permission Models

#### CloudTrail Permission Model

CloudTrail uses **three-layer permission model**:

```
1. IAM Policy
   ↓ (Principal must have cloudtrail:* permissions)
   
2. Trail Configuration
   ↓ (Trail must be enabled and properly configured)
   
3. S3 Bucket Policy
   ↓ (Bucket must allow CloudTrail to write logs)
   
Result: CloudTrail can log if ALL three layers allow
```

#### Config Permission Model

AWS Config uses **role-based permission model**:

```
1. Config Service Role
   ↓ (Role with permissions to evaluate resources)
   
2. Config Recorder
   ↓ (Must be enabled to record configurations)
   
3. Delivery Channel
   ↓ (S3 bucket for configuration snapshots)
   
Result: Config can monitor if all components configured
```

---

### Integration Points

**CloudTrail + Config Integration**:

```
AWS Resource Configuration Change
                ↓
         CloudTrail logs API call
                ↓
         Config detects change via API
                ↓
         Config evaluates compliance rule
                ↓
    ┌──────────────┬────────────────┐
    │              │                │
Non-Compliant  Compliant      EventBridge Trigger
    │              │                │
    └──────────────┴────────┬───────┘
                           ↓
                  Lambda Function
                  (Auto-remediation)
                           ↓
                  Config logs remediation
                           ↓
                  CloudTrail logs Lambda invocation
```

---

## Detailed Examples

### Example 1: Setting Up CloudTrail for Security Auditing

**Scenario**: Organization needs to track all API calls for compliance and security incident investigation.

**Step 1: Create S3 Bucket for CloudTrail Logs**

```bash
# Create S3 bucket for CloudTrail logs
aws s3 mb s3://my-org-cloudtrail-logs-$(date +%s) \
  --region us-east-1

# Enable versioning for data protection
BUCKET_NAME="my-org-cloudtrail-logs-$(date +%s)"
aws s3api put-bucket-versioning \
  --bucket $BUCKET_NAME \
  --versioning-configuration Status=Enabled

# Enable encryption
aws s3api put-bucket-encryption \
  --bucket $BUCKET_NAME \
  --server-side-encryption-configuration '{
    "Rules": [
      {
        "ApplyServerSideEncryptionByDefault": {
          "SSEAlgorithm": "AES256"
        }
      }
    ]
  }'

# Block public access
aws s3api put-public-access-block \
  --bucket $BUCKET_NAME \
  --public-access-block-configuration \
    BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true
```

**Step 2: Create S3 Bucket Policy for CloudTrail**

```bash
# Create bucket policy allowing CloudTrail to write logs
aws s3api put-bucket-policy \
  --bucket $BUCKET_NAME \
  --policy '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Sid": "AWSCloudTrailAclCheck",
        "Effect": "Allow",
        "Principal": {
          "Service": "cloudtrail.amazonaws.com"
        },
        "Action": "s3:GetBucketAcl",
        "Resource": "arn:aws:s3:::'$BUCKET_NAME'"
      },
      {
        "Sid": "AWSCloudTrailWrite",
        "Effect": "Allow",
        "Principal": {
          "Service": "cloudtrail.amazonaws.com"
        },
        "Action": "s3:PutObject",
        "Resource": "arn:aws:s3:::'$BUCKET_NAME'/AWSLogs/*",
        "Condition": {
          "StringEquals": {
            "s3:x-amz-acl": "bucket-owner-full-control"
          }
        }
      }
    ]
  }'
```

**Step 3: Create CloudTrail**

```bash
# Create organization trail (logs all regions and accounts)
aws cloudtrail create-trail \
  --name OrganizationTrail \
  --s3-bucket-name $BUCKET_NAME \
  --is-multi-region-trail \
  --include-global-service-events \
  --enable-log-file-validation

TRAIL_ARN=$(aws cloudtrail describe-trails \
  --trail-name-list OrganizationTrail \
  --query 'trailList[0].TrailARN' \
  --output text)

# Start logging
aws cloudtrail start-logging --trail-name OrganizationTrail

# Enable management and data events
aws cloudtrail put-event-selectors \
  --trail-name OrganizationTrail \
  --advanced-event-selectors '[
    {
      "Field": "eventCategory",
      "Equals": ["Management"]
    },
    {
      "Field": "eventCategory",
      "Equals": ["Data"],
      "SelectorsForResources": [
        {
          "Field": "resources.type",
          "Equals": ["AWS::S3::Object"]
        }
      ]
    }
  ]'
```

**Step 4: Analyze CloudTrail Logs with Athena**

```bash
# Create Athena table for CloudTrail logs
aws athena start-query-execution \
  --query-string "
    CREATE EXTERNAL TABLE IF NOT EXISTS cloudtrail_logs (
      eventversion STRING,
      useridentity STRUCT<
        type:STRING,
        principalid:STRING,
        arn:STRING,
        accountid:STRING,
        invokedby:STRING,
        accesskeyid:STRING,
        userName:STRING,
        sessioncontext:STRUCT<
          attributes:STRUCT<
            mfaauthenticated:STRING,
            creationdate:STRING>,
          sessionissuer:STRUCT<
            type:STRING,
            principalId:STRING,
            arn:STRING,
            accountId:STRING,
            userName:STRING>>>,
      eventtime STRING,
      eventsource STRING,
      eventname STRING,
      awsregion STRING,
      sourceipaddress STRING,
      useragent STRING,
      errorcode STRING,
      errormessage STRING,
      requestparameters STRING,
      responseelements STRING,
      additionaleventdata STRING,
      requestid STRING,
      eventid STRING,
      resources ARRAY<STRUCT<
        ARN:STRING,
        accountId:STRING,
        type:STRING>>,
      eventtype STRING,
      recipientaccountid STRING,
      sharedEventID STRING,
      vpcendpointid STRING
    )
    PARTITIONED BY (region STRING, year STRING, month STRING, day STRING)
    ROW FORMAT SERDE 'com.amazon.emr.hive.serde.CloudTrailSerde'
    STORED AS INPUTFORMAT 'com.amazon.emr.cloudtrail.CloudTrailInputFormat'
    OUTPUTFORMAT 'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat'
    LOCATION 's3://$BUCKET_NAME/AWSLogs/'
  " \
  --query-execution-context Database=default \
  --result-configuration OutputLocation=s3://$BUCKET_NAME/athena-results/

# Query for failed authentication attempts
aws athena start-query-execution \
  --query-string "
    SELECT 
      eventtime,
      useridentity.principalid,
      sourceipaddress,
      eventname,
      errormessage
    FROM cloudtrail_logs
    WHERE errorcode = 'UnauthorizedOperation'
      AND year = '2024'
      AND month = '01'
    ORDER BY eventtime DESC
  " \
  --query-execution-context Database=default \
  --result-configuration OutputLocation=s3://$BUCKET_NAME/athena-results/
```

**Python Application for CloudTrail Analysis**:

```python
import boto3
import json
from datetime import datetime, timedelta
from collections import defaultdict

class CloudTrailAnalyzer:
    def __init__(self, trail_name='OrganizationTrail'):
        self.cloudtrail = boto3.client('cloudtrail')
        self.trail_name = trail_name
    
    def get_recent_events(self, hours=24, event_name=None):
        """Retrieve recent CloudTrail events"""
        
        start_time = datetime.now() - timedelta(hours=hours)
        
        params = {
            'TrailName': self.trail_name,
            'StartTime': start_time,
            'MaxResults': 50
        }
        
        if event_name:
            params['LookupAttributes'] = [
                {
                    'AttributeKey': 'EventName',
                    'AttributeValue': event_name
                }
            ]
        
        try:
            response = self.cloudtrail.lookup_events(**params)
            return response['Events']
        except Exception as e:
            print(f"Error retrieving events: {e}")
            return []
    
    def analyze_failed_logins(self):
        """Analyze failed login attempts"""
        
        events = self.get_recent_events(hours=24)
        failed_logins = defaultdict(int)
        
        for event in events:
            cloud_trail_event = json.loads(event['CloudTrailEvent'])
            
            if (cloud_trail_event.get('eventName') == 'ConsoleLogin' and
                cloud_trail_event.get('errorCode')):
                
                source_ip = cloud_trail_event.get('sourceIPAddress')
                user = cloud_trail_event.get('userIdentity', {}).get('principalId')
                failed_logins[f"{user}@{source_ip}"] += 1
        
        return dict(failed_logins)
    
    def detect_privilege_escalation(self):
        """Detect potential privilege escalation attempts"""
        
        escalation_events = [
            'PutUserPolicy',
            'PutRolePolicy',
            'AttachUserPolicy',
            'AttachRolePolicy',
            'CreateAccessKey',
            'UpdateAssumeRolePolicy'
        ]
        
        suspicious_activity = []
        
        for event_name in escalation_events:
            events = self.get_recent_events(event_name=event_name)
            
            for event in events:
                cloud_trail_event = json.loads(event['CloudTrailEvent'])
                
                suspicious_activity.append({
                    'timestamp': event['EventTime'],
                    'event': event_name,
                    'principal': cloud_trail_event.get('userIdentity', {}).get('arn'),
                    'resource': cloud_trail_event.get('requestParameters'),
                    'source_ip': cloud_trail_event.get('sourceIPAddress')
                })
        
        return suspicious_activity
    
    def identify_root_account_usage(self):
        """Identify root account usage (security risk)"""
        
        events = self.get_recent_events(hours=72)
        root_usage = []
        
        for event in events:
            cloud_trail_event = json.loads(event['CloudTrailEvent'])
            user_identity = cloud_trail_event.get('userIdentity', {})
            
            if user_identity.get('type') == 'Root' and user_identity.get('invokedBy') is None:
                root_usage.append({
                    'timestamp': event['EventTime'],
                    'event_name': event['EventName'],
                    'source_ip': cloud_trail_event.get('sourceIPAddress'),
                    'user_agent': cloud_trail_event.get('userAgent')
                })
        
        return root_usage
    
    def audit_api_throttling(self):
        """Detect API throttling and rate limiting issues"""
        
        events = self.get_recent_events(hours=24)
        throttling_events = []
        
        for event in events:
            cloud_trail_event = json.loads(event['CloudTrailEvent'])
            
            if cloud_trail_event.get('errorCode') in ['Throttling', 'RequestLimitExceeded']:
                throttling_events.append({
                    'timestamp': event['EventTime'],
                    'event': event['EventName'],
                    'error': cloud_trail_event.get('errorCode'),
                    'principal': cloud_trail_event.get('userIdentity', {}).get('arn'),
                    'service': event['EventSource']
                })
        
        return throttling_events

# Usage
analyzer = CloudTrailAnalyzer()

print("=== Failed Login Attempts ===")
failed_logins = analyzer.analyze_failed_logins()
for user_ip, count in sorted(failed_logins.items(), key=lambda x: x[1], reverse=True):
    print(f"{user_ip}: {count} attempts")

print("\n=== Privilege Escalation Attempts ===")
escalation = analyzer.detect_privilege_escalation()
for activity in escalation:
    print(f"{activity['timestamp']}: {activity['principal']} performed {activity['event']}")

print("\n=== Root Account Usage ===")
root_usage = analyzer.identify_root_account_usage()
print(f"Found {len(root_usage)} root account usages")

print("\n=== API Throttling Events ===")
throttling = analyzer.audit_api_throttling()
print(f"Found {len(throttling)} throttling events")
```

---

### Example 2: AWS Config for Compliance Monitoring

**Scenario**: Organization needs to ensure all S3 buckets have encryption, versioning, and logging enabled.

**Step 1: Enable AWS Config**

```bash
# Create IAM role for Config service
aws iam create-role \
  --role-name ConfigServiceRole \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Principal": {
          "Service": "config.amazonaws.com"
        },
        "Action": "sts:AssumeRole"
      }
    ]
  }'

# Attach managed policy
aws iam attach-role-policy \
  --role-name ConfigServiceRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/ConfigRole

# Create S3 bucket for Config snapshots
aws s3 mb s3://my-config-bucket-$(date +%s) --region us-east-1

CONFIG_BUCKET="my-config-bucket-$(date +%s)"

# Create delivery channel
aws configservice put-delivery-channel \
  --delivery-channel-name default \
  --s3-bucket-name $CONFIG_BUCKET \
  --sns-topic-arn arn:aws:sns:us-east-1:123456789012:config-notifications

# Create configuration recorder
aws configservice put-config-recorder \
  --config-recorder-name default \
  --role-arn arn:aws:iam::123456789012:role/ConfigServiceRole \
  --recording-group allSupported=true,includeGlobalResources=true

# Start recorder
aws configservice start-config-recorder --config-recorder-name default
```

**Step 2: Deploy Compliance Rules**

```bash
# Enable managed rule: S3 bucket encryption
aws configservice put-config-rule \
  --config-rule '{
    "ConfigRuleName": "s3-bucket-server-side-encryption-enabled",
    "Source": {
      "Owner": "AWS",
      "SourceIdentifier": "S3_BUCKET_SERVER_SIDE_ENCRYPTION_ENABLED"
    },
    "Scope": {
      "ComplianceResourceTypes": ["AWS::S3::Bucket"]
    }
  }'

# Enable managed rule: S3 versioning
aws configservice put-config-rule \
  --config-rule '{
    "ConfigRuleName": "s3-bucket-versioning-enabled",
    "Source": {
      "Owner": "AWS",
      "SourceIdentifier": "S3_BUCKET_VERSIONING_ENABLED"
    },
    "Scope": {
      "ComplianceResourceTypes": ["AWS::S3::Bucket"]
    }
  }'

# Enable managed rule: S3 logging
aws configservice put-config-rule \
  --config-rule '{
    "ConfigRuleName": "s3-bucket-logging-enabled",
    "Source": {
      "Owner": "AWS",
      "SourceIdentifier": "S3_BUCKET_LOGGING_ENABLED"
    },
    "Scope": {
      "ComplianceResourceTypes": ["AWS::S3::Bucket"]
    }
  }'

# Create custom rule for S3 public access
aws configservice put-config-rule \
  --config-rule '{
    "ConfigRuleName": "s3-public-access-blocked",
    "Source": {
      "Owner": "CUSTOM_LAMBDA",
      "SourceIdentifier": "arn:aws:lambda:us-east-1:123456789012:function:s3-public-access-check",
      "SourceDetails": [
        {
          "EventSource": "aws.config",
          "MessageType": "ConfigurationItemChangeNotification"
        }
      ]
    },
    "Scope": {
      "ComplianceResourceTypes": ["AWS::S3::Bucket"]
    }
  }'
```

**Step 3: Create Conformance Pack for PCI DSS**

```bash
# Deploy conformance pack for PCI DSS
aws configservice put-conformance-pack \
  --conformance-pack-name pci-dss-conformance \
  --template-s3-uri s3://aws-config-conformance-packs/pci-dss-3.2.1.yaml \
  --conformance-pack-input-parameters '
    [
      {
        "ParameterName": "S3BucketNames",
        "ParameterValue": "my-pci-bucket-1,my-pci-bucket-2"
      }
    ]
  '

# Check conformance pack status
aws configservice describe-conformance-packs \
  --conformance-pack-names pci-dss-conformance \
  --query 'ConformancePackDetails[0]'
```

**Python Application for Config Management**:

```python
import boto3
import json
from datetime import datetime

class AWSConfigCompliance:
    def __init__(self):
        self.config = boto3.client('config')
        self.s3 = boto3.client('s3')
    
    def get_compliance_summary(self):
        """Get overall compliance summary"""
        
        response = self.config.describe_compliance_by_config_rule()
        
        compliance_summary = {
            'compliant': 0,
            'non_compliant': 0,
            'not_applicable': 0,
            'rules': {}
        }
        
        for rule in response['ComplianceByConfigRules']:
            rule_name = rule['ConfigRuleName']
            compliance = rule['Compliance']['ComplianceType']
            
            compliance_summary['rules'][rule_name] = compliance
            
            if compliance == 'COMPLIANT':
                compliance_summary['compliant'] += 1
            elif compliance == 'NON_COMPLIANT':
                compliance_summary['non_compliant'] += 1
            else:
                compliance_summary['not_applicable'] += 1
        
        return compliance_summary
    
    def get_non_compliant_resources(self, rule_name):
        """Get resources non-compliant with specific rule"""
        
        try:
            response = self.config.get_compliance_details_by_config_rule(
                ConfigRuleName=rule_name,
                ComplianceTypes=['NON_COMPLIANT']
            )
            
            non_compliant = []
            for detail in response['EvaluationResults']:
                non_compliant.append({
                    'resource_type': detail['EvaluationResultIdentifier']['EvaluationResultQualifier']['ResourceType'],
                    'resource_id': detail['EvaluationResultIdentifier']['EvaluationResultQualifier']['ResourceId'],
                    'compliance': detail['ComplianceType'],
                    'annotation': detail.get('Annotation', 'N/A')
                })
            
            return non_compliant
        except Exception as e:
            print(f"Error getting details: {e}")
            return []
    
    def remediate_s3_encryption(self, bucket_name):
        """Auto-remediation: Enable S3 encryption"""
        
        try:
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
            print(f"Enabled encryption on {bucket_name}")
            return True
        except Exception as e:
            print(f"Failed to enable encryption: {e}")
            return False
    
    def remediate_s3_versioning(self, bucket_name):
        """Auto-remediation: Enable S3 versioning"""
        
        try:
            self.s3.put_bucket_versioning(
                Bucket=bucket_name,
                VersioningConfiguration={'Status': 'Enabled'}
            )
            print(f"Enabled versioning on {bucket_name}")
            return True
        except Exception as e:
            print(f"Failed to enable versioning: {e}")
            return False
    
    def generate_compliance_report(self):
        """Generate comprehensive compliance report"""
        
        report = {
            'timestamp': datetime.now().isoformat(),
            'summary': self.get_compliance_summary(),
            'non_compliant_by_rule': {}
        }
        
        # Get details for each rule
        for rule_name in report['summary']['rules'].keys():
            if report['summary']['rules'][rule_name] == 'NON_COMPLIANT':
                non_compliant = self.get_non_compliant_resources(rule_name)
                report['non_compliant_by_rule'][rule_name] = non_compliant
        
        return report
    
    def setup_auto_remediation_lambda(self):
        """Setup Lambda for automated remediation"""
        
        lambda_function = """
import json
import boto3

config = boto3.client('config')
s3 = boto3.client('s3')

def lambda_handler(event, context):
    # Parse Config change notification
    configuration_item = json.loads(event['configuration_item_capture_json'])
    
    resource_id = configuration_item['resourceId']
    resource_type = configuration_item['resourceType']
    
    if resource_type == 'AWS::S3::Bucket':
        # Check encryption compliance
        try:
            encryption = s3.get_bucket_encryption(Bucket=resource_id)
        except s3.exceptions.ServerSideEncryptionConfigurationNotFoundError:
            # Enable encryption if not present
            s3.put_bucket_encryption(
                Bucket=resource_id,
                ServerSideEncryptionConfiguration={
                    'Rules': [{'ApplyServerSideEncryptionByDefault': {'SSEAlgorithm': 'AES256'}}]
                }
            )
            print(f"Enabled encryption on {resource_id}")
        
        # Check versioning
        versioning = s3.get_bucket_versioning(Bucket=resource_id)
        if versioning.get('Status') != 'Enabled':
            s3.put_bucket_versioning(
                Bucket=resource_id,
                VersioningConfiguration={'Status': 'Enabled'}
            )
            print(f"Enabled versioning on {resource_id}")
    
    return {
        'statusCode': 200,
        'body': json.dumps('Remediation completed')
    }
"""
        return lambda_function

# Usage
compliance = AWSConfigCompliance()

print("=== Compliance Summary ===")
summary = compliance.get_compliance_summary()
print(f"Compliant: {summary['compliant']}")
print(f"Non-Compliant: {summary['non_compliant']}")

print("\n=== Non-Compliant Resources ===")
non_compliant = compliance.get_non_compliant_resources('s3-bucket-server-side-encryption-enabled')
for resource in non_compliant:
    print(f"{resource['resource_id']}: {resource['annotation']}")

print("\n=== Compliance Report ===")
report = compliance.generate_compliance_report()
print(json.dumps(report, indent=2))
```

---

### Example 3: Combining CloudTrail and Config for Complete Audit Trail

**Scenario**: Detect configuration changes, identify who made them, and trigger remediation.

**Architecture**:

```
Resource Configuration Change
    ↓
CloudTrail records API call
    ↓
Config detects change & evaluates compliance
    ↓
EventBridge rule triggers Lambda
    ↓
Lambda function:
  1. Queries CloudTrail for change details
  2. Identifies user/role
  3. Checks Config compliance
  4. Triggers remediation if needed
    ↓
Notification to security team
```

**Implementation**:

```python
import boto3
import json
from datetime import datetime, timedelta

class AuditAndComplianceOrchestration:
    def __init__(self):
        self.cloudtrail = boto3.client('cloudtrail')
        self.config = boto3.client('config')
        self.sns = boto3.client('sns')
        self.s3 = boto3.client('s3')
    
    def lambda_handler(self, event, context):
        """
        AWS Lambda function triggered by EventBridge when Config detects
        non-compliant resource
        """
        
        # Parse Config non-compliance event
        detail = event['detail']
        config_rule_name = detail['configRuleName']
        resource_type = detail['resourceType']
        resource_id = detail['resourceId']
        compliance_type = detail['newEvaluationResult']['complianceType']
        
        if compliance_type != 'NON_COMPLIANT':
            return {'statusCode': 200}
        
        # Get change details from CloudTrail
        change_details = self.get_change_details(resource_type, resource_id)
        
        # Generate audit report
        audit_report = self.generate_audit_report(
            config_rule_name,
            resource_type,
            resource_id,
            change_details
        )
        
        # Store audit report
        self.store_audit_report(audit_report)
        
        # Attempt remediation
        remediation_result = self.auto_remediate(
            config_rule_name,
            resource_type,
            resource_id
        )
        
        # Send notification
        self.send_notification(audit_report, remediation_result)
        
        return {
            'statusCode': 200,
            'body': json.dumps('Audit and remediation completed')
        }
    
    def get_change_details(self, resource_type, resource_id):
        """Query CloudTrail for resource change details"""
        
        try:
            response = self.cloudtrail.lookup_events(
                LookupAttributes=[
                    {
                        'AttributeKey': 'ResourceName',
                        'AttributeValue': resource_id
                    }
                ],
                StartTime=datetime.now() - timedelta(hours=1)
            )
            
            changes = []
            for event in response['Events'][:5]:  # Last 5 related events
                ct_event = json.loads(event['CloudTrailEvent'])
                changes.append({
                    'timestamp': str(event['EventTime']),
                    'event_name': event['EventName'],
                    'principal': event.get('Username', 'Unknown'),
                    'source_ip': ct_event.get('sourceIPAddress'),
                    'user_agent': ct_event.get('userAgent'),
                    'request_params': ct_event.get('requestParameters'),
                    'response_elements': ct_event.get('responseElements')
                })
            
            return changes
        except Exception as e:
            print(f"Error getting change details: {e}")
            return []
    
    def generate_audit_report(self, rule_name, resource_type, resource_id, changes):
        """Generate comprehensive audit report"""
        
        report = {
            'timestamp': datetime.now().isoformat(),
            'rule_violated': rule_name,
            'resource': {
                'type': resource_type,
                'id': resource_id
            },
            'changes': changes,
            'compliance_status': 'NON_COMPLIANT',
            'investigation_required': len(changes) > 0
        }
        
        # Analyze change patterns
        if len(changes) > 0:
            latest_change = changes[0]
            report['latest_change'] = latest_change
            
            # Check for suspicious activity
            if 'delete' in latest_change['event_name'].lower():
                report['risk_level'] = 'HIGH'
            elif latest_change['source_ip'].startswith('203.0.113'):  # Example IP
                report['risk_level'] = 'MEDIUM'
            else:
                report['risk_level'] = 'LOW'
        
        return report
    
    def store_audit_report(self, report):
        """Store audit report in S3"""
        
        report_key = f"audit-reports/{datetime.now().strftime('%Y/%m/%d')}/{report['resource']['id']}-{datetime.now().timestamp()}.json"
        
        try:
            self.s3.put_object(
                Bucket='my-audit-reports-bucket',
                Key=report_key,
                Body=json.dumps(report, indent=2),
                ContentType='application/json'
            )
            print(f"Stored audit report: {report_key}")
        except Exception as e:
            print(f"Error storing report: {e}")
    
    def auto_remediate(self, rule_name, resource_type, resource_id):
        """Attempt auto-remediation based on rule"""
        
        remediation_actions = {
            's3-bucket-server-side-encryption-enabled': self.remediate_s3_encryption,
            's3-bucket-versioning-enabled': self.remediate_s3_versioning,
            's3-bucket-logging-enabled': self.remediate_s3_logging
        }
        
        if rule_name in remediation_actions and resource_type == 'AWS::S3::Bucket':
            try:
                action = remediation_actions[rule_name]
                action(resource_id)
                return {
                    'status': 'success',
                    'rule': rule_name,
                    'action': 'auto-remediation applied'
                }
            except Exception as e:
                return {
                    'status': 'failed',
                    'rule': rule_name,
                    'error': str(e)
                }
        
        return {
            'status': 'no_action',
            'reason': f'No auto-remediation for rule: {rule_name}'
        }
    
    def remediate_s3_encryption(self, bucket_name):
        """Remediate: Enable S3 encryption"""
        self.s3.put_bucket_encryption(
            Bucket=bucket_name,
            ServerSideEncryptionConfiguration={
                'Rules': [{
                    'ApplyServerSideEncryptionByDefault': {'SSEAlgorithm': 'AES256'}
                }]
            }
        )
    
    def remediate_s3_versioning(self, bucket_name):
        """Remediate: Enable S3 versioning"""
        self.s3.put_bucket_versioning(
            Bucket=bucket_name,
            VersioningConfiguration={'Status': 'Enabled'}
        )
    
    def remediate_s3_logging(self, bucket_name):
        """Remediate: Enable S3 logging"""
        self.s3.put_bucket_logging(
            Bucket=bucket_name,
            BucketLoggingStatus={
                'LoggingEnabled': {
                    'TargetBucket': f'{bucket_name}-logs',
                    'TargetPrefix': f'{bucket_name}/'
                }
            }
        )
    
    def send_notification(self, audit_report, remediation_result):
        """Send SNS notification with audit findings"""
        
        message = f"""
AWS Audit and Compliance Report
================================

Resource: {audit_report['resource']['type']} - {audit_report['resource']['id']}
Rule Violated: {audit_report['rule_violated']}
Risk Level: {audit_report.get('risk_level', 'UNKNOWN')}
Timestamp: {audit_report['timestamp']}

Latest Change:
{json.dumps(audit_report.get('latest_change'), indent=2)}

Remediation Result:
{json.dumps(remediation_result, indent=2)}

Investigation Required: {audit_report['investigation_required']}
"""
        
        try:
            self.sns.publish(
                TopicArn='arn:aws:sns:us-east-1:123456789012:security-alerts',
                Subject=f'[{audit_report.get("risk_level", "INFO")}] Config Non-Compliance Alert',
                Message=message
            )
        except Exception as e:
            print(f"Error sending notification: {e}")
```

---

## Top 5 Interview Questions

### Question 1: Explain the Key Differences Between CloudTrail and AWS Config

**Scenario**: "Your company needs to implement comprehensive audit and compliance tracking. Explain when you would use CloudTrail vs AWS Config, and can they work together?"

**Answer Structure**:

**Fundamental Differences**:

| Aspect | CloudTrail | AWS Config |
|--------|-----------|-----------|
| **Focus** | API activity logging | Resource configuration state |
| **Question Answered** | "Who did what, when, and from where?" | "What is the current state and is it compliant?" |
| **Event Type** | API calls and actions | Configuration snapshots |
| **Trigger** | Every API call | Configuration change or scheduled |
| **Use Case** | Security auditing, incident response | Compliance enforcement, inventory |
| **Historical Data** | Event-level detail | Configuration version history |

**When to Use CloudTrail**:
- Investigating security incidents ("Who deleted the database?")
- Tracking API usage and performance
- Detecting unauthorized access attempts
- Auditing user and application activity
- Compliance with regulations requiring activity logs
- Multi-account and multi-region activity tracking

**When to Use AWS Config**:
- Ensuring resources meet compliance standards
- Tracking configuration drift
- Discovering resources and their relationships
- Automated remediation of non-compliant resources
- Creating resource inventory
- Compliance reporting for frameworks (PCI DSS, HIPAA, etc.)

**Integration**:

```
CloudTrail logs API call: "User modified S3 bucket policy"
                ↓
Config detects change in S3 bucket configuration
                ↓
Config evaluates compliance against rule "s3-bucket-public-access-blocked"
                ↓
Resource marked as NON_COMPLIANT
                ↓
EventBridge triggers Lambda function
                ↓
Lambda queries CloudTrail to identify:
  - Which user made the change
  - When the change occurred
  - Source IP address
  - Change parameters
                ↓
Lambda logs findings and triggers remediation
```

**Practical Example**:

```python
# CloudTrail: Answers WHO and WHEN
# "On 2024-01-15 at 14:32:15, user arn:aws:iam::123456789012:user/john.doe 
#  made API call PutBucketPolicy on s3://production-data from IP 203.0.113.45"

# AWS Config: Answers COMPLIANCE
# "S3 bucket 'production-data' is currently non-compliant because 
#  it allows public access, violating rule 's3-block-public-access'"

# Combined: Complete audit trail
# "User john.doe made configuration change that caused non-compliance. 
#  Auto-remediation has corrected the issue. Full change history and 
#  user activity logged for investigation."
```

---

### Question 2: Design a Multi-Account Audit Strategy Using CloudTrail and Config

**Scenario**: "Your organization has 50 AWS accounts across multiple regions. Design a centralized audit and compliance monitoring solution that provides visibility into all accounts while maintaining security boundaries."

**Answer Structure**:

**Architecture Overview**:

```
┌─────────────────────────────────────────────────────────────┐
│ Organization Management Account (Central)                  │
├─────────────────────────────────────────────────────────────┤
│  • CloudTrail Organization Trail                           │
│  • AWS Config Organization Conformance Packs               │
│  • Central S3 Buckets for logs                             │
│  • Security Dashboard (CloudWatch, QuickSight)             │
│  • Alerting (SNS, EventBridge, SecurityHub)               │
└──────────────────────┬──────────────────────────────────────┘
                       │
     ┌─────────────────┼─────────────────┐
     │                 │                 │
┌────▼─────────┐  ┌────▼─────────┐  ┌───▼──────────┐
│ Account 1    │  │ Account 2    │  │ Account N    │
│ (Production) │  │ (Staging)    │  │ (Dev)        │
│              │  │              │  │              │
│ Config Rules │  │ Config Rules │  │ Config Rules │
│ CloudTrail   │  │ CloudTrail   │  │ CloudTrail   │
│ Logs ────────┼─→│ Logs ────────┼─→│ Logs ────────┼──→ Central Bucket
└──────────────┘  └──────────────┘  └──────────────┘
```

**Implementation Steps**:

**Step 1: Setup Organization Trail (Central Account)**

```bash
# Enable organization trail in management account
aws cloudtrail create-organization-trail \
  --name OrganizationTrail \
  --s3-bucket-name org-cloudtrail-logs-bucket \
  --is-multi-region-trail \
  --enable-log-file-validation \
  --include-global-service-events

# Enable for all member accounts
aws organizations enable-trusted-access \
  --service-principal cloudtrail.amazonaws.com

# Start logging
aws cloudtrail start-organization-trail \
  --name OrganizationTrail
```

**Step 2: Setup Config Organization**

```bash
# Create Config aggregator in management account
aws configservice put-configuration-aggregator \
  --configuration-aggregator-name organization-aggregator \
  --account-aggregation-sources '[
    {
      "AllAwsRegions": true,
      "AccountIds": ["111111111111", "222222222222", "333333333333"]
    }
  ]'

# Setup authorization for member accounts (run in each member account)
aws configservice put-aggregation-authorization \
  --authorized-account-id 123456789012 \
  --authorized-aws-region us-east-1
```

**Step 3: Deploy Organization Conformance Packs**

```bash
# Deploy conformance pack across all accounts
aws configservice put-organization-conformance-pack \
  --organization-conformance-pack-name cis-benchmark-conformance \
  --template-s3-uri s3://aws-config-conformance-packs/cis-benchmark-v1.4.yaml \
  --excluded-accounts [] \
  --organization-conformance-pack-input-parameters '
    [
      {
        "ParameterName": "RequiredTags",
        "ParameterValue": "Environment,Owner,CostCenter"
      }
    ]
  '
```

**Step 4: Centralized Monitoring and Alerting**

```python
import boto3
import json
from datetime import datetime

class MultiAccountAuditMonitor:
    def __init__(self, management_account_id):
        self.management_account_id = management_account_id
        self.config = boto3.client('config')
        self.cloudtrail = boto3.client('cloudtrail')
        self.organizations = boto3.client('organizations')
        self.cloudwatch = boto3.client('cloudwatch')
    
    def get_all_member_accounts(self):
        """Get all accounts in organization"""
        paginator = self.organizations.get_paginator('list_accounts')
        accounts = []
        
        for page in paginator.paginate():
            for account in page['Accounts']:
                if account['Status'] == 'ACTIVE':
                    accounts.append(account['Id'])
        
        return accounts
    
    def aggregate_compliance_status(self):
        """Get aggregated compliance status across all accounts"""
        
        response = self.config.get_aggregate_compliance_details_by_config_rule(
            ConfigurationAggregatorName='organization-aggregator',
            AwsRegions=['us-east-1', 'eu-west-1'],
            ComplianceType='NON_COMPLIANT'
        )
        
        compliance_summary = {
            'total_non_compliant': 0,
            'by_account': {},
            'by_rule': {},
            'by_region': {}
        }
        
        for detail in response['AggregateEvaluationResults']:
            account = detail['EvaluationResultIdentifier']['EvaluationResultQualifier']['AccountId']
            region = detail['EvaluationResultIdentifier']['EvaluationResultQualifier']['AwsRegion']
            rule = detail['EvaluationResultIdentifier']['EvaluationResultQualifier']['ConfigRuleName']
            
            compliance_summary['total_non_compliant'] += 1
            compliance_summary['by_account'][account] = compliance_summary['by_account'].get(account, 0) + 1
            compliance_summary['by_rule'][rule] = compliance_summary['by_rule'].get(rule, 0) + 1
            compliance_summary['by_region'][region] = compliance_summary['by_region'].get(region, 0) + 1
        
        return compliance_summary
    
    def detect_cross_account_anomalies(self):
        """Detect unusual API activity across all accounts"""
        
        # Query CloudTrail for failed API calls in last 24 hours
        response = self.cloudtrail.lookup_events(
            LookupAttributes=[
                {
                    'AttributeKey': 'EventName',
                    'AttributeValue': 'AssumeRole'
                }
            ]
        )
        
        anomalies = []
        for event in response['Events']:
            ct_event = json.loads(event['CloudTrailEvent'])
            
            # Alert if cross-account assume role from unexpected source
            if ct_event.get('errorCode') == 'AccessDenied':
                anomalies.append({
                    'timestamp': str(event['EventTime']),
                    'event': event['EventName'],
                    'principal': event.get('Username'),
                    'source_ip': ct_event.get('sourceIPAddress'),
                    'error': ct_event.get('errorCode')
                })
        
        return anomalies
    
    def create_compliance_dashboard(self):
        """Create CloudWatch dashboard for compliance monitoring"""
        
        dashboard_body = {
            'widgets': [
                {
                    'type': 'metric',
                    'properties': {
                        'metrics': [
                            ['AWS/Config', 'ComplianceScore', {'stat': 'Average'}]
                        ],
                        'period': 300,
                        'stat': 'Average',
                        'region': 'us-east-1',
                        'title': 'Overall Compliance Score'
                    }
                },
                {
                    'type': 'metric',
                    'properties': {
                        'metrics': [
                            ['AWS/Config', 'NonCompliantResourceCount', {'stat': 'Sum'}]
                        ],
                        'period': 300,
                        'stat': 'Sum',
                        'region': 'us-east-1',
                        'title': 'Non-Compliant Resources'
                    }
                }
            ]
        }
        
        self.cloudwatch.put_dashboard(
            DashboardName='OrganizationComplianceDashboard',
            DashboardBody=json.dumps(dashboard_body)
        )
```

---

### Question 3: Troubleshoot CloudTrail Configuration and Event Missing Issues

**Scenario**: "You suspect that some CloudTrail events are not being logged properly. Walk through your troubleshooting methodology to identify and resolve the issue."

**Answer Structure**:

**Troubleshooting Flowchart**:

```
CloudTrail Events Missing
    ↓
Step 1: Verify Trail Status
├─ Is trail enabled?
├─ Is logging active?
└─ Correct region/bucket?
    ↓
Step 2: Check S3 Bucket
├─ Bucket exists?
├─ CloudTrail has write permissions?
├─ Bucket policy correct?
└─ Encryption configured?
    ↓
Step 3: Review Event Selectors
├─ Management events enabled?
├─ Data events configured?
└─ Include global service events?
    ↓
Step 4: Check Permissions and Roles
├─ CloudTrail service role exists?
├─ IAM policy has required permissions?
└─ Trust relationship correct?
    ↓
Step 5: Analyze Event History
├─ Look for API errors
├─ Check for access denied messages
└─ Review CloudTrail logs
    ↓
Step 6: Verify KMS Encryption
├─ If using custom KMS key
├─ Key policy allows CloudTrail
└─ Key is accessible
```

**Diagnostic Commands**:

```bash
#!/bin/bash

TRAIL_NAME="OrganizationTrail"
REGION="us-east-1"

echo "=== CloudTrail Troubleshooting ==="

# 1. Check trail status
echo "1. Checking trail status..."
aws cloudtrail describe-trails \
  --trail-name-list $TRAIL_NAME \
  --region $REGION \
  --query 'trailList[0].[TrailARN, HasCustomEventSelectors, IsMultiRegionTrail, HomeRegion, HasInsightSelectors]'

# 2. Check if logging is active
echo "2. Checking if trail is logging..."
aws cloudtrail get-trail-status \
  --name $TRAIL_NAME \
  --region $REGION

# 3. Check event selectors
echo "3. Checking event selectors..."
aws cloudtrail get-event-selectors \
  --trail-name $TRAIL_NAME \
  --region $REGION

# 4. List recent events
echo "4. Recent CloudTrail events..."
aws cloudtrail lookup-events \
  --trail-name $TRAIL_NAME \
  --max-results 10 \
  --region $REGION

# 5. Check S3 bucket permissions
echo "5. Checking S3 bucket policy..."
S3_BUCKET=$(aws cloudtrail describe-trails \
  --trail-name-list $TRAIL_NAME \
  --region $REGION \
  --query 'trailList[0].S3BucketName' \
  --output text)

aws s3api get-bucket-policy --bucket $S3_BUCKET | jq '.'

# 6. Verify bucket exists and is accessible
echo "6. Verifying bucket accessibility..."
aws s3api head-bucket --bucket $S3_BUCKET && echo "Bucket accessible" || echo "Bucket not accessible"

# 7. Check for errors in event history
echo "7. Looking for errors in event history..."
aws cloudtrail lookup-events \
  --max-results 50 \
  --region $REGION | jq '.Events[] | select(.CloudTrailEvent | contains("Error"))'

# 8. Check for access denied messages
echo "8. Checking for access denied..."
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=AccessDenied \
  --max-results 10 \
  --region $REGION

# 9. Test trail with manual API call
echo "9. Making test API call to generate event..."
aws ec2 describe-instances --max-results 1

# 10. Check if test event was logged
echo "10. Checking if test event was logged..."
sleep 5
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=DescribeInstances \
  --max-results 5
```

**Python Troubleshooting Script**:

```python
import boto3
import json
from datetime import datetime, timedelta

class CloudTrailTroubleshooter:
    def __init__(self, trail_name, region='us-east-1'):
        self.trail_name = trail_name
        self.region = region
        self.cloudtrail = boto3.client('cloudtrail', region_name=region)
        self.s3 = boto3.client('s3')
        self.iam = boto3.client('iam')
    
    def comprehensive_audit(self):
        """Run comprehensive CloudTrail audit"""
        
        issues_found = []
        
        # Check 1: Trail exists and enabled
        print("Checking trail configuration...")
        trail = self.get_trail_config()
        if not trail:
            issues_found.append("Trail not found")
            return issues_found
        
        # Check 2: Logging status
        print("Checking logging status...")
        trail_status = self.get_trail_status()
        if not trail_status.get('IsLogging'):
            issues_found.append("Trail is not actively logging")
        
        if trail_status.get('LatestDeliveryError'):
            issues_found.append(f"Delivery error: {trail_status['LatestDeliveryError']}")
        
        # Check 3: S3 bucket permissions
        print("Checking S3 bucket permissions...")
        s3_issues = self.check_s3_permissions(trail['S3BucketName'])
        issues_found.extend(s3_issues)
        
        # Check 4: Event selectors
        print("Checking event selectors...")
        event_selector_issues = self.check_event_selectors()
        issues_found.extend(event_selector_issues)
        
        # Check 5: IAM role permissions
        print("Checking IAM permissions...")
        iam_issues = self.check_iam_permissions()
        issues_found.extend(iam_issues)
        
        # Check 6: Recent events
        print("Checking recent events...")
        event_issues = self.check_recent_events()
        issues_found.extend(event_issues)
        
        # Check 7: KMS key (if applicable)
        if trail.get('KMSKeyId'):
            print("Checking KMS key...")
            kms_issues = self.check_kms_permissions(trail['KMSKeyId'])
            issues_found.extend(kms_issues)
        
        return issues_found
    
    def get_trail_config(self):
        """Get trail configuration"""
        try:
            response = self.cloudtrail.describe_trails(
                trailNameList=[self.trail_name]
            )
            return response['trailList'][0] if response['trailList'] else None
        except Exception as e:
            print(f"Error getting trail config: {e}")
            return None
    
    def get_trail_status(self):
        """Get trail logging status"""
        try:
            response = self.cloudtrail.get_trail_status(name=self.trail_name)
            return response
        except Exception as e:
            print(f"Error getting trail status: {e}")
            return {}
    
    def check_s3_permissions(self, bucket_name):
        """Verify S3 bucket has CloudTrail permissions"""
        issues = []
        
        try:
            # Check if bucket exists
            self.s3.head_bucket(Bucket=bucket_name)
        except self.s3.exceptions.NoSuchBucket:
            issues.append(f"S3 bucket '{bucket_name}' does not exist")
            return issues
        except Exception as e:
            issues.append(f"Cannot access S3 bucket: {e}")
            return issues
        
        # Check bucket policy
        try:
            policy = self.s3.get_bucket_policy(Bucket=bucket_name)
            policy_doc = json.loads(policy['Policy'])
            
            has_cloudtrail_access = False
            for statement in policy_doc.get('Statement', []):
                if statement.get('Principal', {}).get('Service') == 'cloudtrail.amazonaws.com':
                    if 's3:PutObject' in statement.get('Action', []):
                        has_cloudtrail_access = True
            
            if not has_cloudtrail_access:
                issues.append("S3 bucket policy does not grant CloudTrail PutObject permission")
        
        except self.s3.exceptions.NoSuchBucketPolicy:
            issues.append("S3 bucket policy not found")
        except Exception as e:
            issues.append(f"Error checking S3 policy: {e}")
        
        return issues
    
    def check_event_selectors(self):
        """Verify event selectors are properly configured"""
        issues = []
        
        try:
            response = self.cloudtrail.get_event_selectors(trailName=self.trail_name)
            
            # Check if at least management events are enabled
            management_enabled = False
            for selector in response['EventSelectors']:
                if selector.get('ReadWriteType') == 'All' and selector.get('IncludeManagementEvents'):
                    management_enabled = True
            
            if not management_enabled:
                issues.append("Management events not properly enabled in event selectors")
        
        except Exception as e:
            issues.append(f"Error checking event selectors: {e}")
        
        return issues
    
    def check_iam_permissions(self):
        """Verify IAM role has required permissions"""
        issues = []
        
        # This would require checking the role policy
        # Implementation depends on your specific setup
        return issues
    
    def check_recent_events(self):
        """Check for recent events and errors"""
        issues = []
        
        try:
            response = self.cloudtrail.lookup_events(
                maxResults=50,
                StartTime=datetime.now() - timedelta(hours=24)
            )
            
            error_count = 0
            for event in response['Events']:
                ct_event = json.loads(event['CloudTrailEvent'])
                if 'errorCode' in ct_event:
                    error_count += 1
            
            if error_count > 10:
                issues.append(f"High number of errors in recent events ({error_count})")
        
        except Exception as e:
            issues.append(f"Error checking recent events: {e}")
        
        return issues
    
    def check_kms_permissions(self, kms_key_id):
        """Verify KMS key allows CloudTrail"""
        issues = []
        
        # Implementation would check KMS key policy
        # Check if CloudTrail service has decrypt permissions
        
        return issues

# Usage
troubleshooter = CloudTrailTroubleshooter('OrganizationTrail')
issues = troubleshooter.comprehensive_audit()

if issues:
    print("Issues found:")
    for issue in issues:
        print(f"  - {issue}")
else:
    print("No issues found. CloudTrail is properly configured.")
```

---

### Question 4: Design AWS Config Remediation Strategy for Continuous Compliance

**Scenario**: "Design an automated remediation system that detects non-compliant resources, attempts remediation, and notifies the security team if auto-remediation fails."

**Answer Structure**:

**Remediation Strategy**:

```
Non-Compliant Resource Detected
    ↓
EventBridge captures Config change
    ↓
Lambda evaluates remediability
    ├─ Can be auto-remediated? → YES
    │   ├─ Attempt remediation
    │   ├─ If success → Log and notify
    │   └─ If failure → Alert security team
    │
    └─ Cannot be auto-remediated? → NO
        └─ Escalate to security team

Remediation Success Path:
    Resource non-compliant
        ↓
    Lambda applies remediation
        ↓
    Resource becomes compliant
        ↓
    Config verifies compliance
        ↓
    Notification (audit trail)

Remediation Failure Path:
    Resource non-compliant
        ↓
    Lambda attempts remediation
        ↓
    Remediation fails (permission denied, etc.)
        ↓
    Alert security team with details
        ↓
    Create ticket in incident management
        ↓
    Security team reviews and manually remediates
```

**Implementation**:

```python
import boto3
import json
from datetime import datetime
from enum import Enum

class RemediationStatus(Enum):
    SUCCESS = "success"
    PARTIAL = "partial"
    FAILED = "failed"
    NOT_APPLICABLE = "not_applicable"

class ConfigRemediationEngine:
    def __init__(self):
        self.config = boto3.client('config')
        self.s3 = boto3.client('s3')
        self.ec2 = boto3.client('ec2')
        self.iam = boto3.client('iam')
        self.sns = boto3.client('sns')
        self.dynamodb = boto3.client('dynamodb')
    
    def lambda_handler(self, event, context):
        """EventBridge-triggered Lambda for auto-remediation"""
        
        # Parse Config non-compliance event
        detail = event['detail']
        config_rule_name = detail['configRuleName']
        resource_type = detail['resourceType']
        resource_id = detail['resourceId']
        
        print(f"Processing non-compliance: {config_rule_name} on {resource_id}")
        
        # Get remediation strategy
        remediation_strategy = self.get_remediation_strategy(
            config_rule_name,
            resource_type
        )
        
        if not remediation_strategy:
            self.escalate_to_security(
                config_rule_name,
                resource_id,
                "No auto-remediation strategy defined"
            )
            return
        
        # Attempt remediation
        result = self.execute_remediation(
            config_rule_name,
            resource_type,
            resource_id,
            remediation_strategy
        )
        
        # Log remediation attempt
        self.log_remediation_attempt(
            config_rule_name,
            resource_id,
            result
        )
        
        # Send notifications
        if result['status'] == RemediationStatus.SUCCESS:
            self.send_success_notification(config_rule_name, resource_id)
        else:
            self.escalate_to_security(
                config_rule_name,
                resource_id,
                result['error'],
                result
            )
    
    def get_remediation_strategy(self, rule_name, resource_type):
        """Get remediation strategy for specific rule"""
        
        strategies = {
            's3-bucket-server-side-encryption-enabled': {
                'resource_type': 'AWS::S3::Bucket',
                'action': 'enable_s3_encryption',
                'auto_remediate': True
            },
            's3-bucket-versioning-enabled': {
                'resource_type': 'AWS::S3::Bucket',
                'action': 'enable_s3_versioning',
                'auto_remediate': True
            },
            's3-bucket-public-read-prohibited': {
                'resource_type': 'AWS::S3::Bucket',
                'action': 'block_s3_public_access',
                'auto_remediate': True
            },
            'iam-password-policy-check': {
                'resource_type': 'AWS::IAM::PasswordPolicy',
                'action': 'update_password_policy',
                'auto_remediate': True
            },
            'restricted-ssh': {
                'resource_type': 'AWS::EC2::SecurityGroup',
                'action': 'restrict_ssh_access',
                'auto_remediate': False,  # Requires manual approval
                'reason': 'May impact running services'
            },
            'ec2-imdsv2-check': {
                'resource_type': 'AWS::EC2::Instance',
                'action': 'enable_imdsv2',
                'auto_remediate': False,  # Requires instance restart
                'reason': 'May impact running applications'
            }
        }
        
        return strategies.get(rule_name)
    
    def execute_remediation(self, rule_name, resource_type, resource_id, strategy):
        """Execute remediation action"""
        
        if not strategy.get('auto_remediate'):
            return {
                'status': RemediationStatus.NOT_APPLICABLE,
                'reason': strategy.get('reason', 'Not configured for auto-remediation')
            }
        
        action_map = {
            'enable_s3_encryption': self.enable_s3_encryption,
            'enable_s3_versioning': self.enable_s3_versioning,
            'block_s3_public_access': self.block_s3_public_access,
            'update_password_policy': self.update_password_policy,
            'restrict_ssh_access': self.restrict_ssh_access,
            'enable_imdsv2': self.enable_imdsv2
        }
        
        action = action_map.get(strategy['action'])
        if not action:
            return {
                'status': RemediationStatus.FAILED,
                'error': f"Unknown remediation action: {strategy['action']}"
            }
        
        try:
            result = action(resource_id)
            return {
                'status': RemediationStatus.SUCCESS,
                'result': result
            }
        except Exception as e:
            return {
                'status': RemediationStatus.FAILED,
                'error': str(e),
                'error_type': type(e).__name__
            }
    
    def enable_s3_encryption(self, bucket_name):
        """Remediation: Enable S3 server-side encryption"""
        self.s3.put_bucket_encryption(
            Bucket=bucket_name,
            ServerSideEncryptionConfiguration={
                'Rules': [{
                    'ApplyServerSideEncryptionByDefault': {'SSEAlgorithm': 'AES256'}
                }]
            }
        )
        return f"Enabled AES256 encryption on {bucket_name}"
    
    def enable_s3_versioning(self, bucket_name):
        """Remediation: Enable S3 versioning"""
        self.s3.put_bucket_versioning(
            Bucket=bucket_name,
            VersioningConfiguration={'Status': 'Enabled'}
        )
        return f"Enabled versioning on {bucket_name}"
    
    def block_s3_public_access(self, bucket_name):
        """Remediation: Block all S3 public access"""
        self.s3.put_public_access_block(
            Bucket=bucket_name,
            PublicAccessBlockConfiguration={
                'BlockPublicAcls': True,
                'IgnorePublicAcls': True,
                'BlockPublicPolicy': True,
                'RestrictPublicBuckets': True
            }
        )
        return f"Blocked all public access on {bucket_name}"
    
    def update_password_policy(self, _):
        """Remediation: Update IAM password policy"""
        self.iam.update_account_password_policy(
            MinimumPasswordLength=14,
            RequireSymbols=True,
            RequireNumbers=True,
            RequireUppercaseCharacters=True,
            RequireLowercaseCharacters=True,
            AllowUsersToChangePassword=True,
            ExpirePasswords=True,
            MaxPasswordAge=90,
            PasswordReusePrevention=24,
            HardExpiry=False
        )
        return "Updated IAM password policy"
    
    def restrict_ssh_access(self, sg_id):
        """Remediation: Restrict SSH access to security group"""
        # Remove overly permissive SSH rule
        self.ec2.revoke_security_group_ingress(
            GroupId=sg_id,
            IpPermissions=[
                {
                    'IpProtocol': 'tcp',
                    'FromPort': 22,
                    'ToPort': 22,
                    'IpRanges': [{'CidrIp': '0.0.0.0/0', 'Description': 'SSH from anywhere'}]
                }
            ]
        )
        return f"Restricted SSH access on {sg_id}"
    
    def enable_imdsv2(self, instance_id):
        """Remediation: Enable IMDSv2 on EC2 instance"""
        self.ec2.modify_instance_metadata_options(
            InstanceId=instance_id,
            HttpTokens='required',
            HttpPutResponseHopLimit=1
        )
        return f"Enabled IMDSv2 on {instance_id}"
    
    def log_remediation_attempt(self, rule_name, resource_id, result):
        """Log remediation attempt to DynamoDB for audit"""
        
        self.dynamodb.put_item(
            TableName='CloudConfigRemediationLogs',
            Item={
                'rule_name': {'S': rule_name},
                'resource_id': {'S': resource_id},
                'timestamp': {'S': datetime.now().isoformat()},
                'status': {'S': result['status'].value},
                'details': {'S': json.dumps(result)}
            }
        )
    
    def send_success_notification(self, rule_name, resource_id):
        """Send success notification"""
        
        message = f"""
AWS Config Auto-Remediation Success
====================================

Rule: {rule_name}
Resource: {resource_id}
Status: COMPLIANT (auto-remediated)
Timestamp: {datetime.now().isoformat()}

The non-compliant resource has been automatically remediated and is now compliant.
"""
        
        self.sns.publish(
            TopicArn='arn:aws:sns:us-east-1:123456789012:config-remediation-success',
            Subject=f'[SUCCESS] Config Auto-Remediation: {rule_name}',
            Message=message
        )
    
    def escalate_to_security(self, rule_name, resource_id, error, result=None):
        """Escalate to security team for manual intervention"""
        
        message = f"""
AWS Config Remediation Required - Manual Intervention
=======================================================

Rule: {rule_name}
Resource: {resource_id}
Status: REQUIRES MANUAL REMEDIATION
Timestamp: {datetime.now().isoformat()}

Error: {error}
"""
        
        if result:
            message += f"\nDetails: {json.dumps(result, indent=2)}"
        
        self.sns.publish(
            TopicArn='arn:aws:sns:us-east-1:123456789012:security-escalation',
            Subject=f'[ESCALATION] Config Remediation Required: {rule_name}',
            Message=message
        )
```

---

### Question 5: Analyze and Investigate Security Incident Using CloudTrail and Config

**Scenario**: "You detect an unauthorized IAM policy attachment through AWS Config. Walk through how you would investigate the incident using both CloudTrail and Config to determine what happened, who did it, and how to prevent it."

**Answer Structure**:

**Investigation Methodology**:

```
Detection: Unauthorized IAM Policy Attached
    ↓
Step 1: Get Non-Compliance Event Details (Config)
├─ Rule violated: iam-policy-check
├─ Resource: IAM Role
├─ Non-compliance time: 2024-01-15 14:32:00 UTC
    ↓
Step 2: Query CloudTrail for Related Events
├─ Look for: AttachRolePolicy, PutRolePolicyDocument
├─ Time range: ±30 minutes around compliance event
├─ Filter by resource ID
    ↓
Step 3: Identify the Principal (User/Role)
├─ Extract: username/role ARN
├─ Check: Is this a human user or application?
├─ Verify: Does this principal have authorization?
    ↓
Step 4: Analyze the Context
├─ Source IP address
├─ User agent (console vs API vs CLI)
├─ Timestamp analysis
├─ Related API calls
    ↓
Step 5: Determine Root Cause
├─ Accidental change?
├─ Compromised credentials?
├─ Insider threat?
├─ Misconfiguration?
    ↓
Step 6: Implement Preventive Measures
├─ Tighten IAM policies
├─ Enable MFA for sensitive operations
├─ Implement SCPs
└─ Update Config rules
```

**Practical Investigation**:

```python
import boto3
import json
from datetime import datetime, timedelta
from collections import defaultdict

class SecurityIncidentInvestigator:
    def __init__(self):
        self.cloudtrail = boto3.client('cloudtrail')
        self.config = boto3.client('config')
        self.iam = boto3.client('iam')
    
    def investigate_unauthorized_policy_attachment(self, role_name, compliance_change_time):
        """Complete investigation of unauthorized policy attachment"""
        
        investigation = {
            'incident': 'Unauthorized IAM Policy Attachment',
            'target_resource': role_name,
            'detection_time': compliance_change_time.isoformat(),
            'investigation_start': datetime.now().isoformat(),
            'findings': {}
        }
        
        # Step 1: Get Config compliance details
        print("Step 1: Getting Config compliance event details...")
        config_details = self.get_config_event_details(role_name)
        investigation['findings']['config_details'] = config_details
        
        # Step 2: Query CloudTrail for policy attachment events
        print("Step 2: Querying CloudTrail for policy attachment events...")
        ct_events = self.query_policy_attachment_events(
            role_name,
            compliance_change_time
        )
        investigation['findings']['cloudtrail_events'] = ct_events
        
        # Step 3: Identify the principal
        print("Step 3: Identifying the principal...")
        principal_info = self.identify_principal(ct_events)
        investigation['findings']['principal'] = principal_info
        
        # Step 4: Analyze context
        print("Step 4: Analyzing context...")
        context_analysis = self.analyze_context(ct_events, principal_info)
        investigation['findings']['context_analysis'] = context_analysis
        
        # Step 5: Determine root cause
        print("Step 5: Determining root cause...")
        root_cause = self.determine_root_cause(
            principal_info,
            context_analysis,
            ct_events
        )
        investigation['findings']['root_cause'] = root_cause
        
        # Step 6: Recommend preventive measures
        print("Step 6: Recommending preventive measures...")
        preventive_measures = self.recommend_preventive_measures(root_cause)
        investigation['findings']['preventive_measures'] = preventive_measures
        
        return investigation
    
    def query_policy_attachment_events(self, role_name, event_time, hours_before=1):
        """Query CloudTrail for IAM policy attachment events"""
        
        start_time = event_time - timedelta(hours=hours_before)
        end_time = event_time + timedelta(minutes=30)
        
        policy_events = []
        
        for event_name in ['AttachRolePolicy', 'PutRolePolicyDocument', 'PutUserPolicy', 'AttachUserPolicy']:
            try:
                response = self.cloudtrail.lookup_events(
                    LookupAttributes=[
                        {
                            'AttributeKey': 'EventName',
                            'AttributeValue': event_name
                        }
                    ],
                    StartTime=start_time,
                    EndTime=end_time,
                    MaxResults=50
                )
                
                for event in response['Events']:
                    ct_event = json.loads(event['CloudTrailEvent'])
                    
                    # Check if event targets our role
                    request_params = ct_event.get('requestParameters', {})
                    if 'roleName' in request_params and request_params['roleName'] == role_name:
                        policy_events.append({
                            'event_name': event['EventName'],
                            'timestamp': str(event['EventTime']),
                            'principal': event.get('Username'),
                            'source_ip': ct_event.get('sourceIPAddress'),
                            'user_agent': ct_event.get('userAgent'),
                            'request_params': request_params,
                            'response_elements': ct_event.get('responseElements'),
                            'error_code': ct_event.get('errorCode'),
                            'raw_event': ct_event
                        })
            
            except Exception as e:
                print(f"Error querying {event_name}: {e}")
        
        return policy_events
    
    def identify_principal(self, ct_events):
        """Identify who made the change"""
        
        if not ct_events:
            return None
        
        # Get most relevant event (typically the most recent)
        event = ct_events[0]
        principal = event['principal']
        
        principal_info = {
            'identifier': principal,
            'source_ip': event['source_ip'],
            'user_agent': event['user_agent'],
            'timestamp': event['timestamp']
        }
        
        # Try to resolve principal ARN
        try:
            if ':root' in principal:
                principal_info['type'] = 'root_account'
            elif 'assumed-role' in principal:
                principal_info['type'] = 'assumed_role'
            elif '@' in principal:
                principal_info['type'] = 'iam_user'
            else:
                principal_info['type'] = 'unknown'
        except:
            pass
        
        return principal_info
    
    def analyze_context(self, ct_events, principal_info):
        """Analyze context of the change"""
        
        context = {
            'event_count': len(ct_events),
            'time_span': None,
            'events': ct_events,
            'patterns': []
        }
        
        if ct_events:
            timestamps = [datetime.fromisoformat(e['timestamp'].replace('Z', '+00:00')) for e in ct_events]
            context['time_span'] = (max(timestamps) - min(timestamps)).total_seconds()
            
            # Detect patterns
            ips = [e['source_ip'] for e in ct_events]
            if len(set(ips)) > 1:
                context['patterns'].append('Multiple source IPs detected')
            
            # Check for off-hours activity
            for event in ct_events:
                event_hour = datetime.fromisoformat(event['timestamp'].replace('Z', '+00:00')).hour
                if event_hour < 6 or event_hour > 22:
                    context['patterns'].append(f"Off-hours activity at {event_hour}:00")
            
            # Check for console vs API
            user_agents = [e['user_agent'] for e in ct_events]
            if any('console.aws.amazon.com' in ua for ua in user_agents):
                context['patterns'].append('Change made via AWS Console')
            if any('aws-cli' in ua for ua in user_agents):
                context['patterns'].append('Change made via AWS CLI')
        
        return context
    
    def determine_root_cause(self, principal_info, context_analysis, ct_events):
        """Determine likely root cause"""
        
        root_cause = {
            'likelihood': 'unknown',
            'scenarios': [],
            'confidence': 0
        }
        
        if not principal_info:
            root_cause['scenarios'].append({
                'scenario': 'Unknown Principal',
                'description': 'Unable to determine who made the change',
                'severity': 'HIGH'
            })
            return root_cause
        
        # Analyze principal type
        if principal_info['type'] == 'root_account':
            root_cause['scenarios'].append({
                'scenario': 'Root Account Usage',
                'description': 'AWS root account was used to make the change',
                'severity': 'CRITICAL',
                'recommendation': 'Root account should only be used for specific tasks with MFA'
            })
        
        # Check for off-hours activity
        patterns = context_analysis.get('patterns', [])
        if 'Off-hours activity' in str(patterns):
            root_cause['scenarios'].append({
                'scenario': 'Off-Hours Change',
                'description': 'Change made during off-business hours',
                'severity': 'HIGH',
                'recommendation': 'Investigate if change was authorized'
            })
        
        # Check source IP
        source_ip = principal_info['source_ip']
        if source_ip.startswith('203.0.113'):  # Example: documentation/test IP
            root_cause['scenarios'].append({
                'scenario': 'Unknown Source IP',
                'description': f'Change from unknown IP: {source_ip}',
                'severity': 'HIGH',
                'recommendation': 'Verify if IP is expected'
            })
        
        # Multiple events suggest automation or script
        if context_analysis['event_count'] > 3:
            root_cause['scenarios'].append({
                'scenario': 'Automated/Script Activity',
                'description': f'{context_analysis["event_count"]} related events detected',
                'severity': 'MEDIUM',
                'recommendation': 'Check if this was part of planned automation'
            })
        
        # Determine overall likelihood
        if len(root_cause['scenarios']) == 0:
            root_cause['likelihood'] = 'accidental'
            root_cause['confidence'] = 0.3
        elif any(s['severity'] == 'CRITICAL' for s in root_cause['scenarios']):
            root_cause['likelihood'] = 'compromised_credentials'
            root_cause['confidence'] = 0.8
        elif any(s['severity'] == 'HIGH' for s in root_cause['scenarios']):
            root_cause['likelihood'] = 'insider_threat'
            root_cause['confidence'] = 0.6
        
        return root_cause
    
    def recommend_preventive_measures(self, root_cause):
        """Recommend preventive measures based on root cause"""
        
        measures = {
            'immediate_actions': [],
            'preventive_controls': [],
            'detective_controls': []
        }
        
        likelihood = root_cause.get('likelihood', 'unknown')
        
        # Immediate actions
        if likelihood in ['compromised_credentials', 'insider_threat']:
            measures['immediate_actions'].append('Review and reset affected IAM credentials')
            measures['immediate_actions'].append('Review all recent actions by affected principal')
            measures['immediate_actions'].append('Consider temporary suspension of IAM user')
        
        # Preventive controls
        measures['preventive_controls'].append('Implement MFA for all IAM users')
        measures['preventive_controls'].append('Use Service Control Policies (SCPs) to restrict policy attachment')
        measures['preventive_controls'].append('Require approval for sensitive IAM changes')
        measures['preventive_controls'].append('Implement least privilege access')
        
        # Detective controls
        measures['detective_controls'].append('Enable CloudTrail logging with log file validation')
        measures['detective_controls'].append('Create Config rules for unauthorized policy detection')
        measures['detective_controls'].append('Setup EventBridge rules for policy change notifications')
        measures['detective_controls'].append('Monitor for off-hours IAM changes')
        
        return measures
    
    def get_config_event_details(self, role_name):
        """Get Config compliance event details"""
        
        # This would query Config for the specific non-compliance event
        return {
            'resource_type': 'AWS::IAM::Role',
            'resource_id': role_name,
            'compliance_status': 'NON_COMPLIANT',
            'rule_name': 'iam-policy-restricted-attached'
        }

# Usage
investigator = SecurityIncidentInvestigator()

# Investigate incident
investigation = investigator.investigate_unauthorized_policy_attachment(
    role_name='MyApplicationRole',
    compliance_change_time=datetime.now() - timedelta(hours=2)
)

print(json.dumps(investigation, indent=2))
```

---

## References

[1] AWS CloudTrail User Guide - AWS Documentation. https://docs.aws.amazon.com/awscloudtrail/latest/userguide/

[2] AWS Config User Guide - AWS Documentation. https://docs.aws.amazon.com/config/latest/userguide/

[3] Logging AWS Audit Manager API calls with CloudTrail - AWS Audit Manager. https://docs.aws.amazon.com/audit-manager/latest/userguide/logging-using-cloudtrail.html

[4] AWS Config vs CloudTrail: Reporting, Configuration & Pricing - Cloudwards. https://www.cloudwards.net/aws-config-vs-cloudtrail/

[5] Using AWS CloudTrail Data Events to Audit Your Amazon SNS and Amazon SQS Workloads - AWS Blog. https://aws.amazon.com/blogs/mt/using-aws-cloudtrail-data-events-to-audit-your-amazon-sns-and-amazon-sqs-workloads/

[6] Best Practices for Using AWS CloudTrail in 2024 - KirkpatrickPrice. https://kirkpatrickprice.com/blog/cloud-security-blog/aws-cloudtrail-logs/

[7] Manage Continuous Compliance by Using AWS Config Configuration Recorder - AWS Blog. https://aws.amazon.com/blogs/mt/manage-continuous-compliance-by-using-aws-config-configuration-recorder-resource-type/

[8] AWS Config Features - AWS Documentation. https://aws.amazon.com/config/features/

[9] CloudTrail Logging Evasion: Where Policy Size Matters - Permiso. https://permiso.io/blog/cloudtrail-logging-evasion-where-policy-size-matters

[10] AWS Config Overview - KodeKloud Notes. https://notes.kodekloud.com/docs/AWS-Certified-SysOps-Administrator-Associate/Domain-4-Security-and-Compliance/AWS-Config-Overview
