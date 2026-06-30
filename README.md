# Kentik AWS CloudFormation Deployment

Deploys Kentik IAM roles and policies across AWS accounts for two collection types:

- **Metadata collection** — network topology via EC2, DirectConnect, ELB, IAM, Network Manager, and CloudWatch
- **Flow collection** — VPC Flow Logs read from centralized regional S3 buckets

---

## Architecture

### Metadata Collection — Hub/Spoke

Kentik connects to one hub account, which assumes roles in each spoke account to collect metadata. The spoke account list is registered with Kentik during POC onboarding — no `organizations:ListAccounts` permission is required.

```
Kentik AWS Account (834693425129)
        │
        │  sts:AssumeRole
        │  + ExternalId (Kentik Company ID)
        │  + PrincipalArn (eks-ingest-node)
        ▼
   Hub Account
   KentikMetadataPrimaryRole
        │
        │  sts:AssumeRole
        ├──────────────────────────────────┐
        ▼                                  ▼
   Spoke Account 1              Spoke Account N
   KentikMetadataSecondaryRole  KentikMetadataSecondaryRole
   (read-only metadata)         (read-only metadata)
```

### Flow Log Collection — Direct to Centralized Logging Account

Kentik connects directly to the centralized logging account where all spoke accounts ship their VPC Flow Logs. The hub role is not involved.

```
Kentik AWS Account (834693425129)
        │
        │  sts:AssumeRole
        │  + ExternalId (Kentik Company ID)
        │  + PrincipalArn (eks-ingest-node)
        ▼
   Central Logging Account
   KentikFlowRole
        │
        │  s3:GetObject / s3:ListBucket / s3:GetBucketLocation
        ├──────────────────────┬────────────────────────┐
        ▼                      ▼                        ▼
   s3://bucket-us-east-1  s3://bucket-us-east-2  s3://bucket-us-west-2
   (flow logs from         (flow logs from        (flow logs from
    all spoke accounts)     all spoke accounts)    all spoke accounts)
```

---

## Templates

| File | Purpose | Deployed In |
|---|---|---|
| `kentik-hub-account-cfn.yaml` | Creates the hub role Kentik assumes | Hub account — once |
| `kentik-spoke-account-cfn.yaml` | Creates secondary role for metadata collection | Every spoke account — via StackSets |
| `kentik-standard-account-flow-cfn.yaml` | Creates flow role for centralized S3 access | Central logging account — once |

---

## Prerequisites

- AWS CLI configured with profiles for the hub account, spoke accounts, and central logging account
- **Kentik Company ID** — found in the Kentik portal under **Settings → Licenses** (the "Account #" field)
- **Hub account ID** — retrieved from step 1 stack outputs
- **Spoke account IDs** — the accounts registered with Kentik during POC onboarding; must match exactly
- **S3 bucket names** — the regional centralized flow log bucket names

---

## Deployment Summary

### Step 1 — Deploy Hub Account

```bash
aws cloudformation deploy \
  --profile <hub-account-profile> \
  --template-file kentik-hub-account-cfn.yaml \
  --stack-name kentik-metadata-hub \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-east-1 \
  --parameter-overrides KentikCompanyID=<your-kentik-company-id>
```

### Step 2 — Deploy Spoke Accounts via StackSets

Run from the **management (master) account**. The spoke account IDs must match the accounts registered with Kentik during onboarding.

```bash
aws cloudformation create-stack-set \
  --profile <management-account-profile> \
  --stack-set-name kentik-metadata-spoke \
  --template-body file://kentik-spoke-account-cfn.yaml \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameters ParameterKey=HubAccountId,ParameterValue=<hub-account-id>

aws cloudformation create-stack-instances \
  --profile <management-account-profile> \
  --stack-set-name kentik-metadata-spoke \
  --accounts 111122223333 444455556666 777788889999 \
  --regions us-east-1
```

### Step 3 — Deploy Central Logging Account (Flow Logs)

```bash
aws cloudformation deploy \
  --profile <logging-account-profile> \
  --template-file kentik-standard-account-flow-cfn.yaml \
  --stack-name kentik-flow \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-east-1 \
  --parameter-overrides file://flow-params.json
```

### Step 4 — Configure Kentik Portal

1. Log into the [Kentik portal](https://portal.kentik.com)
2. Navigate to **Settings → Cloud → AWS**
3. Add a new AWS account and enter the **hub role ARN** from step 1 outputs
4. Enter the **flow role ARN** from step 3 outputs for flow log collection

---

For full parameter reference, parameter file examples, security details, and troubleshooting see [DEPLOYMENT.md](DEPLOYMENT.md).

---

## Reference

- [Kentik KB: Metadata Configuration](https://kb.kentik.com/docs/metadata-configuration-aws)
- [Kentik KB: AWS Endpoints List](https://kb.kentik.com/docs/aws-endpoints-list)
