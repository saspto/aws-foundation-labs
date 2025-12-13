Here are the instructions for Lab 2.

### Lab Overview

  * **Goal:** Deploy a Python Lambda function inside a private VPC subnet that logs to CloudWatch and writes a file to the private S3 bucket.
  * **Key Challenge:** Running Lambda in a private subnet removes default internet access. To reach AWS services (S3 and CloudWatch Logs), we must ensure the correct VPC Endpoints are in place.

-----

### Phase 1: Networking & IAM Preparation

#### 1\. Set Environment Variables

Reuse the variables from Lab 1, and add a new one for the Lambda function.

```bash
export LAB_BUCKET_NAME="<YOUR_BUCKET_NAME_FROM_LAB_1>"
export VPC_ID="<YOUR_VPC_ID>"
export SUBNET_ID="<YOUR_PRIVATE_SUBNET_ID>" # Same subnet as EC2
export LAMBDA_ROLE_NAME="Lambda-Private-Role"
export REGION="us-east-1"
```

#### 2\. Create/Identify Security Group

The Lambda needs a Security Group to function within the VPC.

```bash
export SECURITY_GROUP_ID=$(aws ec2 create-security-group \
    --group-name LambdaSecurityGroup \
    --description "SG for Private Lambda" \
    --vpc-id $VPC_ID \
    --output text)

echo "Lambda Security Group: $SECURITY_GROUP_ID"
```

#### 3\. Update IAM Role (Existing Role + Inline Policy)

We will assume `Lambda-Private-Role` is the existing service role. We will add an inline policy that covers three specific needs: S3 access, Logging, and **VPC Network Interface creation** (required for private Lambda).

```bash
# 1. Create the Trust Policy (if role doesn't exist yet)
cat > lambda-trust.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "lambda.amazonaws.com" },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF
aws iam create-role --role-name $LAMBDA_ROLE_NAME --assume-role-policy-document file://lambda-trust.json

# 2. Create the Inline Policy
cat > lambda-inline-policy.json <<EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:PutObject",
                "logs:CreateLogGroup",
                "logs:CreateLogStream",
                "logs:PutLogEvents"
            ],
            "Resource": "*"
        }
    ]
}
EOF

# 3. Attach Inline Policy to the Role
aws iam put-role-policy \
    --role-name $LAMBDA_ROLE_NAME \
    --policy-name Lab2-Lambda-Privileges \
    --policy-document file://lambda-inline-policy.json
```

#### 4\. Critical Networking Step: CloudWatch Logs Endpoint

Because your subnet is **private**, the Lambda function cannot reach the public CloudWatch Logs API to write "Hello World". You must create a **VPC Interface Endpoint** for logs.

*(If you skip this, the Lambda will time out trying to send logs).*

```bash
aws ec2 create-vpc-endpoint \
    --vpc-id $VPC_ID \
    --service-name com.amazonaws.$REGION.logs \
    --vpc-endpoint-type Interface \
    --subnet-ids $SUBNET_ID \
    --security-group-ids $SECURITY_GROUP_ID
```

*(Note: Ensure your S3 Gateway Endpoint from Lab 1 is still associated with the Route Table for `$SUBNET_ID`.)*

-----

### Phase 2: Function Development

#### 1\. Create the Python Script

Create a file named `lambda_function.py`.

```python
import json
import boto3
import os

def lambda_handler(event, context):
    # 1. Generate Log Output
    print("Hello world from lambda")
    
    # 2. Define S3 Client
    s3 = boto3.client('s3')
    bucket_name = os.environ['BUCKET_NAME']
    file_name = "lambda_artifact.txt"
    file_content = "This file was created by a private Lambda function."
    
    try:
        # 3. Upload to S3
        response = s3.put_object(
            Bucket=bucket_name,
            Key=file_name,
            Body=file_content
        )
        return {
            'statusCode': 200,
            'body': json.dumps(f"File uploaded successfully to {bucket_name}")
        }
    except Exception as e:
        print(f"Error: {e}")
        return {
            'statusCode': 500,
            'body': json.dumps("Error uploading file")
        }
```

#### 2\. Package the Function

Zip the python file.

```bash
zip function.zip lambda_function.py
```

-----

### Phase 3: Deployment & Verification

#### 1\. Create the Lambda Function

We configure the function to connect to the private subnet using the Security Group created earlier.

```bash
aws lambda create-function \
    --function-name PrivateLabFunction \
    --runtime python3.9 \
    --zip-file fileb://function.zip \
    --handler lambda_function.lambda_handler \
    --role arn:aws:iam::$(aws sts get-caller-identity --query Account --output text):role/$LAMBDA_ROLE_NAME \
    --timeout 15 \
    --vpc-config SubnetIds=$SUBNET_ID,SecurityGroupIds=$SECURITY_GROUP_ID \
    --environment Variables={BUCKET_NAME=$LAB_BUCKET_NAME}
```

#### 2\. Invoke the Function

Trigger the function manually.

```bash
aws lambda invoke --function-name PrivateLabFunction response.json
```

#### 3\. Verification

**A. Check the Response**
Check the `response.json` file. You should see a status code of 200.

```bash
cat response.json
```

**B. Verify S3 Upload**
Check if the file `lambda_artifact.txt` exists in your bucket.

```bash
aws s3 ls s3://$LAB_BUCKET_NAME/
```

**C. Verify CloudWatch Logs**
Since we are in a private network, use the CLI to fetch the latest logs.

```bash
# Get the Log Group Name
LOG_GROUP="/aws/lambda/PrivateLabFunction"

# Find the latest Log Stream
STREAM_NAME=$(aws logs describe-log-streams \
    --log-group-name $LOG_GROUP \
    --order-by LastEventTime \
    --descending \
    --limit 1 \
    --query "logStreams[0].logStreamName" \
    --output text)

# Get the Log Events
aws logs get-log-events \
    --log-group-name $LOG_GROUP \
    --log-stream-name $STREAM_NAME \
    --query "events[*].message"
```

**Expected Output:** You should see `"Hello world from lambda"` in the log output.
