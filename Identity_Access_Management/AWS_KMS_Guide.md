# AWS KMS (Key Management Service) — Encryption Key Lifecycle Management

## Detailed Explanation

### What is AWS KMS?

AWS Key Management Service (KMS) is a managed cryptographic service that enables you to create, control, and use encryption keys to protect data across AWS services and your applications[1]. It provides centralized key management, compliance support, and integration with AWS services for encrypting data at rest and in transit.

**Key Characteristics:**
- **Managed Service**: AWS handles the HSM (Hardware Security Module) infrastructure
- **FIPS 140-2 Compliance**: Level 2 and Level 3 certified
- **CloudTrail Logging**: All KMS operations are logged for audit trails
- **Regional Service**: Keys exist in specific regions and cannot be replicated
- **Envelope Encryption**: Uses hierarchical encryption model for efficiency

### AWS KMS Key Types

| Key Type | AWS Managed | Customer Managed | AWS Owned |
|----------|:----------:|:----------------:|:---------:|
| Creation | AWS creates | User creates | AWS services create |
| Cost | Free | $1/month + API calls | Free |
| Key Material Source | AWS KMS generated | User choice (generated or imported) | Service generated |
| Key Rotation | Automatic (1 year) | Optional automatic or manual | Service managed |
| Deletion | Not allowed | User controlled (with 7-30 day waiting period) | Cannot delete |
| Use Case | Default encryption | Sensitive workloads, compliance | CloudWatch, S3 (default) |
| CMK vs KMS Key | Called CMK | Now called KMS key | N/A |

### Key Lifecycle Stages

\begin{enumerate}
\item **Creation**: Generate new KMS key with aliases and tags
\item **Activation**: Key becomes available for encryption/decryption operations
\item **Rotation**: New cryptographic material generated (automatic or manual)
\item **Deactivation**: Temporarily disable key for compliance purposes
\item **Scheduled Deletion**: 7 to 30-day waiting period before actual deletion
\item **Deletion**: Key and all material permanently destroyed
\end{enumerate}

### Envelope Encryption Model

Envelope encryption is the foundation of AWS KMS security:

**Process:**
1. **Data Key Generation**: KMS generates a unique data key for each object
2. **Data Encryption**: Application encrypts data using plaintext data key
3. **Key Encryption**: Application requests KMS to encrypt the data key
4. **Storage**: Stores encrypted data key alongside encrypted data
5. **Decryption**: KMS decrypts the encrypted data key, application uses plaintext key to decrypt data

**Benefits:**
- Efficient large-scale encryption without loading KMS with data encryption
- Reduces KMS API call overhead for massive datasets
- Historical decryption capability with key versions

### Permission Model

AWS KMS uses a hybrid permission model:

\begin{itemize}
\item **IAM Policies**: Control who can call KMS API operations
\item **Key Policies**: Control who can use specific KMS keys (JSON-based resource policies)
\item **Grants**: Provide temporary permissions for specific operations on keys
\item **Encryption Context**: Optional plaintext key-value pairs for additional audit trail
\end{itemize}

**Evaluation Logic**: All three layers must allow the action. If any layer denies, the overall decision is deny.

### Key Rotation Strategy

**Automatic Key Rotation:**
- **AWS Managed Keys**: Rotated automatically every year (365 days)
- **Customer Managed Keys**: Optional, must be explicitly enabled
- **Process**: AWS generates new cryptographic material while retaining old material
- **No Re-encryption**: Previous data remains accessible via old key material
- **Default Rotation Period**: 365 days (customizable via API)

**Manual Key Rotation:**
- Create new customer managed key
- Update application code or service to use new key ID/alias
- Maintain old key in active state for historical data decryption
- Common pattern: Use aliases to abstract key rotation from applications

---

## Detailed Examples

### Example 1: Creating and Configuring a Customer Managed Key

**Scenario**: Financial services company needs to encrypt transaction data in S3 with full control over key lifecycle.

**AWS Console Configuration:**

```
1. Navigate to AWS KMS Console
2. Select "Create Key"
3. Choose Key Type: Symmetric (for S3 encryption)
4. Add Key Alias: "financial-transactions-key"
5. Add Description: "Encrypts sensitive transaction records"
6. Add Tags:
   - Environment: Production
   - Application: FinanceApp
   - ComplianceLevel: PCI-DSS
7. Define Key Policy:
   - Root account full permissions
   - Finance team can use for encrypt/decrypt
   - DBA can manage key rotation
8. Enable Automatic Key Rotation: YES (annual)
9. Review and Create
```

**AWS CLI Configuration:**

```bash
# Create customer managed key
aws kms create-key \
  --description "Financial transaction encryption key" \
  --key-usage ENCRYPT_DECRYPT \
  --origin AWS_KMS \
  --region us-east-1

# Output: KeyId = arn:aws:kms:us-east-1:123456789012:key/12345678-1234-1234-1234-123456789012

# Create alias for easier reference
aws kms create-alias \
  --alias-name alias/financial-transactions-key \
  --target-key-id 12345678-1234-1234-1234-123456789012

# Enable automatic key rotation
aws kms enable-key-rotation \
  --key-id 12345678-1234-1234-1234-123456789012

# Verify key policy
aws kms get-key-policy \
  --key-id alias/financial-transactions-key \
  --policy-name default
```

**Key Policy (JSON):**

