# Kentik AWS Metadata Collection — CloudFormation Deployment

Deploys the Kentik IAM roles and policies across AWS accounts. Two architectures are supported:

- **Standard** — Kentik connects directly to each account. Use this when you have one account or don't need a centralized hub.
- **Nested (Hub/Spoke)** — Kentik connects to one hub account, which assumes roles in each spoke account. Use this for multi-account environments.

---

## Standard Configuration

Deploy `kentik-standard-account-cfn.yaml` independently in every account you want Kentik to monitor.

### Prerequisites

- AWS CLI configured with credentials for the target account
- Your **Kentik Company ID** — found in the Kentik portal under **Settings → Licenses** (the "Account #" field)

### Deploy

Run this once per account:

```bash
aws cloudformation deploy \
  --template-file kentik-standard-account-cfn.yaml \
  --stack-name kentik-metadata \
  --parameter-overrides KentikCompanyID=<your-company-id> \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-east-1
```

If Kentik needs access to a VPC Flow Log S3 bucket in that account, add:

```
S3BucketName=<your-flow-log-bucket-name>
```

After deployment, grab the role ARN from the stack outputs:

```bash
aws cloudformation describe-stacks \
  --stack-name kentik-metadata \
  --query "Stacks[0].Outputs"
```

### Configure Kentik

1. Log into the [Kentik portal](https://portal.kentik.com)
2. Navigate to **Settings → Cloud → AWS**
3. Add a new AWS account and enter the **role ARN** from the stack outputs
4. Repeat for each additional account

### Resources Created

| Resource | Purpose |
|---|---|
| `KentikMetadataPolicy` | Read-only metadata permissions (EC2, CloudWatch, etc.) |
| `KentikMetadataRole` | Assumed directly by Kentik's AWS account (`834693425129`) |

---

## Nested (Hub/Spoke) Configuration

Deploys IAM roles across multiple AWS accounts using a hub-and-spoke architecture. Kentik connects to one **hub** account, which assumes roles in each **spoke** account to collect metadata.

### Prerequisites

- AWS CLI configured with credentials for each target account (or permission to use CloudFormation StackSets)
- Your **Kentik Company ID** — found in the Kentik portal under **Settings → Licenses** (the "Account #" field)
- AWS Organizations enabled if you plan to deploy via StackSets

### Step 1 — Deploy the Hub Account

Run this once in the account Kentik will connect to directly.

```bash
aws cloudformation deploy \
  --template-file kentik-hub-account-cfn.yaml \
  --stack-name kentik-metadata-hub \
  --parameter-overrides KentikCompanyID=<your-company-id> \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-east-1
```

After deployment, grab the hub account ID from the stack outputs — you'll need it in Step 2:

```bash
aws cloudformation describe-stacks \
  --stack-name kentik-metadata-hub \
  --query "Stacks[0].Outputs"
```

### Step 2 — Deploy to Spoke Accounts

#### Option A: Single account (AWS CLI)

Repeat for each spoke account, substituting the correct credentials and hub account ID:

```bash
aws cloudformation deploy \
  --template-file kentik-spoke-account-cfn.yaml \
  --stack-name kentik-metadata-spoke \
  --parameter-overrides HubAccountId=<hub-account-id> \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-east-1
```

If Kentik needs access to a VPC Flow Log S3 bucket in that account, add:

```
S3BucketName=<your-flow-log-bucket-name>
```

#### Option B: All accounts via StackSets (recommended for large orgs)

This deploys to an entire AWS Organization OU in one command.

```bash
# 1. Create the StackSet
aws cloudformation create-stack-set \
  --stack-set-name kentik-metadata-spoke \
  --template-body file://kentik-spoke-account-cfn.yaml \
  --parameters ParameterKey=HubAccountId,ParameterValue=<hub-account-id> \
  --capabilities CAPABILITY_NAMED_IAM \
  --permission-model SERVICE_MANAGED \
  --auto-deployment Enabled=true,RetainStacksOnAccountRemoval=false

# 2. Deploy to an OU (replace ou-xxxx-xxxxxxxx with your target OU ID)
aws cloudformation create-stack-instances \
  --stack-set-name kentik-metadata-spoke \
  --deployment-targets OrganizationalUnitIds=ou-xxxx-xxxxxxxx \
  --regions us-east-1 \
  --operation-preferences FailureToleranceCount=2,MaxConcurrentCount=5
```

### Step 3 — Configure Kentik

1. Log into the [Kentik portal](https://portal.kentik.com)
2. Navigate to **Settings → Cloud → AWS**
3. Add a new AWS account and enter the **hub role ARN** from the Step 1 stack outputs
4. Kentik will automatically discover and assume the spoke roles

### Resources Created

| Template | Resource | Purpose |
|---|---|---|
| Hub | `KentikMetadataPrimaryPolicy` | Allows hub role to assume spoke roles + list org accounts |
| Hub | `KentikMetadataPrimaryRole` | Assumed by Kentik's AWS account (`834693425129`) |
| Spoke | `KentikMetadataSecondaryPolicy` | Read-only metadata permissions (EC2, CloudWatch, etc.) |
| Spoke | `KentikMetadataSecondaryRole` | Assumed by the hub role to collect spoke account metadata |

---

## Reference

- [Kentik KB: Metadata Configuration](https://kb.kentik.com/docs/metadata-configuration-aws)
