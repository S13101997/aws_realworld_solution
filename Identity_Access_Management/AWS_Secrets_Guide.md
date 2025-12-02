# AWS Secrets Manager & Parameter Store — Secure Secrets Management

## Detailed Explanation

### Overview: Secrets Management in AWS

AWS provides two complementary services for managing sensitive data and configuration information:

1. **AWS Secrets Manager**: Purpose-built for managing secrets (credentials, API keys, database passwords)
2. **AWS Systems Manager Parameter Store**: General-purpose configuration management with secret support

Both services provide encryption, audit logging, and fine-grained access control, but with different strengths and use cases.

### AWS Secrets Manager

#### What is AWS Secrets Manager?

AWS Secrets Manager is a fully managed service for protecting secrets used to access databases, APIs, and other services. It enables you to rotate secrets automatically, manage access through fine-grained permissions, and audit secret usage through CloudTrail[1].

**Key Characteristics:**
- **Purpose-Built for Secrets**: Optimized for database credentials, API keys, OAuth tokens
- **Automatic Rotation**: Built-in Lambda integration for routine credential updates
- **Cross-Account Access**: Resource-based policies for multi-account deployments
- **Versioning**: Maintains secret history with labeled versions
- **Cost**: $0.40 per secret per month + $0.05 per 10,000 API calls
- **Limit**: 500,000 secrets per region per account
- **Max Secret Size**: 64 KB

**Architecture Flow**:

```
Application → Secret Request → Secrets Manager → KMS Decryption
    ↓                              ↓                   ↓
Query Secret         Find Latest Version        Decrypt with Key
                     Check Permissions          Return Plaintext
                     CloudTrail Audit Log       (temporary in memory)
```

#### AWS Systems Manager Parameter Store

Parameter Store is a secure, hierarchical configuration store that manages both plaintext and encrypted configuration data and secrets[2].

**Key Characteristics:**
- **Dual Purpose**: Manages configurations AND secrets
- **Hierarchical Organization**: Path-based parameter naming (/app/prod/db/password)
- **Manual Rotation**: No built-in automatic rotation (requires custom Lambda)
- **Cost-Effective**: Standard tier = free (up to 10,000 parameters)
- **Limit**: 10,000 standard parameters per region (unlimited advanced)
- **Max Parameter Size**: 4 KB standard tier, 8 KB advanced tier
- **Parameter Types**:
  - String: Plaintext configuration values
  - StringList: Comma-separated values
  - SecureString: Encrypted using AWS KMS

---

### Feature Comparison Matrix

| Feature | Secrets Manager | Parameter Store |
|---------|:---------------:|:---------------:|
| **Automatic Rotation** | ✅ (Lambda-based) | ❌ (manual only) |
| **Cross-Account Access** | ✅ (resource policy) | ⚠️ (limited) |
| **RDS Auto Rotation** | ✅ (native) | ❌ |
| **Versioning** | ✅ (automatic + labeled) | ⚠️ (last 100 versions) |
| **Size Limit** | 64 KB | 4 KB (standard) / 8 KB (advanced) |
| **Cost** | $0.40/secret + API calls | Free (standard) / $0.05/secret (advanced) |
| **TTL Support** | ❌ | ✅ (parameter expiration) |
| **Hierarchy** | ❌ (flat structure) | ✅ (path-based) |
| **Multi-Region Replication** | ✅ | ❌ |
| **Public/Private Secrets** | ✅ | ❌ |
| **Best For** | Database credentials, API keys | Configuration management, feature flags |

---

### Permission Models

#### Secrets Manager Permission Model

Secrets Manager uses **four-layer permission model**:

\begin{enumerate}
\item **IAM Policy**: Principal must have secretsmanager API permissions
\item **Resource Policy**: Secret's policy grants/denies access to principals
\item **VPC Endpoint Policy**: Restricts access if using VPC endpoint
\item **Encryption Key Policy**: KMS key must allow decrypt operation
\end{enumerate}

**Evaluation**: All four layers must allow the action.

#### Parameter Store Permission Model

Parameter Store uses **hierarchical permission model**:

\begin{itemize}
\item **IAM Policy**: Principal must have ssm:GetParameter/ssm:PutParameter permission
\item **Parameter Tier**: Standard vs Advanced affects permissions and limits
\item **SecureString Encryption**: Requires kms:Decrypt in addition to ssm:GetParameter
\item **Path-Based Access**: Can grant access to parameter hierarchies (e.g., /prod/*)
\end{itemize}

---

### Secret Rotation Strategy

#### Secrets Manager Rotation

**Rotation Process (Automatic)**:

1. **Schedule**: Define rotation frequency (e.g., every 30 days)
2. **Lambda Invocation**: Secrets Manager calls Lambda rotation function
3. **Create New Secret**: Function generates new credentials
4. **Test**: Verify new credentials work with target service
5. **Finish**: Update secret version and mark as current
6. **Clean Up**: Previous versions remain accessible for rollback

**Rotation Strategies**:

| Strategy | Use Case | Implementation |
|----------|----------|-----------------|
| **Single User** | Simple applications, APIs | Create new secret, update in one operation |
| **Alternating Users** | Databases (RDS, Aurora) | Maintain two users, rotate alternately to avoid lock-outs |
| **Custom** | Complex services | Write bespoke Lambda rotation logic |

**Supported Native Rotations** (Lambda templates provided):
- Amazon RDS (MySQL, PostgreSQL, Aurora)
- Amazon Redshift
- Amazon DocumentDB
- Amazon DynamoDB
- Generic secret (HTTP API updates)

#### Parameter Store Rotation

Parameter Store does NOT have automatic rotation. Options:

\begin{enumerate}
\item **EventBridge + Lambda**: Trigger Lambda on schedule to update parameter
\item **API Update**: Application periodically updates parameter value
\item **Infrastructure-as-Code**: Terraform/CloudFormation manages rotation
\item **Manual Process**: Team manually updates parameter on schedule
\end{enumerate}

---

## Detailed Examples

### Example 1: Creating and Managing Database Credentials with Secrets Manager

**Scenario**: Application needs to securely store RDS database credentials with automatic rotation.

**AWS Console Setup**:

```
1. Navigate to Secrets Manager
2. Click "Store a new secret"
3. Select "Credentials for RDS database"
4. Configure:
   - Username: admin_user
   - Password: GenerateSecurePassword
   - Database: production_db
   - Host: prod-db.c9akciq32.us-east-1.rds.amazonaws.com
5. Enable "Automatic rotation"
6. Set rotation interval: 30 days
7. Choose Lambda function: SecretsManager-RDS-MySQL
8. Attach IAM permissions
9. Store secret
```

**AWS CLI Creation**:

```bash
# Step 1: Create the secret
aws secretsmanager create-secret \
  --name prod/rds/admin \
  --description "Production RDS admin credentials" \
  --secret-string '{
    "username":"admin_user",
    "password":"InitialSecurePassword123!",
    "engine":"mysql",
    "host":"prod-db.c9akciq32.us-east-1.rds.amazonaws.com",
    "port":3306,
    "dbname":"production"
  }' \
  --kms-key-id arn:aws:kms:us-east-1:123456789012:key/key-id \
  --tags Key=Environment,Value=Production Key=Application,Value=AppName

# Step 2: Configure automatic rotation
aws secretsmanager rotate-secret \
  --secret-id prod/rds/admin \
  --rotation-lambda-arn arn:aws:lambda:us-east-1:123456789012:function:SecretsManager-RDS-MySQL \
  --rotation-rules "AutomaticallyAfterDays=30,Duration=2,DurationUnit=HOURS"

# Step 3: Grant Lambda permission to rotate
aws lambda add-permission \
  --function-name SecretsManager-RDS-MySQL \
  --statement-id AllowSecretsManager \
  --action lambda:InvokeFunction \
  --principal secretsmanager.amazonaws.com

# Step 4: Grant RDS modify permissions to Lambda role
aws iam put-role-policy \
  --role-name SecretsManager-RDS-Rotation-Role \
  --policy-name RDSRotation \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Action": [
          "rds:ModifyDBInstance",
          "rds:DescribeDBInstances"
        ],
        "Resource": "arn:aws:rds:us-east-1:123456789012:db:production_db"
      }
    ]
  }'

# Step 5: Retrieve secret in application
aws secretsmanager get-secret-value \
  --secret-id prod/rds/admin \
  --query 'SecretString' \
  --output text | jq -r '.password'
```

**Application Integration (Python)**:

```python
import boto3
import json
from botocore.exceptions import ClientError

class RDSCredentialsManager:
    def __init__(self, secret_name, region_name='us-east-1'):
        self.secret_name = secret_name
        self.region_name = region_name
        self.client = boto3.client('secretsmanager', region_name=region_name)
        self.secret_cache = None
        self.cache_timestamp = None
        self.cache_ttl = 3600  # 1 hour cache
    
    def get_secret(self, use_cache=True):
        """Retrieve database credentials from Secrets Manager"""
        
        import time
        current_time = time.time()
        
        # Use cached secret if valid
        if use_cache and self.secret_cache and (current_time - self.cache_timestamp) < self.cache_ttl:
            return self.secret_cache
        
        try:
            response = self.client.get_secret_value(
                SecretId=self.secret_name
            )
            
            if 'SecretString' in response:
                secret = json.loads(response['SecretString'])
                self.secret_cache = secret
                self.cache_timestamp = current_time
                return secret
            else:
                raise ValueError("Binary secret not supported for RDS")
                
        except ClientError as e:
            if e.response['Error']['Code'] == 'ResourceNotFoundException':
                print(f"Secret {self.secret_name} not found")
            elif e.response['Error']['Code'] == 'InvalidRequestException':
                print("Invalid request to retrieve secret")
            else:
                print(f"Error retrieving secret: {e}")
            raise
    
    def get_connection_string(self):
        """Build database connection string"""
        secret = self.get_secret()
        
        connection_string = (
            f"mysql+pymysql://{secret['username']}:{secret['password']}@"
            f"{secret['host']}:{secret['port']}/{secret['dbname']}"
        )
        return connection_string
    
    def rotate_immediately(self):
        """Force immediate secret rotation"""
        try:
            response = self.client.rotate_secret(
                SecretId=self.secret_name,
                RotateImmediately=True
            )
            print(f"Rotation initiated: {response['ARN']}")
            return True
        except ClientError as e:
            print(f"Rotation failed: {e}")
            return False

# Usage
creds_manager = RDSCredentialsManager('prod/rds/admin')
db_url = creds_manager.get_connection_string()

# Use with SQLAlchemy
from sqlalchemy import create_engine
engine = create_engine(db_url)
connection = engine.connect()
```

---

### Example 2: Parameter Store Hierarchical Configuration with SecureStrings

**Scenario**: Application uses Parameter Store to manage environment-specific configurations including database passwords and API keys.

**Hierarchical Structure**:

```
/myapp/                                    # Root
├── /myapp/prod/                           # Production environment
│   ├── /myapp/prod/db/host               # SecureString
│   ├── /myapp/prod/db/password           # SecureString
│   ├── /myapp/prod/api/key               # SecureString
│   └── /myapp/prod/cache/ttl             # String
├── /myapp/staging/                        # Staging environment
│   ├── /myapp/staging/db/host            # SecureString
│   ├── /myapp/staging/db/password        # SecureString
│   └── /myapp/staging/features/new_ui    # String (boolean)
└── /myapp/dev/                            # Development
    ├── /myapp/dev/db/host                # String (plaintext OK in dev)
    └── /myapp/dev/debug_mode             # String
```

**AWS CLI Parameter Creation**:

```bash
#!/bin/bash

# Production Database Configuration
aws ssm put-parameter \
  --name /myapp/prod/db/host \
  --value "prod-db.c9akciq32.us-east-1.rds.amazonaws.com" \
  --type SecureString \
  --key-id arn:aws:kms:us-east-1:123456789012:key/key-id \
  --tags Key=Environment,Value=Production Key=Component,Value=Database

aws ssm put-parameter \
  --name /myapp/prod/db/password \
  --value "SecurePassword123!@#" \
  --type SecureString \
  --key-id arn:aws:kms:us-east-1:123456789012:key/key-id \
  --tags Key=Environment,Value=Production Key=Component,Value=Database

aws ssm put-parameter \
  --name /myapp/prod/db/username \
  --value "produser" \
  --type String \
  --tags Key=Environment,Value=Production Key=Component,Value=Database

# Production API Configuration
aws ssm put-parameter \
  --name /myapp/prod/api/key \
  --value "sk_live_abc123xyz..." \
  --type SecureString \
  --key-id arn:aws:kms:us-east-1:123456789012:key/key-id \
  --tags Key=Environment,Value=Production Key=Component,Value=API

# Production Feature Flags (plaintext)
aws ssm put-parameter \
  --name /myapp/prod/features/new_ui \
  --value "true" \
  --type String \
  --tags Key=Environment,Value=Production Key=Component,Value=Features

# Staging Configuration
aws ssm put-parameter \
  --name /myapp/staging/db/host \
  --value "staging-db.c9akciq32.us-east-1.rds.amazonaws.com" \
  --type SecureString \
  --key-id arn:aws:kms:us-east-1:123456789012:key/key-id \
  --tags Key=Environment,Value=Staging

# Get all parameters in production
aws ssm get-parameters-by-path \
  --path /myapp/prod \
  --recursive \
  --with-decryption
```

**Application Integration (Node.js with Lambda Extension)**:

```javascript
const AWS = require('aws-sdk');
const ssm = new AWS.SSM({ region: 'us-east-1' });

class ConfigurationManager {
  constructor(environment = 'prod') {
    this.environment = environment;
    this.parameterCache = new Map();
    this.cacheTTL = 3600000; // 1 hour in milliseconds
  }

  /**
   * Get parameter by path (hierarchical)
   */
  async getConfigurationByPath(basePath, decrypt = true) {
    const fullPath = `/myapp/${this.environment}${basePath}`;
    
    try {
      const params = {
        Path: fullPath,
        Recursive: true,
        WithDecryption: decrypt
      };
      
      const response = await ssm.getParametersByPath(params).promise();
      
      // Convert flat parameter list to nested object
      const config = {};
      response.Parameters.forEach(param => {
        const keys = param.Name.replace(fullPath + '/', '').split('/');
        let obj = config;
        for (let i = 0; i < keys.length - 1; i++) {
          obj[keys[i]] = obj[keys[i]] || {};
          obj = obj[keys[i]];
        }
        obj[keys[keys.length - 1]] = param.Value;
      });
      
      return config;
    } catch (error) {
      console.error(`Failed to retrieve configuration from ${fullPath}:`, error);
      throw error;
    }
  }

  /**
   * Get single parameter with caching
   */
  async getParameter(parameterName, decrypt = true) {
    const cacheKey = `${parameterName}:${decrypt}`;
    const now = Date.now();
    
    // Check cache
    if (this.parameterCache.has(cacheKey)) {
      const { value, timestamp } = this.parameterCache.get(cacheKey);
      if (now - timestamp < this.cacheTTL) {
        console.log(`Cache hit for ${parameterName}`);
        return value;
      }
    }
    
    try {
      const response = await ssm.getParameter({
        Name: `/myapp/${this.environment}${parameterName}`,
        WithDecryption: decrypt
      }).promise();
      
      const value = response.Parameter.Value;
      this.parameterCache.set(cacheKey, { value, timestamp: now });
      
      return value;
    } catch (error) {
      console.error(`Failed to retrieve parameter ${parameterName}:`, error);
      throw error;
    }
  }

  /**
   * Put parameter (update configuration)
   */
  async putParameter(parameterName, value, encrypt = false) {
    try {
      const response = await ssm.putParameter({
        Name: `/myapp/${this.environment}${parameterName}`,
        Value: value,
        Type: encrypt ? 'SecureString' : 'String',
        Overwrite: true
      }).promise();
      
      // Invalidate cache
      this.parameterCache.clear();
      
      console.log(`Parameter updated: ${parameterName}, Version: ${response.Version}`);
      return response;
    } catch (error) {
      console.error(`Failed to update parameter ${parameterName}:`, error);
      throw error;
    }
  }
}