```json
{
  "Sid": "Enable IAM policies",
  "Effect": "Allow",
  "Principal": {
    "AWS": "arn:aws:iam::123456789012:root"
  },
  "Action": "kms:*",
  "Resource": "*"
},
{
  "Sid": "Allow Finance Team to use key",
  "Effect": "Allow",
  "Principal": {
    "AWS": "arn:aws:iam::123456789012:role/FinanceTeamRole"
  },
  "Action": [
    "kms:Decrypt",
    "kms:Encrypt",
    "kms:GenerateDataKey",
    "kms:DescribeKey"
  ],
  "Resource": "*"
},
{
  "Sid": "Allow DBA to manage rotation",
  "Effect": "Allow",
  "Principal": {
    "AWS": "arn:aws:iam::123456789012:role/DBATeamRole"
  },
  "Action": [
    "kms:EnableKeyRotation",
    "kms:DisableKeyRotation",
    "kms:GetKeyRotationStatus"
  ],
  "Resource": "*"
}
```

---

### Example 2: Envelope Encryption with Application Integration

**Scenario**: E-commerce application encrypting customer payment data with KMS.

**Application Code (Python with boto3):**

```python
import boto3
import base64
from datetime import datetime

# Initialize KMS client
kms_client = boto3.client('kms', region_name='us-east-1')

class PaymentDataEncryptor:
    def __init__(self, key_id):
        self.key_id = key_id
        self.kms_client = kms_client
    
    # Step 1: Generate Data Key
    def generate_data_key(self):
        response = self.kms_client.generate_data_key(
            KeyId=self.key_id,
            KeySpec='AES_256',
            EncryptionContext={
                'purpose': 'payment_encryption',
                'service': 'ecommerce-app',
                'timestamp': datetime.now().isoformat()
            }
        )
        
        plaintext_key = response['Plaintext']
        encrypted_key = response['CiphertextBlob']
        
        return plaintext_key, encrypted_key
    
    # Step 2: Encrypt Payment Data
    def encrypt_payment_data(self, payment_info):
        # Generate data key for this transaction
        plaintext_key, encrypted_key = self.generate_data_key()
        
        # Use plaintext key to encrypt payment data
        from cryptography.fernet import Fernet
        cipher = Fernet(base64.urlsafe_b64encode(plaintext_key[:32]))
        
        encrypted_data = cipher.encrypt(
            payment_info.encode('utf-8')
        )
        
        # Store encrypted key with encrypted data
        return {
            'encrypted_data': base64.b64encode(encrypted_data).decode(),
            'encrypted_key': base64.b64encode(encrypted_key).decode(),
            'timestamp': datetime.now().isoformat()
        }
    
    # Step 3: Decrypt Payment Data
    def decrypt_payment_data(self, encrypted_payload):
        # Decrypt the data key using KMS
        encrypted_key = base64.b64decode(
            encrypted_payload['encrypted_key']
        )
        
        response = self.kms_client.decrypt(
            CiphertextBlob=encrypted_key,
            EncryptionContext={
                'purpose': 'payment_encryption',
                'service': 'ecommerce-app'
            }
        )
        
        plaintext_key = response['Plaintext']
        
        # Use plaintext key to decrypt data
        from cryptography.fernet import Fernet
        cipher = Fernet(base64.urlsafe_b64encode(plaintext_key[:32]))
        
        encrypted_data = base64.b64decode(
            encrypted_payload['encrypted_data']
        )
        
        decrypted_data = cipher.decrypt(encrypted_data)
        
        return decrypted_data.decode('utf-8')

# Usage
encryptor = PaymentDataEncryptor(
    key_id='arn:aws:kms:us-east-1:123456789012:key/12345678-1234-1234-1234-123456789012'
)

# Encrypt
payment_info = '{"card": "4532-XXXX-XXXX-1234", "amount": 99.99}'
encrypted = encryptor.encrypt_payment_data(payment_info)

# Store encrypted payload in database
# Later: Decrypt
decrypted = encryptor.decrypt_payment_data(encrypted)
print(f"Decrypted payment info: {decrypted}")
```

---

### Example 3: Manual Key Rotation Strategy

**Scenario**: Healthcare provider must rotate encryption keys quarterly for HIPAA compliance.

**Manual Rotation Process:**

```bash
# Step 1: Create new KMS key
NEW_KEY=$(aws kms create-key \
  --description "Medical records encryption - Q1 2024 rotation" \
  --tags TagKey=rotation-quarter,TagValue=Q1-2024 \
  --query 'KeyMetadata.KeyId' \
  --output text)

echo "New Key ID: $NEW_KEY"

# Step 2: Create alias for new key (use timestamp pattern)
aws kms create-alias \
  --alias-name alias/medical-records-key-2024-q1 \
  --target-key-id $NEW_KEY

# Step 3: Copy key policy from old key to new key
OLD_KEY="arn:aws:kms:us-east-1:123456789012:key/old-key-id"

# Get old key policy
OLD_POLICY=$(aws kms get-key-policy \
  --key-id $OLD_KEY \
  --policy-name default \
  --query 'Policy' \
  --output text)

# Apply to new key
aws kms put-key-policy \
  --key-id $NEW_KEY \
  --policy-name default \
  --policy "$OLD_POLICY"

# Step 4: Update application configuration
# Update environment variable or config file
echo "MEDICAL_ENCRYPTION_KEY_ID=$NEW_KEY" > /etc/app/kms-config.env

# Step 5: Restart application (zero-downtime with rolling deployment)
# Application will use new key for all new encryptions
# Old key remains active for decrypting existing records

# Step 6: Monitor CloudWatch metrics
aws cloudwatch get-metric-statistics \
  --namespace AWS/KMS \
  --metric-name UserErrorCount \
  --dimensions Name=KeyId,Value=$NEW_KEY \
  --start-time 2024-01-01T00:00:00Z \
  --end-time 2024-01-02T00:00:00Z \
  --period 3600 \
  --statistics Sum

# Step 7: Schedule old key deletion (disable first, then schedule deletion)
aws kms disable-key \
  --key-id $OLD_KEY

aws kms schedule-key-deletion \
  --key-id $OLD_KEY \
  --pending-window-in-days 30

# Verify deletion schedule
aws kms describe-key --key-id $OLD_KEY
```

