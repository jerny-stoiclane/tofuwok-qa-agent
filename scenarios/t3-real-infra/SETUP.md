# T3 Real Infra: Setup Requirements

T3 scenarios test against real AWS. Before running any T3 scenario, these prerequisites must be met.

## Test Directory

T3 uses `test/companies/_qa/aws/us-east-1/` in the `tf-orchestrator-gha` repo. This dir must exist on main with:

```
test/companies/_qa/
├── ci.env                          # AWS auth config (real role ARN)
└── aws/
    └── us-east-1/
        ├── backend.tf              # local backend (or S3 for backend tests)
        ├── ci.s3.tfbackend         # S3 backend config (for backend tests)
        ├── main.tf                 # empty or minimal — scenarios add resources
        ├── variables.tf
        └── .terraform-version
```

## Auth Config

The `ci.env` must have a real IAM role that the tofuwok-runner can assume via OIDC:

```
AWS_ACCOUNT_ID=814369619170
AWS_REGION=us-east-1
IAM_ROLE=arn:aws:iam::814369619170:role/github-actions/github-actions-tf-orchestrator-gha
```

OR the tofuwok repo config must have `default_aws_role` set.

The runner's `awsauth.ResolveAWSCredentials()` must be able to find and assume this role.

## IAM Role Permissions

The test role needs ONLY:
- `ssm:PutParameter`, `ssm:GetParameter`, `ssm:DeleteParameter` (for SSM tests)
- `iam:GetRole`, `iam:CreateRole`, `iam:DeleteRole` (for IAM tests)
- `sts:GetCallerIdentity` (for auth verification)
- `s3:GetObject`, `s3:PutObject` on the state bucket (if using S3 backend)

It should NOT have: EC2, RDS, EKS, Lambda, or anything expensive. Use an SCP or permission boundary to prevent cost.

## Verify Prerequisites

Before running T3, the QA agent runs these checks:
1. `test/companies/_qa/aws/us-east-1/` dir exists in the repo
2. Auth config resolves a valid role ARN
3. `sts:GetCallerIdentity` succeeds via the runner
4. The role ARN matches expected account/role

If any check fails, T3 is blocked with a clear message about what's missing.
