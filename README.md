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

## Step 4 — Configure GitHub Repository Secret and Variable

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


## Repository Variable

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

## Step 5 — Find GitHub Owner ID and Repository ID

The immutable OIDC `sub` claim requires two GitHub IDs:

- **Owner ID**
- **Repository ID**

These IDs can be retrieved from the GitHub repository metadata.

### Option 1 — GitHub REST API

Run:

```bash
curl https://api.github.com/repos/Jaishree97/github-actions-oidc-demo
```

The response contains the repository ID and owner ID:

```json
{
  "id": 1338257986,
  "owner": {
    "login": "Jaishree97",
    "id": 222460494
  }
}
```

For this hands-on repository:

| Value | ID |
|---|---|
| Owner | `Jaishree97` |
| Owner ID | `222460494` |
| Repository | `github-actions-oidc-demo` |
| Repository ID | `1338257986` |

### Option 2 — GitHub Actions Context

The IDs can also be retrieved directly from the GitHub Actions workflow context:

```YAML
- name: Get GitHub IDs
  run: |
    echo "Repository: ${{ github.repository }}"
    echo "Owner ID: ${{ github.repository_owner_id }}"
    echo "Repository ID: ${{ github.repository_id }}"
```
This approach was used during troubleshooting to confirm the exact IDs required for the immutable OIDC trust policy.

> **Note: The owner ID and repository ID are stable identifiers used by the immutable OIDC subject format.**

---

## Step 6 — Build the Immutable OIDC Subject

GitHub uses the owner ID and repository ID to construct the immutable OIDC `sub` claim.

For this hands-on repository:

```text
Owner: Jaishree97
Owner ID: 222460494
Repository: github-actions-oidc-demo
Repository ID: 1338257986
Branch: main
```

### Immutable Subject Structure:

```text
repo:<OWNER>@<OWNER_ID>/<REPOSITORY>@<REPOSITORY_ID>:ref:refs/heads/<BRANCH>
```

### Subject for This Repository:

```text
repo:Jaishree97@222460494/github-actions-oidc-demo@1338257986:ref:refs/heads/main
```
Breakdown:

```text
repo:
  └── Jaishree97
       └── @222460494
            └── /github-actions-oidc-demo
                 └── @1338257986
                      └── :ref:refs/heads/main
```
This subject identifies:

- GitHub owner: Jaishree97
- Immutable owner ID: 222460494
- Repository: github-actions-oidc-demo
- Immutable repository ID: 1338257986
- Branch: main

> **Key Learning: The immutable subject uses stable owner and repository IDs instead of relying only on repository and owner names.**

---

## Step 7 — Configure IAM Trust Policy

The IAM trust policy determines which GitHub Actions OIDC token is allowed to assume the IAM role.

The policy validates two OIDC claims:

- `aud` — confirms the token is intended for AWS STS.
- `sub` — identifies the specific GitHub repository and `main` branch.

Replace `<AWS_ACCOUNT_ID>` with your AWS account ID before applying the policy.

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
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
          "token.actions.githubusercontent.com:sub": "repo:Jaishree97@222460494/github-actions-oidc-demo@1338257986:ref:refs/heads/main"
        }
      }
    }
  ]
}
```
### Understanding the Trust Policy

1. Federated Principal

```json
"Principal": {
  "Federated": "arn:aws:iam::<AWS_ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com"
}
```
This specifies that the trusted identity provider is GitHub's OIDC provider.

2. Assume Role Action

```json
"Action": "sts:AssumeRoleWithWebIdentity"
```
AWS STS uses this action to allow GitHub's OIDC token to assume the IAM role.

3. Audience Condition

```json
"token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
```
This ensures that the OIDC token is intended to be used with AWS STS.

4. Immutable Subject Condition

```json
"token.actions.githubusercontent.com:sub": "repo:Jaishree97@222460494/github-actions-oidc-demo@1338257986:ref:refs/heads/main"
```
This restricts role assumption to:

```text
Owner: Jaishree97
Owner ID: 222460494
Repository: github-actions-oidc-demo
Repository ID: 1338257986
Branch: main
```
### Why StringEquals?   

Both claims are matched against exact expected values:

```text
aud = sts.amazonaws.com
sub = repo:Jaishree97@222460494/github-actions-oidc-demo@1338257986:ref:refs/heads/main
```
Using exact matching keeps the trust policy restrictive.

> **Security Tip: Avoid broad wildcard patterns when the exact repository and branch are known. Restricting the sub claim to the required repository and branch follows the principle of least privilege.**


### Trust Relationship

The resulting trust relationship is:

```text
GitHub Actions
      │
      │ OIDC Token
      ▼