---

### Example 4: Key Policy and IAM Integration

**Scenario**: Multi-team organization with separate Finance and Engineering teams accessing same KMS key for different purposes.

**Comprehensive Key Policy:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "Enable IAM Root Access",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:root"
      },
      "Action": "kms:*",
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "aws:PrincipalOrgID": "o-exampleorgid"
        }
      }
    },
    {
      "Sid": "Finance Team - Encrypt/Decrypt",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:role/FinanceTeamRole"
      },
      "Action": [
        "kms:Decrypt",
        "kms:Encrypt",
        "kms:GenerateDataKey",
        "kms:DescribeKey"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "kms:EncryptionContext:Department": "Finance"
        }
      }
    },
    {
      "Sid": "Engineering Team - Decrypt Only",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:role/EngineeringTeamRole"
      },
      "Action": [
        "kms:Decrypt",
        "kms:DescribeKey"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "kms:ViaService": "s3.us-east-1.amazonaws.com"
        }
      }
    },
    {
      "Sid": "DBA Team - Key Administration",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:role/DBATeamRole"
      },
      "Action": [
        "kms:EnableKeyRotation",
        "kms:DisableKeyRotation",
        "kms:GetKeyRotationStatus",
        "kms:PutKeyPolicy",
        "kms:GetKeyPolicy",
        "kms:CreateGrant",
        "kms:RetireGrant"
      ],
      "Resource": "*"
    },
    {
      "Sid": "Deny Unencrypted Operations",
      "Effect": "Deny",
      "Principal": "*",
      "Action": [
        "kms:Decrypt",
        "kms:Encrypt"
      ],
      "Resource": "*",
      "Condition": {
        "Bool": {
          "aws:SecureTransport": "false"
        }
      }
    }
  ]
}
```

**Corresponding IAM Role Trust Policy (Finance Team):**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

**IAM Inline Policy (Finance Team):**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "EncryptFinancialData",
      "Effect": "Allow",
      "Action": [
        "kms:Encrypt",
        "kms:Decrypt",
        "kms:GenerateDataKey",
        "kms:DescribeKey"
      ],
      "Resource": "arn:aws:kms:us-east-1:123456789012:key/financial-key-id",
      "Condition": {
        "StringEquals": {
          "kms:EncryptionContext:Department": "Finance"
        }
      }
    }
  ]
}
```

---

### Example 5: KMS Integration with AWS Services

**Scenario**: Complete encrypted data pipeline using S3, RDS, and EBS with KMS.

**S3 Server-Side Encryption Configuration:**

```bash
# Enable default encryption on S3 bucket
aws s3api put-bucket-encryption \
  --bucket my-secure-bucket \
  --server-side-encryption-configuration '{
    "Rules": [
      {
        "ApplyServerSideEncryptionByDefault": {
          "SSEAlgorithm": "aws:kms",
          "KMSMasterKeyID": "arn:aws:kms:us-east-1:123456789012:key/12345678-1234"
        },
        "BucketKeyEnabled": true
      }
    ]
  }'

# Upload object with KMS encryption
aws s3 cp sensitive-data.txt s3://my-secure-bucket/ \
  --sse aws:kms \
  --sse-kms-key-id arn:aws:kms:us-east-1:123456789012:key/12345678-1234
```

**RDS Database Encryption:**

```bash
# Create encrypted RDS instance
aws rds create-db-instance \
  --db-instance-identifier secure-prod-db \
  --db-instance-class db.t3.large \
  --engine mysql \
  --master-username admin \
  --master-user-password MySecurePassword123! \
  --allocated-storage 100 \
  --storage-type gp3 \
  --storage-encrypted \
  --kms-key-id arn:aws:kms:us-east-1:123456789012:key/12345678-1234 \
  --backup-retention-period 30 \
  --enable-cloudwatch-logs-exports error,general,slowquery
```

**EBS Volume Encryption:**

```bash
# Create encrypted EBS volume
aws ec2 create-volume \
  --size 100 \
  --volume-type gp3 \
  --availability-zone us-east-1a \
  --encrypted \
  --kms-key-id arn:aws:kms:us-east-1:123456789012:key/12345678-1234 \
  --tag-specifications 'ResourceType=volume,Tags=[{Key=Name,Value=EncryptedDataVolume}]'

# Enable EBS encryption by default
aws ec2 enable-ebs-encryption-by-default \
  --region us-east-1
```

**Application Layer (Node.js with encryption context):**

```javascript
const AWS = require('aws-sdk');
const kms = new AWS.KMS({ region: 'us-east-1' });

async function encryptSensitiveData(plaintext, context) {
  try {
    const params = {
      KeyId: 'alias/my-encryption-key',
      Plaintext: plaintext,
      EncryptionContext: context
    };
    
    const result = await kms.encrypt(params).promise();
    return result.CiphertextBlob.toString('base64');
    
  } catch (error) {
    console.error('Encryption failed:', error);
    throw error;
  }
}

async function decryptData(ciphertext, context) {
  try {
    const params = {
      CiphertextBlob: Buffer.from(ciphertext, 'base64'),
      EncryptionContext: context
    };
    
    const result = await kms.decrypt(params).promise();
    return result.Plaintext.toString('utf-8');
    
  } catch (error) {
    console.error('Decryption failed:', error);
    throw error;
  }
}

// Usage
const encryptionContext = {
  userId: '12345',
  dataType: 'customer_payment',
  environment: 'production'
};

const encrypted = await encryptSensitiveData('{"card": "4532"}', encryptionContext);
const decrypted = await decryptData(encrypted, encryptionContext);
```

