# Automated S3 Bucket (CloudFormation + GitHub Actions)

This repository deploys one secure S3 bucket using CloudFormation when code is pushed to `main`.

## What gets deployed

- One S3 bucket from `bucket.yaml`.
- Bucket name is read from SSM Parameter Store path `/my-app/bucket-name`.
- Baseline security enabled:
- Public access blocked.
- AES256 server-side encryption.
- Versioning enabled.

## Prerequisites

- AWS CLI installed and authenticated.
- GitHub repository access.
- AWS account where the stack will be deployed.

## 1) IAM user for local CLI (learning path)

For learning purposes only, create an IAM user with programmatic access and attach `AdministratorAccess`.

Then configure AWS CLI locally:

```powershell
aws configure
```

Use:

- Access key ID from IAM user
- Secret access key from IAM user
- Default region: `us-east-1`
- Output format: `json`

Validate your identity:

```powershell
aws sts get-caller-identity
```

## 2) Create the SSM parameter for bucket name

Choose a globally unique bucket name, then create/update the parameter:

```powershell
$bucketName = "my-secure-bucket-<unique-suffix>"

aws ssm put-parameter \
  --name "/my-app/bucket-name" \
  --type String \
  --value $bucketName \
  --overwrite \
  --region us-east-1
```

Confirm value:

```powershell
aws ssm get-parameter \
  --name "/my-app/bucket-name" \
  --region us-east-1
```

## 3) Deploy locally once (baseline check)

```powershell
aws cloudformation deploy \
  --template-file bucket.yaml \
  --stack-name MySecureStack \
  --region us-east-1 \
  --no-fail-on-empty-changeset
```

## 4) Configure GitHub OIDC role (recommended)

The workflow uses OIDC, so no static AWS keys are stored in GitHub.

### Trust policy example

Replace `<ACCOUNT_ID>`, `<OWNER>`, and `<REPO>`.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:<OWNER>/<REPO>:ref:refs/heads/main"
        }
      }
    }
  ]
}
```

### Permissions policy (starter)

Use least privilege in production. This starter policy is enough for this challenge:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "cloudformation:CreateStack",
        "cloudformation:UpdateStack",
        "cloudformation:DescribeStacks",
        "cloudformation:DescribeStackEvents",
        "cloudformation:GetTemplateSummary",
        "cloudformation:ValidateTemplate"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:CreateBucket",
        "s3:PutBucketPublicAccessBlock",
        "s3:PutEncryptionConfiguration",
        "s3:PutBucketVersioning",
        "s3:GetBucketPublicAccessBlock",
        "s3:GetEncryptionConfiguration",
        "s3:GetBucketVersioning"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "ssm:GetParameter",
        "ssm:GetParameters"
      ],
      "Resource": "arn:aws:ssm:us-east-1:<ACCOUNT_ID>:parameter/my-app/bucket-name"
    }
  ]
}
```

## 5) Add GitHub secret

In the GitHub repository settings, add:

- Secret name: `AWS_ROLE_TO_ASSUME`
- Value: ARN of the IAM role trusted for GitHub OIDC

## 6) Push to main to trigger CI deploy

The workflow at `.github/workflows/deploy.yml` runs automatically on push to `main` and executes:

1. Template validation
2. Stack deployment for `MySecureStack`

## Troubleshooting

- Bucket name already taken: update `/my-app/bucket-name` with a new globally unique value.
- Access denied in workflow: verify OIDC trust policy subject and role permissions.
- No changes to deploy: expected if the stack is already up to date.
