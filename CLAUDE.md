# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is the **AWS Networking Lab 02** repository: a CloudFormation root stack that builds a VPC with one public and one private subnet in `us-east-1a`, two SSH-only security groups, and one EC2 instance in each subnet. The root stack deploys every resource through nested stacks whose templates live in an S3 bucket, not in this repo.

**Key characteristics:**

- Root stack template with nested stacks referenced via `TemplateURL`
- Nested templates are stored in the S3 bucket named by `NestedStacksS3BucketName` (`us-east-1`)
- Public SSH is open to `0.0.0.0/0`; private SSH is allowed only from the public SSH security group
- Conventional commits drive semantic versioning and the changelog

## Project Structure

```text
cloudformation/
├── template.yaml                  # Root stack template (nested stacks, parameters, outputs)
├── parameters.json                # Parameter values for the lab
└── stack-config.json              # Stack name and template/parameter file paths

media/
└── architecture-diagram.jpg       # Diagram used in README.md

.github/workflows/
├── ci.yaml                        # CI build (manual trigger only; calls a reusable workflow), changelog and release jobs
├── create-branch.yaml             # Auto-create feature branches from issues
├── claude.yaml                    # Claude Code workflow
├── claude-code-review.yaml        # Claude Code review workflow
├── notify.yaml                    # Notification workflow
└── setup-environments.yaml        # GitHub environment setup

scripts/plugins/
├── release.config.js              # Semantic-release configuration
└── (other release plugins)        # Commit analysis, notes generation, publish

.releaserc.json                    # Semantic-release plugins (branch: main)
.claude/                           # Claude Code settings and repo skills
.devcontainer/devcontainer.json    # Dev container setup
CHANGELOG.md                       # Generated release changelog
README.md                          # Lab documentation
```

## Nested Stacks

The root stack (`cloudformation/template.yaml`) creates these nested stacks. Each one's `TemplateURL` is `https://${TemplateBucket}.s3.us-east-1.amazonaws.com/<key>`.

| Root resource | S3 key | Purpose |
| ------------- | ------ | ------- |
| `Vpc` | `cfn-nested-aws-vpc/template.yaml` | VPC and Internet Gateway |
| `PublicSubnets` | `cfn-nested-aws-subnets/template.yaml` | Public subnet (`SubnetQualifier: public`), given the IGW ID |
| `PrivateSubnets` | `cfn-nested-aws-subnets/template.yaml` | Private subnet (`SubnetQualifier: private`), no IGW passed |
| `PublicSg` | `cfn-nested-aws-ec2-security-group/template.yaml` | SSH from `0.0.0.0/0` |
| `PrivateSg` | `cfn-nested-aws-ec2-security-group/template.yaml` | SSH from the public SSH security group |
| `PublicEc2` | `cfn-nested-aws-ec2-instance/template.yaml` | EC2 instance in the public subnet |
| `PrivateEc2` | `cfn-nested-aws-ec2-instance/template.yaml` | EC2 instance in the private subnet |

The nested templates are not in this repo. Changes to them must be uploaded to the bucket separately.

## Key Files

### `cloudformation/template.yaml`

**Purpose:** Root stack. Wires the nested stacks together and exposes outputs.

**Key parameters:**

- `TemplateBucket`: Bucket that holds the nested templates
- `Project` (default `netwk-lab02`), `Env` (`devl`, `stag`, `prod`), `CiSuffix`
- `VpcCidr`, `PublicCidrs`, `PrivateCidrs`
- `PublicSgName`, `PrivateSgName`: Base names for the security groups
- `AmiParam` (SSM parameter name, latest Amazon Linux 2023; pass the name, not an AMI ID), `InstanceType` (`t3.micro`, `t3.small`, `t3.medium`, `t2.micro`), `KeyName` (default `networking-lab`)