---

## Top 5 Interview Questions

### Question 1: Explain AWS KMS Key Lifecycle and Rotation Strategy Design

**Scenario**: "Design a comprehensive key lifecycle management strategy for a healthcare organization that processes sensitive patient data. The organization has multiple AWS accounts across regions and requires compliance with HIPAA and SOC 2 standards."

**Expected Answer Structure**:

**Key Lifecycle Stages**:

\begin{enumerate}
\item **Creation Phase**
   - Create customer-managed KMS keys in each region where data resides
   - Implement centralized key policy management in management account
   - Use CloudFormation or Terraform for infrastructure-as-code approach
   - Tag keys appropriately: Environment, DataClassification, ComplianceRequirement

\item **Activation Phase**
   - Configure key policy to restrict access to authorized IAM roles
   - Enable CloudTrail logging for all key operations
   - Set up CloudWatch alarms for suspicious key usage patterns
   - Create aliases for easier application integration

\item **Rotation Phase**
   - **For automatic rotation**: Enable for all customer-managed keys with 365-day period
   - **For compliance audits**: Implement quarterly manual rotation alongside automatic
   - Process: Create new key → Update alias → Validate → Schedule old key deletion
   - AWS KMS retains all key material versions automatically for backward decryption

\item **Deactivation Phase**
   - When key reaches end-of-life, first disable it for 30 days
   - Monitor CloudWatch metrics to ensure no active decryption operations
   - Verify all data encrypted with this key has been re-encrypted if necessary

\item **Deletion Phase**
   - Schedule key deletion with 7-30 day waiting period
   - Receive AWS notifications before actual deletion
   - Key material is securely destroyed after waiting period expires
\end{enumerate}

**Multi-Account Architecture**:

```
Headquarters Account (Management)
├── Master KMS Key Policy (read-only)
├── CloudTrail centralized logging
└── SNS notifications

Regional Accounts (Dev, Staging, Prod)
├── Production Account
│   ├── Customer-managed KMS key for patient data
│   ├── Automatic annual rotation enabled
│   └── Cross-account access from audit account
├── Staging Account
│   ├── Separate KMS key for non-sensitive testing
│   └── Manual rotation for compliance testing
└── Development Account
    ├── AWS-managed key (cost-effective)
    └── No compliance requirements
```

**Rotation Strategy Implementation**:

| Rotation Type | Schedule | Use Case | Implementation |
|---------------|----------|----------|-----------------|
| **Automatic** | Annual (365 days) | Regular compliance | Enable key rotation setting |
| **Quarterly Manual** | Every 90 days | Audit requirements | Create new key, update alias |
| **Emergency Rotation** | On-demand | Security incident | Immediate key replacement, encrypted re-keying |

**Encryption Context for Audit Trail**:

```json
{
  "Department": "Healthcare",
  "DataType": "PatientRecords",
  "ComplianceLevel": "HIPAA",
  "Environment": "Production",
  "CreatedDate": "2024-01-15"
}
```

**Monitoring and Alerts**:

- CloudWatch metric: `UserErrorCount` for failed decryption attempts
- Lambda function: Triggered on key deletion schedule to notify compliance team
- EventBridge rule: Alert on unauthorized key usage patterns
- Monthly CloudTrail report: Audit all KMS operations

**Cross-Account Access Pattern**:

Account A (owns key) grants permissions to Account B (uses key) through key policy and STS AssumeRole mechanism.

---

### Question 2: Troubleshoot KMS Permission Denied Errors and Permission Model

**Scenario**: "An application running on EC2 instance is receiving 'AccessDenied' error when trying to decrypt S3 objects. The same application works fine in another AWS account. Walk through how you would diagnose and resolve this issue."

**Answer Structure**:

**Step 1: Verify Three-Layer Permission Model**:

\begin{itemize}
\item **Layer 1 - IAM Policy**: Does the EC2 instance role have permission to call kms:Decrypt?
\item **Layer 2 - Key Policy**: Does the KMS key policy allow the EC2 instance role to decrypt?
\item **Layer 3 - Encryption Context**: Does the operation include correct encryption context if required?
\end{itemize}

**Diagnostic Checklist**:

```bash
# 1. Get EC2 instance IAM role
aws ec2 describe-instances --instance-ids i-1234567890abcdef0 \
  --query 'Reservations[0].Instances[0].IamInstanceProfile'

# 2. Get IAM role policies
ROLE_NAME="ec2-app-role"
aws iam list-attached-role-policies --role-name $ROLE_NAME
aws iam list-role-policies --role-name $ROLE_NAME

# 3. Get inline policies
aws iam get-role-policy --role-name $ROLE_NAME --policy-name KmsPolicy

# 4. Get KMS key details
KEY_ID="arn:aws:kms:us-east-1:123456789012:key/12345678"
aws kms describe-key --key-id $KEY_ID

# 5. Get KMS key policy
aws kms get-key-policy --key-id $KEY_ID --policy-name default

# 6. Check if key is enabled
aws kms get-key-rotation-status --key-id $KEY_ID
```

**Common Permission Denied Scenarios and Solutions**:

| Scenario | Root Cause | Solution |
|----------|-----------|----------|
| EC2 role missing kms:Decrypt permission | IAM policy doesn't grant action | Add kms:Decrypt to role's inline/managed policy |
| Key policy doesn't trust EC2 role's account | Cross-account access not allowed | Add EC2 role ARN to key policy Principal |
| Encryption context mismatch | App encrypts with context but decrypts without | Ensure encryption context matches exactly |
| S3 bucket key disabled but old object uses data key | Encryption method mismatch | Re-encrypt objects with consistent method |
| Deny statement in key policy overrides Allow | Explicit Deny takes precedence | Remove Deny condition for trusted principals |

