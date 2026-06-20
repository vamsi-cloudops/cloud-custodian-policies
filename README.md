# Cloud Custodian Policies

> Production-ready Cloud Custodian policy library for automated compliance, governance, and remediation across enterprise multi-account AWS environments — covering resource tagging, security guardrails, cost controls, and automated remediation workflows.

![Cloud Custodian](https://img.shields.io/badge/Cloud%20Custodian-%3E%3D0.9-blue?logo=python)
![AWS](https://img.shields.io/badge/AWS-Multi--Account-orange?logo=amazon-aws)
![Python](https://img.shields.io/badge/Python-%3E%3D3.11-blue?logo=python)
![License](https://img.shields.io/badge/License-MIT-green)

---

## Overview

This repository contains a curated set of [Cloud Custodian](https://cloudcustodian.io/) policies for enforcing organizational cloud governance at scale. Policies are organized by domain (tagging, security, cost, compliance) and designed for deployment via AWS Organizations using Lambda-based execution with SQS event streams.

Key capabilities:
- **Resource tagging enforcement** — detect, alert, and quarantine untagged or mistagged resources
- **Security guardrails** — automated response to misconfigurations (open security groups, public S3, exposed secrets)
- **Cost controls** — identify and act on idle, oversized, and orphaned resources
- **Remediation workflows** — notification, tagging, stopping, and termination pipelines with approval gates

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                       AWS Organizations                             │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │              Cloud Custodian Orchestration Account          │   │
│  │                                                             │   │
│  │   ┌──────────────┐    ┌──────────────┐    ┌─────────────┐  │   │
│  │   │   c7n-org    │───▶│  IAM Roles   │───▶│  S3 Output  │  │   │
│  │   │  (CLI tool)  │    │  (per acct)  │    │  (results)  │  │   │
│  │   └──────────────┘    └──────────────┘    └─────────────┘  │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                │                                    │
│               ┌────────────────┼────────────────┐                  │
│               │                │                │                  │
│  ┌────────────▼───┐  ┌─────────▼──────┐  ┌─────▼──────────┐       │
│  │   Dev Account  │  │ Prod Account   │  │ Staging Account│       │
│  │                │  │                │  │                │       │
│  │  Lambda (c7n)  │  │  Lambda (c7n)  │  │  Lambda (c7n)  │       │
│  │  CloudWatch    │  │  CloudWatch    │  │  CloudWatch    │       │
│  │  EventBridge   │  │  EventBridge   │  │  EventBridge   │       │
│  └────────────────┘  └────────────────┘  └────────────────┘       │
│               │                │                │                  │
│               └────────────────┼────────────────┘                  │
│                                │                                    │
│                    ┌───────────▼──────────┐                         │
│                    │  SNS / Slack / JIRA  │  (Notifications)        │
│                    └──────────────────────┘                         │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Policy Domains

### Tagging Policies
| Policy | Action | Trigger |
|---|---|---|
| `require-mandatory-tags` | Notify → quarantine after 7 days | Daily scheduled |
| `detect-untagged-ec2` | Tag with `owner:unknown`, notify | Real-time (CloudTrail) |
| `enforce-tag-values` | Notify team lead | Daily scheduled |
| `auto-tag-creator` | Tag with `created-by` on launch | Real-time (CloudTrail) |

### Security Guardrails
| Policy | Action | Trigger |
|---|---|---|
| `remediate-open-ssh` | Revoke SG rule + notify | Real-time (CloudTrail) |
| `remediate-open-rdp` | Revoke SG rule + notify | Real-time (CloudTrail) |
| `block-public-s3-bucket` | Set bucket ACL to private | Real-time (CloudTrail) |
| `detect-root-login` | Alert + notify security team | Real-time (CloudTrail) |
| `disable-unused-iam-keys` | Disable key after 90 days | Daily scheduled |
| `revoke-overly-permissive-roles` | Notify + flag for review | Daily scheduled |
| `detect-ecr-public-image` | Notify DevOps team | Real-time (EventBridge) |

### Cost Controls
| Policy | Action | Trigger |
|---|---|---|
| `stop-idle-ec2` | Notify → stop after 48h | Daily scheduled |
| `delete-unattached-ebs` | Notify → snapshot → delete after 7d | Daily scheduled |
| `delete-unused-eips` | Release after 24h | Daily scheduled |
| `rightsize-overprovisioned-ec2` | Generate resize recommendation | Weekly scheduled |
| `delete-old-snapshots` | Delete snapshots > 90 days | Weekly scheduled |
| `stop-dev-resources-offhours` | Stop EC2/RDS in dev accounts at 8pm | Scheduled (cron) |
| `delete-unused-nat-gateways` | Notify → delete after 48h | Daily scheduled |

### Compliance
| Policy | Action | Trigger |
|---|---|---|
| `enforce-s3-encryption` | Enable AES256 encryption | Real-time (CloudTrail) |
| `enforce-ebs-encryption` | Notify + remediate | Real-time (CloudTrail) |
| `require-vpc-flow-logs` | Enable flow logs | Daily scheduled |
| `enforce-cloudtrail-enabled` | Re-enable if disabled | Real-time (CloudTrail) |
| `detect-config-disabled` | Alert security team | Real-time (EventBridge) |
| `enforce-mfa-delete-s3` | Notify bucket owner | Daily scheduled |

---

## Repository Structure

```
cloud-custodian-policies/
├── policies/
│   ├── tagging/
│   │   ├── require-mandatory-tags.yml
│   │   ├── auto-tag-creator.yml
│   │   └── enforce-tag-values.yml
│   ├── security/
│   │   ├── remediate-open-sg.yml
│   │   ├── block-public-s3.yml
│   │   ├── detect-root-login.yml
│   │   ├── disable-unused-iam-keys.yml
│   │   └── ecr-public-image.yml
│   ├── cost/
│   │   ├── stop-idle-ec2.yml
│   │   ├── delete-unattached-ebs.yml
│   │   ├── delete-old-snapshots.yml
│   │   ├── stop-dev-offhours.yml
│   │   └── rightsize-recommendations.yml
│   └── compliance/
│       ├── enforce-s3-encryption.yml
│       ├── enforce-ebs-encryption.yml
│       ├── require-vpc-flow-logs.yml
│       └── enforce-cloudtrail.yml
├── deployment/
│   ├── c7n-org-runner.sh
│   ├── accounts.yml
│   └── lambda-deploy.sh
├── notifications/
│   ├── slack-notify.yml
│   └── email-notify.yml
├── tests/
│   └── pytest/
│       ├── test_tagging_policies.py
│       └── test_security_policies.py
├── terraform/
│   ├── c7n-lambda-role.tf
│   └── sqs-dead-letter.tf
└── docs/
    ├── policy-authoring-guide.md
    ├── remediation-approval-flow.md
    └── org-deployment-guide.md
```

---

## Prerequisites

| Requirement | Version |
|---|---|
| Python | >= 3.11 |
| Cloud Custodian (`c7n`) | >= 0.9 |
| `c7n-org` | >= 0.6 |
| AWS CLI | >= 2.x |
| Terraform | >= 1.5 (for Lambda deployment) |

### Installation

```bash
python -m venv .venv
source .venv/bin/activate
pip install c7n c7n-org
```

---

## Usage

### Run a Single Policy (Single Account)

```bash
custodian run \
  --output-dir=output/ \
  policies/cost/stop-idle-ec2.yml
```

### Run Across All Accounts (Org-wide)

```bash
c7n-org run \
  --accounts accounts.yml \
  --output-dir s3://company-c7n-output/ \
  --region us-east-1 \
  -p policies/security/remediate-open-sg.yml
```

### Dry Run (No Actions Taken)

```bash
custodian run \
  --output-dir=output/ \
  --dryrun \
  policies/tagging/require-mandatory-tags.yml
```

### Generate a Report

```bash
custodian report \
  --output-dir=output/ \
  --format csv \
  policies/cost/delete-unattached-ebs.yml > report.csv
```

---

## Sample Policies

### Detect and Notify on Untagged EC2

```yaml
policies:
  - name: detect-untagged-ec2
    resource: ec2
    description: |
      Identify EC2 instances missing mandatory tags and notify the responsible team.
      After 7 days, the instance is stopped; after 14 days, it is terminated.
    filters:
      - "tag:team": absent
      - "tag:environment": absent
      - "tag:cost-center": absent
      - State.Name: running
    actions:
      - type: tag
        tags:
          Custodian: "untagged-ec2"
      - type: notify
        template: default
        priority_header: "2"
        subject: "[Cloud Custodian] Untagged EC2 detected - action required"
        to:
          - resource-owner
          - slack://cloud-governance
        transport:
          type: sqs
          queue: https://sqs.us-east-1.amazonaws.com/123456789/c7n-mailer
```

### Remediate Open SSH Security Group

```yaml
policies:
  - name: remediate-open-ssh
    resource: security-group
    description: |
      Detect and immediately revoke security group rules that allow SSH (22)
      from 0.0.0.0/0 or ::/0. Notify the owner and log to security channel.
    mode:
      type: cloudtrail
      role: arn:aws:iam::{account_id}:role/CloudCustodian
      events:
        - source: ec2.amazonaws.com
          event: AuthorizeSecurityGroupIngress
          ids: "requestParameters.groupId"
    filters:
      - type: ingress
        Ports: [22]
        Cidr:
          value: "0.0.0.0/0"
    actions:
      - type: remove-permissions
        ingress: matched
      - type: notify
        subject: "[SECURITY] Open SSH SG remediated"
        to:
          - resource-owner
          - slack://security-alerts
        transport:
          type: sqs
          queue: https://sqs.us-east-1.amazonaws.com/123456789/c7n-mailer
```

### Stop Idle EC2 Instances

```yaml
policies:
  - name: stop-idle-ec2
    resource: ec2
    description: |
      Stop EC2 instances with avg CPU < 5% for 7 days. Notifies owner 48h before action.
    filters:
      - State.Name: running
      - type: metrics
        name: CPUUtilization
        statistics: Average
        days: 7
        value: 5
        op: less-than
      - "tag:environment":
          not-in: [prod, staging]
    actions:
      - type: notify
        subject: "[Cost] Idle EC2 will be stopped in 48h"
        to:
          - resource-owner
          - slack://cost-governance
        transport:
          type: sqs
          queue: https://sqs.us-east-1.amazonaws.com/123456789/c7n-mailer
      - type: stop
        # grace period handled by separate 48h-delay policy
```

---

## Notification Setup

Notifications use the `c7n-mailer` Lambda. Configure Slack and email:

```yaml
# notifications/slack-notify.yml
queue_url: https://sqs.us-east-1.amazonaws.com/123456789/c7n-mailer
slack_token: "xoxb-your-slack-token"
from_address: cloud-governance@company.com
contact_tags:
  - owner
  - team
```

Deploy the mailer:

```bash
c7n-mailer --config notifications/slack-notify.yml --update-lambda
```

---

## Org-wide Deployment

Define target accounts and assume-role ARNs:

```yaml
# deployment/accounts.yml
accounts:
  - name: dev-account
    account_id: "111111111111"
    role: arn:aws:iam::111111111111:role/CloudCustodian
    regions: [us-east-1, eu-west-1]

  - name: prod-account
    account_id: "222222222222"
    role: arn:aws:iam::222222222222:role/CloudCustodian
    regions: [us-east-1, eu-west-1, ap-southeast-1]
```

Run all security policies org-wide:

```bash
c7n-org run \
  --accounts deployment/accounts.yml \
  --output-dir s3://company-c7n-output/$(date +%Y-%m-%d)/ \
  -p policies/security/
```

---

## IAM Role Requirements

Each target account must have a cross-account role with the minimum permissions needed per policy domain. A Terraform module is provided:

```bash
cd terraform/
terraform apply -var="management_account_id=123456789012"
```

The role grants read and selective write access:

```json
{
  "Effect": "Allow",
  "Action": [
    "ec2:Describe*",
    "ec2:StopInstances",
    "ec2:RevokeSecurityGroupIngress",
    "s3:GetBucketAcl",
    "s3:PutBucketAcl",
    "iam:ListAccessKeys",
    "iam:UpdateAccessKey",
    "cloudtrail:GetTrailStatus",
    "cloudtrail:StartLogging"
  ],
  "Resource": "*"
}
```

---

## Testing

Policies are unit-tested using `pytest` with mocked AWS resources via `placebo`:

```bash
pip install pytest placebo
pytest tests/pytest/ -v
```

---

## Contributing

1. Add new policies under the appropriate domain folder
2. Include a `description` field in every policy — this is required for the policy registry
3. Test with `--dryrun` before enabling real actions
4. For destructive policies (terminate, delete), add a `notify` action with a 48h+ lead time
5. Submit a PR with the blast radius and affected resource types documented

---

## License

MIT — see [LICENSE](LICENSE) for details.