**Key outputs:** `VpcId`, `IgwId`, `PublicSubnetId`, `PrivateSubnetId`, `PublicSgId`, `PublicSgName`, `PrivateSgId`, `PrivateSgName`, `PublicEc2Id`, `PrivateEc2Id`.

### `cloudformation/parameters.json`

**Purpose:** Parameter values used for the lab. It sets `Project` to `netwk-lab02`, `VpcCidr` to `10.11.0.0/16`, the public and private CIDRs to `10.11.0.0/20` and `10.11.48.0/20`, `KeyName` to `networking-lab`, and `CiSuffix` to empty. Other parameters use their template defaults.

### `cloudformation/stack-config.json`

**Purpose:** Read by CI to get the stack name (`aws-lab02-private-subnet-ec2-access-stack`), the template path, and the parameter file path.

## Naming Convention

The VPC name tag follows this pattern:

```bash
{ProjectName}-{Environment}-{Region}[-{CiSuffix}]
```

Example: `networking-devl-us-east-1` (with the values in `parameters.json`)

## Testing & Validation

**Validate the template:**

```bash
aws cloudformation validate-template --template-body file://cloudformation/template.yaml
```

**Deploy the root stack:**

```bash
aws cloudformation deploy \
  --template-file cloudformation/template.yaml \
  --stack-name aws-lab02-private-subnet-ec2-access-stack \
  --region us-east-1 \
  --parameter-overrides ProjectName=networking
```

Pass other parameters as `Name=Value` pairs in `--parameter-overrides`. The nested templates must already be in the S3 bucket, and the EC2 key pair named by `KeyPairName` must exist in the account and region.

**Verify outputs and clean up:**

```bash
aws cloudformation describe-stacks --stack-name aws-lab02-private-subnet-ec2-access-stack --query 'Stacks[0].Outputs' --output table
aws cloudformation delete-stack --stack-name aws-lab02-private-subnet-ec2-access-stack
```

## CI Workflow

`.github/workflows/ci.yaml` (named "CI Build Workflow") reads `cloudformation/stack-config.json` and calls the reusable workflow `subhamay-bhattacharyya-gha/cfn-ci-build-reusable-wf` with environment `ci`. Its push and pull request triggers are commented out, so it currently runs only on manual dispatch. The same file also has changelog and release jobs.

**GitHub environment variables required in the `ci` environment** (used by the reusable workflow):

- `AWS_REGION`, `AWS_ACCOUNT_ID`, `OIDC_ROLE_NAME`, `CFN_TEMPLATES_S3_BUCKET`

**OIDC setup:** The workflow uses AWS OIDC for keyless auth (`id-token: write`). The GitHub OIDC provider must trust the role named by `OIDC_ROLE_NAME`.

## Conventional Commits & Release Flow

Commits to `main` drive semantic-release through `.releaserc.json` (`@semantic-release/commit-analyzer`, release notes, changelog, git, and GitHub plugins).

```text
feat: add support for X       → MINOR release
fix: correct behavior of Y    → PATCH release
BREAKING CHANGE: ...          → MAJOR release
chore: / docs: ...            → no release
```

Feature branches follow the same commit format. Releases happen on merge to `main`.

## When Modifying the Template

1. **Edit** `cloudformation/template.yaml`
2. **Update** `cloudformation/parameters.json` if you add or change parameters that the lab sets
3. **Validate** with `aws cloudformation validate-template`
4. **Update** `README.md` if resources, parameters or outputs change
5. **Commit** with a conventional commit message (e.g., `feat: add encryption key parameter`)
6. **Merge to `main`** → the release workflow creates the tag and GitHub release

## Dev Container

Pre-configured with Node.js 20 and the GitHub Copilot extension. The repo no longer has a `package.json`, so there are no npm dependencies to install.

## Current Branch

`main` is the release branch. Feature work branches from `main` and merges back via PR. Branch naming follows `{type}/CFN-{issue-number}-{slug}` (e.g., `feature/CFN-42-add-encryption`).