**Detailed Diagnostic Example**:

**Error**: Application receives "User: arn:aws:iam::123456789012:role/ec2-app-role is not authorized to perform: kms:Decrypt"

**Investigation**:

```bash
# 1. Check IAM policy attached to role
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject"  # ← Missing kms:Decrypt!
      ],
      "Resource": "arn:aws:s3:::my-bucket/*"
    }
  ]
}

# 2. Check KMS key policy
{
  "Sid": "Allow S3 service role",
  "Effect": "Allow",
  "Principal": {
    "AWS": "arn:aws:iam::123456789012:role/aws-s3-service-role"
  },
  "Action": "kms:Decrypt"
  # ← EC2 role not listed!
}

# RESOLUTION: Update IAM policy
{
  "Effect": "Allow",
  "Action": [
    "s3:GetObject",
    "kms:Decrypt",           # ← ADD THIS
    "kms:DescribeKey"        # ← ADD THIS
  ],
  "Resource": [
    "arn:aws:s3:::my-bucket/*",
    "arn:aws:kms:us-east-1:123456789012:key/12345678"
  ]
}

# Update KMS key policy
{
  "Sid": "Allow EC2 app role to decrypt",
  "Effect": "Allow",
  "Principal": {
    "AWS": "arn:aws:iam::123456789012:role/ec2-app-role"  # ← ADD THIS
  },
  "Action": [
    "kms:Decrypt",
    "kms:DescribeKey"
  ],
  "Resource": "*"
}
```

**Validation After Fix**:

```bash
# Test with AWS CLI assuming the EC2 role
aws sts assume-role \
  --role-arn arn:aws:iam::123456789012:role/ec2-app-role \
  --role-session-name test-session

# Use temporary credentials to test KMS operation
export AWS_ACCESS_KEY_ID=...
export AWS_SECRET_ACCESS_KEY=...
export AWS_SESSION_TOKEN=...

aws kms decrypt \
  --key-id arn:aws:kms:us-east-1:123456789012:key/12345678 \
  --ciphertext-blob fileb://encrypted-data
```

---

### Question 3: Design KMS Strategy for Multi-Region, Multi-Account Environment with Compliance Requirements

**Scenario**: "Your organization is expanding globally with AWS infrastructure across us-east-1, eu-west-1, and ap-southeast-1 regions. You must ensure compliance with regional data residency laws (GDPR for EU, etc.). How would you design a KMS strategy that balances security, compliance, and operational efficiency?"

**Answer Structure**:

**Architecture Design**:

```
Global Organization
├── Management Account (Billing/Centralized Logging)
│   ├── CloudTrail - Central logging for all KMS operations
│   ├── EventBridge - Rules for security monitoring
│   └── SNS - Notifications across regions
│
├── US Region (us-east-1)
│   ├── Production Account
│   │   ├── KMS Key 1 (Financial data - customer-managed)
│   │   ├── KMS Key 2 (Application secrets - AWS-managed)
│   │   └── RDS/S3/EBS - All encrypted with Key 1
│   │
│   ├── Development Account
│   │   └── KMS Key (Development - AWS-managed)
│   │
│   └── Compliance/Audit Account
│       └── Read-only access to all keys
│
├── EU Region (eu-west-1) - GDPR Compliance
│   ├── Production Account
│   │   ├── KMS Key (EU-only, no cross-region replication)
│   │   ├── Data residency: All data must stay in EU
│   │   └── Quarterly key rotation mandated
│   │
│   └── Backup Account
│       └── Secondary key for cross-region replication
│
└── APAC Region (ap-southeast-1) - Data Residency Required
    ├── Production Account
    │   ├── KMS Key (APAC-only)
    │   └── No data replication to other regions
    │
    └── Development Account
        └── Test key
```

**Key Strategy Matrix**:

| Region | Compliance | Key Rotation | Multi-Account Access | Data Residency |
|--------|-----------|--------------|----------------------|-----------------|
| **US** | SOC 2, PCI-DSS | Annual auto | Cross-account allowed | US regions only |
| **EU** | GDPR | Quarterly manual | EU accounts only | EU regions only |
| **APAC** | Data Protection Act | Semi-annual | APAC accounts only | APAC regions only |

**Implementation Details**:

**1. Regional KMS Key Policies with Strict Access Controls**:

```json
{
  "Sid": "GDPR - EU Data Residency Enforcement",
  "Effect": "Deny",
  "Principal": "*",
  "Action": [
    "kms:Decrypt",
    "kms:Encrypt"
  ],
  "Resource": "*",
  "Condition": {
    "StringNotEquals": {
      "aws:RequestedRegion": "eu-west-1"
    }
  }
}
```

**2. Automated Key Rotation by Region**:

