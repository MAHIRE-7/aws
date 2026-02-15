# 🚀 S3 to S3 Cross-Account Data Transfer (Production Approach)

This document explains how to securely transfer data between **Amazon S3 buckets in different AWS accounts** using **IAM Roles**, without making buckets public or sharing access keys.

---

## 📌 Problem Statement

- Source S3 bucket exists in **Account A**
- Destination S3 bucket exists in **Account B**
- Data must be transferred **securely**
- ❌ No public access
- ❌ No hardcoded credentials
- 
---

## 🧠 Core Concept (Very Important)

> **Cross-account S3 access is achieved using IAM Roles + STS AssumeRole.**

S3 buckets never trust users directly — they trust **IAM identities**.

---



## 🏗️ Architecture Diagram (Single View)
```
    ┌─────────────────────────┐
    │     Account A            │
    │  Source S3 Bucket        │
    │  EC2 / Jenkins / EKS     │
    └───────────┬─────────────┘
                │
                │  sts:AssumeRole
                ▼
    ┌─────────────────────────┐
    │     Account B            │
    │  IAM Role (S3 Access)    │
    │  Destination S3 Bucket  │
    └─────────────────────────┘
```
    
---

## 🔁 End-to-End Flow

1. **Account B** creates IAM Role with trust policy for Account A
2. **Account A** assumes the role using STS
3. **Account A** gets temporary credentials
4. **Account A** transfers data using temporary credentials

---

## ✅ Method Used (Recommended & Interview-Friendly)

### **IAM Role + aws s3 sync**

Best for:
- One-time transfer
- Jenkins pipelines
- EC2 / EKS workloads
- Controlled & auditable access

---

## 🔧 Implementation Steps

### 1️⃣ Account B: Create IAM Role
- Trusted entity: **Another AWS Account**
- Trusted account ID: **Account A**

**Trust Policy**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::<ACCOUNT_A_ID>:root"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

**Permission Policy**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::destination-bucket",
        "arn:aws:s3:::destination-bucket/*"
      ]
    }
  ]
}
```

---

### 2️⃣ Account A: Assume the Role
```bash
aws sts assume-role \
  --role-arn arn:aws:iam::<ACCOUNT_B_ID>:role/S3TransferRole \
  --role-session-name s3-transfer
```

**Export Temporary Credentials**
```bash
export AWS_ACCESS_KEY_ID="<AccessKeyId>"
export AWS_SECRET_ACCESS_KEY="<SecretAccessKey>"
export AWS_SESSION_TOKEN="<SessionToken>"
```

---

### 3️⃣ Transfer Data
```bash
aws s3 sync s3://source-bucket s3://destination-bucket
```

---

## 🔄 Alternative Methods

### Method 2: S3 Cross-Region Replication (CRR)
**Best for**: Continuous sync, disaster recovery

```json
{
  "Role": "arn:aws:iam::<ACCOUNT_A_ID>:role/replication-role",
  "Rules": [
    {
      "Status": "Enabled",
      "Prefix": "",
      "Destination": {
        "Bucket": "arn:aws:s3:::destination-bucket",
        "AccessControlTranslation": {
          "Owner": "Destination"
        },
        "Account": "<ACCOUNT_B_ID>"
      }
    }
  ]
}
```

### Method 3: S3 Batch Operations
**Best for**: Large-scale transfers, complex transformations

---

## 🛠️ Complete Implementation Script

```bash
#!/bin/bash

# Variables
SOURCE_BUCKET="source-bucket"
DEST_BUCKET="destination-bucket"
ACCOUNT_B_ID="123456789012"
ROLE_NAME="S3TransferRole"

# Step 1: Assume role
echo "Assuming role..."
ROLE_CREDS=$(aws sts assume-role \
  --role-arn "arn:aws:iam::${ACCOUNT_B_ID}:role/${ROLE_NAME}" \
  --role-session-name "s3-transfer-$(date +%s)" \
  --output json)

# Step 2: Extract credentials
export AWS_ACCESS_KEY_ID=$(echo $ROLE_CREDS | jq -r '.Credentials.AccessKeyId')
export AWS_SECRET_ACCESS_KEY=$(echo $ROLE_CREDS | jq -r '.Credentials.SecretAccessKey')
export AWS_SESSION_TOKEN=$(echo $ROLE_CREDS | jq -r '.Credentials.SessionToken')

# Step 3: Transfer data
echo "Starting transfer..."
aws s3 sync s3://${SOURCE_BUCKET} s3://${DEST_BUCKET} \
  --delete \
  --exact-timestamps

