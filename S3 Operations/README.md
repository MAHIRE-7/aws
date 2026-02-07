# 🚀 S3 to S3 Cross-Account Data Transfer (Production Approach)

This document explains how to securely transfer data between **Amazon S3 buckets in different AWS accounts** using **IAM Roles**, without making buckets public or sharing access keys.

---

## 📌 Problem Statement

- Source S3 bucket exists in **Account A**
- Destination S3 bucket exists in **Account B**
- Data must be transferred **securely**
- ❌ No public access
- ❌ No hardcoded credentials

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

## 📚 Additional Resources

- [AWS S3 Cross-Account Access](https://docs.aws.amazon.com/s3/latest/userguide/cross-account-access.html)
- [IAM Roles for Cross-Account Access](https://docs.aws.amazon.com/IAM/latest/UserGuide/tutorial_cross-account-with-roles.html)
- [S3 Transfer Acceleration](https://docs.aws.amazon.com/s3/latest/userguide/transfer-acceleration.html)

---

## 📄 License

This project is open source and available under the [MIT License](../LICENSE).