// Lambda Handler
exports.handler = async (event) => {
  const config = new ConfigurationManager('prod');
  
  try {
    // Get all database configuration
    const dbConfig = await config.getConfigurationByPath('/db', true);
    console.log('DB Config:', dbConfig);
    
    // Get specific parameter
    const apiKey = await config.getParameter('/api/key', true);
    console.log('API Key retrieved successfully');
    
    // Update feature flag
    await config.putParameter('/features/new_ui', 'false', false);
    
    return {
      statusCode: 200,
      body: JSON.stringify({ message: 'Configuration loaded successfully' })
    };
  } catch (error) {
    return {
      statusCode: 500,
      body: JSON.stringify({ error: error.message })
    };
  }
};
```

---

### Example 3: Secrets Manager with Custom Lambda Rotation

**Scenario**: Rotate API keys for third-party service (not RDS or native rotation).

**Lambda Rotation Function**:

```python
import boto3
import json
import requests
from datetime import datetime

secretsmanager_client = boto3.client('secretsmanager')

def lambda_handler(event, context):
    """
    Custom rotation for third-party API key rotation
    
    Rotation stages:
    1. create - Generate new secret
    2. set - Update service with new credentials
    3. test - Verify new credentials work
    4. finish - Mark new version as current
    """
    
    service_client = event['ClientRequestToken']
    secret_id = event['SecretId']
    step = event['Step']
    secret_version = event['ClientRequestToken']
    
    # Get secret metadata
    metadata = secretsmanager_client.describe_secret(SecretId=secret_id)
    
    if not metadata['RotationEnabled']:
        raise ValueError(f"Secret {secret_id} is not enabled for rotation")
    
    versions = metadata['VersionIdsToStages']
    
    if secret_version not in versions:
        raise ValueError(f"Secret version {secret_version} not found")
    
    if len(versions[secret_version]) == 1 and 'AWSCURRENT' in versions[secret_version]:
        print(f"Secret version {secret_version} is already set as AWSCURRENT")
        return
    
    if 'AWSCURRENT' not in versions:
        raise ValueError(f"Secret {secret_id} does not have AWSCURRENT version")
    
    current_dict = get_secret_dict(
        secretsmanager_client, 
        secret_id, 
        'AWSCURRENT', 
        secret_version
    )
    
    # Step 1: Create new secret
    if step == "create":
        new_api_key = generate_new_api_key()
        
        new_secret_dict = {
            "api_key": new_api_key,
            "service_name": "third-party-api",
            "created_date": datetime.now().isoformat(),
            "rotated_by": "lambda-rotation"
        }
        
        try:
            secretsmanager_client.put_secret_value(
                SecretId=secret_id,
                ClientRequestToken=secret_version,
                Stages=['AWSPENDING'],
                SecretString=json.dumps(new_secret_dict)
            )
            print(f"Created new secret version {secret_version}")
        except ClientError as e:
            if e.response['Error']['Code'] == 'InvalidParameterException':
                print(f"Secret version {secret_version} already exists")
            else:
                raise
    
    # Step 2: Set (register new API key with service)
    elif step == "set":
        pending_dict = get_secret_dict(
            secretsmanager_client,
            secret_id,
            'AWSPENDING',
            secret_version
        )
        
        # Call third-party API to register new key
        if register_api_key_with_service(pending_dict['api_key']):
            print(f"Registered new API key with service")
        else:
            raise ValueError("Failed to register API key with service")
    
    # Step 3: Test (verify new credentials work)
    elif step == "test":
        pending_dict = get_secret_dict(
            secretsmanager_client,
            secret_id,
            'AWSPENDING',
            secret_version
        )
        
        if not test_api_key(pending_dict['api_key']):
            raise ValueError("New API key does not work")
        
        print(f"Successfully tested new API key")
    
    # Step 4: Finish (make pending version current)
    elif step == "finish":
        versions = secretsmanager_client.describe_secret(SecretId=secret_id)['VersionIdsToStages']
        
        if secret_version not in versions:
            raise ValueError(f"Secret version {secret_version} not found")
        
        if 'AWSCURRENT' in versions[secret_version]:
            print(f"Secret version {secret_version} already marked as AWSCURRENT")
            return
        
        if 'AWSPENDING' not in versions[secret_version]:
            raise ValueError(f"Secret version {secret_version} not in AWSPENDING stage")
        
        # Update version stages
        secretsmanager_client.update_secret_version_stage(
            SecretId=secret_id,
            VersionStage='AWSCURRENT',
            MoveToVersionId=secret_version,
            RemoveFromVersionId=versions['AWSCURRENT'][0]
        )
        
        # Revoke old API key
        for version_id in versions:
            if 'AWSPREVIOUS' in versions[version_id]:
                old_secret = get_secret_dict(
                    secretsmanager_client,
                    secret_id,
                    'AWSPREVIOUS',
                    version_id
                )
                revoke_api_key_from_service(old_secret['api_key'])
        
        print(f"Finished rotation for secret version {secret_version}")
    
    else:
        raise ValueError(f"Invalid rotation step: {step}")


def get_secret_dict(client, secret_id, stage, token):
    """Retrieve secret at specific stage"""
    secret = client.get_secret_value(
        SecretId=secret_id,
        VersionId=token,
        VersionStage=stage
    )
    
    return json.loads(secret['SecretString'])


def generate_new_api_key():
    """Generate new API key (replace with actual generation logic)"""
    import secrets
    return f"sk_{secrets.token_hex(32)}"


def register_api_key_with_service(api_key):
    """Register API key with third-party service"""
    try:
        response = requests.post(
            'https://api.thirdparty.com/keys',
            headers={'Authorization': 'Bearer admin-token'},
            json={'api_key': api_key, 'name': f'rotated-{datetime.now().isoformat()}'}
        )
        return response.status_code == 201
    except Exception as e:
        print(f"Failed to register key: {e}")
        return False


def test_api_key(api_key):
    """Test if API key works"""
    try:
        response = requests.get(
            'https://api.thirdparty.com/health',
            headers={'Authorization': f'Bearer {api_key}'}
        )
        return response.status_code == 200
    except Exception as e:
        print(f"Failed to test key: {e}")
        return False