echo "Transfer completed!"
```

---

## 🔐 Security Best Practices

- ✅ Use IAM Roles instead of access keys
- ✅ Apply least privilege permissions
- ✅ Enable CloudTrail for audit logging
- ✅ Use temporary credentials (STS)
- ✅ Implement bucket policies for additional security
- ✅ Enable S3 server-side encryption

---

## 📊 Monitoring & Logging

### CloudWatch Metrics
- Monitor transfer progress
- Track API call counts
- Set up alarms for failures

### CloudTrail Events
```json
{
  "eventName": "AssumeRole",
  "sourceIPAddress": "<IP>",
  "userIdentity": {
    "type": "IAMUser",
    "principalId": "<PRINCIPAL_ID>",
    "arn": "arn:aws:iam::<ACCOUNT_A_ID>:user/<USERNAME>"
  }
}
```

---

## 🚨 Troubleshooting

### Common Issues

**Access Denied Error**
```bash
An error occurred (AccessDenied) when calling the AssumeRole operation
```
**Solution**: Check trust policy in Account B role

**Invalid Bucket Policy**
```bash
An error occurred (AccessDenied) when calling the PutObject operation
```
**Solution**: Verify S3 permissions in role policy

**Session Token Expired**
```bash
The provided token has expired
```
**Solution**: Re-assume the role to get fresh credentials

---

## 🎯 Production Considerations

- **Cost Optimization**: Use S3 Transfer Acceleration for large transfers
- **Performance**: Implement multipart uploads for large files
- **Reliability**: Add retry logic and error handling
- **Compliance**: Ensure data residency requirements are met
- **Backup**: Maintain source data until transfer verification

---

## 🧠 Interview Questions & Answers

**Q: Why not make S3 buckets public for transfer?**
A: Security risk, violates least privilege, no audit trail

**Q: What's the difference between IAM User and IAM Role?**
A: Users have permanent credentials, roles provide temporary credentials via STS

**Q: How do you handle large file transfers?**
A: Use multipart upload, S3 Transfer Acceleration, or AWS DataSync

---

## 🔄 S3 Bucket Lifecycle Management

### 📌 What is S3 Lifecycle?

> **S3 Lifecycle policies automatically transition or delete objects based on age or criteria.**

Use cases:
- Cost optimization (move old data to cheaper storage)
- Compliance (auto-delete after retention period)
- Data archival (move to Glacier)
- Cleanup (delete incomplete multipart uploads)

---

## 🏗️ S3 Storage Classes (Cost vs Access)

| Storage Class | Cost | Retrieval | Use Case |
|--------------|------|-----------|----------|
| S3 Standard | High | Instant | Frequent access |
| S3 Intelligent-Tier | Auto | Instant | Unknown pattern |
| S3 Standard-IA | Medium | Instant | Infrequent access |
| S3 One Zone-IA | Low | Instant | Non-critical |
| S3 Glacier Instant | Lower | Instant | Archive |
| S3 Glacier Flexible | Lowest | Minutes-Hrs | Long-term |
| S3 Glacier Deep | Cheapest | 12 hours | Compliance |

---

## 🔧 Lifecycle Policy Configuration

### Basic Lifecycle Policy (Transition + Expiration)

```json
{
  "Rules": [
    {
      "Id": "Move-to-IA-then-Glacier",
      "Status": "Enabled",
      "Filter": {
        "Prefix": "logs/"
      },
      "Transitions": [
        {
          "Days": 30,
          "StorageClass": "STANDARD_IA"
        },
        {
          "Days": 90,
          "StorageClass": "GLACIER"
        },
        {
          "Days": 365,
          "StorageClass": "DEEP_ARCHIVE"
        }
      ],
      "Expiration": {
        "Days": 730
      }
    }
  ]
}
```

**Timeline:**
- Day 0-30: S3 Standard
- Day 30-90: S3 Standard-IA
- Day 90-365: Glacier Flexible
- Day 365-730: Glacier Deep Archive
- Day 730+: Deleted

---

## 📋 Common Lifecycle Scenarios

### Scenario 1: Log File Management
```json
{
  "Rules": [
    {
      "Id": "log-lifecycle",
      "Status": "Enabled",
      "Filter": {
        "Prefix": "application-logs/"
      },
      "Transitions": [
        {
          "Days": 7,
          "StorageClass": "STANDARD_IA"
        },
        {
          "Days": 30,
          "StorageClass": "GLACIER"
        }
      ],
      "Expiration": {
        "Days": 90
      }
    }
  ]
}
```

### Scenario 2: Backup Retention
```json
{
  "Rules": [
    {
      "Id": "backup-retention",
      "Status": "Enabled",
      "Filter": {
        "Prefix": "backups/"
      },
      "Transitions": [
        {
          "Days": 1,
          "StorageClass": "GLACIER_IR"
        },
        {
          "Days": 90,
          "StorageClass": "DEEP_ARCHIVE"
        }
      ],
      "Expiration": {
        "Days": 2555
      }
    }
  ]
}
```

### Scenario 3: Cleanup Incomplete Uploads
```bash
aws s3api put-bucket-lifecycle-configuration \
  --bucket my-bucket \
  --lifecycle-configuration '{
    "Rules": [{
      "Id": "cleanup-incomplete-uploads",
      "Status": "Enabled",
      "Filter": {},
      "AbortIncompleteMultipartUpload": {
        "DaysAfterInitiation": 7
      }
    }]
  }'