GitHub OIDC Provider
      │
      │ aud = sts.amazonaws.com
      │ sub = immutable repository + branch
      ▼
AWS IAM Trust Policy
      │
      │ Allow
      ▼
AWS STS
      │
      ▼
Temporary AWS Credentials
```
> **Key Learning: The IAM trust policy is the security boundary that determines whether a GitHub Actions OIDC token is allowed to assume the AWS IAM role.**
---

## Step 8 — Create GitHub Actions Workflow

Create:

```text
.github/workflows/aws-oidc-challenge.yml
```

Use:

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

### Workflow Flow

```text
GitHub Actions
      ↓
Request OIDC Token
      ↓
AWS STS
      ↓
Validate IAM Trust Policy
      ↓
Assume IAM Role
      ↓
Temporary AWS Credentials
      ↓
Verify Identity
      ↓
Validate S3 Access
```

### Key Configuration

```yaml
permissions:
  id-token: write
  contents: read
```

- `id-token: write` allows GitHub Actions to request an OIDC token.
- `contents: read` allows the repository to be checked out.

The AWS credentials action uses the GitHub OIDC token to assume the IAM role through AWS STS.

> **Key Learning:** GitHub Actions authenticates to AWS using OIDC and temporary credentials instead of storing long-lived AWS access keys.

---

## Step 9 — Run the Workflow

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

1. Checkout the repository.
2. Authenticate to AWS using GitHub OIDC.
3. Assume the IAM role through AWS STS.
4. Verify the AWS identity.
5. Validate S3 read access.

![Successful GitHub Actions OIDC Workflow](./images/06-github-oidc-workflow-success.png)

---

## Step 10 — Verify AWS Identity

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

> **The `assumed-role` ARN confirms that GitHub Actions successfully assumed the IAM role through AWS STS.**

---

## Step 11 — Validate S3 Access

The IAM role has:

```text
AmazonS3ReadOnlyAccess
```
![AWS STS Get Caller Identity Output](./images/07-aws-sts-get-caller-identity.png)

---

## Step 12 — The workflow runs:

```bash
aws s3 ls
```
![AWS S3 Read Access Output](./images/08-aws-s3-read-access.png)

> **This confirms that the temporary AWS credentials have the expected S3 read permissions.**

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

### Troubleshooting Result

The first workflow run failed because the IAM trust policy used the legacy GitHub OIDC `sub` format.

After updating the trust policy to use the immutable owner ID and repository ID, the workflow successfully:

- Assumed the IAM role through AWS STS.
- Obtained temporary AWS credentials.
- Verified the AWS identity.
- Accessed S3 using read-only permissions.

> **This hands-on helped me understand that GitHub OIDC authentication depends on the IAM trust policy matching the `sub` claim issued by GitHub.**

---

## Final Takeaway

> **GitHub Actions can authenticate to AWS securely using OIDC without storing long-lived AWS access keys.**

> **The IAM trust policy determines which GitHub workflow is allowed to assume the AWS role.**

> **Immutable OIDC subject claims strengthen this trust relationship by including immutable GitHub owner and repository IDs.**

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