```python
import boto3
from datetime import datetime

class MultiRegionKeyRotation:
    def __init__(self):
        self.rotation_config = {
            'us-east-1': {'days': 365, 'level': 'SOC2'},
            'eu-west-1': {'days': 90, 'level': 'GDPR'},
            'ap-southeast-1': {'days': 180, 'level': 'DataResidency'}
        }
    
    def rotate_keys(self):
        for region, config in self.rotation_config.items():
            kms_client = boto3.client('kms', region_name=region)
            
            # List all customer-managed keys in region
            keys = kms_client.list_keys()
            
            for key in keys['Keys']:
                key_id = key['KeyId']
                status = kms_client.get_key_rotation_status(
                    KeyId=key_id
                )
                
                # Enable rotation if not already enabled
                if not status['KeyRotationEnabled']:
                    kms_client.enable_key_rotation(KeyId=key_id)
                    
                    # Log rotation enablement
                    print(f"Enabled rotation for {key_id} in {region}")
                    
                    # Send notification
                    self._send_notification(
                        region=region,
                        key_id=key_id,
                        event='rotation_enabled'
                    )
    
    def _send_notification(self, region, key_id, event):
        sns = boto3.client('sns')
        sns.publish(
            TopicArn='arn:aws:sns:us-east-1:123456789012:kms-alerts',
            Subject=f'KMS Key Event: {event} in {region}',
            Message=f'Key {key_id} - {event} at {datetime.now()}'
        )
```

**3. Cross-Account Access Pattern (Same Region Only)**:

```json
{
  "Sid": "Allow EU Production Account to decrypt",
  "Effect": "Allow",
  "Principal": {
    "AWS": "arn:aws:iam::111111111111:role/eu-prod-app-role"
  },
  "Action": [
    "kms:Decrypt",
    "kms:GenerateDataKey",
    "kms:DescribeKey"
  ],
  "Resource": "*",
  "Condition": {
    "StringEquals": {
      "aws:RequestedRegion": "eu-west-1",
      "aws:SourceAccount": "111111111111"
    }
  }
}
```

**4. Compliance Monitoring and Audit**:

```bash
# Create EventBridge rule for GDPR compliance monitoring
aws events put-rule \
  --name kms-gdpr-compliance-monitor \
  --event-pattern '{
    "source": ["aws.kms"],
    "detail-type": ["AWS API Call via CloudTrail"],
    "detail": {
      "eventName": ["ScheduleKeyDeletion", "DisableKey"],
      "requestParameters": {
        "region": ["eu-west-1"]
      }
    }
  }' \
  --state ENABLED

# Alert on suspicious activities
aws events put-targets \
  --rule kms-gdpr-compliance-monitor \
  --targets "Id"="1","Arn"="arn:aws:sns:eu-west-1:123456789012:kms-alerts"
```

---

### Question 4: KMS Best Practices and Security Hardening

**Scenario**: "Your security team has flagged concerns about KMS key management. They want you to audit current practices and recommend hardening measures. What are the critical security best practices?"

**Answer Structure**:

**KMS Security Best Practices**:

\begin{enumerate}
\item **Key Isolation**
   - One key per application/service per environment
   - Separate keys for different data classifications
   - Avoid "golden key" shared across multiple services

\item **Access Control (Defense in Depth)**
   - Combine IAM policies + KMS key policies + grants
   - Use encryption context for additional audit trail
   - Implement least privilege principle
   - Use condition keys for fine-grained control (VPC endpoints, IP ranges, time-based)

\item **Key Rotation**
   - Enable automatic rotation for all customer-managed keys
   - Implement quarterly manual rotation for compliance
   - Monitor rotation dates proactively
   - Maintain all key material versions for backward compatibility

\item **Monitoring and Logging**
   - Enable CloudTrail for all KMS operations
   - Set up CloudWatch alarms for unauthorized access attempts
   - Monitor metrics: UserErrorCount, ThrottledCount
   - Review CloudTrail logs weekly for suspicious patterns

\item **Encryption Context**
   - Always include encryption context for sensitive operations
   - Verify context during decryption
   - Never include secrets in encryption context
   - Use consistent context naming across organization

\item **Key Deletion Protection**
   - Use 30-day waiting period before actual deletion
   - Set up SNS notifications for pending deletions
   - Implement SCPs to prevent accidental key deletion
   - Require multi-person approval for key deletion

\item **Compliance Integration**
   - Enable key-level tagging for compliance tracking
   - Use AWS Config for KMS key compliance monitoring
   - Regular audits with AWS CloudTrail Insights
   - Document key lifecycle for compliance reports

\item **Disaster Recovery**
   - Maintain key replicas in secondary region (manual pattern)
   - Test decryption with backup keys periodically
   - Document key recovery procedures
   - Implement automated backup verification
\end{enumerate}

**Security Hardening Checklist**:

```bash
#!/bin/bash
# KMS Security Audit Script

echo "=== KMS Security Audit ==="

# 1. Check for AWS-managed keys (should be minimized)
echo "1. Listing customer-managed keys..."
aws kms list-keys --query 'Keys[*].KeyId' | while read key; do
  key_status=$(aws kms describe-key --key-id "$key" --query 'KeyMetadata.KeyManager')
  if [ "$key_status" != "CUSTOMER" ]; then
    echo "WARNING: $key is AWS-managed"
  fi
done

# 2. Verify key rotation is enabled
echo "2. Checking key rotation status..."
aws kms list-keys --query 'Keys[*].KeyId' | while read key; do
  rotation=$(aws kms get-key-rotation-status --key-id "$key" --query 'KeyRotationEnabled')
  if [ "$rotation" != "true" ]; then
    echo "CRITICAL: $key does not have rotation enabled"
  fi
done

# 3. Verify encryption context usage
echo "3. Checking CloudTrail for encryption context..."
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=ResourceName,AttributeValue=kms-key \
  --max-results 100 | grep -i "encryptioncontext"

# 4. Check for overly permissive key policies
echo "4. Auditing key policies..."
aws kms list-keys --query 'Keys[*].KeyId' | while read key; do
  policy=$(aws kms get-key-policy --key-id "$key" --policy-name default)
  if echo "$policy" | grep -q '"Principal": "\*"'; then
    echo "WARNING: $key has wildcard principal"
  fi
done

# 5. Check for keys pending deletion
echo "5. Checking for pending key deletions..."
aws kms list-keys --query 'Keys[*].KeyId' | while read key; do
  deletion_date=$(aws kms describe-key --key-id "$key" --query 'KeyMetadata.DeletionDate')
  if [ "$deletion_date" != "null" ]; then
    echo "KEY ALERT: $key scheduled for deletion on $deletion_date"
  fi
done

# 6. Verify VPC endpoint for KMS (if applicable)
echo "6. Checking VPC endpoints..."
aws ec2 describe-vpc-endpoints --query 'VpcEndpoints[?ServiceName==`com.amazonaws.us-east-1.kms`]'
```