```

---

## 🛠️ Implementing Lifecycle Policies

### AWS CLI
```bash
# Apply lifecycle policy
aws s3api put-bucket-lifecycle-configuration \
  --bucket my-bucket \
  --lifecycle-configuration file://lifecycle.json

# View lifecycle rules
aws s3api get-bucket-lifecycle-configuration \
  --bucket my-bucket

# Delete lifecycle policy
aws s3api delete-bucket-lifecycle \
  --bucket my-bucket
```

### Terraform
```hcl
resource "aws_s3_bucket_lifecycle_configuration" "example" {
  bucket = aws_s3_bucket.example.id

  rule {
    id     = "archive-rule"
    status = "Enabled"

    filter {
      prefix = "logs/"
    }

    transition {
      days          = 30
      storage_class = "STANDARD_IA"
    }

    transition {
      days          = 90
      storage_class = "GLACIER"
    }

    expiration {
      days = 365
    }
  }
}
```

---

## 💰 Cost Optimization Examples

### Application Logs (90-day retention)
```
Before: 1TB in S3 Standard = $23/month

After Lifecycle:
- 7 days Standard = $4.60
- 23 days Standard-IA = $10.24
- 60 days Glacier = $4.10
Total = ~$19/month (17% savings)
```

### Database Backups (7-year retention)
```
Before: 5TB in S3 Standard = $115/month

After Lifecycle:
- 1 day Standard = $3.45
- 89 days Glacier IR = $18
- Rest in Deep Archive = $5
Total = ~$26/month (77% savings)
```

---

## ⚠️ Important Considerations

### Minimum Storage Duration
- **Standard-IA**: 30 days minimum
- **One Zone-IA**: 30 days minimum
- **Glacier Flexible**: 90 days minimum
- **Deep Archive**: 180 days minimum

**Early deletion = charged for full minimum duration**

### Minimum Object Size
- **Standard-IA/One Zone-IA**: 128KB minimum
- **Glacier**: 40KB minimum

### Valid Transitions
```
Standard → Standard-IA → Glacier → Deep Archive ✅
Standard → Intelligent-Tiering → Glacier ✅
Glacier → Standard ❌ (use restore instead)
```

---

## 🔍 Monitoring Lifecycle Policies

```bash
# List objects by storage class
aws s3api list-objects-v2 \
  --bucket my-bucket \
  --query 'Contents[].{Key:Key,StorageClass:StorageClass,Size:Size}'

# Check storage metrics
aws cloudwatch get-metric-statistics \
  --namespace AWS/S3 \
  --metric-name BucketSizeBytes \
  --dimensions Name=BucketName,Value=my-bucket \
  --start-time 2024-01-01T00:00:00Z \
  --end-time 2024-01-31T23:59:59Z \
  --period 86400 \
  --statistics Average
```

---

## 🎯 Production Best Practices

✅ Start with logs and backups (easy wins)  
✅ Use S3 Intelligent-Tiering for unknown access patterns  
✅ Set up CloudWatch alarms for storage costs  
✅ Test policies on non-production buckets first  
✅ Clean up incomplete multipart uploads (hidden costs)  
✅ Enable versioning with lifecycle for old versions

---

## 🧠 Lifecycle Interview Questions

**Q: What happens if you delete a Glacier object after 30 days?**  
A: You're charged for the full 90-day minimum storage duration

**Q: Can you transition from Glacier back to Standard automatically?**  
A: No, you must manually restore objects. Lifecycle only moves to colder storage

**Q: How do lifecycle policies affect versioned buckets?**  
A: Use NoncurrentVersionTransition and NoncurrentVersionExpiration for old versions

---

## 📚 Additional Resources

- [AWS S3 Cross-Account Access](https://docs.aws.amazon.com/s3/latest/userguide/cross-account-access.html)
- [IAM Roles for Cross-Account Access](https://docs.aws.amazon.com/IAM/latest/UserGuide/tutorial_cross-account-with-roles.html)
- [S3 Transfer Acceleration](https://docs.aws.amazon.com/s3/latest/userguide/transfer-acceleration.html)
- [S3 Lifecycle Configuration](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lifecycle-mgmt.html)
- [S3 Storage Classes](https://aws.amazon.com/s3/storage-classes/)

---

## 📄 License

This project is open source and available under the [MIT License](../LICENSE).