Here are the instructions for Lab 5.

### Lab Overview

  * **Goal:** Configure AWS Config to track resource changes, detect non-compliance (using a Managed Rule), and use **EventBridge** to trigger an email alert via SNS.
  * **Constraints:** Use existing roles (`account-role-1`), use inline policies, and avoid SES.
  * **Architecture:** `Resource Change` → `AWS Config (Records & Evaluates)` → `Compliance Change Event` → `EventBridge Rule` → `SNS Topic` → `Email`.

-----

### Phase 1: Environment & SNS Setup

#### 1\. Set Environment Variables

```bash
export LAB_BUCKET_NAME="lab5-config-history-$(date +%s)"
export TOPIC_NAME="ConfigAlertsTopic"
export CONFIG_ROLE_NAME="account-role-1"
export REGION="us-east-1"
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export EMAIL_ADDRESS="your-email@example.com" # <--- REPLACE THIS
```

#### 2\. Create SNS Topic & Subscription

We create the topic and subscribe your email immediately so you can confirm the subscription link while setting up the rest.

```bash
# Create Topic
export TOPIC_ARN=$(aws sns create-topic --name $TOPIC_NAME --region $REGION --query 'TopicArn' --output text)

# Subscribe Email
aws sns subscribe \
    --topic-arn $TOPIC_ARN \
    --protocol email \
    --notification-endpoint $EMAIL_ADDRESS \
    --region $REGION

echo "Check your inbox for $EMAIL_ADDRESS and confirm the subscription now."
```

#### 3\. Authorize EventBridge to Publish to SNS

Instead of an IAM role, SNS uses a **Resource-Based Policy** to allow EventBridge to push messages.

```bash
cat > sns-policy.json <<EOF
{
  "Version": "2008-10-17",
  "Id": "__default_policy_ID",
  "Statement": [
    {
      "Sid": "AllowEventBridge",
      "Effect": "Allow",
      "Principal": { "Service": "events.amazonaws.com" },
      "Action": "sns:Publish",
      "Resource": "$TOPIC_ARN"
    }
  ]
}
EOF

aws sns set-topic-attributes \
    --topic-arn $TOPIC_ARN \
    --attribute-name Policy \
    --attribute-value file://sns-policy.json \
    --region $REGION
```

-----

### Phase 2: Configure IAM Role for AWS Config

We will repurpose `account-role-1`. AWS Config needs to assume this role to record your resource configuration and write history to S3.

#### 1\. Update Trust Policy (Allow Config Service)

```bash
cat > config-trust.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "config.amazonaws.com" },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

aws iam update-assume-role-policy --role-name $CONFIG_ROLE_NAME --policy-document file://config-trust.json
```

#### 2\. Add Inline Policy (S3 & Config Permissions)

Config needs access to the history bucket and permission to describe resources.

```bash
# Create S3 Bucket for Config History
aws s3api create-bucket --bucket $LAB_BUCKET_NAME --region $REGION

cat > config-inline-policy.json <<EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:PutObject",
                "s3:GetBucketAcl",
                "config:Put*",
                "config:Get*",
                "config:List*",
                "config:Start*",
                "config:Stop*",
                "ec2:Describe*",
                "s3:ListAllMyBuckets"
            ],
            "Resource": "*"
        }
    ]
}
EOF

aws iam put-role-policy \
    --role-name $CONFIG_ROLE_NAME \
    --policy-name ConfigServicePermissions \
    --policy-document file://config-inline-policy.json
```

-----

### Phase 3: Setup AWS Config

#### 1\. Create Configuration Recorder

This tells AWS Config to start tracking resources (we will track **All Supported** resources for this lab).

```bash
aws configservice put-configuration-recorder \
    --configuration-recorder name=default,roleARN=arn:aws:iam::$ACCOUNT_ID:role/$CONFIG_ROLE_NAME \
    --recording-group allSupported=true,includeGlobalResourceTypes=true \
    --region $REGION
```

#### 2\. Create Delivery Channel

This tells Config where to store the history files (S3).

```bash
aws configservice put-delivery-channel \
    --delivery-channel name=default,s3BucketName=$LAB_BUCKET_NAME \
    --region $REGION
```

#### 3\. Start Recording

```bash
aws configservice start-configuration-recorder --configuration-recorder-name default --region $REGION
```

#### 4\. Add a Compliance Rule

We will add a managed rule: **`s3-bucket-public-read-prohibited`**. This rule checks if any S3 bucket allows public read access.