def revoke_api_key_from_service(api_key):
    """Revoke old API key from service"""
    try:
        requests.delete(
            f'https://api.thirdparty.com/keys/{api_key}',
            headers={'Authorization': 'Bearer admin-token'}
        )
        print(f"Revoked old API key from service")
    except Exception as e:
        print(f"Failed to revoke key: {e}")
```

**Lambda Configuration**:

```bash
# Create Lambda execution role
aws iam create-role \
  --role-name SecretsManager-CustomRotation-Role \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Principal": {"Service": "lambda.amazonaws.com"},
        "Action": "sts:AssumeRole"
      }
    ]
  }'

# Attach policies
aws iam put-role-policy \
  --role-name SecretsManager-CustomRotation-Role \
  --policy-name SecretsManagerAccess \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Action": [
          "secretsmanager:DescribeSecret",
          "secretsmanager:GetSecretValue",
          "secretsmanager:PutSecretValue",
          "secretsmanager:UpdateSecretVersionStage"
        ],
        "Resource": "arn:aws:secretsmanager:us-east-1:123456789012:secret:*"
      }
    ]
  }'

# Create Lambda function
aws lambda create-function \
  --function-name SecretsManager-CustomRotation \
  --runtime python3.11 \
  --role arn:aws:iam::123456789012:role/SecretsManager-CustomRotation-Role \
  --handler index.lambda_handler \
  --zip-file fileb://rotation_function.zip

# Configure rotation
aws secretsmanager rotate-secret \
  --secret-id thirdparty-api-key \
  --rotation-lambda-arn arn:aws:lambda:us-east-1:123456789012:function:SecretsManager-CustomRotation \
  --rotation-rules "AutomaticallyAfterDays=90"
```

---

### Example 4: Cross-Account Secret Access

**Scenario**: Application in Account B needs to access secrets in Account A (centralized secrets management).

**Account A (Secret Owner) - Resource Policy**:

```bash
# Create secret in Account A
aws secretsmanager create-secret \
  --name /shared/database/credentials \
  --secret-string '{
    "username": "dbadmin",
    "password": "SecurePassword123!"
  }'

# Grant Account B access via resource policy
aws secretsmanager put-resource-policy \
  --secret-id /shared/database/credentials \
  --resource-policy '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Principal": {
          "AWS": "arn:aws:iam::222222222222:role/ApplicationRole"
        },
        "Action": [
          "secretsmanager:GetSecretValue",
          "secretsmanager:DescribeSecret"
        ],
        "Resource": "*"
      }
    ]
  }'
```

**Account B (Secret Consumer) - IAM Policy**:

```bash
# Add IAM policy to Account B role
aws iam put-role-policy \
  --role-name ApplicationRole \
  --policy-name AccessSharedSecrets \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Action": [
          "secretsmanager:GetSecretValue",
          "secretsmanager:DescribeSecret"
        ],
        "Resource": "arn:aws:secretsmanager:us-east-1:111111111111:secret:/shared/database/credentials*"
      },
      {
        "Effect": "Allow",
        "Action": "kms:Decrypt",
        "Resource": "arn:aws:kms:us-east-1:111111111111:key/key-id"
      }
    ]
  }'
```

**Application in Account B (Retrieval)**:

```python
import boto3

class CrossAccountSecretsClient:
    def __init__(self, secret_arn, role_arn, session_name='cross-account-access'):
        self.secret_arn = secret_arn
        self.role_arn = role_arn
        self.session_name = session_name
        self.secret_client = None
    
    def assume_role_and_get_credentials(self):
        """Assume role in Account A and get credentials"""
        sts_client = boto3.client('sts')
        
        response = sts_client.assume_role(
            RoleArn=self.role_arn,
            RoleSessionName=self.session_name
        )
        
        credentials = response['Credentials']
        
        # Create Secrets Manager client with assumed role credentials
        self.secret_client = boto3.client(
            'secretsmanager',
            region_name='us-east-1',
            aws_access_key_id=credentials['AccessKeyId'],
            aws_secret_access_key=credentials['SecretAccessKey'],
            aws_session_token=credentials['SessionToken']
        )
    
    def get_secret(self):
        """Retrieve secret from Account A"""
        if not self.secret_client:
            self.assume_role_and_get_credentials()
        
        response = self.secret_client.get_secret_value(SecretId=self.secret_arn)
        return response

# Usage
client = CrossAccountSecretsClient(
    secret_arn='arn:aws:secretsmanager:us-east-1:111111111111:secret:/shared/database/credentials',
    role_arn='arn:aws:iam::111111111111:role/CrossAccountAccessRole'
)

secret = client.get_secret()
print(secret['SecretString'])
```

---

### Example 5: Parameter Store with CloudFormation Integration

**Scenario**: CloudFormation template references Parameter Store parameters for dynamic configuration.

**CloudFormation Template**:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'Application Stack with Parameter Store References'

Parameters:
  Environment:
    Type: String
    Default: prod
    AllowedValues: [dev, staging, prod]

Resources:
  # Lambda Execution Role
  LambdaExecutionRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: lambda.amazonaws.com
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
      Policies:
        - PolicyName: ParameterStoreAccess
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action:
                  - ssm:GetParameter
                  - ssm:GetParameters
                Resource:
                  - !Sub 'arn:aws:ssm:${AWS::Region}:${AWS::AccountId}:parameter/myapp/${Environment}/*'
              - Effect: Allow
                Action: kms:Decrypt
                Resource: !Sub 'arn:aws:kms:${AWS::Region}:${AWS::AccountId}:key/key-id'

  # Lambda Function using Parameter Store
  ApplicationFunction:
    Type: AWS::Lambda::Function
    Properties:
      FunctionName: !Sub 'app-function-${Environment}'
      Runtime: python3.11
      Role: !GetAtt LambdaExecutionRole.Arn
      Environment:
        Variables:
          ENVIRONMENT: !Ref Environment
          # Reference Parameter Store as environment variables
          DB_HOST: !Sub '{{resolve:ssm:/myapp/${Environment}/db/host}}'
          DB_USER: !Sub '{{resolve:ssm:/myapp/${Environment}/db/username}}'
          DB_PASSWORD: !Sub '{{resolve:ssm-secure:/myapp/${Environment}/db/password}}'
          API_KEY: !Sub '{{resolve:ssm-secure:/myapp/${Environment}/api/key}}'
      Code:
        ZipFile: |
          import os
          import boto3
          
          def handler(event, context):
              # Parameters loaded from environment via CloudFormation
              db_host = os.environ['DB_HOST']
              api_key = os.environ['API_KEY']
              
              return {
                  'statusCode': 200,
                  'body': 'Configuration loaded successfully'
              }

  # RDS Database using Parameter Store
  DatabaseSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Database security group
      VpcId: !Sub '{{resolve:ssm:/myapp/${Environment}/vpc/id}}'

Outputs:
  LambdaFunctionArn:
    Description: Lambda Function ARN
    Value: !GetAtt ApplicationFunction.Arn
    Export:
      Name: !Sub '${Environment}-ApplicationFunctionArn'
```

**Syntax Notes**:

| Syntax | Purpose | Decryption |
|--------|---------|-----------|
| `{{resolve:ssm:parameter-name}}` | Reference plaintext parameter | N/A |
| `{{resolve:ssm-secure:parameter-name}}` | Reference encrypted parameter | Automatic |
| `{{resolve:secretsmanager:secret-name}}` | Reference Secrets Manager secret | Automatic |

---

## Top 5 Interview Questions

