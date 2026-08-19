# GitHub Actions OIDC – Immutable Subject Claims with AWS IAM

Hands-on demonstration of GitHub Actions authentication to AWS using **OpenID Connect (OIDC)**, **AWS IAM**, and **AWS Security Token Service (STS)** with GitHub's **immutable `sub` claim format**.

**Repository:** https://github.com/Jaishree97/github-actions-oidc-demo

---

## Overview

GitHub introduced **immutable subject (`sub`) claims** for GitHub Actions OIDC tokens.

Repositories using the immutable subject format include the **GitHub owner ID** and **repository ID** in the OIDC `sub` claim.

This changes how AWS IAM trust policies must identify GitHub Actions workflows.

This project demonstrates how to:

- Configure GitHub as an AWS IAM OIDC identity provider.
- Create an IAM role for GitHub Actions.
- Configure an IAM trust policy using immutable subject claims.
- Authenticate GitHub Actions to AWS without long-lived access keys.
- Obtain temporary AWS credentials through AWS STS.
- Verify the assumed IAM role.
- Validate Amazon S3 read access.

> **Important:** This repository was created in **August 2026**, after GitHub's **July 15, 2026** change.  
> Therefore, this lab uses the immutable GitHub OIDC subject claim format.

---

## Objective

- Understand GitHub Actions OIDC authentication.
- Understand immutable OIDC subject (`sub`) claims.
- Compare legacy and immutable subject formats.
- Configure an AWS IAM OIDC identity provider.
- Configure an IAM role trust policy using immutable IDs.
- Authenticate GitHub Actions using OIDC.
- Use temporary AWS credentials through AWS STS.
- Access AWS resources without storing long-lived AWS access keys.

---

## Architecture

```text
GitHub Actions
      │
      │ OIDC Token
      ▼
GitHub OIDC Provider
      │
      ▼
AWS IAM Trust Policy
      │
      ▼
AWS STS
AssumeRoleWithWebIdentity
      │
      ▼
Temporary AWS Credentials
      │
      ▼
Amazon S3
```

---

## Legacy vs Immutable Subject Claims

### Legacy Subject Format

```text
repo:<OWNER>/<REPOSITORY>:ref:refs/heads/main
```

Example:

```text
repo:octocat/my-app:ref:refs/heads/main
```

### Immutable Subject Format

```text
repo:<OWNER>@<OWNER_ID>/<REPOSITORY>@<REPOSITORY_ID>:ref:refs/heads/main
```

Example:

```text
repo:octocat@12345678/my-app@987654321:ref:refs/heads/main
```

### Why the Change Matters

An IAM trust policy written only for the legacy subject format may fail when GitHub sends an immutable `sub` claim.

A typical authentication failure is:

```text
Not authorized to perform sts:AssumeRoleWithWebIdentity
```

The trust policy must match the subject claim actually included in the GitHub OIDC token.

---

## Repository Details

| Setting | Value |
|---|---|
| Repository | `Jaishree97/github-actions-oidc-demo` |
| Visibility | Public |
| Default Branch | `main` |
| OIDC Provider | `token.actions.githubusercontent.com` |
| AWS Audience | `sts.amazonaws.com` |
| AWS Permission | `AmazonS3ReadOnlyAccess` |

---

# Implementation

## Step 1 — GitHub Repository

The repository used for this hands-on lab:

```text
Repository: github-actions-oidc-demo
Owner: Jaishree97
Visibility: Public
Default branch: main
```

Repository:

https://github.com/Jaishree97/github-actions-oidc-demo

> The repository was created in **August 2026**, after the **July 15, 2026** GitHub immutable subject claims change.

Therefore, the IAM trust policy uses the immutable subject format.

---

## Step 2 — Create GitHub OIDC Identity Provider

Navigate to:

```text
AWS Console
→ IAM
→ Identity Providers
→ Add Provider
```

Configure:

| Setting | Value |
|---|---|
| Provider Type | OpenID Connect |
| Provider URL | `https://token.actions.githubusercontent.com` |
| Audience | `sts.amazonaws.com` |

The provider allows AWS IAM to trust OIDC tokens issued by GitHub Actions.

---

## Step 3 — Create IAM Role

Navigate to:

```text
AWS Console
→ IAM
→ Roles
→ Create Role
```

Configure:

```text
Trusted entity: Web Identity
Identity Provider: token.actions.githubusercontent.com
Audience: sts.amazonaws.com
```

Example role name:

```text
github-oidc-challenge-role
```

Attach:

```text
AmazonS3ReadOnlyAccess
```

Role description:

```text
IAM role for GitHub Actions OIDC authentication with Amazon S3 read-only access.
```

