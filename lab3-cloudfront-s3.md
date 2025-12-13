Here are the instructions for Lab 3.

### Lab Overview

  * **Goal:** Serve a private S3 bucket via CloudFront using a custom domain resolved within a Private VPC.
  * **Concepts:** CloudFront Origin Access Identity (OAI), Private Route 53 Hosted Zones, ACM Import (Self-Signed), and S3 Bucket Policies.
  * **Critical Network Requirement:** While the DNS resolution is private, **CloudFront is a public service**. Your testing EC2 instance inside the VPC **must have outbound internet access** (via a NAT Gateway or Internet Gateway) to reach the CloudFront edge locations. The VPC Endpoints from Lab 1 are not sufficient for CloudFront traffic.

-----

### Phase 1: Preparation & S3 Setup

#### 1\. Set Environment Variables

```bash
export LAB_BUCKET_NAME="lab3-cloudfront-source-$(date +%s)"
export DOMAIN_NAME="secure.internal.lab"  # The custom private domain
export REGION="us-east-1"                 # CloudFront certs MUST be in us-east-1
export VPC_ID="<YOUR_VPC_ID>"             # From previous labs
```

#### 2\. Create and Lock Down S3 Bucket

We create the bucket and block all public access immediately.

```bash
# Create Bucket
aws s3api create-bucket --bucket $LAB_BUCKET_NAME --region $REGION

# Block Public Access (Strict)
aws s3api put-public-access-block \
    --bucket $LAB_BUCKET_NAME \
    --public-access-block-configuration "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"
```

#### 3\. Upload Sample Website

```bash
echo "<html><body><h1>Hello from CloudFront via Private DNS!</h1></body></html>" > index.html
aws s3 cp index.html s3://$LAB_BUCKET_NAME/index.html
```

-----

### Phase 2: Security (Certificates & OAI)

#### 1\. Generate Self-Signed Certificate

CloudFront requires the certificate to support the specific domain name (`secure.internal.lab`).

```bash
# Generate Private Key
openssl genrsa -out private-key.pem 2048

# Generate Certificate (Valid for 365 days)
openssl req -new -x509 -nodes -days 365 \
    -key private-key.pem \
    -out certificate.pem \
    -subj "/C=US/ST=VA/L=LintonHall/O=MyOrg/CN=$DOMAIN_NAME"
```

#### 2\. Import Certificate to ACM

**Crucial:** CloudFront only accepts certificates from the **us-east-1** region (N. Virginia), even if your infrastructure is elsewhere.

```bash
export CERT_ARN=$(aws acm import-certificate \
    --certificate fileb://certificate.pem \
    --private-key fileb://private-key.pem \
    --region us-east-1 \
    --query 'CertificateArn' --output text)

echo "Certificate ARN: $CERT_ARN"
```

#### 3\. Create Origin Access Identity (OAI)

This special user identity allows CloudFront to read from your bucket while keeping the bucket private from everyone else.

```bash
export OAI_ID=$(aws cloudfront create-cloud-front-origin-access-identity \
    --cloud-front-origin-access-identity-config CallerReference="lab3-oai-$(date +%s)",Comment="Lab3 OAI" \
    --query 'CloudFrontOriginAccessIdentity.Id' --output text)

echo "OAI ID: $OAI_ID"
```

-----

### Phase 3: CloudFront Distribution

#### 1\. Create Distribution Config

Creating a distribution with aliases and custom certs via CLI is complex. We will generate a JSON skeleton.

```bash
cat > dist-config.json <<EOF
{
  "CallerReference": "lab3-dist-$(date +%s)",
  "Aliases": {
    "Quantity": 1,
    "Items": ["$DOMAIN_NAME"]
  },
  "DefaultRootObject": "index.html",
  "Origins": {
    "Quantity": 1,
    "Items": [
      {
        "Id": "S3-Origin",
        "DomainName": "$LAB_BUCKET_NAME.s3.amazonaws.com",
        "S3OriginConfig": {
          "OriginAccessIdentity": "origin-access-identity/cloudfront/$OAI_ID"
        }
      }
    ]
  },
  "DefaultCacheBehavior": {
    "TargetOriginId": "S3-Origin",
    "ViewerProtocolPolicy": "redirect-to-https",
    "MinTTL": 0,
    "AllowedMethods": {
      "Quantity": 2,
      "Items": ["HEAD", "GET"],
      "CachedMethods": { "Quantity": 2, "Items": ["HEAD", "GET"] }
    },
    "ForwardedValues": {
      "QueryString": false,
      "Cookies": { "Forward": "none" }
    },
    "TrustedSigners": { "Enabled": false, "Quantity": 0 }
  },
  "CacheBehaviors": { "Quantity": 0 },
  "ViewerCertificate": {
    "ACMCertificateArn": "$CERT_ARN",
    "SSLSupportMethod": "sni-only",
    "MinimumProtocolVersion": "TLSv1.2_2021"
  },
  "Enabled": true,
  "Comment": "Lab 3 Private DNS Distribution"
}
EOF
```