**Security Hardening Implementation**:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "RootAccountAccess",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:root"
      },
      "Action": "kms:*",
      "Resource": "*"
    },
    {
      "Sid": "DenyInsecureTransport",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "kms:*",
      "Resource": "*",
      "Condition": {
        "Bool": {
          "aws:SecureTransport": "false"
        }
      }
    },
    {
      "Sid": "DenyUnencryptedContext",
      "Effect": "Deny",
      "Principal": "*",
      "Action": [
        "kms:Encrypt",
        "kms:GenerateDataKey"
      ],
      "Resource": "*",
      "Condition": {
        "Null": {
          "kms:EncryptionContext": "true"
        }
      }
    },
    {
      "Sid": "AllowOnlyVPCEndpoint",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "kms:*",
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "aws:SourceVpce": "vpce-12345678"
        }
      }
    },
    {
      "Sid": "DenyKeyDeletion",
      "Effect": "Deny",
      "Principal": "*",
      "Action": [
        "kms:ScheduleKeyDeletion",
        "kms:DisableKey"
      ],
      "Resource": "*"
    }
  ]
}
```

---

### Question 5: Real-World KMS Integration Troubleshooting - Performance and Cost Optimization

**Scenario**: "A high-traffic application encrypting millions of objects daily in S3 is experiencing throttling errors from KMS and facing unexpectedly high costs. The engineering team is considering caching decrypted data, but security wants to prevent this. How would you optimize KMS usage for performance and cost?"

**Answer Structure**:

**Problem Analysis**:

| Issue | Root Cause | Impact |
|-------|-----------|--------|
| **KMS Throttling** | Exceeding API rate limits (10,000 requests/second per key) | Failed encryptions, application errors |
| **High Costs** | $0.03 per 10,000 requests + per-key-month fee | Budget overruns |
| **Security Concerns** | Caching decrypted data violates compliance | Increased breach risk |

**Solution: S3 Bucket Key Feature**

S3 Bucket Key reduces KMS API calls and costs by ~99% while maintaining security:

```bash
# Enable S3 Bucket Key (default encryption)
aws s3api put-bucket-encryption \
  --bucket my-high-traffic-bucket \
  --server-side-encryption-configuration '{
    "Rules": [
      {
        "ApplyServerSideEncryptionByDefault": {
          "SSEAlgorithm": "aws:kms",
          "KMSMasterKeyID": "arn:aws:kms:us-east-1:123456789012:key/key-id"
        },
        "BucketKeyEnabled": true
      }
    ]
  }'

# For existing objects, enable with PutObject
aws s3 cp large-dataset.txt s3://my-high-traffic-bucket/ \
  --sse aws:kms \
  --sse-kms-key-id arn:aws:kms:us-east-1:123456789012:key/key-id \
  --metadata "bucket-key-enabled=true"
```

**Cost Optimization Math**:

```
WITHOUT S3 Bucket Key:
- 1M objects/day × 2 operations (encrypt + decrypt) = 2M KMS API calls
- 2M ÷ 10,000 = 200 API request units
- Cost: 200 × $0.03 = $6/day = $180/month
- Plus: $1/month per key = $181/month total

WITH S3 Bucket Key:
- S3 derives data keys from bucket key (no KMS API call per object)
- ~100 bucket key rotations/month = 100 API calls
- Cost: 0.01 × $0.03 = $0.0003/month
- Plus: $1/month per key = $1.0003/month total

SAVINGS: ~99.4% reduction ($180 → ~$1/month)
```

**Performance Optimization Architecture**:

```
Application Load Balancer
    ↓
Auto Scaling Group (EC2 instances)
    ↓
EC2 instances with IAM role (kms:Decrypt permission)
    ↓
Encrypted S3 bucket (with Bucket Key enabled)
    ↓
Single KMS Key (Regional)
    ├── Automatic annual rotation
    └── CloudWatch monitoring

Benefits:
- 1 KMS operation per bucket key derivation (~daily)
- Instead of 1 KMS operation per object (~1M/day)
- 99%+ reduction in KMS API calls
- <1ms latency for data key derivation
```

**Detailed Implementation with Data Key Caching**:

```python
import boto3
import time
from functools import lru_cache
from datetime import datetime, timedelta

