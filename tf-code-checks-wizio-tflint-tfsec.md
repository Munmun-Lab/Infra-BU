# Terraform Code Quality & Security Checks using TFLint, TFSec & Wizio

## Overview

This document explains how to implement Terraform code validation, linting, and security scanning using:

* Terraform Validate
* TFLint
* TFSec
* Wizio Policy Checks
* CI/CD Pipeline Integration

The purpose of these checks is to improve:

* Infrastructure code quality
* Security compliance
* Terraform best practices
* Deployment reliability
* DevSecOps governance

This guide can be used for:

* GitLab projects
* GitHub repositories
* Enterprise DevOps onboarding
* Cloud governance implementation
* Infrastructure automation projects

---

# Architecture Flow

```text
Developer Pushes Terraform Code
            ↓
      Git Repository
            ↓
        CI/CD Pipeline
            ↓
 ┌──────────────────────┐
 │ terraform fmt        │
 │ terraform validate   │
 │ tflint               │
 │ tfsec                │
 │ wizio checks         │
 └──────────────────────┘
            ↓
      Quality Reports
            ↓
   Merge / Deployment Approval
```

---

# Prerequisites

Before implementation ensure:

* Terraform installed
* Git installed
* GitLab or GitHub repository
* CI/CD runner configured
* AWS credentials configured (if required)

Recommended versions:

| Tool          | Recommended Version |
| ------------- | ------------------- |
| Terraform     | >= 1.5              |
| TFLint        | Latest              |
| TFSec         | Latest              |
| GitLab Runner | Latest              |

---

# Project Structure

Example Terraform repository structure:

```text
terraform-project/
│
├── main.tf
├── variables.tf
├── outputs.tf
├── provider.tf
├── terraform.tfvars
│
├── modules/
│   ├── vpc/
│   └── ec2/
│
├── environments/
│   ├── dev/
│   ├── qa/
│   └── prod/
│
├── .gitlab-ci.yml
├── .tflint.hcl
└── README.md
```

---

# Step 1: Terraform Format Check

Terraform formatting ensures code consistency.

Run locally:

```bash
terraform fmt -recursive
```

Validate formatting:

```bash
terraform fmt -check -recursive
```

Purpose:

* Standardized code formatting
* Easier code reviews
* Consistent indentation
* Improved readability

---

# Step 2: Terraform Validate

Initialize Terraform:

```bash
terraform init
```

Run validation:

```bash
terraform validate
```

Purpose:

* Detect syntax errors
* Validate provider configuration
* Verify module references
* Catch invalid variable usage

Example Output:

```text
Success! The configuration is valid.
```

---

# Step 3: Install TFLint

## Mac

```bash
brew install tflint
```

## Linux

```bash
curl -s https://raw.githubusercontent.com/terraform-linters/tflint/master/install_linux.sh | bash
```

## Windows

Use Chocolatey:

```powershell
choco install tflint
```

Verify installation:

```bash
tflint --version
```

---

# Step 4: Configure TFLint

Create configuration file:

```bash
touch .tflint.hcl
```

Example configuration:

```hcl
plugin "aws" {
  enabled = true
  version = "0.32.0"
  source  = "github.com/terraform-linters/tflint-ruleset-aws"
}

rule "terraform_unused_declarations" {
  enabled = true
}

rule "terraform_unused_required_providers" {
  enabled = true
}
```

Initialize plugins:

```bash
tflint --init
```

Run TFLint:

```bash
tflint
```

Recursive scan:

```bash
tflint --recursive
```

---

# Step 5: TFLint Use Cases

TFLint helps identify:

* Unused variables
* Deprecated syntax
* Incorrect instance types
* Invalid provider configuration
* Missing tags
* Terraform best practice violations

Example Issue:

```text
Warning: terraform_unused_declarations
```

Benefits:

* Cleaner Terraform code
* Reduced configuration drift
* Better cloud governance

---

# Step 6: Install TFSec

## Mac

```bash
brew install tfsec
```

## Linux

```bash
curl -s https://raw.githubusercontent.com/aquasecurity/tfsec/master/scripts/install_linux.sh | bash
```

## Windows

```powershell
choco install tfsec
```

Verify installation:

```bash
tfsec --version
```

---

# Step 7: Run TFSec

Run scan:

```bash
tfsec .
```

Scan recursively:

```bash
tfsec ./modules
```

Example output:

```text
Result #1 HIGH Security Group allows ingress from 0.0.0.0/0
```

---

# Step 8: TFSec Security Checks

TFSec validates Terraform security posture.

Common checks:

