# Automated AWS Infrastructure as Code with GitHub Actions CI/CD

[![CI/CD Pipeline](https://img.shields.io/badge/CI%2FCD-GitHub%20Actions-blue?logo=github-actions&logoColor=white)](.github/workflows/deploy.yml)
[![Infrastructure as Code](https://img.shields.io/badge/IaC-AWS%20CloudFormation-orange?logo=amazon-aws&logoColor=white)](cloudformation.yaml)
[![Cloud Provider](https://img.shields.io/badge/Cloud-AWS-232F3E?logo=amazon-aws&logoColor=white)](https://aws.amazon.com/)
[![OS](https://img.shields.io/badge/OS-Ubuntu%2024.04%20LTS-E95420?logo=ubuntu&logoColor=white)](https://ubuntu.com/)
[![Web Server](https://img.shields.io/badge/Server-Nginx-009639?logo=nginx&logoColor=white)](https://nginx.org/)
[![Security](https://img.shields.io/badge/Security-OIDC%20Keyless%20Auth-green?logo=openid&logoColor=white)](iam/)

> **Portfolio Project**  
> **Author:** [Safkat Hossain](https://github.com/safkat000)  
> **Repository:** [`safkat000/bongodev-cloudformation-github-actions`](https://github.com/safkat000/bongodev-cloudformation-github-actions)  


---

## Executive Summary & Architecture Overview

This project implements an end-to-end **GitOps continuous delivery (CI/CD) pipeline** that automates the provisioning and lifecycle management of cloud infrastructure on Amazon Web Services (AWS). 

Instead of manual resource creation through the AWS Management Console or risky local CLI deployments, this repository establishes **Git as the single source of truth** for cloud infrastructure. Pushing code changes to the `main` branch automatically triggers GitHub Actions to validate and reconcile AWS resources via **AWS CloudFormation**.

A cornerstone of this implementation is **zero-trust, keyless cloud authentication**. By establishing an **OpenID Connect (OIDC)** identity federation between GitHub Actions and AWS Identity and Access Management (IAM), the deployment pipeline assumes an IAM role dynamically using short-lived security tokens. **No long-lived AWS access keys (`AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY`) are stored or exposed.**

```text
Developer (VS Code / Terminal)
        │
        │  git push origin main
        ▼
GitHub Repository (safkat000/bongodev-cloudformation-github-actions)
        │
        │  Trigger on push to: cloudformation.yaml, .github/workflows/deploy.yml
        ▼
GitHub Actions Runner (ubuntu-latest)
        │  1. Checkout code
        │  2. Exchange GitHub OIDC JWT with AWS STS for temporary credentials
        │  3. Verify caller identity (aws sts get-caller-identity)
        │  4. Validate syntax & schema (aws cloudformation validate-template)
        │  5. Deploy / Reconcile stack (aws cloudformation deploy)
        ▼
AWS CloudFormation Stack: bongodev-github-iac-dev (Region: us-west-2)
        │
        ├── AWS::EC2::VPC ("DemoVPC" - 10.42.0.0/16)
        ├── AWS::EC2::Subnet ("PublicSubnet" - 10.42.1.0/24, MapPublicIpOnLaunch)
        ├── AWS::EC2::InternetGateway & VPCGatewayAttachment
        ├── AWS::EC2::RouteTable & DefaultRoute (0.0.0.0/0 via IGW)
        ├── AWS::EC2::SecurityGroup ("WebSecurityGroup" - Inbound HTTP/80)
        └── AWS::EC2::Instance ("UbuntuNginxServer" - Ubuntu 24.04 LTS t3.micro)
                 │
                 └── EC2 User Data bootstrap:
                       ├── apt-get update & apt-get install nginx
                       ├── Deploy custom HTML site
                       └── systemctl enable --now nginx
```

---

## Project Verification & Submission Checklist

All milestones specified in the [Submission Checklist](docs/student-submission-checklist.md) have been successfully accomplished and validated against AWS:

| Milestone / Requirement | Status | Evidence / Reference |
|---|:---:|---|
| GitHub repository containing `cloudformation.yaml` and `.github/workflows/deploy.yml` | Completed | [`cloudformation.yaml`](cloudformation.yaml), [`.github/workflows/deploy.yml`](.github/workflows/deploy.yml) |
| Successful initial GitHub Actions workflow (green check, OIDC exchange, stack create) | Completed | [Screenshot 02](#1-github-actions-initial-deployment) |
| CloudFormation stack `bongodev-github-iac-dev` in `CREATE_COMPLETE` / `UPDATE_COMPLETE` | Completed | [Screenshot 03](#2-cloudformation-stack-provisioned) & [Screenshot 07](#6-github-actions-second-push-update) |
| Running Ubuntu 24.04 EC2 instance in custom VPC with tags | Completed | [Screenshot 04](#3-ubuntu-ec2-instance-running) |
| Live Nginx website serving `Hello from bongoDev!` in `Environment: dev` | Completed | [Screenshot 05](#4-live-nginx-web-server-response) |
| GitHub commit URL modifying EC2 `Course` tag triggering automated deployment | Completed | Commit [`3c988a0`](https://github.com/safkat000/bongodev-cloudformation-github-actions/commit/3c988a0f446712596f406b266ee8af4aa50e799d) & [Screenshot 06](#5-iac-code-change-second-push) |
| Verification of modified `Course` tag in EC2 console after automated stack update | Completed | [Screenshot 08](#7-verified-ec2-tag-update) |
| Clean teardown and stack deletion confirmation | Completed | [Screenshot 09](#8-stack-deletion-in-progress) & [Screenshot 10](#9-complete-stack-deletion--cleanup) |
| Deep-dive technical explanations (GitHub Actions vs. CloudFormation, OIDC security) | Completed | [Section: Technical Deep Dive & Q&A](#technical-deep-dive--architectural-questions) |

---

## Visual Implementation Showcase

Below is the chronological visual evidence documenting the deployment, live verification, infrastructure update, and cleanup phases.

### 1. GitHub Actions Initial Deployment
The initial push to the repository triggered the pipeline. GitHub Actions authenticated via OIDC, validated the template, and executed `aws cloudformation deploy`, creating all resources automatically without any manual AWS console interaction.

<p align="center">
  <img src="Screenshots/02.Github Action Successful.png" alt="GitHub Actions Initial Deployment Successful" width="90%" />
</p>

---

### 2. CloudFormation Stack Provisioned
In the AWS CloudFormation Console (`us-west-2`), the stack `bongodev-github-iac-dev` reached `CREATE_COMPLETE`, outputting the VPC ID, public IP address, instance ID, and website URL.

<p align="center">
  <img src="Screenshots/03.CloudFormation Stack.png" alt="CloudFormation Stack CREATE_COMPLETE" width="90%" />
</p>

---

### 3. Ubuntu EC2 Instance Running
The `t3.micro` EC2 instance launched inside the dedicated VPC (`10.42.0.0/16`) and public subnet (`10.42.1.0/24`), powered by Canonical's official Ubuntu 24.04 LTS AMI resolved dynamically via AWS SSM Parameter Store.

<p align="center">
  <img src="Screenshots/04.Ubuntu EC2 Instance.png" alt="Ubuntu EC2 Instance in AWS Console" width="90%" />
</p>

---

### 4. Live Nginx Web Server Response
Navigating to the instance's public IPv4 address renders the custom web page bootstrapped automatically by the EC2 User Data script during first launch.

<p align="center">
  <img src="Screenshots/05.Nginx Result.png" alt="Live Nginx Website" width="90%" />
</p>

---

### 5. IaC Code Change (Second Push)
To demonstrate continuous deployment and CloudFormation's update mechanics, the EC2 instance tag was updated in `cloudformation.yaml` from `bongoDev-DevOps` to `bongoDev-DevOps-Homework-V2` and committed to `main` ([Commit 3c988a0](https://github.com/safkat000/bongodev-cloudformation-github-actions/commit/3c988a0f446712596f406b266ee8af4aa50e799d)).

<p align="center">
  <img src="Screenshots/06.Modified EC2 Tag.png" alt="Modified EC2 Tag Git Commit" width="90%" />
</p>

---

### 6. GitHub Actions Second Push (Update)
The second push triggered the workflow once again. Instead of failing or attempting to duplicate the infrastructure, CloudFormation generated a changeset and applied an in-place update to the running stack.

<p align="center">
  <img src="Screenshots/07.Update After 2nd Push.png" alt="GitHub Actions Pipeline Updating Stack" width="90%" />
</p>

---

### 7. Verified EC2 Tag Update
Checking the EC2 instance properties in the AWS Console confirmed that the `Course` tag updated to `bongoDev-DevOps-Homework-V2` while preserving the exact same instance ID without downtime or recreation.

<p align="center">
  <img src="Screenshots/08.EC2 Tag Is Modified.png" alt="EC2 Tag Updated to bongoDev-DevOps-Homework-V2" width="90%" />
</p>

---

### 8. Stack Deletion in Progress
Following verification, the stack deletion was initiated to clean up all allocated AWS billable resources (VPC, IGW, Route Table, Security Group, EC2 instance, EBS volumes, and public IP).

<p align="center">
  <img src="Screenshots/09.Stack Deletion In Progress.png" alt="CloudFormation Stack Deletion in Progress" width="90%" />
</p>

---

### 9. Complete Stack Deletion & Cleanup
All resources provisioned by the IaC template were cleanly terminated, leaving no orphaned resources or lingering costs.

<p align="center">
  <img src="Screenshots/10.Stack Deleted.png" alt="CloudFormation Stack Cleanup Confirmed" width="90%" />
</p>

---

### 10. Repository Workspace Setup
Local development environment and repository structure showing tracked IaC files, workflows, IAM policies, and documentation.

<p align="center">
  <img src="Screenshots/01.Git Repo.png" alt="Repository Workspace Structure" width="90%" />
</p>

---

## Technical Deep Dive & Architectural Questions

### 1. What is the fundamental difference between GitHub Actions and AWS CloudFormation?
* **GitHub Actions** is a **CI/CD orchestration engine**. It automates the pipeline lifecycle: responding to Git triggers, spinning up ephemeral virtual runners, running linting/validation tests, negotiating credentials with external identity providers, and executing CLI commands. It does not natively manage AWS resource states.
* **AWS CloudFormation** is an **Infrastructure as Code (IaC) provisioning and state engine**. It takes declarative templates describing the desired cloud state and handles dependency graph resolution, resource provisioning, change sets, rollbacks on failure, and state tracking directly within the AWS control plane.
* **Synthesis:** GitHub Actions *orchestrates the delivery workflow*, while CloudFormation *executes and manages the cloud infrastructure state*.

### 2. Why can a `git push` deploy infrastructure without anyone logging into the AWS Console?
Because GitHub Actions leverages **federated authentication via OpenID Connect (OIDC)**. When a commit is pushed to `main`, GitHub Actions requests a signed JSON Web Token (JWT) from GitHub's OIDC token service (`token.actions.githubusercontent.com`). The workflow presents this token to AWS Security Token Service (STS) using `AssumeRoleWithWebIdentity`. AWS validates the token signature against GitHub's public keys and checks the IAM role's trust policy. Once validated, AWS STS issues temporary security credentials (session token, access key ID, and secret access key) scoped exclusively to that workflow run, allowing it to invoke CloudFormation APIs directly.

### 3. What does OIDC do, and why is it safer than committing static AWS access keys?
* **Elimination of Long-Lived Secrets:** Traditional CI/CD stored permanent `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` pairs in GitHub Secrets. If leaked, stolen, or accidentally printed in build logs, attackers gain indefinite access until keys are revoked.
* **Short-Lived Ephemeral Credentials:** OIDC produces temporary AWS credentials valid only for the duration of the job (e.g., 15–60 minutes), after which they expire automatically.
* **Cryptographic Trust Conditions:** The AWS IAM trust policy explicitly validates the claims inside the GitHub token:
  * `aud`: Ensures the token was minted specifically for `sts.amazonaws.com`.
  * `sub`: Matches the exact repository, numeric owner ID, numeric repository ID, and Git branch (`repo:safkat000@.../bongodev-cloudformation-github-actions@...:ref:refs/heads/main`). No other repository, fork, or branch can assume the role.

### 4. On a second push, why is the existing stack updated instead of creating a new one?
The deployment workflow executes `aws cloudformation deploy --stack-name bongodev-github-iac-dev`. The `deploy` command is **idempotent**:
* If a stack with that name does not exist, CloudFormation creates it.
* If a stack with that name already exists, CloudFormation calculates a **Change Set** (the differential between the current state and the newly submitted template) and applies only the required updates in-place.
* The `--no-fail-on-empty-changeset` flag ensures that if a run contains no infrastructure changes, the workflow gracefully exits with success instead of throwing an error.

### 5. Is CloudFormation stack creation complete always the same as Nginx ready? Why not?
No. CloudFormation marks an `AWS::EC2::Instance` resource as `CREATE_COMPLETE` as soon as AWS finishes provisioning the virtual machine, allocating network interfaces, and signaling to the hypervisor that the instance is booted. 

However, **EC2 User Data scripts execute asynchronously inside the guest OS** during the initial boot via `cloud-init`. The package manager (`apt-get update`, `apt-get install nginx`) and web server daemon startup take additional time (typically 1–3 minutes) after CloudFormation reports `CREATE_COMPLETE`. To guarantee synchronization in enterprise setups, one would use `cfn-signal` and a CloudFormation `CreationPolicy`.

### 6. Why does this project update an EC2 tag instead of modifying the webpage HTML to demonstrate updates?
EC2 User Data scripts only execute **once on the initial launch** of an instance by default. If you edit the `UserData` block in an existing CloudFormation template, CloudFormation will update the instance metadata, but **the running Ubuntu OS will not rerun the bootstrap script**. Updating the HTML would therefore not reflect on the live website unless the instance is terminated and replaced.

Updating a resource tag (such as `Course: bongoDev-DevOps-Homework-V2`) safely tests the IaC reconciliation engine because:
* CloudFormation recognizes the tag diff in its Change Set.
* It applies the tag update directly via the EC2 API without causing service disruption or instance replacement (`Update requires: No interruption`).

### 7. What should happen to AWS resources when testing and verification are complete?
In any cloud DevOps workflow, non-production test environments must be promptly decommissioned to enforce **cost governance** and **security hygiene**:
* Prevent continuous billing for compute instances (`t3.micro`), public IPv4 addresses, and EBS gp3 storage volumes.
* Eliminate abandoned attack surfaces (e.g., exposed HTTP port 80).
* Ensure that the IaC templates are truly reproducible from scratch at any time.

---

## Repository Architecture & Directory Structure

```text
bongodev-cloudformation-github-actions/
├── README.md                            <-- Project documentation & portfolio presentation
├── cloudformation.yaml                  <-- Declarative AWS IaC template (VPC, EC2, Nginx)
├── .github/
│   └── workflows/
│       └── deploy.yml                   <-- GitHub Actions CI/CD automation workflow
├── iam/
│   ├── deployment-policy-lab.json       <-- Scoped IAM deployment permissions policy
│   ├── trust-policy-new-repos-2026.json <-- OIDC trust policy for modern GitHub repositories
│   └── trust-policy-legacy-repos.json   <-- Fallback OIDC trust policy for legacy repositories
├── docs/
│   └── student-submission-checklist.md  <-- Verified course submission requirements
├── Screenshots/                         <-- Photographic proof of deployment and verification
│   ├── 01.Git Repo.png
│   ├── 02.Github Action Successful.png
│   ├── 03.CloudFormation Stack.png
│   ├── 04.Ubuntu EC2 Instance.png
│   ├── 05.Nginx Result.png
│   ├── 06.Modified EC2 Tag.png
│   ├── 07.Update After 2nd Push.png
│   ├── 08.EC2 Tag Is Modified.png
│   ├── 09.Stack Deletion In Progress.png
│   └── 10.Stack Deleted.png
└── .gitignore
```

---

## Infrastructure as Code Specification (`cloudformation.yaml`)

The template defines a complete networking stack and application server:

| Resource Logical ID | AWS Resource Type | Configuration & Purpose |
|---|---|---|
| `DemoVPC` | `AWS::EC2::VPC` | Dedicated VPC with CIDR `10.42.0.0/16`, DNS support and hostnames enabled. |
| `InternetGateway` | `AWS::EC2::InternetGateway` | Provides internet ingress and egress for the VPC. |
| `GatewayAttachment` | `AWS::EC2::VPCGatewayAttachment` | Attaches the Internet Gateway to `DemoVPC`. |
| `PublicSubnet` | `AWS::EC2::Subnet` | Public subnet with CIDR `10.42.1.0/24`, configured with `MapPublicIpOnLaunch: true`. |
| `PublicRouteTable` | `AWS::EC2::RouteTable` | Custom routing table for public subnets. |
| `DefaultRoute` | `AWS::EC2::Route` | Route directing outbound `0.0.0.0/0` traffic to the Internet Gateway. |
| `PublicRouteAssociation` | `AWS::EC2::SubnetRouteTableAssociation` | Associates `PublicSubnet` with `PublicRouteTable`. |
| `WebSecurityGroup` | `AWS::EC2::SecurityGroup` | Inbound TCP port 80 allowed for HTTP traffic. Zero SSH ingress for improved security. |
| `UbuntuNginxServer` | `AWS::EC2::Instance` | `t3.micro` EC2 instance with IMDSv2 enforced (`HttpTokens: required`). AMI dynamically queried from Canonical's official SSM parameter `/aws/service/canonical/ubuntu/server/noble/stable/current/amd64/hvm/ebs-gp3/ami-id`. |
| `UserData` | Bash Script | Automates `apt-get` updates, installs Nginx, writes custom HTML, and enables the systemd service. |

### Stack Outputs
* `WebsiteURL`: Direct HTTP link to the live Nginx website (`http://${PublicIp}`).
* `InstanceId`: Physical ID of the provisioned EC2 instance.
* `PublicIPAddress`: Public IPv4 address assigned to the instance.
* `VpcId`: Physical ID of the provisioned VPC.

---

## CI/CD Pipeline Implementation (`deploy.yml`)

The automation workflow is defined in [`.github/workflows/deploy.yml`](.github/workflows/deploy.yml):

```yaml
name: Deploy Ubuntu Nginx with CloudFormation

on:
  push:
    branches:
      - main
    paths:
      - cloudformation.yaml
      - .github/workflows/deploy.yml
  workflow_dispatch:

permissions:
  id-token: write
  contents: read

concurrency:
  group: bongodev-cloudformation-dev
  cancel-in-progress: false

jobs:
  deploy:
    name: Validate and deploy infrastructure
    runs-on: ubuntu-latest
    timeout-minutes: 30
    env:
      STACK_NAME: bongodev-github-iac-dev
      AWS_REGION: ${{ vars.AWS_REGION }}

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Configure temporary AWS credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v6.3.0
        with:
          role-to-assume: ${{ vars.AWS_ROLE_ARN }}
          aws-region: ${{ vars.AWS_REGION }}
          role-session-name: bongodev-iac-lab

      - name: Confirm AWS identity
        run: aws sts get-caller-identity --query '{Account:Account,Arn:Arn}' --output json

      - name: Validate CloudFormation template
        run: aws cloudformation validate-template --template-body file://cloudformation.yaml --region "$AWS_REGION"

      - name: Create or update CloudFormation stack
        run: |
          aws cloudformation deploy \
            --template-file cloudformation.yaml \
            --stack-name "$STACK_NAME" \
            --parameter-overrides Environment=dev \
            --region "$AWS_REGION" \
            --no-fail-on-empty-changeset

      - name: Print website and infrastructure outputs
        run: |
          aws cloudformation describe-stacks \
            --stack-name "$STACK_NAME" \
            --region "$AWS_REGION" \
            --query 'Stacks[0].Outputs[*].[OutputKey,OutputValue]' \
            --output table

      - name: Lab completion message
        run: echo 'The stack is deployed. EC2 User Data may need additional minutes to install Nginx.'
```

### Key Security & Operational Highlights
1. **Concurrency Control:** `concurrency.group` guarantees that parallel pushes do not trigger simultaneous CloudFormation updates, preventing race conditions or stack locking conflicts (`UPDATE_IN_PROGRESS`).
2. **Minimal GitHub Permissions:** Only `id-token: write` (for OIDC token acquisition) and `contents: read` are granted.
3. **Template Pre-Validation:** Syntax and schema errors are caught via `aws cloudformation validate-template` before initiating changesets on AWS.

---

## Reproduction & Setup Guide

To replicate this deployment in your own AWS sandbox environment:

### Step 1: Configure IAM OpenID Connect Provider
In **AWS IAM → Identity Providers**:
* **Provider Type:** OpenID Connect
* **Provider URL:** `https://token.actions.githubusercontent.com`
* **Audience:** `sts.amazonaws.com`

### Step 2: Create IAM Policy & Deployment Role
1. Create the customer-managed policy `bongodev-github-iac-lab-policy` using the JSON provided in [`iam/deployment-policy-lab.json`](iam/deployment-policy-lab.json).
2. Create an IAM role named `bongodev-github-iac-deploy-role` with **Web Identity** federation pointing to the GitHub OIDC provider.
3. Attach the permissions policy and edit the **Trust Policy** using [`iam/trust-policy-new-repos-2026.json`](iam/trust-policy-new-repos-2026.json), substituting your AWS Account ID, GitHub Owner, Owner ID, Repository Name, and Repository ID:

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
          "token.actions.githubusercontent.com:sub": "repo:<GITHUB_OWNER>@<OWNER_ID>/<REPO_NAME>@<REPO_ID>:ref:refs/heads/main"
        }
      }
    }
  ]
}
```

### Step 3: Configure GitHub Repository Variables
In your GitHub repository (**Settings → Secrets and variables → Actions → Variables**), create:
* `AWS_ROLE_ARN`: `arn:aws:iam::<AWS_ACCOUNT_ID>:role/bongodev-github-iac-deploy-role`
* `AWS_REGION`: `us-west-2`

### Step 4: Deploy & Verify
Push to `main`. GitHub Actions will automatically trigger, validate the CloudFormation template, provision the stack, and output the website URL.

### Step 5: Clean Teardown
To avoid unnecessary AWS cloud costs after verification:
```bash
aws cloudformation delete-stack \
  --stack-name bongodev-github-iac-dev \
  --region us-west-2

aws cloudformation wait stack-delete-complete \
  --stack-name bongodev-github-iac-dev \
  --region us-west-2
```

---

## Troubleshooting Guide

| Issue / Error | Root Cause | Solution |
|---|---|---|
| `Not authorized to perform sts:AssumeRoleWithWebIdentity` | Mismatched OIDC `sub` claim in IAM Trust Policy. | Modern repositories (post-July 2026) include numeric Owner ID and Repo ID in the subject claim. Verify the exact subject using `gh api repos/<OWNER>/<REPO> --jq '"\(.owner.login)@\(.owner.id)/\(.name)@\(.id)"'`. |
| `AccessDenied` on `ec2:*` or `cloudformation:*` | IAM role missing lab deployment policy. | Verify `bongodev-github-iac-lab-policy` is attached to `bongodev-github-iac-deploy-role`. Check for SCP or permission boundary restrictions. |
| Workflow does not trigger on commit | Changed files outside workflow trigger path. | The workflow filters on `cloudformation.yaml` and `.github/workflows/deploy.yml`. Changes exclusively to markdown files do not trigger deployments. Use **Run workflow** for manual testing. |
| `WebsiteURL` times out (`ERR_CONNECTION_TIMED_OUT`) | EC2 User Data still installing Nginx or opening via HTTPS. | Ensure you are navigating via plain **HTTP** (`http://<IP>`). Wait 2–3 minutes for the `apt-get` bootstrap script to finish. Check `/var/log/bongodev-bootstrap.log`. |
| CloudFormation enters `ROLLBACK_COMPLETE` | Resource provisioning error (e.g., subnet quota, AMI availability). | Inspect CloudFormation **Events** tab for the first failing resource. Delete the rolled-back stack before redeploying. |

---

## References & Official Documentation

* [GitHub Actions: Configuring OpenID Connect in Amazon Web Services](https://docs.github.com/en/actions/how-tos/secure-your-work/security-harden-deployments/oidc-in-aws)
* [AWS Configure AWS Credentials Action (`aws-actions/configure-aws-credentials`)](https://github.com/aws-actions/configure-aws-credentials)
* [AWS CloudFormation CLI Deploy Reference](https://docs.aws.amazon.com/cli/latest/reference/cloudformation/deploy/index.html)
* [Canonical: Ubuntu Server CloudFormation AMIs](https://ubuntu.com/aws/docs/aws-how-to/instances/build-cloudformation-templates/)
* [AWS EC2 User Data & Shell Scripts](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/user-data.html)

---