---

## Step 4 — Configure GitHub Repository Secret

Navigate to:

```text
GitHub Repository
→ Settings
→ Secrets and variables
→ Actions
```

Create a repository secret:

```text
Name: AWS_ROLE_ARN
```

Example value:

```text
arn:aws:iam::<AWS_ACCOUNT_ID>:role/github-oidc-challenge-role
```

The role ARN is stored as a GitHub repository secret instead of being hardcoded in the workflow.

---

## Step 5 — Configure GitHub Repository Variable

Create a repository variable:

```text
Name: AWS_REGION
Value: us-east-1
```

The workflow reads the AWS region using:

```yaml
aws-region: ${{ vars.AWS_REGION }}
```

---

## Step 6 — Find GitHub Owner ID and Repository ID

The immutable subject requires:

- GitHub Owner ID
- GitHub Repository ID

These can be retrieved from GitHub repository metadata.

Example:

```bash
curl https://api.github.com/repos/<OWNER>/<REPOSITORY>
```

Example:

```bash
curl https://api.github.com/repos/Jaishree97/github-actions-oidc-demo
```

The response contains:

```json
{
  "id": 1338257986,
  "owner": {
    "login": "Jaishree97",
    "id": 222460494
  }
}
```

Required values:

| Value | Example |
|---|---|
| Owner | `Jaishree97` |
| Owner ID | `222460494` |
| Repository | `github-actions-oidc-demo` |
| Repository ID | `1338257986` |
| Branch | `main` |

---

## Step 7 — Immutable OIDC Subject

For this repository:

```text
Owner: Jaishree97
Owner ID: 222460494
Repository: github-actions-oidc-demo
Repository ID: 1338257986
Branch: main
```

The immutable subject is:

```text
repo:Jaishree97@222460494/github-actions-oidc-demo@1338257986:ref:refs/heads/main
```

The structure is:

```text
repo:<OWNER>@<OWNER_ID>/<REPOSITORY>@<REPOSITORY_ID>:ref:refs/heads/main
```

---

## Step 8 — Configure IAM Trust Policy

Replace the IAM role trust relationship with:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<AWS_ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:Jaishree97@222460494/github-actions-oidc-demo@1338257986:ref:refs/heads/main"
        }
      }
    }
  ]
}
```

### Important

The `aud` claim belongs under:

```json
"StringEquals"
```

The immutable `sub` claim belongs under:

```json
"StringLike"
```

Do **not** duplicate the `aud` condition under `StringLike`.

### Security Benefit

The trust policy restricts role assumption to:

```text
Jaishree97
        ↓
Owner ID: 222460494
        ↓
github-actions-oidc-demo
        ↓
Repository ID: 1338257986
        ↓
main branch
```

This creates a repository- and branch-specific trust relationship.

---

## Step 9 — GitHub Actions Workflow

Workflow file:

```text
.github/workflows/aws-oidc-challenge.yml
```

```yaml
name: GitHub OIDC Immutable Subject Claims

on:
  workflow_dispatch:

permissions:
  id-token: write
  contents: read

jobs:
  oidc-immutable-demo:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Configure AWS Credentials using OIDC
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          role-session-name: GitHubActions
          aws-region: ${{ vars.AWS_REGION }}

      - name: Verify AWS Identity
        run: aws sts get-caller-identity

      - name: Validate S3 Read Access
        run: aws s3 ls