| Check                 | Example                 |
| --------------------- | ----------------------- |
| Public S3 Buckets     | Bucket exposed publicly |
| Open Security Groups  | 0.0.0.0/0 exposure      |
| Unencrypted Resources | Missing encryption      |
| IAM Wildcards         | Over-permissive IAM     |
| Logging Disabled      | CloudTrail disabled     |
| Public Databases      | Public RDS access       |

Benefits:

* Shift-left security
* DevSecOps implementation
* Compliance readiness
* Reduced cloud risk

---

# Step 9: Install Wizio

Wizio can be integrated as part of Terraform governance and policy validation.

Example installation:

```bash
curl -fsSL https://app.wizio.ai/install.sh | sh
```

Authenticate:

```bash
wizio login
```

Verify:

```bash
wizio --version
```

---

# Step 10: Run Wizio Checks

Example command:

```bash
wizio scan .
```

Policy validation example:

```bash
wizio policy check
```

Typical validations:

* Security policy checks
* Compliance rules
* Organizational governance
* Cloud best practices
* Cost optimization rules

---

# Step 11: GitLab CI/CD Integration

Create:

```text
.gitlab-ci.yml
```

Example pipeline:

```yaml
stages:
  - validate
  - lint
  - security

terraform_validate:
  stage: validate
  image: hashicorp/terraform:latest
  script:
    - terraform init
    - terraform validate

terraform_fmt:
  stage: validate
  image: hashicorp/terraform:latest
  script:
    - terraform fmt -check -recursive

terraform_tflint:
  stage: lint
  image:
    name: ghcr.io/terraform-linters/tflint
    entrypoint: [""]
  script:
    - tflint --init
    - tflint --recursive

terraform_tfsec:
  stage: security
  image:
    name: aquasec/tfsec
    entrypoint: [""]
  script:
    - tfsec .
```

---

# Step 12: GitHub Actions Integration

Create:

```text
.github/workflows/terraform-checks.yml
```

Example workflow:

```yaml
name: Terraform Checks

on:
  push:
    branches:
      - main

jobs:
  terraform-checks:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3

      - name: Terraform Init
        run: terraform init

      - name: Terraform Validate
        run: terraform validate

      - name: Terraform Format
        run: terraform fmt -check -recursive

      - name: Install TFLint
        run: |
          curl -s https://raw.githubusercontent.com/terraform-linters/tflint/master/install_linux.sh | bash

      - name: Run TFLint
        run: |
          tflint --init
          tflint --recursive

      - name: Install TFSec
        run: |
          curl -s https://raw.githubusercontent.com/aquasecurity/tfsec/master/scripts/install_linux.sh | bash

      - name: Run TFSec
        run: tfsec .
```

---

# Step 13: Example Terraform Issues Detected

## Example 1: Open Security Group

Bad Example:

```hcl
cidr_blocks = ["0.0.0.0/0"]
```

Detected by:

* tfsec
* security policies

---

## Example 2: Unused Variable

Bad Example:

```hcl
variable "unused_var" {}
```

Detected by:

* tflint

---

## Example 3: Unencrypted S3 Bucket

Bad Example:

```hcl
resource "aws_s3_bucket" "demo" {}
```

Detected by:

* tfsec

---

# Step 14: DevSecOps Best Practices

Recommended practices:

* Run checks before merge
* Block insecure Terraform changes
* Use branch protection rules
* Enable pull request reviews
* Maintain reusable Terraform modules
* Scan all environments
* Integrate policy-as-code

---

# Step 15: Benefits of Terraform Code Checks

| Area        | Benefit                        |
| ----------- | ------------------------------ |
| Security    | Detect insecure infrastructure |
| Quality     | Improve Terraform standards    |
| Governance  | Enforce policies               |
| Automation  | Reduce manual review effort    |
| Compliance  | Improve audit readiness        |
| Reliability | Prevent deployment failures    |

---

# Troubleshooting

## TFLint Plugin Error

Run:

```bash
tflint --init
```

---

## Terraform Validate Failure

Run:

```bash
terraform init
```

---

## TFSec False Positives

Ignore example:

```hcl
#tfsec:ignore:AWS006
```

Use carefully and document the reason.

---

# End-to-End Workflow Example

```text
Developer writes Terraform code
        ↓
Git push to repository
        ↓
CI/CD pipeline triggered
        ↓
terraform fmt check
        ↓
terraform validate
        ↓
tflint scan
        ↓
tfsec security scan
        ↓
wizio policy validation
        ↓
Merge approval
        ↓
Terraform deployment
```

---

# Conclusion

Terraform code quality and security checks are essential components of modern Infrastructure as Code and DevSecOps practices. Integrating Terraform validation, TFLint, TFSec, and Wizio into CI/CD pipelines helps organizations improve security posture, enforce standards, reduce operational risks, and maintain scalable cloud governance.

These implementations support enterprise-grade infrastructure automation while enabling faster and safer cloud deployments.
