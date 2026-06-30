# Kentik AWS CloudFormation Deployment

Deploys the Kentik IAM roles and policies across AWS accounts for two collection types:

- **Metadata collection** — EC2, DirectConnect, ELB, IAM, Network Firewall, Network Manager, and optional CloudWatch metrics
- **Flow collection** — VPC Flow Logs read from an S3 bucket

Three architectures are supported:

- **Standard** — Kentik connects directly to each account. Use this when you have one account or don't need a centralized hub.
- **Nested (Hub/Spoke)** — Kentik connects to one hub account, which assumes roles in each spoke account. Use this for multi-account environments where the hub account is the AWS Organizations management account.
- **Nested (Hub/Spoke, Non-Org Account)** — Same as above, but the hub account is not the Organizations management account. Requires an additional role in the management account to delegate `organizations:ListAccounts`.

---

## Templates

| File | Purpose |
|---|---|
| `kentik-standard-account-cfn.yaml` | Metadata collection — standard (direct) architecture |
| `kentik-standard-account-flow-cfn.yaml` | Flow collection — standard (direct) architecture |
| `kentik-hub-account-cfn.yaml` | Hub account — nested architecture |
| `kentik-spoke-account-cfn.yaml` | Spoke accounts — metadata, nested architecture |
| `kentik-org-account-cfn.yaml` | Management account — nested non-org architecture only |

---

## Customizing Permissions

The metadata policy grants permissions for every AWS networking service Kentik supports. If your environment doesn't use certain services, you can remove the corresponding actions from the policy before deploying to follow least-privilege.

Edit `kentik-standard-account-cfn.yaml` and/or `kentik-spoke-account-cfn.yaml` and delete any comment block and its actions that don't apply:

| Comment block | Actions | Remove if… |
|---|---|---|
| `# CloudWatch (optional)` | `cloudwatch:ListMetrics`, `GetMetricStatistics`, `GetMetricData` | You don't need CloudWatch metrics in Kentik |
| `# DirectConnect` | `directconnect:Describe*` | You don't use AWS Direct Connect |
| `# Network Firewall` | `network-firewall:List*`, `Describe*` | You don't use AWS Network Firewall |
| `# Network Manager` | `networkmanager:List*`, `Get*` | You don't use AWS Cloud WAN |

The EC2 and ELB actions are required for core metadata collection and should not be removed.

> **Tip:** Save a copy of any removed blocks before deploying. If you later adopt one of these services, you can add the actions back and update the stack with `aws cloudformation deploy` — no role recreation required.

---

## Standard Configuration

Deploy templates directly in each account you want Kentik to monitor. Metadata and flow collection use separate roles and can be deployed independently.

### Prerequisites

- AWS CLI configured with credentials for the target account
- Your **Kentik Company ID** — found in the Kentik portal under **Settings → Licenses** (the "Account #" field)
- For flow collection: the name of the S3 bucket where VPC Flow Logs are stored

### Deploy — Metadata

Run once per account:

```bash
aws cloudformation deploy \
  --template-file kentik-standard-account-cfn.yaml \
  --stack-name kentik-metadata \
  --parameter-overrides KentikCompanyID=<your-company-id> \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-east-1
```

### Deploy — Flow Collection

Run once per account that has VPC Flow Logs to collect:

```bash
aws cloudformation deploy \
  --template-file kentik-standard-account-flow-cfn.yaml \
  --stack-name kentik-flow \
  --parameter-overrides \
    KentikCompanyID=<your-company-id> \
    S3BucketName=<your-flow-log-bucket-name> \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-east-1
```

After deployment, retrieve the role ARNs from the stack outputs:

```bash
aws cloudformation describe-stacks \
  --stack-name kentik-metadata \
  --query "Stacks[0].Outputs"

aws cloudformation describe-stacks \
  --stack-name kentik-flow \
  --query "Stacks[0].Outputs"
```

### Configure Kentik

