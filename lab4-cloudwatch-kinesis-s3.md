Here are the instructions for Lab 4.

### Lab Overview

  * **Goal:** Create a pipeline that streams CloudWatch Logs to an S3 bucket for long-term storage and daily organization.
  * **Architecture:** `CloudWatch Logs (Source)` → `Subscription Filter` → `Kinesis Data Firehose` → `S3 Bucket`.
      * *Note:* We use **Kinesis Data Firehose** (now Amazon Data Firehose) because it is the specific Kinesis service designed to "ingest to S3" with automatic time-based partitioning (Daily/Hourly).
  * **Key Concept:** CloudWatch Subscription Filters allow real-time scanning of logs. Only logs matching your filter are sent to the Kinesis stream.

-----

### Phase 1: Infrastructure Setup (S3 & IAM)

Here are the updated Lab 4 instructions.

### Lab Roles

  * **Constraint:** Use existing IAM roles (`account-role-1` and `account-role-2`) instead of creating new ones.
  * **Strategy:** We will **update the Trust Relationship** of these roles to allow the AWS services (Firehose and CloudWatch) to assume them, and attach **Inline Policies** for the specific permissions required.
  * **Role Assignment:**
      * **`account-role-1`**: Used by **CloudWatch Logs** to push data to Firehose.
      * **`account-role-2`**: Used by **Kinesis Firehose** to write data to S3.

-----

### Phase 1: Environment & S3 Setup

#### 1\. Set Environment Variables

```bash
export LAB_BUCKET_NAME="lab4-logs-archive-$(date +%s)"
export FIREHOSE_NAME="MyLogIngestionStream"
# Using EXISTING roles as requested
export CW_LOG_ROLE="account-role-1"
export FH_S3_ROLE="account-role-2"
export REGION="us-east-1"
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
```

#### 2\. Create the Archive S3 Bucket

```bash
aws s3api create-bucket --bucket $LAB_BUCKET_NAME --region $REGION
```

-----

### Phase 2: Configure "Firehose to S3" Role (`account-role-2`)

We must configure `account-role-2` so Kinesis Firehose can assume it and write to your new bucket.

#### 1\. Update Trust Policy (Allow Firehose to Assume)

*Warning: This overwrites the existing trust policy for this lab session.*

```bash
cat > firehose-trust.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "firehose.amazonaws.com" },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

aws iam update-assume-role-policy --role-name $FH_S3_ROLE --policy-document file://firehose-trust.json
```

#### 2\. Add Inline Permission Policy (Allow S3 Write)

```bash
cat > firehose-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
          "s3:AbortMultipartUpload",
          "s3:GetBucketLocation",
          "s3:GetObject",
          "s3:ListBucket",
          "s3:ListBucketMultipartUploads",
          "s3:PutObject"
      ],
      "Resource": [
          "arn:aws:s3:::$LAB_BUCKET_NAME",
          "arn:aws:s3:::$LAB_BUCKET_NAME/*"
      ]
    }
  ]
}
EOF

aws iam put-role-policy --role-name $FH_S3_ROLE --policy-name S3AccessInline --policy-document file://firehose-policy.json
```

-----

### Phase 3: Create Kinesis Firehose

Now we create the stream using the configured existing role.

```bash
# Get the ARN of account-role-2
export FH_ROLE_ARN=$(aws iam get-role --role-name $FH_S3_ROLE --query 'Role.Arn' --output text)

aws firehose create-delivery-stream \
    --delivery-stream-name $FIREHOSE_NAME \
    --delivery-stream-type DirectPut \
    --s3-destination-configuration \
    "RoleARN=$FH_ROLE_ARN,BucketARN=arn:aws:s3:::$LAB_BUCKET_NAME,BufferingHints={SizeInMBs=1,IntervalInSeconds=60},Prefix=daily-logs/YYYY/MM/DD/,ErrorOutputPrefix=errors/"
```

-----