class OptimizedS3Encryption:
    def __init__(self, bucket_name, key_id):
        self.s3_client = boto3.client('s3')
        self.kms_client = boto3.client('kms')
        self.bucket_name = bucket_name
        self.key_id = key_id
        self.cache_duration = 3600  # 1 hour
        self.data_key_cache = {}
    
    def get_data_key(self, cache_key='default'):
        """
        Cache data keys for up to 1 hour to reduce KMS API calls
        while maintaining compliance (keys rotated regularly)
        """
        current_time = time.time()
        
        if cache_key in self.data_key_cache:
            cached_key, created_at = self.data_key_cache[cache_key]
            if current_time - created_at < self.cache_duration:
                return cached_key  # Use cached key
        
        # Generate new data key only if cache expired
        response = self.kms_client.generate_data_key(
            KeyId=self.key_id,
            KeySpec='AES_256',
            EncryptionContext={
                'purpose': 's3_encryption',
                'cache_key': cache_key
            }
        )
        
        plaintext_key = response['Plaintext']
        self.data_key_cache[cache_key] = (plaintext_key, current_time)
        
        return plaintext_key
    
    def upload_encrypted_object(self, key_name, file_path):
        """
        Upload with S3 Bucket Key (no KMS API call needed)
        """
        try:
            with open(file_path, 'rb') as f:
                self.s3_client.put_object(
                    Bucket=self.bucket_name,
                    Key=key_name,
                    Body=f,
                    ServerSideEncryption='aws:kms',
                    SSEKMSKeyId=self.key_id,
                    Metadata={
                        'timestamp': datetime.now().isoformat(),
                        'bucket-key-enabled': 'true'
                    }
                )
            return True
        except Exception as e:
            print(f"Upload failed: {e}")
            return False
    
    def get_encrypted_object(self, key_name):
        """
        Download and decrypt using S3 Bucket Key
        (S3 handles decryption with KMS)
        """
        try:
            response = self.s3_client.get_object(
                Bucket=self.bucket_name,
                Key=key_name
            )
            return response['Body'].read()
        except Exception as e:
            print(f"Download failed: {e}")
            return None
    
    def monitor_kms_performance(self):
        """
        Monitor KMS metrics to verify optimization
        """
        cloudwatch = boto3.client('cloudwatch')
        
        metrics = cloudwatch.get_metric_statistics(
            Namespace='AWS/KMS',
            MetricName='UserErrorCount',
            StartTime=datetime.now() - timedelta(hours=24),
            EndTime=datetime.now(),
            Period=3600,
            Statistics=['Sum'],
            Dimensions=[{'Name': 'KeyId', 'Value': self.key_id}]
        )
        
        return metrics['Datapoints']

# Usage
optimizer = OptimizedS3Encryption(
    bucket_name='my-high-traffic-bucket',
    key_id='arn:aws:kms:us-east-1:123456789012:key/key-id'
)

# Batch upload (all use same Bucket Key, minimal KMS overhead)
for i in range(1000000):
    optimizer.upload_encrypted_object(
        key_name=f'data/object-{i}.bin',
        file_path=f'local/object-{i}.bin'
    )

# Monitor performance
metrics = optimizer.monitor_kms_performance()
print(f"KMS errors in last 24h: {sum(m['Sum'] for m in metrics)}")
```

**Advanced Cost Tracking with AWS Cost Explorer**:

```bash
# Create cost allocation tags for KMS usage
aws kms tag-resource \
  --key-id arn:aws:kms:us-east-1:123456789012:key/key-id \
  --tags TagKey=CostCenter,TagValue=Engineering \
           TagKey=Application,TagValue=HighTraffic

# Use AWS Cost Explorer API to track costs by tag
aws ce get-cost-and-usage \
  --time-period Start=2024-01-01,End=2024-01-31 \
  --granularity MONTHLY \
  --metrics "UnblendedCost" \
  --group-by Type=DIMENSION,Key=SERVICE \
           Type=TAG,Key=Application \
  --filter file://kms-filter.json
```

**Performance Monitoring Dashboard**:

```python
# CloudWatch Dashboard for KMS monitoring
dashboard_body = {
    "widgets": [
        {
            "type": "metric",
            "properties": {
                "metrics": [
                    ["AWS/KMS", "UserErrorCount", {"stat": "Sum"}],
                    ["AWS/KMS", "ThrottledCount", {"stat": "Sum"}],
                    ["AWS/S3", "4xxErrors", {"stat": "Sum"}],
                    ["AWS/S3", "5xxErrors", {"stat": "Sum"}]
                ],
                "period": 300,
                "stat": "Average",
                "region": "us-east-1",
                "title": "KMS & S3 Performance"
            }
        }
    ]
}

cloudwatch = boto3.client('cloudwatch')
cloudwatch.put_dashboard(
    DashboardName='KMS-S3-Optimization',
    DashboardBody=str(dashboard_body)
)
```

---

## References

[1] AWS Key Management Service - AWS Documentation. https://docs.aws.amazon.com/kms/

[2] AWS KMS Best Practices - AWS Prescriptive Guidance. https://docs.aws.amazon.com/prescriptive-guidance/aws-kms-best-practices/

[3] Key Management Lifecycle Best Practices - Cloud Security Alliance. https://cloudsecurityalliance.org/blog/2024/02/02/key-management-lifecycle-best-practices/

[4] AWS KMS Key Rotation - AWS Documentation. https://docs.aws.amazon.com/kms/latest/developerguide/rotating-keys.html

[5] S3 Bucket Key Optimization - AWS Security Blog. https://aws.amazon.com/blogs/security/aws-kms-how-many-keys-do-i-need/

[6] AWS KMS Concepts and Encryption Context - AWS Documentation. https://docs.aws.amazon.com/kms/latest/developerguide/concepts.html

[7] Envelope Encryption Pattern - AWS KMS Developer Guide. https://docs.aws.amazon.com/kms/latest/developerguide/envelope-encryption.html

[8] KMS Key Policies and IAM Integration - AWS Documentation. https://docs.aws.amazon.com/kms/latest/developerguide/key-policies.html

[9] CloudTrail Logging for KMS - AWS Security Best Practices. https://docs.aws.amazon.com/kms/latest/developerguide/logging-using-cloudtrail.html

[10] Multi-Region KMS Strategy - AWS Architecture Guide. https://docs.aws.amazon.com/prescriptive-guidance/patterns/multi-account-multi-region-architecture/