### Question 1: When Should You Use Secrets Manager vs Parameter Store?

**Scenario**: "Your organization needs to manage multiple types of data: database credentials, API keys, feature flags, and application configuration. How would you decide what goes in Secrets Manager and what goes in Parameter Store?"

**Answer Structure**:

**Decision Matrix**:

| Use Case | Service | Reason |
|----------|---------|--------|
| **Database Credentials** | Secrets Manager | Built-in auto-rotation for RDS/Aurora/Redshift |
| **API Keys** | Secrets Manager | Needs rotation, cross-account access |
| **OAuth Tokens** | Secrets Manager | Time-sensitive, requires versioning |
| **Feature Flags** | Parameter Store | Plaintext, hierarchical, cost-effective |
| **App Configuration** | Parameter Store | Non-sensitive, hierarchical paths |
| **Environment URLs** | Parameter Store | Plaintext, change infrequently |
| **Cache TTL Values** | Parameter Store | String/Integer values, free tier |
| **Encryption Keys** | KMS (not here) | Never store keys in either service |

**Detailed Decision Logic**:

**Choose Secrets Manager when**:
\begin{itemize}
\item Secret requires automatic rotation (RDS, API keys, OAuth)
\item You need multi-region replication with automatic failover
\item Cross-account access required for centralized secrets
\item Compliance needs strict versioning and audit trails
\item Secret size < 64 KB and cost is acceptable
\end{itemize}

**Choose Parameter Store when**:
\begin{itemize}
\item Configuration is mostly application settings
\item Need hierarchical path-based organization (/app/prod/db/host)
\item Cost is critical (first 10,000 free in standard tier)
\item Manual rotation via Lambda acceptable
\item Size < 4 KB (standard) or 8 KB (advanced)
\item Integration with Systems Manager, CloudFormation needed
\end{itemize}

**Implementation Example**:

```
Organization Structure:

Secrets Manager:
├── prod/rds/admin          (auto-rotate every 30 days)
├── prod/api/stripe         (auto-rotate quarterly)
└── prod/oauth/github       (auto-rotate annually)

Parameter Store:
├── /app/prod/db/host       (SecureString - never changes)
├── /app/prod/cache/ttl     (String - updated via Lambda)
├── /app/prod/features/v2   (String - toggle via CloudFormation)
└── /app/prod/replica/count (String - infrastructure config)
```

**Cost Comparison for 100 production secrets**:

```
Scenario 1: All in Secrets Manager
- 100 secrets × $0.40/month = $40/month
- 100 secrets × 8,640 API calls/month = 864,000 calls
- 864,000 ÷ 10,000 × $0.05 = $4.32/month
- Monthly Cost: $44.32/month

Scenario 2: 30 secrets in SM, 70 in Parameter Store
- Secrets Manager: 30 × $0.40 = $12/month
- Parameter Store Standard: Free (first 10,000 params free)
- API calls Secrets Manager: 30 × 8,640 = 259,200 calls
- Cost: $12 + ($12.96) = ~$25/month
- Savings: 44% reduction
```

---

### Question 2: Design Secure Secret Rotation Strategy with Zero Downtime

**Scenario**: "Design a secret rotation strategy for a production database that serves millions of requests per day. The rotation must occur without downtime, with automatic rollback capability, and must maintain audit trails for compliance."

**Answer Structure**:

**Rotation Architecture**:

```
Week 1-2: Normal Operation
┌─────────────────┐
│ Application     │──────┐
│ (v1 Password)   │      │
└─────────────────┘      │
                         ▼
                   ┌────────────────┐
                   │ RDS Database   │
                   │ (Active User)  │
                   └────────────────┘

Week 2: Rotation Initiated
┌─────────────────────────────────────────┐
│ Secrets Manager Rotation Lambda          │
├─────────────────────────────────────────┤
│ 1. Create new DB user (standby)          │
│ 2. Grant same permissions as old user    │
│ 3. Update secret to new credentials      │
│ 4. Store old credentials in AWSPREVIOUS  │
└─────────────────────────────────────────┘

Week 2: Testing Phase (5 minute grace period)
┌──────────────┐              ┌──────────────┐
│ 90% Traffic  │              │ 10% Traffic  │
│ Old Password │──┐       ┌───│ New Password │
└──────────────┘  │       │   └──────────────┘
                  │       │
                  ▼       ▼
            ┌──────────────────┐
            │ RDS Database     │
            │ Both users work  │
            └──────────────────┘

Week 2: Gradual Migration (Canary Deployment)
Time  │ Old Traffic │ New Traffic │ Errors
──────┼─────────────┼─────────────┼────────
0:00  │    100%     │     0%      │  0.00%
0:15  │     90%     │    10%      │  0.00%
0:30  │     75%     │    25%      │  0.00%
1:00  │     50%     │    50%      │  0.00%
2:00  │     25%     │    75%      │  0.00%
4:00  │      0%     │   100%      │  0.00%  ← Fully migrated

Week 2: Cleanup
┌──────────────────────────────────────────┐
│ Disable old DB user after 24-hour buffer │
│ Keep secret in AWSPREVIOUS for 7 days    │
│ Mark rotation as complete in audit logs  │
└──────────────────────────────────────────┘
```

**Lambda Rotation Implementation with Canary**:

```python
import boto3
import time
from datetime import datetime, timedelta

class SecureRotationWithCanary:
    def __init__(self):
        self.sm = boto3.client('secretsmanager')
        self.rds = boto3.client('rds')
        self.cloudwatch = boto3.client('cloudwatch')
    
    def rotate_secret_with_canary(self, secret_id, db_instance_id):
        """
        Rotate secret with canary deployment strategy
        Minimizes risk by gradually shifting traffic
        """
        
        # Stage 1: Create new credentials
        current_secret = self.sm.get_secret_value(
            SecretId=secret_id,
            VersionStage='AWSCURRENT'
        )
        
        current_creds = json.loads(current_secret['SecretString'])
        
        # Generate new password
        new_password = self._generate_secure_password()
        
        # Create new DB user
        self._create_db_user(
            db_instance_id,
            f"appuser_{int(time.time())}",
            new_password,
            current_creds['username']
        )
        
        # Stage 2: Store new secret in AWSPENDING
        pending_version = self.sm.put_secret_value(
            SecretId=secret_id,
            Stages=['AWSPENDING'],
            SecretString=json.dumps({
                'username': f"appuser_{int(time.time())}",
                'password': new_password,
                'engine': current_creds['engine'],
                'host': current_creds['host']
            })
        )
        
        # Stage 3: Canary deployment with monitoring
        canary_config = {
            'duration_minutes': 240,  # 4-hour migration
            'error_threshold_percent': 0.1,  # 0.1% error rate threshold
            'rollback_on_error': True
        }
        
        try:
            success = self._execute_canary_deployment(
                secret_id,
                pending_version['VersionId'],
                canary_config
            )
            
            if success:
                # Stage 4: Promote new secret to AWSCURRENT
                self.sm.update_secret_version_stage(
                    SecretId=secret_id,
                    VersionStage='AWSCURRENT',
                    MoveToVersionId=pending_version['VersionId'],
                    RemoveFromVersionId=current_secret['VersionId']
                )
                
                # Keep old version in AWSPREVIOUS for 7 days
                self.sm.update_secret_version_stage(
                    SecretId=secret_id,
                    VersionStage='AWSPREVIOUS',
                    MoveToVersionId=current_secret['VersionId']
                )
                
                print("Secret rotation completed successfully")
                return True
            else:
                # Rollback on error
                self._rollback_rotation(secret_id, current_secret['VersionId'])
                return False
                
        except Exception as e:
            print(f"Rotation failed: {e}")
            self._rollback_rotation(secret_id, current_secret['VersionId'])
            raise
    
    def _execute_canary_deployment(self, secret_id, new_version_id, config):
        """
        Execute canary deployment with traffic gradual shift
        """
        
        duration = config['duration_minutes']
        error_threshold = config['error_threshold_percent']
        
        # Traffic migration schedule
        migration_schedule = [
            (0, 100),      # 0% new traffic
            (15, 10),      # 10% new traffic
            (30, 25),      # 25% new traffic
            (60, 50),      # 50% new traffic
            (120, 75),     # 75% new traffic
            (240, 100),    # 100% new traffic
        ]
        
        start_time = time.time()
        
        for offset, new_traffic_percent in migration_schedule:
            elapsed = time.time() - start_time
            
            if elapsed < offset * 60:
                time.sleep((offset * 60) - elapsed)
            
            # Update traffic shift (via load balancer/API Gateway)
            self._shift_traffic_percentage(new_traffic_percent, new_version_id)
            
            # Monitor error rate
            error_rate = self._get_error_rate()
            
            print(f"Traffic shift: {new_traffic_percent}% new, Error rate: {error_rate}%")
            
            if error_rate > error_threshold:
                print(f"ERROR RATE EXCEEDED ({error_rate}% > {error_threshold}%)")
                return False
            
            time.sleep(60)  # Observe for 1 minute
        
        return True
    
    def _get_error_rate(self):
        """Get error rate from CloudWatch"""
        response = self.cloudwatch.get_metric_statistics(
            Namespace='ApplicationMetrics',
            MetricName='DBConnectionErrors',
            StartTime=datetime.now() - timedelta(minutes=5),
            EndTime=datetime.now(),
            Period=300,
            Statistics=['Sum', 'Average']
        )
        
        if response['Datapoints']:
            return response['Datapoints'][0].get('Average', 0)
        return 0
    
    def _generate_secure_password(self):
        """Generate cryptographically secure password"""
        import secrets
        return f"Rotate{secrets.token_hex(16)}!@#"
    
    def _create_db_user(self, instance_id, username, password, template_user):
        """Create new DB user with same permissions as template user"""
        # Implementation would connect to DB and create user
        pass
    
    def _shift_traffic_percentage(self, new_traffic_percent, new_version):
        """Shift traffic to new secret version"""
        # Implementation would update load balancer/API Gateway
        pass
    
    def _rollback_rotation(self, secret_id, previous_version_id):
        """Rollback to previous secret version"""
        self.sm.update_secret_version_stage(
            SecretId=secret_id,
            VersionStage='AWSCURRENT',
            MoveToVersionId=previous_version_id
        )
```

**Monitoring & Alerting**:

```python
class RotationMonitoring:
    def __init__(self):
        self.cloudwatch = boto3.client('cloudwatch')
        self.sns = boto3.client('sns')
    
    def create_rotation_alarms(self, secret_id):
        """Create alarms for rotation monitoring"""
        
        # Alarm 1: Database connection errors spike
        self.cloudwatch.put_metric_alarm(
            AlarmName=f'secret-rotation-{secret_id}-errors',
            MetricName='DBConnectionErrors',
            Namespace='RDS',
            Statistic='Sum',
            Period=300,
            EvaluationPeriods=2,
            Threshold=100,
            ComparisonOperator='GreaterThanThreshold',
            AlarmActions=[f'arn:aws:sns:us-east-1:123456789012:rotation-alerts']
        )
        
        # Alarm 2: Lambda rotation function failures
        self.cloudwatch.put_metric_alarm(
            AlarmName=f'secret-rotation-{secret_id}-lambda-errors',
            MetricName='Errors',
            Namespace='AWS/Lambda',
            Statistic='Sum',
            Period=300,
            EvaluationPeriods=1,
            Threshold=1,
            ComparisonOperator='GreaterThanThreshold'
        )
```

---

### Question 3: Implement Least Privilege Access Control for Secrets

**Scenario**: "Your organization has developers, DBAs, security teams, and applications accessing the same secrets. Design access control that enforces least privilege while allowing necessary operations."

**Answer Structure**:

**Access Control Tiers**:

| Role | Secrets Manager | Parameter Store | Operations |
|------|:---------------:|:---------------:|------------|
| **Developer** | View, Decrypt | View, Decrypt | Read secrets for local testing |
| **DBA** | Rotate, Manage | Admin | Manage database secret rotation |
| **Security Team** | Admin, Tag | Admin | Audit, compliance, policy |
| **Application** | Decrypt only | Decrypt only | Runtime secret retrieval |
| **CI/CD Pipeline** | Decrypt | Decrypt | Secret injection during deployment |

**Fine-Grained IAM Policies**:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DeveloperReadSecretsForTesting",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:role/DeveloperRole"
      },
      "Action": [
        "secretsmanager:GetSecretValue",
        "secretsmanager:DescribeSecret"
      ],
      "Resource": [
        "arn:aws:secretsmanager:us-east-1:123456789012:secret:dev/*"
      ],
      "Condition": {
        "StringEquals": {
          "aws:SourceVpc": "vpc-dev-12345"
        },
        "IpAddress": {
          "aws:SourceIp": "10.0.0.0/8"
        }
      }
    },
    {
      "Sid": "DeveloperDenyProdSecrets",
      "Effect": "Deny",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:role/DeveloperRole"
      },
      "Action": "*",
      "Resource": [
        "arn:aws:secretsmanager:us-east-1:123456789012:secret:prod/*"
      ]
    },
    {
      "Sid": "ApplicationRuntimeAccess",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:role/ApplicationRole"
      },
      "Action": [
        "secretsmanager:GetSecretValue"
      ],
      "Resource": [
        "arn:aws:secretsmanager:us-east-1:123456789012:secret:prod/app/*"
      ],
      "Condition": {
        "StringEquals": {
          "aws:SourceVpc": "vpc-prod-67890"
        }
      }
    },
    {
      "Sid": "DBAManageRotation",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:role/DBARole"
      },
      "Action": [
        "secretsmanager:RotateSecret",
        "secretsmanager:PutSecretValue",
        "secretsmanager:UpdateSecretVersionStage",
        "secretsmanager:DescribeSecret",
        "secretsmanager:GetSecretValue"
      ],
      "Resource": [
        "arn:aws:secretsmanager:us-east-1:123456789012:secret:prod/database/*"
      ]
    }
  ]
}
```

**Parameter Store Hierarchical Access**:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ApplicationAccessProdParameters",
      "Effect": "Allow",
      "Action": [
        "ssm:GetParameter",
        "ssm:GetParameters"
      ],
      "Resource": "arn:aws:ssm:us-east-1:123456789012:parameter/myapp/prod/*"
    },
    {
      "Sid": "DeveloperAccessDevParameters",
      "Effect": "Allow",
      "Action": [
        "ssm:GetParameter",
        "ssm:GetParameters",
        "ssm:PutParameter"
      ],
      "Resource": "arn:aws:ssm:us-east-1:123456789012:parameter/myapp/dev/*"
    },
    {
      "Sid": "DenyDecryptProductionSecrets",
      "Effect": "Deny",
      "Action": "ssm:GetParameter",
      "Resource": "arn:aws:ssm:us-east-1:123456789012:parameter/myapp/prod/*/password",
      "Condition": {
        "StringEquals": {
          "ssm:Decrypt": "true"
        },
        "NotIpAddress": {
          "aws:SourceIp": "10.1.0.0/16"
        }
      }
    }
  ]
}
```