```

---

## Step 10 — Run the Workflow

Navigate to:

```text
GitHub Repository
→ Actions
→ GitHub OIDC Immutable Subject Claims
→ Run workflow
```

Select:

```text
Branch: main
```

The workflow performs:

1. Checkout repository.
2. Request an OIDC token.
3. Authenticate to AWS using OIDC.
4. Assume the IAM role through AWS STS.
5. Obtain temporary AWS credentials.
6. Verify the AWS identity.
7. Validate S3 read access.

---

## Step 11 — Verify AWS Identity

Open the successful workflow run.

Expand:

```text
Verify AWS Identity
```

Command:

```bash
aws sts get-caller-identity
```

Expected output:

```json
{
  "UserId": "AROAXXXXXXXXXXXXX:GitHubActions",
  "Account": "<AWS_ACCOUNT_ID>",
  "Arn": "arn:aws:sts::<AWS_ACCOUNT_ID>:assumed-role/github-oidc-challenge-role/GitHubActions"
}
```

The `assumed-role` ARN confirms that GitHub Actions successfully assumed the IAM role through AWS STS.

---

## Step 12 — Validate S3 Access

The IAM role has:

```text
AmazonS3ReadOnlyAccess
```

The workflow runs:

```bash
aws s3 ls
```

This validates that the temporary credentials obtained through OIDC can perform the permitted S3 read operation.

---

# Troubleshooting

## Authentication Failure

During the first workflow run, the authentication failed with:

```text
Not authorized to perform sts:AssumeRoleWithWebIdentity
```

### Root Cause

The IAM trust policy was initially configured with the legacy subject:

```text
repo:Jaishree97/github-actions-oidc-demo:ref:refs/heads/main
```

However, the repository uses the immutable subject format.

The required subject is:

```text
repo:Jaishree97@222460494/github-actions-oidc-demo@1338257986:ref:refs/heads/main
```

### Resolution

The IAM trust policy was updated to use the immutable GitHub owner ID and repository ID.

After updating the trust policy, GitHub Actions successfully assumed the AWS IAM role.

---

## Additional Troubleshooting

A GitHub OIDC debugger action was also tested, but it failed during its Docker build.

Instead of relying on the debugger, GitHub Actions context values were used to retrieve the required IDs.

Example:

```yaml
- name: Get GitHub IDs
  run: |
    echo "Repository: ${{ github.repository }}"
    echo "Owner ID: ${{ github.repository_owner_id }}"
    echo "Repository ID: ${{ github.repository_id }}"
```

This provided the exact GitHub IDs required to configure the immutable OIDC trust policy.

> **Key learning:** The authentication failure was caused by the IAM trust policy subject condition, not by AWS access keys or the GitHub Actions workflow.

---

# Security Benefits

- No long-lived AWS access keys stored in GitHub.
- Uses temporary credentials issued by AWS STS.
- Repository-specific IAM trust policy.
- Branch-specific role assumption.
- Immutable GitHub owner and repository IDs.
- Supports least-privilege IAM permissions.
- Reduces the risk of credential leakage.
- Uses GitHub Actions native OIDC authentication.

---

# AWS Services and Technologies

| Technology | Purpose |
|---|---|
| GitHub Actions | CI/CD workflow execution |
| OpenID Connect | Federated authentication |
| AWS IAM | Identity and access management |
| AWS STS | Temporary AWS credentials |
| Amazon S3 | Read-access validation |

---

# Key Learning Outcomes

After completing this hands-on lab, I learned how to:

- Configure GitHub Actions OIDC with AWS.
- Create an AWS IAM OIDC identity provider.
- Create an IAM role for GitHub Actions.
- Understand legacy vs immutable OIDC subjects.
- Configure IAM trust policies using immutable IDs.
- Restrict role assumption to a specific repository and branch.
- Use AWS STS to obtain temporary credentials.
- Verify the assumed IAM role.
- Access AWS resources without long-lived access keys.
- Troubleshoot `AssumeRoleWithWebIdentity` authorization failures.

---

# Project Structure

```text
github-actions-oidc-demo/
│
├── .github/
│   └── workflows/
│       └── aws-oidc-challenge.yml
│
└── README.md
```

---

# Result

The hands-on successfully demonstrates:

- [x] GitHub repository created
- [x] GitHub OIDC provider configured in AWS IAM
- [x] IAM role created
- [x] `AmazonS3ReadOnlyAccess` attached
- [x] Immutable OIDC trust policy configured
- [x] GitHub Actions OIDC permissions configured
- [x] AWS role ARN stored as a GitHub repository secret
- [x] AWS region stored as a GitHub repository variable
- [x] GitHub Actions authenticated to AWS using OIDC
- [x] Temporary AWS credentials obtained through AWS STS
- [x] AWS identity verified using `aws sts get-caller-identity`
- [x] S3 read access verified using `aws s3 ls`

---

## Final Takeaway

> **GitHub Actions can authenticate to AWS securely using OIDC without storing long-lived AWS access keys.**

> **The IAM trust policy determines which GitHub workflow is allowed to assume the AWS role.**

> **Immutable OIDC subject claims strengthen this trust relationship by including stable GitHub owner and repository IDs.**

---

# References

## GitHub

- [GitHub OIDC Documentation](https://docs.github.com/en/actions/concepts/security/openid-connect)
- [GitHub OIDC Reference](https://docs.github.com/en/actions/reference/security/oidc)
- [Immutable Subject Claims Announcement](https://github.blog/changelog/2026-04-23-immutable-subject-claims-for-github-actions-oidc-tokens/)

## AWS

- [AWS IAM Documentation](https://docs.aws.amazon.com/iam/)
- [AWS STS Documentation](https://docs.aws.amazon.com/STS/)
- [AWS IAM Web Identity Federation](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_providers_oidc.html)
