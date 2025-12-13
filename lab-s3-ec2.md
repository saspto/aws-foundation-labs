Here are the AWS lab instructions designed to meet your specific networking and security constraints.

### Lab Overview

  * **Goal:** Deploy a private EC2 instance that serves a static website from a secured S3 bucket.
  * **Constraint Checklist:**
      * [x] Private Networking (VPC Endpoints/Private Subnet).
      * [x] Dynamic AMI selection (Latest organizational build).
      * [x] Least Privilege IAM Inline Policy.
      * [x] S3 Static Web Hosting enabled.

### Prerequisites

  * **AWS CLI** installed and configured on your local machine/admin terminal.
  * **VPC ID** and **Private Subnet ID** where the EC2 will reside.
  * **VPC Gateway Endpoint for S3** attached to the VPC (Required for the private EC2 to talk to S3 without a NAT Gateway).

-----

### Phase 1: Infrastructure Setup (Local Terminal)

#### 1\. Set Environment Variables

Set these variables to streamline the commands. Replace the placeholders with your actual IDs.

```bash
export LAB_BUCKET_NAME="my-private-lab-bucket-$(date +%s)"
export LAB_ROLE_NAME="EC2-S3-Lab-Role"
export LAB_REGION="us-east-1"  # Change to your region
export VPC_ID="vpc-xxxxxx"
export SUBNET_ID="subnet-xxxxxx" # Must be a Private Subnet
export VPC_ENDPOINT_ID="vpce-xxxxxx" # ID of the S3 Gateway Endpoint
```

#### 2\. Identify the Latest Organization AMI

This command filters images owned by "self" (your org), sorts by creation date, and grabs the latest Image ID.

```bash
export AMI_ID=$(aws ec2 describe-images \
    --owners self \
    --filters "Name=state,Values=available" \
    --query "Images | sort_by(@, &CreationDate) | [-1].ImageId" \
    --output text \
    --region $LAB_REGION)

echo "Using AMI: $AMI_ID"
```

#### 3\. Create IAM Role and Inline Policy

Create a role that allows the EC2 to assume it, and attach an inline policy restricted to the specific bucket.

**A. Create Trust Policy**

```bash
cat > trust-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "ec2.amazonaws.com" },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

aws iam create-role --role-name $LAB_ROLE_NAME --assume-role-policy-document file://trust-policy.json
```

**B. Create Inline Permissions Policy**

```bash
cat > s3-inline-policy.json <<EOF
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
                "arn:aws:s3:::$LAB_BUCKET_NAME",
                "arn:aws:s3:::$LAB_BUCKET_NAME/*"
            ]
        }
    ]
}
EOF

aws iam put-role-policy --role-name $LAB_ROLE_NAME --policy-name S3AccessPolicy --policy-document file://s3-inline-policy.json
```

**C. Create Instance Profile**

```bash
aws iam create-instance-profile --instance-profile-name $LAB_ROLE_NAME
aws iam add-role-to-instance-profile --instance-profile-name $LAB_ROLE_NAME --role-name $LAB_ROLE_NAME
```

#### 4\. Create and Secure the S3 Bucket

We will create the bucket and apply a policy that **only** allows access from your VPC Endpoint. This satisfies the "Private Networking" requirement while still enabling hosting.

```bash
# Create Bucket
aws s3api create-bucket --bucket $LAB_BUCKET_NAME --region $LAB_REGION

# Disable "Block Public Access" (Required for Website Hosting feature, even if restricted by policy)
aws s3api put-public-access-block \
    --bucket $LAB_BUCKET_NAME \
    --public-access-block-configuration "BlockPublicAcls=false,IgnorePublicAcls=false,BlockPublicPolicy=false,RestrictPublicBuckets=false"

# Apply VPC-Restricted Policy
cat > bucket-policy.json <<EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "Allow-VPC-Only",
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::$LAB_BUCKET_NAME/*",
            "Condition": {
                "StringEquals": {
                    "aws:sourceVpce": "$VPC_ENDPOINT_ID"
                }
            }
        }
    ]
}
EOF

aws s3api put-bucket-policy --bucket $LAB_BUCKET_NAME --policy file://bucket-policy.json
```

#### 5\. Launch EC2 Instance

We launch the instance into the private subnet with the role attached.
*Note: Ensure your Security Group allows inbound SSH or SSM access.*

```bash
aws ec2 run-instances \
    --image-id $AMI_ID \
    --count 1 \
    --instance-type t3.micro \
    --key-name YourKeyPairName \
    --subnet-id $SUBNET_ID \
    --iam-instance-profile Name=$LAB_ROLE_NAME \
    --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=S3-Lab-Instance}]'
```

-----

### Phase 2: Lab Execution (On the EC2 Instance)

Connect to your instance (via SSM Session Manager or Bastion Host).

#### 1\. Install AWS CLI

*Note: If your organizational AMI is Amazon Linux 2 or 2023, AWS CLI is pre-installed. If not:*

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

#### 2\. Create the Hello World Webpage

Since we are in a private network, we will generate the files locally rather than downloading from the internet.

```bash
# Create index.html
echo '<html><head><link rel="stylesheet" type="text/css" href="style.css"></head><body><h1>Hello World from Private EC2!</h1></body></html>' > index.html

# Create style.css
echo 'body { background-color: #f0f0f0; font-family: Arial; color: #333; } h1 { color: #0066cc; }' > style.css
```

#### 3\. Upload to S3

Use the instance's IAM role to upload the files.

```bash
aws s3 cp index.html s3://<YOUR_BUCKET_NAME>/index.html
aws s3 cp style.css s3://<YOUR_BUCKET_NAME>/style.css
```

#### 4\. Enable Web Hosting

Configure the bucket to act as a website.

```bash
aws s3 website s3://<YOUR_BUCKET_NAME>/ --index-document index.html
```

#### 5\. Verify Connectivity (The "Curl" Test)

Since standard S3 Website endpoints (`http://bucket.s3-website...`) resolve publicly, they often fail from a truly private EC2 without a NAT Gateway. However, because we configured the **VPC Endpoint**, we can verify the site using the S3 API path, which routes through the private AWS network backbone.

Run this curl command to verify the file is accessible and the CSS content is served:

```bash
# Verify the HTML content
curl https://<YOUR_BUCKET_NAME>.s3.<YOUR_REGION>.amazonaws.com/index.html

# Verify the CSS content
curl https://<YOUR_BUCKET_NAME>.s3.<YOUR_REGION>.amazonaws.com/style.css
```

**Expected Output:** You should see the raw HTML and CSS code printed to your terminal.

Would you like me to generate a Python script to automate the "Find Latest AMI" step for your automation pipelines?
