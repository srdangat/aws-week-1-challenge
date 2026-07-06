# AWS GitHub OIDC Challenge

Configure secure GitHub Actions authentication to AWS using OpenID Connect (OIDC), IAM Roles, and AWS Security Token Service (STS) without storing long-lived AWS access keys in GitHub Secrets.

---

## Overview

This project demonstrates how GitHub Actions securely authenticates to AWS using OpenID Connect (OIDC). Instead of storing AWS access keys in GitHub Secrets, GitHub requests a short-lived OIDC token, which AWS validates before issuing temporary credentials through AWS STS.

---

## Objective

* Configure GitHub as an OIDC Identity Provider in AWS IAM.
* Create an IAM Role for GitHub Actions.
* Configure a secure IAM Trust Policy.
* Authenticate GitHub Actions using OIDC.
* Access AWS resources using temporary credentials.
* Eliminate the need for long-lived AWS access keys.

---

## Architecture

```text
GitHub Actions
      │
      │ OIDC Token
      ▼
AWS IAM OIDC Provider
      │
      ▼
AWS STS AssumeRoleWithWebIdentity
      │
Temporary Credentials
      ▼
AWS Resources (S3, EC2, etc.)
```

---

## Prerequisites

* AWS Account
* GitHub Account
* IAM permissions to create:
  * Identity Provider
  * IAM Role
* GitHub Repository

---

## Implementation Steps

### Step 1 — Create the GitHub OIDC Provider

Navigate to:

```text
AWS Console
→ IAM
→ Identity Providers
→ Add Provider
```

Configure the provider:

| Setting       | Value                                         |
| ------------- | --------------------------------------------- |
| Provider Type | OpenID Connect                                |
| Provider URL  | `https://token.actions.githubusercontent.com` |
| Audience      | `sts.amazonaws.com`                           |

Click **Add Provider**.

---

### Step 2 — Create an IAM Role

Navigate to:

```text
AWS Console
→ IAM
→ Roles
→ Create Role
```

Configuration:

* Trusted Entity: **Web Identity**
* Identity Provider: **token.actions.githubusercontent.com**
* Audience: **sts.amazonaws.com**

Attach a permission policy such as:

* `AmazonS3ReadOnlyAccess`

Create the IAM Role.

---

### Step 3 — Create GitHub Repository Variables

Navigate to:

```text
Repository
→ Settings
→ Secrets and variables
→ Actions
→ Variables
```

Create the following repository variables:

| Variable       | Description                    | Example                                                     |
| -------------- | ------------------------------ | ----------------------------------------------------------- |
| `AWS_ROLE_ARN` | IAM Role ARN created in Step 2 | `arn:aws:iam::123456789012:role/github-oidc-challenge-role` |
| `AWS_REGION`   | AWS Region                     | `ap-south-1`                                                |


---

### Step 4 — Configure the IAM Trust Policy

Replace the IAM Role trust relationship with:

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
          "token.actions.githubusercontent.com:sub": "repo:<GITHUB_USERNAME>/<REPOSITORY_NAME>:*"
        }
      }
    }
  ]
}
```

> **Security Tip:** Restrict the trust policy to a specific branch for improved security.

Example:

```text
repo:<GITHUB_USERNAME>/<REPOSITORY_NAME>:ref:refs/heads/main
```

---

### Step 5 — Create the GitHub Actions Workflow

Create the following file:

```text
.github/workflows/aws-oidc-challenge.yml
```

```yaml
name: AWS OIDC Challenge

on:
  workflow_dispatch:

permissions:
  id-token: write
  contents: read

jobs:
  aws-oidc-challenge:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Configure AWS Credentials using OIDC
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ vars.AWS_ROLE_ARN }}
          aws-region: ${{ vars.AWS_REGION }}

      - name: Verify AWS Identity
        run: aws sts get-caller-identity

      - name: Validate S3 Read Access
        run: aws s3 ls
```

---

## Security Benefits

* No long-lived AWS access keys stored in GitHub.
* Uses temporary credentials issued by AWS STS.
* Supports least-privilege IAM permissions.
* Repository-specific IAM trust policy.
* Optional branch-level restrictions.
* Reduces the risk of credential exposure.

---

## AWS Services Used

* AWS Identity and Access Management (IAM)
* OpenID Connect (OIDC)
* AWS Security Token Service (STS)
* Amazon S3
* GitHub Actions

---

## Key Learning Outcomes

* Understand GitHub OIDC authentication with AWS.
* Configure an IAM OIDC Identity Provider.
* Create secure IAM Roles for CI/CD.
* Configure IAM Trust Policies.
* Authenticate GitHub Actions without access keys.
* Use temporary AWS credentials with AWS STS.
* Improve CI/CD security by eliminating long-lived AWS credentials.

---