```bash
aws configservice put-config-rule \
    --config-rule "{\"ConfigRuleName\":\"check-s3-public-read\",\"Source\":{\"Owner\":\"AWS\",\"SourceIdentifier\":\"S3_BUCKET_PUBLIC_READ_PROHIBITED\"},\"Scope\":{\"ComplianceResourceTypes\":[\"AWS::S3::Bucket\"]}}" \
    --region $REGION
```

-----

### Phase 4: Configure EventBridge for Alerts

We will create a rule that listens specifically for **Config Compliance Changes**.

#### 1\. Create EventBridge Rule

This pattern matches when a resource's compliance status changes (e.g., from `COMPLIANT` to `NON_COMPLIANT`).

```bash
cat > event-pattern.json <<EOF
{
  "source": ["aws.config"],
  "detail-type": ["Config Rules Compliance Change"],
  "detail": {
    "messageType": ["ComplianceChangeNotification"],
    "configRuleName": ["check-s3-public-read"]
  }
}
EOF

aws events put-rule \
    --name "Capture-Config-NonCompliance" \
    --event-pattern file://event-pattern.json \
    --state ENABLED \
    --region $REGION
```

#### 2\. Add SNS Target

```bash
aws events put-targets \
    --rule "Capture-Config-NonCompliance" \
    --targets "Id"="1","Arn"="$TOPIC_ARN" \
    --region $REGION
```

-----

### Phase 5: Test & Verify

To trigger the alert, we need to create a resource that violates the `s3-bucket-public-read-prohibited` rule.

#### 1\. Create Non-Compliant Resource

Create a bucket with public read access enabled.

```bash
export BAD_BUCKET="bad-bucket-public-$(date +%s)"

# Create bucket
aws s3api create-bucket --bucket $BAD_BUCKET --region $REGION

# Remove Public Access Block (Danger!)
aws s3api delete-public-access-block --bucket $BAD_BUCKET

# Add Public Read Policy
cat > public-policy.json <<EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "PublicRead",
            "Effect": "Allow",
            "Principal": "*",
            "Action": ["s3:GetObject"],
            "Resource": ["arn:aws:s3:::$BAD_BUCKET/*"]
        }
    ]
}
EOF
aws s3api put-bucket-policy --bucket $BAD_BUCKET --policy file://public-policy.json
```

#### 2\. Trigger Evaluation

AWS Config detects changes periodically, but we can force an evaluation.

```bash
aws configservice start-config-rules-evaluation --config-rule-names check-s3-public-read --region $REGION
```

#### 3\. Verification

  * **Wait:** It may take 2-5 minutes for Config to evaluate and EventBridge to fire.
  * **Check Email:** You should receive an email from "AWS Notifications" with JSON details showing `newEvaluationResult` as `NON_COMPLIANT`.

-----

### Lab 5 Cleanup Script

```bash
#!/bin/bash
echo "--- STARTING CLEANUP ---"

# 1. Remove EventBridge Rule & Targets
aws events remove-targets --rule "Capture-Config-NonCompliance" --ids "1" --region $REGION
aws events delete-rule --name "Capture-Config-NonCompliance" --region $REGION

# 2. Stop & Remove AWS Config
aws configservice delete-config-rule --config-rule-name check-s3-public-read --region $REGION
aws configservice stop-configuration-recorder --configuration-recorder-name default --region $REGION
aws configservice delete-delivery-channel --delivery-channel-name default --region $REGION
aws configservice delete-configuration-recorder --configuration-recorder-name default --region $REGION

# 3. Delete SNS Topic
aws sns delete-topic --topic-arn $TOPIC_ARN --region $REGION

# 4. Delete Buckets (History & Bad Bucket)
aws s3 rb s3://$LAB_BUCKET_NAME --force
aws s3 rb s3://$BAD_BUCKET --force

# 5. Clean IAM Role (Remove Inline Policy)
aws iam delete-role-policy --role-name $CONFIG_ROLE_NAME --policy-name ConfigServicePermissions

# Note: We leave the Trust Policy as is, or you can manually revert it if needed.
echo "--- CLEANUP COMPLETE ---"
```

[AWS Config and EventBridge Integration Guide](https://www.google.com/search?q=https://www.youtube.com/watch%3Fv%3Dd_uWz0Rk5FI)
This video provides a visual walkthrough of integrating AWS Config with EventBridge, reinforcing the concepts used in the lab to route compliance notifications.