#### 2\. Deploy Distribution

```bash
export DIST_ID=$(aws cloudfront create-distribution-with-tags \
    --distribution-config-with-tags "DistributionConfig=$(cat dist-config.json),Tags={Items=[{Key=Name,Value=Lab3-Dist}]}" \
    --query 'Distribution.Id' --output text)

export CF_DOMAIN=$(aws cloudfront get-distribution --id $DIST_ID --query 'Distribution.DomainName' --output text)

echo "Creating Distribution: $DIST_ID"
echo "CloudFront Domain: $CF_DOMAIN"
echo "Wait (~5-10 mins) for Status to change from InProgress to Deployed."
```

-----

### Phase 4: Permissions & Networking

#### 1\. Update S3 Bucket Policy

Now that we have the OAI and Bucket Name, we permit *only* the OAI to read the files.

```bash
cat > s3-oai-policy.json <<EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowCloudFrontOAI",
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::cloudfront:user/CloudFront Origin Access Identity $OAI_ID"
            },
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::$LAB_BUCKET_NAME/*"
        }
    ]
}
EOF

aws s3api put-bucket-policy --bucket $LAB_BUCKET_NAME --policy file://s3-oai-policy.json
```

#### 2\. Create Private Route 53 Zone

This creates a DNS zone that only exists inside your VPC.

```bash
# Create the Private Zone
export ZONE_ID=$(aws route53 create-hosted-zone \
    --name internal.lab \
    --vpc VPCRegion=$REGION,VPCId=$VPC_ID \
    --caller-reference "lab3-zone-$(date +%s)" \
    --hosted-zone-config Comment="Private Lab Zone" \
    --query 'HostedZone.Id' --output text)

# Create the DNS Record (Alias to CloudFront)
cat > dns-record.json <<EOF
{
  "Comment": "Alias to CloudFront",
  "Changes": [
    {
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "$DOMAIN_NAME",
        "Type": "A",
        "AliasTarget": {
          "HostedZoneId": "Z2FDTNDATAQYW2", 
          "DNSName": "$CF_DOMAIN",
          "EvaluateTargetHealth": false
        }
      }
    }
  ]
}
EOF
# Note: Z2FDTNDATAQYW2 is the fixed Hosted Zone ID for all CloudFront distributions.

aws route53 change-resource-record-sets --hosted-zone-id $ZONE_ID --change-batch file://dns-record.json
```

-----

### Phase 5: Verification (From EC2)

Connect to your EC2 instance inside the VPC.

**Note:** The EC2 must have internet access (NAT Gateway) to run this test, because it must reach `secure.internal.lab` -\> CloudFront IP (Public Internet).

```bash
# Test 1: Curl the domain
# We use -k (insecure) because curl doesn't trust our self-signed CA.
# We use -L to follow redirects (HTTP -> HTTPS if configured, though we set redirect-to-https).

curl -k -L https://secure.internal.lab/index.html
```

**Expected Output:** `<h1>Hello from CloudFront via Private DNS!</h1>`

### Lab 3 Cleanup Script

```bash
#!/bin/bash
# 1. Delete Route 53 Record
aws route53 change-resource-record-sets --hosted-zone-id $ZONE_ID --change-batch '{"Changes":[{"Action":"DELETE","ResourceRecordSet":{"Name":"'"$DOMAIN_NAME"'","Type":"A","AliasTarget":{"HostedZoneId":"Z2FDTNDATAQYW2","DNSName":"'"$CF_DOMAIN"'","EvaluateTargetHealth":false}}}]}'

# 2. Delete Hosted Zone
aws route53 delete-hosted-zone --id $ZONE_ID

# 3. Disable CloudFront (Required before delete)
ETAG=$(aws cloudfront get-distribution --id $DIST_ID --query 'ETag' --output text)
aws cloudfront update-distribution --id $DIST_ID --if-match $ETAG --distribution-config "$(aws cloudfront get-distribution-config --id $DIST_ID --query 'DistributionConfig' | jq '.Enabled=false')" 

echo "Distribution disabled. Wait for 'Deployed' status before deleting manually."

# 4. Delete OAI
ETAG_OAI=$(aws cloudfront get-cloud-front-origin-access-identity --id $OAI_ID --query 'ETag' --output text)
aws cloudfront delete-cloud-front-origin-access-identity --id $OAI_ID --if-match $ETAG_OAI

# 5. Delete Bucket
aws s3 rb s3://$LAB_BUCKET_NAME --force

# 6. Delete Cert
aws acm delete-certificate --certificate-arn $CERT_ARN --region us-east-1
```