**Implementation with Resource Tags**:

```python
class SecretAccessController:
    def __init__(self):
        self.sm = boto3.client('secretsmanager')
        self.iam = boto3.client('iam')
    
    def enforce_least_privilege(self, secret_id, role_name, permissions):
        """
        Enforce least privilege by:
        1. Creating minimal IAM policy
        2. Adding resource tags
        3. Logging all access
        """
        
        # Get secret details
        secret = self.sm.describe_secret(SecretId=secret_id)
        
        # Build minimal policy
        policy = {
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Sid": f"Access{secret_id}",
                    "Effect": "Allow",
                    "Action": permissions,
                    "Resource": secret['ARN'],
                    "Condition": {
                        "StringEquals": {
                            "aws:RequestedRegion": "us-east-1"
                        }
                    }
                }
            ]
        }
        
        # Tag secret for audit
        self.sm.tag_resource(
            SecretId=secret_id,
            Tags=[
                {'Key': 'AccessRole', 'Value': role_name},
                {'Key': 'LastAudited', 'Value': datetime.now().isoformat()},
                {'Key': 'AccessLevel', 'Value': 'LeastPrivilege'}
            ]
        )
        
        # Apply policy to role
        self.iam.put_role_policy(
            RoleName=role_name,
            PolicyName=f'Access{secret_id}',
            PolicyDocument=json.dumps(policy)
        )
```

---

### Question 4: Troubleshoot and Debug Secrets Access Issues

**Scenario**: "An application is receiving 'AccessDenied' errors when trying to retrieve secrets. Walk through your troubleshooting methodology step by step."

**Answer Structure**:

**Troubleshooting Flowchart**:

```
AccessDenied Error
    ↓
┌─────────────────────────────────────────┐
│ Step 1: Verify Four-Layer Permissions  │
├─────────────────────────────────────────┤
│ ✓ IAM Policy grants secretsmanager actions
│ ✓ Secret resource policy trusts principal
│ ✓ VPC endpoint policy allows access (if applicable)
│ ✓ KMS key policy allows decrypt
└─────────────────────────────────────────┘
    ↓
[Check each layer - which fails?]
    ↓
┌─────────────────────────────────────────┐
│ Step 2: Enable CloudTrail Logging      │
│ Review failed API calls for details    │
└─────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────┐
│ Step 3: Check Secret Status            │
│ - Is secret enabled?                   │
│ - Is secret being rotated?             │
│ - Correct region?                      │
└─────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────┐
│ Step 4: Verify Encryption Context      │
│ (if required for encryption operation) │
└─────────────────────────────────────────┘
```

**Diagnostic Commands**:

```bash
#!/bin/bash

SECRET_ID="prod/rds/admin"
PRINCIPAL_ARN="arn:aws:iam::123456789012:role/ApplicationRole"

echo "=== SECRET ACCESS TROUBLESHOOTING ==="

# 1. Check if secret exists
echo "1. Verifying secret exists..."
aws secretsmanager describe-secret \
  --secret-id $SECRET_ID \
  --query '{Name:Name,ARN:ARN,Enabled:! DeletedDate}' || exit 1

# 2. Get secret resource policy
echo "2. Checking resource policy..."
aws secretsmanager get-resource-policy \
  --secret-id $SECRET_ID \
  --query 'ResourcePolicy' | jq '.'

# 3. Check if principal is allowed
echo "3. Checking if principal is in resource policy..."
aws secretsmanager get-resource-policy \
  --secret-id $SECRET_ID \
  --query 'ResourcePolicy' | \
  jq --arg arn "$PRINCIPAL_ARN" '.Statement[] | select(.Principal.AWS | contains($arn))'

# 4. Check KMS key permissions
echo "4. Getting KMS key details..."
SECRET_KMS_KEY=$(aws secretsmanager describe-secret \
  --secret-id $SECRET_ID \
  --query 'KmsKeyId' \
  --output text)

echo "KMS Key: $SECRET_KMS_KEY"

# 5. Check KMS key policy
echo "5. Checking KMS key policy..."
aws kms get-key-policy \
  --key-id $SECRET_KMS_KEY \
  --policy-name default | \
  jq '.Statement[] | select(.Principal.AWS | contains("'$PRINCIPAL_ARN'"))'

# 6. Check IAM policy for principal
echo "6. Checking IAM policies attached to principal..."
ROLE_NAME=$(echo $PRINCIPAL_ARN | cut -d'/' -f2)
aws iam list-attached-role-policies \
  --role-name $ROLE_NAME \
  --query 'AttachedPolicies[].PolicyArn' | \
  xargs -I {} aws iam get-policy-version \
    --policy-arn {} \
    --version-id $(aws iam get-policy --policy-arn {} --query 'Policy.DefaultVersionId' --output text) \
    --query 'PolicyVersion.Document' | jq '.Statement[] | select(.Action | contains("secretsmanager"))'

# 7. Check CloudTrail logs
echo "7. Checking recent failed API calls in CloudTrail..."
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=GetSecretValue \
  --max-results 10 \
  --query 'Events[] | .[] | select(.CloudTrailEvent | contains("AccessDenied"))'

# 8. Test access with assumed role
echo "8. Attempting to retrieve secret with assumed role..."
aws sts assume-role \
  --role-arn $PRINCIPAL_ARN \
  --role-session-name troubleshooting \
  --output json > /tmp/assumed-credentials.json

export AWS_ACCESS_KEY_ID=$(jq -r '.Credentials.AccessKeyId' /tmp/assumed-credentials.json)
export AWS_SECRET_ACCESS_KEY=$(jq -r '.Credentials.SecretAccessKey' /tmp/assumed-credentials.json)
export AWS_SESSION_TOKEN=$(jq -r '.Credentials.SessionToken' /tmp/assumed-credentials.json)

aws secretsmanager get-secret-value \
  --secret-id $SECRET_ID \
  2>&1 | tee /tmp/secret-access-result.txt

# 9. Analyze error
echo "9. Error Analysis..."
if grep -q "AccessDenied" /tmp/secret-access-result.txt; then
  echo "❌ AccessDenied - Check all four layers of permissions"
elif grep -q "ResourceNotFoundException" /tmp/secret-access-result.txt; then
  echo "❌ Secret not found - Verify secret ID and region"
elif grep -q "InvalidRequestException" /tmp/secret-access-result.txt; then
  echo "⚠️  Invalid request - Check parameter format"
else
  echo "✅ Secret retrieved successfully"
fi
```

**Common Issues and Solutions**:

| Error | Root Cause | Solution |
|-------|-----------|----------|
| **AccessDenied** | IAM policy missing | Add `secretsmanager:GetSecretValue` to role |
| **AccessDenied** | Resource policy denies | Add principal to secret's resource policy |
| **AccessDenied** | KMS key policy denies | Add role ARN to KMS key policy |
| **ResourceNotFoundException** | Wrong region | Verify secret exists in queried region |
| **ResourceNotFoundException** | Secret deleted | Check if secret is in deletion schedule |
| **DecryptionFailure** | KMS permission missing | Add `kms:Decrypt` to role for KMS key |
| **InvalidSignatureException** | Secret corrupted | Request new secret creation |

---

### Question 5: Design Multi-Region Secrets Strategy for High Availability