### Phase 4: Configure "CloudWatch to Firehose" Role (`account-role-1`)

We configure `account-role-1` so CloudWatch Logs can assume it and push data to the Firehose stream we just created.

#### 1\. Update Trust Policy (Allow CloudWatch Logs to Assume)

```bash
cat > cw-trust.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "logs.$REGION.amazonaws.com" },
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringLike": { "aws:SourceArn": "arn:aws:logs:$REGION:$ACCOUNT_ID:*" }
      }
    }
  ]
}
EOF

aws iam update-assume-role-policy --role-name $CW_LOG_ROLE --policy-document file://cw-trust.json
```

#### 2\. Add Inline Permission Policy (Allow Firehose Put)

```bash
# Get Firehose ARN
export FH_ARN=$(aws firehose describe-delivery-stream --delivery-stream-name $FIREHOSE_NAME --query 'DeliveryStreamDescription.DeliveryStreamARN' --output text)

cat > cw-policy.json <<EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": ["firehose:PutRecord", "firehose:PutRecordBatch"],
            "Resource": "$FH_ARN"
        }
    ]
}
EOF

aws iam put-role-policy --role-name $CW_LOG_ROLE --policy-name FirehosePutInline --policy-document file://cw-policy.json
```

-----

### Phase 5: Create Subscription Filter & Test

#### 1\. Create Log Group & Subscription

```bash
export TEST_LOG_GROUP="Lab4-App-Logs"
export CW_ROLE_ARN=$(aws iam get-role --role-name $CW_LOG_ROLE --query 'Role.Arn' --output text)

# Create Log Group
aws logs create-log-group --log-group-name $TEST_LOG_GROUP

# Create Subscription Filter
aws logs put-subscription-filter \
    --log-group-name $TEST_LOG_GROUP \
    --filter-name AllLogsToKinesis \
    --filter-pattern "" \
    --destination-arn $FH_ARN \
    --role-arn $CW_ROLE_ARN
```

#### 2\. Generate Data & Verify

```bash
# Create Log Stream
aws logs create-log-stream --log-group-name $TEST_LOG_GROUP --log-stream-name "TestStream1"

# Put Logs
aws logs put-log-events \
    --log-group-name $TEST_LOG_GROUP \
    --log-stream-name "TestStream1" \
    --log-events timestamp=$(date +%s)000,message="[INFO] Using Existing Roles Test"
```

**Verify:** Wait \~60 seconds, then check S3:

```bash
aws s3 ls s3://$LAB_BUCKET_NAME --recursive
```

-----

### Phase 6: Cleanup Script (Preserving Roles)

This script cleans up the resources but **does not delete** the pre-existing roles. It only removes the inline policies we added.

```bash
#!/bin/bash

echo "--- STARTING CLEANUP (PRESERVING ROLES) ---"

# 1. Delete Subscription Filter & Log Group
aws logs delete-subscription-filter --log-group-name $TEST_LOG_GROUP --filter-name AllLogsToKinesis
aws logs delete-log-group --log-group-name $TEST_LOG_GROUP

# 2. Delete Firehose Stream
aws firehose delete-delivery-stream --delivery-stream-name $FIREHOSE_NAME

# 3. Delete S3 Bucket
aws s3 rb s3://$LAB_BUCKET_NAME --force

# 4. Detach Inline Policies from Existing Roles
echo "Removing inline policies from $CW_LOG_ROLE and $FH_S3_ROLE..."
aws iam delete-role-policy --role-name $CW_LOG_ROLE --policy-name FirehosePutInline
aws iam delete-role-policy --role-name $FH_S3_ROLE --policy-name S3AccessInline

# Note: We are NOT deleting the roles ($CW_LOG_ROLE / $FH_S3_ROLE) 
# nor reverting the Trust Policy, as that is complex to rollback automatically.
# They will remain configured to trust CloudWatch/Firehose until manually reset.

echo "--- CLEANUP COMPLETE ---"
```