1. Log into the [Kentik portal](https://portal.kentik.com)
2. Navigate to **Settings → Cloud → AWS**
3. Add a new AWS account and enter the **metadata role ARN** and **flow role ARN** from the stack outputs
4. Repeat for each additional account

### Resources Created

| Template | Resource | Purpose |
|---|---|---|
| Metadata | `KentikMetadataPolicy` | Read-only metadata permissions (EC2, DirectConnect, ELB, etc.) |
| Metadata | `KentikMetadataRole` | Assumed directly by Kentik's AWS account (`834693425129`) |
| Flow | `KentikFlowPolicy` | Read access to the VPC Flow Logs S3 bucket |
| Flow | `KentikFlowRole` | Assumed directly by Kentik's AWS account (`834693425129`) |

---

## Nested (Hub/Spoke) Configuration

Deploys IAM roles across multiple AWS accounts using a hub-and-spoke architecture. Kentik connects to one **hub** account, which assumes roles in each **spoke** account to collect metadata.

The spoke template covers metadata collection only. For flow log collection in spoke accounts, deploy `kentik-standard-account-flow-cfn.yaml` directly in each spoke account using the Standard flow instructions above.

### Prerequisites

- AWS CLI configured with credentials for each target account (or permission to use CloudFormation StackSets)
- Your **Kentik Company ID** — found in the Kentik portal under **Settings → Licenses** (the "Account #" field)
- AWS Organizations enabled if you plan to deploy via StackSets

### Step 1 — Deploy the Hub Account

Run this once in the account Kentik will connect to directly:

```bash
aws cloudformation deploy \
  --template-file kentik-hub-account-cfn.yaml \
  --stack-name kentik-metadata-hub \
  --parameter-overrides KentikCompanyID=<your-company-id> \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-east-1
```

Retrieve the hub account ID from the stack outputs — you'll need it in Step 2:

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

#### Option B: All accounts via StackSets (recommended for large orgs)

This deploys to an entire AWS Organization OU in one command:

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
| Hub | `KentikMetadataPrimaryPolicy` | Allows hub role to assume spoke roles and list org accounts |
| Hub | `KentikMetadataPrimaryRole` | Assumed by Kentik's AWS account (`834693425129`) |
| Spoke | `KentikMetadataSecondaryPolicy` | Read-only metadata permissions (EC2, DirectConnect, ELB, etc.) |
| Spoke | `KentikMetadataSecondaryRole` | Assumed by the hub role to collect spoke account metadata |

---

## Nested Accounts (Non-Org Account)

Use this when you are deploying the hub/spoke architecture and the **hub account is not the AWS Organizations management account**. `organizations:ListAccounts` can only be called from the management account (or a delegated admin). This configuration deploys all three templates: hub, spoke, and an additional role in the management account that the hub assumes to perform account listing.

If your hub account **is** the management account, use the [Nested (Hub/Spoke) Configuration](#nested-hubspoke-configuration) section instead.

### Prerequisites

- AWS CLI profiles or credential contexts for three account types:
  - **Hub account** — the account Kentik connects to directly
  - **Spoke accounts** — accounts Kentik collects metadata from
  - **Management account** — the AWS Organizations management account
- Your **Kentik Company ID** — found in the Kentik portal under **Settings → Licenses** (the "Account #" field)

### Step 1 — Deploy the Hub Account

Switch credentials to the hub account and run:

```bash
aws cloudformation deploy \
  --template-file kentik-hub-account-cfn.yaml \
  --stack-name kentik-metadata-hub \
  --parameter-overrides KentikCompanyID=<your-company-id> \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-east-1
```

Retrieve the hub account ID and primary role name from the stack outputs — you will need them in Steps 2 and 3:

```bash
aws cloudformation describe-stacks \
  --stack-name kentik-metadata-hub \
  --query "Stacks[0].Outputs"
```

Note the `HubAccountId` and `PrimaryRoleArn` values.

### Step 2 — Deploy to Spoke Accounts

#### Option A: Single account (AWS CLI)

Switch credentials to each spoke account and repeat:

```bash
aws cloudformation deploy \
  --template-file kentik-spoke-account-cfn.yaml \
  --stack-name kentik-metadata-spoke \
  --parameter-overrides HubAccountId=<hub-account-id> \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-east-1
```

#### Option B: All accounts via StackSets (recommended for large orgs)

Run from an account with StackSets permissions:

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

### Step 3 — Deploy to the Management Account

Switch credentials to the AWS Organizations management account and run:

```bash
aws cloudformation deploy \
  --template-file kentik-org-account-cfn.yaml \
  --stack-name kentik-metadata-org \
  --parameter-overrides \
    HubAccountId=<hub-account-id> \
    HubRoleName=<hub-role-name> \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-east-1
```

`HubRoleName` defaults to `KentikMetadataPrimaryRole` — only override if you changed it during the hub deployment.

Retrieve the org role ARN from the stack outputs:

```bash
aws cloudformation describe-stacks \
  --stack-name kentik-metadata-org \
  --query "Stacks[0].Outputs"
```

Note the `OrgRoleArn` value — you will provide it to Kentik in Step 4.

### Step 4 — Configure Kentik

1. Log into the [Kentik portal](https://portal.kentik.com)
2. Navigate to **Settings → Cloud → AWS**
3. Add a new AWS account and enter the **hub role ARN** (`PrimaryRoleArn`) from the Step 1 stack outputs
4. When prompted for an organization role, enter the **org role ARN** (`OrgRoleArn`) from the Step 3 stack outputs
5. Kentik will assume the org role to list accounts, then assume the spoke roles to collect metadata

### Resources Created

| Template | Resource | Purpose |
|---|---|---|
| Hub | `KentikMetadataPrimaryPolicy` | Allows hub role to assume spoke roles and the org role |
| Hub | `KentikMetadataPrimaryRole` | Assumed by Kentik's AWS account (`834693425129`) |
| Spoke | `KentikMetadataSecondaryPolicy` | Read-only metadata permissions (EC2, DirectConnect, ELB, etc.) |
| Spoke | `KentikMetadataSecondaryRole` | Assumed by the hub role to collect spoke account metadata |
| Org | `KentikMetadataOrgPolicy` | Grants `organizations:ListAccounts` in the management account |
| Org | `KentikMetadataOrgRole` | Assumed by the hub role to list org accounts cross-account |

---

## Reference

- [Kentik KB: Metadata Configuration](https://kb.kentik.com/docs/metadata-configuration-aws)
- [Kentik KB: AWS Endpoints List](https://kb.kentik.com/docs/aws-endpoints-list)