**Scenario**: "Design a secrets management strategy that ensures high availability across regions, with automatic failover and compliance with data residency requirements."

**Answer Structure**:

**Multi-Region Architecture**:

```
Primary Region (us-east-1)
┌────────────────────────────────┐
│ AWS Secrets Manager            │
│  - prod/rds/password           │
│  - prod/api/key                │
│  - prod/oauth/token            │
│                                │
│ Replication Enabled ──────────┐│
└────────────────────────────────┘│
                                  │
                 Asynchronous      │
                 Replication       │
                 (seconds)         │
                                  │
Secondary Region (eu-west-1)      │
┌────────────────────────────────┐│
│ AWS Secrets Manager            ││
│  - prod/rds/password (read)    ││
│  - prod/api/key (read)         ││
│  - prod/oauth/token (read)     ││
│                                ││
│ Auto Failover Enabled ◄────────┘│
└────────────────────────────────┘
     (If primary unavailable)

Tertiary Region (ap-southeast-1) - Optional
┌────────────────────────────────┐
│ AWS Secrets Manager            │
│  - Manual Region (manual sync)  │
│  - Compliance: Data residency   │
└────────────────────────────────┘
```

**Multi-Region Replication Configuration**:

```bash
# Enable replica regions for secret
aws secretsmanager replicate-secret-to-regions \
  --secret-id prod/rds/admin \
  --add-replica-regions RegionCode=eu-west-1,KmsKeyId=arn:aws:kms:eu-west-1:...:key/... \
                        RegionCode=ap-southeast-1,KmsKeyId=arn:aws:kms:ap-southeast-1:...:key/...

# Verify replication
aws secretsmanager describe-secret \
  --secret-id prod/rds/admin \
  --query 'ReplicationStatus' | jq '.'

# Output:
# [
#   {
#     "Region": "us-east-1",
#     "RegionReplicationStatus": "Succeeded",
#     "KmsKeyId": "arn:aws:kms:us-east-1:...:key/...",
#     "LastAccessedDate": "2024-01-15T10:30:00.000Z",
#     "LastFailureMessage": null
#   },
#   {
#     "Region": "eu-west-1",
#     "RegionReplicationStatus": "Succeeded",
#     "KmsKeyId": "arn:aws:kms:eu-west-1:...:key/...",
#     "LastAccessedDate": "2024-01-15T10:30:05.000Z",
#     "LastFailureMessage": null
#   },
#   {
#     "Region": "ap-southeast-1",
#     "RegionReplicationStatus": "Succeeded",
#     "KmsKeyId": "arn:aws:kms:ap-southeast-1:...:key/...",
#     "LastAccessedDate": "2024-01-15T10:30:10.000Z",
#     "LastFailureMessage": null
#   }
# ]
```

**Application Multi-Region Failover Logic**:

```python
import boto3
from botocore.exceptions import ClientError
import time

class MultiRegionSecretsClient:
    def __init__(self, secret_id, primary_region, replica_regions):
        self.secret_id = secret_id
        self.primary_region = primary_region
        self.replica_regions = replica_regions
        self.all_regions = [primary_region] + replica_regions
        self.current_region_index = 0
        self.region_health = {region: {'healthy': True, 'last_check': 0} for region in self.all_regions}
    
    def get_secret_with_failover(self, max_retries=3):
        """
        Retrieve secret with automatic failover to replica regions
        """
        
        retries = 0
        while retries < max_retries:
            # Try current region
            current_region = self.all_regions[self.current_region_index]
            
            try:
                secret = self._fetch_from_region(current_region)
                self.region_health[current_region]['healthy'] = True
                return secret
            
            except ClientError as e:
                print(f"Failed to get secret from {current_region}: {e}")
                self.region_health[current_region]['healthy'] = False
                
                # Failover to next region
                self.current_region_index = (self.current_region_index + 1) % len(self.all_regions)
                retries += 1
                time.sleep(1)  # Brief backoff
        
        raise Exception(f"Failed to retrieve secret from all regions after {max_retries} attempts")
    
    def _fetch_from_region(self, region):
        """Fetch secret from specific region"""
        
        client = boto3.client('secretsmanager', region_name=region)
        response = client.get_secret_value(SecretId=self.secret_id)
        return json.loads(response['SecretString'])
    
    def get_healthy_regions(self):
        """Get list of currently healthy regions"""
        return [region for region, health in self.region_health.items() if health['healthy']]

# Usage in application
client = MultiRegionSecretsClient(
    secret_id='prod/rds/admin',
    primary_region='us-east-1',
    replica_regions=['eu-west-1', 'ap-southeast-1']
)

try:
    secret = client.get_secret_with_failover()
    print(f"Retrieved from region: {client.all_regions[client.current_region_index]}")
except Exception as e:
    print(f"All regions failed: {e}")
```

**Monitoring Multi-Region Replication**:

```python
class ReplicationHealthMonitor:
    def __init__(self):
        self.sm = boto3.client('secretsmanager')
        self.cloudwatch = boto3.client('cloudwatch')
    
    def monitor_replication_health(self, secret_id):
        """Monitor replication status across regions"""
        
        secret = self.sm.describe_secret(SecretId=secret_id)
        
        for replica in secret['ReplicationStatus']:
            region = replica['Region']
            status = replica['RegionReplicationStatus']
            
            # Log status
            print(f"{region}: {status}")
            
            # Create CloudWatch metric
            self.cloudwatch.put_metric_data(
                Namespace='SecretsManager',
                MetricData=[
                    {
                        'MetricName': 'ReplicationStatus',
                        'Value': 1 if status == 'Succeeded' else 0,
                        'Dimensions': [
                            {'Name': 'SecretId', 'Value': secret_id},
                            {'Name': 'Region', 'Value': region}
                        ]
                    }
                ]
            )
            
            # Alert on failures
            if status != 'Succeeded' and replica['LastFailureMessage']:
                self._send_alert(
                    secret_id=secret_id,
                    region=region,
                    error=replica['LastFailureMessage']
                )
```

---

## References

[1] AWS Secrets Manager - AWS Documentation. https://docs.aws.amazon.com/secretsmanager/

[2] AWS Systems Manager Parameter Store - AWS Documentation. https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-parameter-store.html

[3] How to Choose Between Secrets Manager and Parameter Store - AWS Security Blog. https://aws.amazon.com/blogs/security/how-to-choose-the-right-aws-service-for-managing-secrets-and-configurations/

[4] AWS Secrets Manager Rotation - AWS Documentation. https://docs.aws.amazon.com/secretsmanager/latest/userguide/rotating-secrets.html

[5] Parameter Store SecureString Encryption - AWS Documentation. https://docs.aws.amazon.com/systems-manager/latest/userguide/parameter-store-about-config-encryption.html

[6] Cross-Account Secret Access - AWS Documentation. https://docs.aws.amazon.com/secretsmanager/latest/userguide/manage-secret-access.html

[7] Multi-Region Secrets Manager - AWS Documentation. https://docs.aws.amazon.com/secretsmanager/latest/userguide/replicate-secrets.html

[8] Secrets Manager Best Practices - AWS Prescriptive Guidance. https://docs.aws.amazon.com/secretsmanager/latest/userguide/best-practices.html

[9] Parameter Store Best Practices - AWS Systems Manager Guide. https://docs.aws.amazon.com/systems-manager/latest/userguide/parameter-store-best-practices.html

[10] IAM Policies for Secrets Manager - AWS Documentation. https://docs.aws.amazon.com/secretsmanager/latest/userguide/auth-and-access_identity-based-policies.html