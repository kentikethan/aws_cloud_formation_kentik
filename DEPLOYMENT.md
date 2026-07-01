# Kentik AWS CloudFormation — Detailed Deployment Guide

This guide covers prerequisites, full parameter reference, deployment commands with parameter file examples, security notes, and troubleshooting. For a quick overview see [README.md](README.md).

---

## Table of Contents

1. [Architecture Deep Dive](#architecture-deep-dive)
2. [Templates Reference](#templates-reference)
3. [Prerequisites](#prerequisites)
4. [Step 1 — Create Kentik IAM Role in Hub Account](#step-1--create-kentik-iam-role-in-hub-account)
5. [Step 2 — Create Kentik IAM Roles in Spoke Accounts via StackSets](#step-2--create-kentik-iam-roles-in-spoke-accounts-via-stacksets)
6. [Step 3 — Create Kentik IAM Role in Central Logging Account](#step-3--create-kentik-iam-role-in-central-logging-account)
7. [Step 4 — Kentik Portal Configuration](#step-4--kentik-portal-configuration)
8. [Security Design Notes](#security-design-notes)
9. [Customization](#customization)
10. [Troubleshooting](#troubleshooting)

---

## Architecture Deep Dive

### Metadata Collection

Kentik uses a **hub/spoke IAM chain** to collect network metadata (EC2, DirectConnect, ELB, IAM account alias, CloudWatch, Network Manager) from all spoke accounts.

```
Kentik AWS Account (834693425129)
        │
        │  sts:AssumeRole + ExternalId + PrincipalArn condition
        ▼
Kentik Hub Account (existing)
KentikMetadataPrimaryRole  ← IAM role created by kentik-hub-account-cfn.yaml
  └── Policy: sts:AssumeRole on arn:aws:iam::*:role/${SecondaryRoleName}
        │
        │  sts:AssumeRole
        ├─────────────────────────┬─────────────────────────┐
        ▼                         ▼                         ▼
Spoke Account A            Spoke Account B           Spoke Account N
KentikMetadataSecondaryRole KentikMetadataSecondaryRole  ...
  └── Read-only metadata      └── Read-only metadata
```

**What is collected per spoke account:**

| Service | Actions |
|---|---|
| CloudWatch | GetMetricData, GetMetricStatistics, ListMetrics |
| DirectConnect | Describe\* (all) |
| EC2 | Describe\* (all) |
| Elastic Load Balancing | Describe\* (all) |
| IAM | ListAccountAliases |
| Network Manager | GetCoreNetworkPolicy, DescribeGlobalNetworks, GetLinks, GetDevices, GetSites, ListCoreNetworkPolicyVersions |

### Flow Log Collection

Kentik connects **directly** to the central logging account. The hub role is not involved. All VPC Flow Logs from spoke accounts are routed to regional S3 buckets in the central logging account.

```
Spoke VPCs (multiple accounts/regions)
        │  CloudWatch → Kinesis / VPC Flow Logs → S3
        ▼
Central Logging Account
  ├── s3://flow-bucket-us-east-1/
  │     └── {account-id}/{region}/{vpc-id}/
  ├── s3://flow-bucket-us-east-2/
  │     └── {account-id}/{region}/{vpc-id}/
  └── s3://flow-bucket-us-west-2/
        └── {account-id}/{region}/{vpc-id}/
        │
        │  (Kentik assumes role in this account)
        ▼
KentikFlowRole
  └── Policy: s3:GetBucketLocation + s3:ListBucket + s3:GetObject
              scoped to exact named buckets
```

**Trust chain for both roles:**

Both `KentikMetadataPrimaryRole` (hub) and `KentikFlowRole` (logging account) apply the same double-lock on trust:

1. **ExternalId** (`sts:ExternalId`) — your Kentik Company ID. Prevents confused-deputy attacks if your account ARN were reused by another Kentik customer.
2. **PrincipalArn** (`aws:PrincipalArn`) — locked to `arn:aws:iam::834693425129:role/eks-ingest-node`. Prevents any other principal in Kentik's account (including root) from assuming the role.

---

## Templates Reference

### `kentik-hub-account-cfn.yaml`

Deployed once into your existing Kentik Hub account. Creates the IAM roles — it does not create the account itself. Creates:
- **`KentikMetadataPrimaryRole`** — trusted by Kentik's account; can assume secondary roles in any spoke account
- **`KentikMetadataPrimaryPolicy`** — `sts:AssumeRole` on `arn:aws:iam::*:role/${SecondaryRoleName}`

| Parameter | Type | Default | Description |
|---|---|---|---|
| `KentikCompanyID` | String | _(required)_ | Your Kentik Company ID. Used as `ExternalId` in the trust policy. |
| `PolicyName` | String | `KentikMetadataPrimaryPolicy` | IAM policy name to create. |
| `RoleName` | String | `KentikMetadataPrimaryRole` | IAM role name to create. |
| `SecondaryRoleName` | String | `KentikMetadataSecondaryRole` | Name of the secondary role deployed in spoke accounts. Must match the `RoleName` in the spoke template. |

**Outputs:**

| Output | Description |
|---|---|
| `RoleArn` | ARN of the Kentik hub role — enter this in the Kentik portal |
| `RoleName` | Name of the Kentik hub role |
| `AccountId` | Kentik Hub account ID — required as input for the spoke template |

---

### `kentik-spoke-account-cfn.yaml`

Deployed in every spoke account (via StackSets). Creates:
- **`KentikMetadataSecondaryRole`** — trusted by the hub role ARN; grants read-only metadata access

| Parameter | Type | Default | Description |
|---|---|---|---|
| `HubAccountId` | String | _(required)_ | AWS account ID of the hub account. Used to scope the trust policy to the exact hub role. |
| `HubRoleName` | String | `KentikMetadataPrimaryRole` | Name of the hub role. Combined with `HubAccountId` to form the trust ARN. |
| `PolicyName` | String | `KentikMetadataSecondaryPolicy` | IAM policy name to create. |
| `RoleName` | String | `KentikMetadataSecondaryRole` | IAM role name to create. Must match `SecondaryRoleName` in the hub template. |

**Outputs:**

| Output | Description |
|---|---|
| `RoleArn` | ARN of the secondary role |
| `RoleName` | Name of the secondary role |
| `AccountId` | Spoke account ID |

---

### `kentik-standard-account-flow-cfn.yaml`

Deployed once in the **central logging account** — the account that owns the regional S3 buckets where all spoke VPC Flow Logs are stored. Creates:
- **`KentikFlowRole`** — trusted directly by Kentik's account; grants read-only S3 access
- **`KentikFlowPolicy`** — scoped to exact bucket names (no wildcards)

| Parameter | Type | Default | Description |
|---|---|---|---|
| `KentikCompanyID` | String | _(required)_ | Your Kentik Company ID. Used as `ExternalId` in the trust policy. |
| `PolicyName` | String | `KentikFlowPolicy` | IAM policy name to create. |
| `RoleName` | String | `KentikFlowRole` | IAM role name to create. |
| `S3BucketNames` | CommaDelimitedList | _(required)_ | Comma-separated names of regional S3 buckets (e.g. `bucket-us-east-1,bucket-us-east-2`). |

**Outputs:**

| Output | Description |
|---|---|
| `RoleArn` | ARN of the flow role — enter this in the Kentik portal |
| `RoleName` | Name of the flow role |
| `AccountId` | Logging account ID |

---

## Prerequisites

**Tools:**
- AWS CLI v2 (`aws --version`)
- Permissions to deploy CloudFormation stacks in hub, spoke, and logging accounts
- StackSets enabled in the management account (or delegated admin account) if using StackSets for spoke deployment

**Information to collect before starting:**

| Item | Where to find it |
|---|---|
| Kentik Company ID | Kentik portal → Settings → Licenses → "Account #" field |
| Kentik Hub account ID | Your existing AWS account designated as the Kentik Hub |
| Spoke account IDs | Your manual list of accounts Kentik should monitor |
| Central logging account ID | The account owning the flow log S3 buckets |
| S3 bucket names (flow logs) | S3 console in the logging account |

---

## Step 1 — Create Kentik IAM Role in Hub Account

Your Kentik Hub account already exists. This step deploys the `KentikMetadataPrimaryRole` IAM role into it so that Kentik's AWS account can assume it and reach spoke accounts.

> **Note on `--region`:** IAM is a global AWS service — the role created here works in every region automatically. The `--region` flag only controls where CloudFormation stores the stack state (events, outputs). Pick any region; `us-east-1` is used here as a convention. You only need to run this once.

```bash
aws cloudformation deploy \
  --profile <kentik-hub-account-profile> \
  --template-file kentik-hub-account-cfn.yaml \
  --stack-name kentik-metadata-hub \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-east-1 \
  --parameter-overrides \
    KentikCompanyID=<your-company-id>
```

Or with a parameters file `hub-params.json`:

```json
[
  {
    "ParameterKey": "KentikCompanyID",
    "ParameterValue": "123456"
  }
]
```

```bash
aws cloudformation deploy \
  --profile <kentik-hub-account-profile> \
  --template-file kentik-hub-account-cfn.yaml \
  --stack-name kentik-metadata-hub \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-east-1 \
  --parameter-overrides file://hub-params.json
```

**Retrieve outputs** (needed for step 2):

```bash
aws cloudformation describe-stacks \
  --profile <kentik-hub-account-profile> \
  --stack-name kentik-metadata-hub \
  --region us-east-1 \
  --query "Stacks[0].Outputs"
```

Note the `RoleArn` and `AccountId` values from the output.

---

## Step 2 — Create Kentik IAM Roles in Spoke Accounts via StackSets

Your spoke accounts already exist. This step deploys the `KentikMetadataSecondaryRole` IAM role into each of them.

> **Where these commands run:** All StackSet commands run **from the management (master/root) account**. CloudFormation handles cross-account deployment into each target spoke account.

> **Note on `--regions us-east-1`:** Since only IAM roles are being created (global service), specifying a single region is correct. The role works across all regions in each spoke account. You do not need to repeat this for multiple regions.

There are two ways to target accounts depending on your environment. Choose the one that fits.

---

### Option A — Organizations-integrated (recommended for large deployments)

Use this if you have AWS Organizations and want CloudFormation to pull the account list automatically. You target an OU or the org root — no need to list individual account IDs.

**First, enable trusted access** between CloudFormation and Organizations (one-time, in the management account):

```bash
aws cloudformation activate-organizations-access \
  --profile <management-account-profile>
```

**Create the StackSet with service-managed permissions:**

```bash
aws cloudformation create-stack-set \
  --profile <management-account-profile> \
  --stack-set-name kentik-metadata-spoke \
  --template-body file://kentik-spoke-account-cfn.yaml \
  --capabilities CAPABILITY_NAMED_IAM \
  --permission-model SERVICE_MANAGED \
  --auto-deployment Enabled=true,RetainStacksOnAccountRemoval=false \
  --parameters \
    ParameterKey=HubAccountId,ParameterValue=<kentik-hub-account-id>
```

**Deploy to all accounts in an OU:**

```bash
aws cloudformation create-stack-instances \
  --profile <management-account-profile> \
  --stack-set-name kentik-metadata-spoke \
  --deployment-targets OrganizationalUnitIds=ou-xxxx-xxxxxxxx \
  --regions us-east-1
```

**Or deploy to the entire organization root:**

```bash
aws cloudformation create-stack-instances \
  --profile <management-account-profile> \
  --stack-set-name kentik-metadata-spoke \
  --deployment-targets RootId=r-xxxx \
  --regions us-east-1
```

With `--auto-deployment Enabled=true`, any new account added to the OU is automatically provisioned with the Kentik role — no manual step required.

> To find your OU ID or root ID: AWS Console → AWS Organizations → AWS accounts, or run:
> ```bash
> aws organizations list-roots --profile <management-account-profile>
> aws organizations list-organizational-units-for-parent --parent-id r-xxxx --profile <management-account-profile>
> ```

---

### Option B — Self-managed (small deployments / no Organizations)

Use this if you have a small, fixed list of spoke accounts. Account IDs are passed explicitly.

**Prerequisites:**
- `AWSCloudFormationStackSetAdministrationRole` must exist in the management account
- `AWSCloudFormationStackSetExecutionRole` must exist in each spoke account (trusting the management account)

See [AWS self-managed StackSets setup](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/stacksets-prereqs-self-managed.html) if these roles don't exist yet.

**Create the StackSet:**

```bash
aws cloudformation create-stack-set \
  --profile <management-account-profile> \
  --stack-set-name kentik-metadata-spoke \
  --template-body file://kentik-spoke-account-cfn.yaml \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameters \
    ParameterKey=HubAccountId,ParameterValue=<kentik-hub-account-id>
```

**Deploy to explicit account list:**

```bash
aws cloudformation create-stack-instances \
  --profile <management-account-profile> \
  --stack-set-name kentik-metadata-spoke \
  --accounts 111122223333 444455556666 777788889999 \
  --regions us-east-1
```

---

### Adding or removing accounts later

```bash
# Add accounts (works for both Option A and B)
aws cloudformation create-stack-instances \
  --profile <management-account-profile> \
  --stack-set-name kentik-metadata-spoke \
  --accounts 222233334444 \
  --regions us-east-1

# Remove accounts
aws cloudformation delete-stack-instances \
  --profile <management-account-profile> \
  --stack-set-name kentik-metadata-spoke \
  --accounts 333344445555 \
  --regions us-east-1 \
  --no-retain-stacks
```

> **Spoke account list and Kentik:** The accounts targeted here must match the accounts registered with Kentik during POC onboarding. Kentik stores that list on their side — roles must exist in every account Kentik was told to monitor.

---

## Step 3 — Create Kentik IAM Role in Central Logging Account

Your central logging account already exists and owns the flow log S3 buckets. This step deploys the `KentikFlowRole` IAM role into it so that Kentik can read flow logs directly from those buckets.

> **Note on `--region`:** Same as step 1 — IAM is global. The region only determines where CloudFormation stores the stack state. Run this once in any region.

Create `flow-params.json`:

```json
[
  {
    "ParameterKey": "KentikCompanyID",
    "ParameterValue": "123456"
  },
  {
    "ParameterKey": "S3BucketNames",
    "ParameterValue": "flow-bucket-us-east-1,flow-bucket-us-east-2,flow-bucket-us-west-2"
  }
]
```

Deploy:

```bash
aws cloudformation deploy \
  --profile <logging-account-profile> \
  --template-file kentik-standard-account-flow-cfn.yaml \
  --stack-name kentik-flow \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-east-1 \
  --parameter-overrides file://flow-params.json
```

**Retrieve flow role ARN:**

```bash
aws cloudformation describe-stacks \
  --profile <logging-account-profile> \
  --stack-name kentik-flow \
  --region us-east-1 \
  --query "Stacks[0].Outputs"
```

Note the `RoleArn` value.

**S3 bucket naming convention for VPC Flow Logs:**

Buckets are expected to contain objects with the path structure:
```
{bucket-name}/{account-id}/{region}/{vpc-id}/...
```

Example:
```
flow-bucket-us-east-2/015035640618/us-east-2/vpc-040d5ba53061fd8d7/...
```

The policy grants `s3:GetObject` on `arn:aws:s3:::{bucket-name}/*` which covers this full sub-hierarchy.

---

## Step 4 — Kentik Portal Configuration

1. Log into [portal.kentik.com](https://portal.kentik.com)
2. Navigate to **Settings → Cloud → AWS**
3. Click **Add Cloud Account**
4. Enter the following:
   - **Role ARN** — the `RoleArn` output from step 1 (hub account)
   - **External ID** — your Kentik Company ID (same value used as `KentikCompanyID`)
5. For flow log collection, under **Flow Settings**, add a flow source and enter:
   - **Flow Role ARN** — the `RoleArn` output from step 3 (logging account)
   - **S3 Bucket Names** — same list provided in step 3

---

## Security Design Notes

### Confused-deputy protection

Both hub and flow roles apply two conditions on the trust policy:

```yaml
Condition:
  StringEquals:
    sts:ExternalId: !Ref KentikCompanyID
  ArnEquals:
    aws:PrincipalArn: arn:aws:iam::834693425129:role/eks-ingest-node
```

- `sts:ExternalId` ensures only Kentik's system using your company ID can assume the role, preventing another Kentik customer from using your role ARN.
- `aws:PrincipalArn` locks trust to the specific `eks-ingest-node` role. Even if Kentik's account root or other roles tried to assume the role, they would be denied.

### Spoke role trust is scoped to hub role ARN

The secondary role in each spoke account trusts the **exact hub role ARN**, not the hub account root:

```yaml
Principal:
  AWS: arn:aws:iam::<HubAccountId>:role/<HubRoleName>
```

This means even if another principal in the hub account were compromised, they could not assume the secondary role.

### Hub policy scoped to secondary role name

The hub role can only assume roles named `${SecondaryRoleName}` — it cannot assume arbitrary roles in spoke accounts:

```yaml
Resource: !Sub "arn:aws:iam::*:role/${SecondaryRoleName}"
```

### S3 access scoped to exact bucket names

No wildcard bucket access. The flow policy lists only the exact bucket names provided at deploy time. Adding a new regional bucket requires updating the stack parameter.

### MaxSessionDuration: 3600

All roles cap session duration to 1 hour (the minimum for assumed role sessions in AWS IAM).

### Permissions intentionally excluded

These permissions were evaluated and excluded:

| Permission | Reason excluded |
|---|---|
| `organizations:ListAccounts` | Not needed — spoke account list is provided manually |
| `network-firewall:*` | Not in use in this environment; `DescribeFirewallPolicy` and `DescribeRuleGroup` would expose firewall rule signatures unnecessarily |
| `s3:*` in spoke template | Flow logs are centralized — spoke accounts do not need S3 access via the metadata role |

---

## Customization

### Changing role or policy names

Override the `RoleName` and `PolicyName` parameters. If you change `RoleName` in the hub or spoke template, ensure both templates use the same `SecondaryRoleName` value so the trust chain works correctly.

### Deploying to multiple regions

Repeat the StackSet `create-stack-instances` command with a different `--regions` value. The same IAM role is account-wide (IAM is global), so you only need one region per account for the role itself. Specify multiple regions only if your StackSets configuration requires regional targets for other resources.

### Adding new S3 buckets for flow logs

Update the `S3BucketNames` parameter on the existing stack:

```bash
aws cloudformation deploy \
  --profile <logging-account-profile> \
  --template-file kentik-standard-account-flow-cfn.yaml \
  --stack-name kentik-flow \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-east-1 \
  --parameter-overrides file://flow-params.json
```

Where `flow-params.json` now includes the new bucket name in the comma-separated list. CloudFormation will update the managed policy in place.

### Adding new spoke accounts

```bash
aws cloudformation create-stack-instances \
  --stack-set-name kentik-metadata-spoke \
  --accounts <new-account-id> \
  --regions us-east-1
```

No changes to the hub or StackSet are required.

---

## Troubleshooting

### `CAPABILITY_NAMED_IAM` error

CloudFormation requires this capability flag when creating named IAM resources. Always include `--capabilities CAPABILITY_NAMED_IAM` in deploy commands.

### StackSets operation fails with `AccessDenied`

StackSets needs service-linked roles or self-managed IAM roles for cross-account deployment. Run:

```bash
aws cloudformation activate-organizations-access
```

Or create the `AWSCloudFormationStackSetAdministrationRole` and `AWSCloudFormationStackSetExecutionRole` manually. See [AWS StackSets prerequisites](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/stacksets-prereqs.html).

### Kentik reports `AccessDenied` on `sts:AssumeRole`

Verify:
1. The `KentikCompanyID` parameter matches exactly what is in the Kentik portal (Settings → Licenses)
2. The role ARN entered in the Kentik portal matches the `RoleArn` stack output exactly
3. For the spoke role, confirm the hub account ID in `HubAccountId` matches the account where the hub stack was deployed (`AccountId` output from step 1)

### Stack update fails on IAM policy with `MalformedPolicyDocument`

The `S3BucketNames` parameter must contain valid S3 bucket names (lowercase, no special characters other than hyphens). Ensure there are no spaces around commas in the comma-delimited list.

Valid: `bucket-us-east-1,bucket-us-east-2`
Invalid: `bucket-us-east-1, bucket-us-east-2` (space after comma causes an invalid ARN)

### Hub role can assume the spoke role, but Kentik returns empty metadata

Confirm the spoke role has the correct secondary role name that matches what Kentik is configured to assume. The secondary role name in the Kentik portal must match the `RoleName` parameter (default: `KentikMetadataSecondaryRole`) used when deploying the spoke stack.

---

## Reference

- [Kentik KB: Metadata Configuration — AWS](https://kb.kentik.com/docs/metadata-configuration-aws)
- [Kentik KB: AWS Endpoints List](https://kb.kentik.com/docs/aws-endpoints-list)
- [AWS CloudFormation StackSets Documentation](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/what-is-cfnstacksets.html)
- [AWS IAM — Confused Deputy Prevention](https://docs.aws.amazon.com/IAM/latest/UserGuide/confused-deputy.html)
