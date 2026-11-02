# Terraform CI/CD with GitHub Actions, AWS OIDC, and S3 Backend

A hands-on project that demonstrates how to provision AWS networking infrastructure with Terraform and deploy it through GitHub Actions using OpenID Connect (OIDC).

The workflow separates **Terraform Plan** from **Terraform Apply** and places the Apply job behind a GitHub Environment named `production`.

## Project overview

This repository provisions:

- An Amazon VPC with CIDR block `10.0.0.0/16`.
- One subnet with CIDR block `10.0.1.0/24`, associated with the VPC.
- Name tags: `dev-vpc` and `dev-subnet`.

Terraform state is configured to use an Amazon S3 backend with S3 lockfile support.

## Architecture

```text
Developer
   |
   | Manually dispatch workflow (workflow_dispatch)
   v
GitHub Repository
   |
   v
GitHub Actions - Terraform Plan
   |
   | OIDC token (id-token: write)
   v
AWS IAM OIDC Identity Provider
   |
   v
IAM Role (restricted trust policy)
   |
   | Temporary AWS credentials from STS
   v
Terraform
   |
   +--> S3 remote state
   |
   +--> Terraform plan artifact
             |
             v
      GitHub Environment: production
          (approval gate, if reviewers/rules are configured)
             |
             v
      GitHub Actions - Terraform Apply
             |
             v
      AWS VPC and subnet
```

## Tools and services

| Tool / Service | Purpose |
|---|---|
| Terraform | Infrastructure as Code |
| AWS | Cloud infrastructure provider |
| Amazon VPC | Virtual private cloud |
| Amazon EC2 networking | Subnet resource |
| Amazon S3 | Remote Terraform state |
| AWS IAM OIDC provider | Trusts GitHub-issued OIDC tokens |
| AWS STS | Issues temporary role credentials |
| GitHub Actions | CI/CD automation |
| GitHub Environments | Production deployment gate |
| GitHub Actions artifacts | Transfers the saved Terraform plan between jobs |

## Repository structure

```text
.
├── .github/
│   └── workflows/
│       ├── terraform-cicd.yml
│       └── testoidc.yml
├── backend.tf
├── main.tf
├── provider.tf
├── variables.tf
└── README.md
```

## How it works

The workflow is defined in [`.github/workflows/terraform-cicd.yml`](.github/workflows/terraform-cicd.yml) and is manually triggered using `workflow_dispatch`.

### 1. Terraform Plan job

The first job:

1. Checks out the repository.
2. Requests AWS credentials by assuming the configured IAM role through GitHub OIDC.
3. Verifies the AWS identity using `aws sts get-caller-identity`.
4. Creates the configured state bucket if it is not already owned by the account, then verifies bucket ownership and enables versioning.
5. Installs Terraform version `1.10.5`.
6. Runs `terraform init`.
7. Runs `terraform validate`.
8. Runs `terraform plan -out=tfplan`.
9. Uploads `tfplan` as a GitHub Actions artifact named `terraform-plan` (retained for one day).

### 2. Production approval gate

The Apply job declares:

```yaml
needs: plan
environment: production
```

This makes it depend on a successful Plan job and use the GitHub Environment named `production`. Configure required reviewers or other protection rules under **Repository Settings → Environments → production** if you want a manual approval before deployment. Merely declaring the environment does not by itself guarantee that a reviewer approval is required.

### 3. Terraform Apply job

After the Plan job succeeds and any configured environment protection rules are satisfied, the second job:

1. Checks out the repository.
2. Assumes the AWS IAM role through OIDC again.
3. Verifies the AWS identity.
4. Installs Terraform version `1.10.5`.
5. Runs `terraform init`.
6. Downloads the `terraform-plan` artifact.
7. Applies the saved plan using `terraform apply tfplan`.

The Apply job uses a separate OIDC authentication step because each job runs on its own GitHub-hosted runner.

## Step-by-step setup

### Prerequisites

- An AWS account with permission to configure IAM, S3, and VPC resources.
- A GitHub repository with GitHub Actions enabled.
- Terraform installed locally (optional for local testing).
- AWS CLI installed locally (optional for local testing).

### Step 1: Create the GitHub OIDC identity provider in AWS

In the AWS Console:

1. Open **IAM → Identity providers**.
2. Choose **Add provider**.
3. Select **OpenID Connect**.
4. Set the provider URL to:

   `https://token.actions.githubusercontent.com`

5. Set the audience to:

   `sts.amazonaws.com`

6. Create the provider.

If the GitHub OIDC provider already exists in the AWS account, reuse it rather than creating a duplicate.

### Step 2: Create an IAM role for GitHub Actions

1. Open **IAM → Roles → Create role**.
2. Choose **Web identity** as the trusted entity type.
3. Select the GitHub OIDC identity provider.
4. Set the audience to `sts.amazonaws.com`.
5. Create a role for the repository's GitHub Actions workflow.
6. Attach only the AWS permissions needed by this Terraform project.
7. Save the role ARN.

The workflow expects the role ARN to be stored in the GitHub repository secret `AWS_ROLE_ARN`.

**Least-privilege reminder:** The role needs permissions for the Terraform resources and backend operations used by this project. Avoid attaching broad administrator permissions for routine use.

### Step 3: Configure the IAM role trust policy

The trust policy must match the OIDC subject claim GitHub actually issues.

This repository has GitHub's immutable subject claim enabled. The visible subject prefix is:

```text
repo:YOUR_GITHUB_NAME@YOUR_OWNER_ID/YOUR_REPOSITORY_NAME@YOUR_REPOSITORY_ID
```

For the current workflow, the trust policy should allow the main branch for Plan and the `production` environment for Apply. Confirm the exact subject values against **Repository Settings → Actions → OIDC** and keep the audience restricted to `sts.amazonaws.com`.

Example condition for this repository's current immutable subject format:

```json
{
  "StringEquals": {
    "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
    "token.actions.githubusercontent.com:sub": [
      "repo:YOUR_GITHUB_NAME@YOUR_OWNER_ID/YOUR_REPOSITORY_NAME@YOUR_REPOSITORY_ID:ref:refs/heads/main",
      "repo:YOUR_GITHUB_NAME@YOUR_OWNER_ID/YOUR_REPOSITORY_NAME@YOUR_REPOSITORY_ID:environment:production"
    ]
  }
}
```

Use this condition inside the role trust policy's `Condition.StringEquals` object. Preserve the correct federated principal ARN and the `sts:AssumeRoleWithWebIdentity` action. If the repository owner or repository is renamed, or the subject template changes, review and update the trust policy.

### Step 4: Configure GitHub repository secrets

Open **Repository Settings → Secrets and variables → Actions → New repository secret**.

Create:

| Secret name | Value |
|---|---|
| `AWS_ROLE_ARN` | ARN of the IAM role created for this repository |

Do not put AWS access keys in the workflow. OIDC is used to obtain temporary credentials at runtime. Do not publish secret values in this README, source files, screenshots, or commit history.

### Step 5: Configure the production environment

1. Open **Repository Settings → Environments**.
2. Create an environment named exactly `production`.
3. Optionally configure required reviewers and deployment branch restrictions.
4. Save the environment settings.

The workflow references this exact environment name in the Apply job.

### Step 6: Configure the S3 Terraform backend

The backend is defined in [`backend.tf`](backend.tf):

- Bucket: `YOUR_TERRAFORM_STATE_BUCKET`
- Key: `terraform.tfstate`
- Region: `us-east-1`
- S3 lockfile: enabled
- Expected AWS account: `YOUR_AWS_ACCOUNT_ID`

The Plan job checks for the bucket, creates it if absent, verifies the expected bucket owner, and enables bucket versioning.

The IAM role needs the appropriate S3 permissions for these operations and Terraform state access. Protect the state bucket: Terraform state can contain sensitive infrastructure data. Keep public access blocked and restrict access to authorized principals.

### Step 7: Review Terraform configuration

The current Terraform files define:

- `provider.tf`: AWS provider configuration.
- `variables.tf`: default VPC and subnet CIDR blocks and names.
- `main.tf`: one VPC and one subnet.
- `backend.tf`: S3 remote state configuration.

Current default values:

| Variable | Default |
|---|---|
| `cidr` | `10.0.0.0/16` |
| `name_tags` | `dev-vpc` |
| `subnet_cidr` | `10.0.1.0/24` |
| `subnet_tag` | `dev-subnet` |

### Step 8: Run the workflow

1. Open the repository's **Actions** tab.
2. Select **Terraform Deploy**.
3. Choose **Run workflow**.
4. Select the `main` branch.
5. Start the workflow.

The Plan job runs first. Review the Terraform plan output before allowing the Apply job to proceed. If production environment protection rules are configured, approve the deployment when GitHub requests approval.

### Step 9: Verify resources in AWS

After a successful Apply:

1. Open **AWS Console → VPC → Your VPCs**.
2. Find the VPC named `dev-vpc` and verify its CIDR.
3. Open **Subnets** and verify the subnet named `dev-subnet` is associated with that VPC.
4. Open **Amazon S3 → Buckets** and verify the Terraform state bucket.
5. Check the GitHub Actions run to confirm both Plan and Apply jobs succeeded.

The expected network resources are one VPC (`10.0.0.0/16`) and one subnet (`10.0.1.0/24`).

## Useful Terraform commands

Run these locally from the repository root if you have configured AWS credentials and the required backend access:

```bash
terraform init
terraform fmt -check
terraform validate
terraform plan
terraform apply
```

To remove resources created by this configuration when they are no longer needed:

```bash
terraform destroy
```

Run destroy only when you intentionally want to remove the managed infrastructure. Confirm the selected AWS account, workspace, and plan before approving destructive changes. Do not delete the S3 state bucket while it contains state you still need.

## Security notes

- Use GitHub OIDC and short-lived AWS role credentials instead of long-lived IAM access keys.
- Restrict the IAM trust policy to this repository, the intended branch, and the intended GitHub Environment.
- Keep the audience claim set to `sts.amazonaws.com`.
- Grant only the AWS permissions required by Terraform and the S3 backend.
- Keep the Terraform state bucket private, enable versioning, and limit access.
- Configure GitHub Environment protection rules for production deployments.
- Never commit credentials, secret values, Terraform state files, or sensitive plan files.

## Troubleshooting

| Symptom | What to check |
|---|---|
| `Not authorized to perform sts:AssumeRoleWithWebIdentity` | Verify the IAM OIDC provider, audience, immutable `sub` claim, branch/environment subject, and role ARN secret. |
| Plan works but Apply cannot assume the role | Confirm the trust policy includes the `production` environment subject and the Apply job uses `environment: production`. |
| S3 backend access denied | Check IAM permissions for the bucket and state object, including lockfile operations. |
| State bucket creation fails | Check bucket name availability, account ownership, region, and S3 permissions. |
| Terraform validation or plan fails | Review the failing Terraform output and confirm variables, provider configuration, and backend configuration. |
| Apply cannot find `tfplan` | Confirm the Plan artifact upload and Apply artifact download use the same artifact name. |

## Learning outcomes

By completing this project, you practise:

- Configuring GitHub Actions to request OIDC tokens.
- Establishing trust between GitHub Actions and AWS IAM.
- Using AWS STS temporary role credentials.
- Managing Terraform state remotely in Amazon S3.
- Separating infrastructure planning and deployment.
- Adding a GitHub Environment protection gate to a deployment workflow.
- Provisioning AWS VPC networking resources with Terraform.

---

**Project:** Terraform CI/CD using GitHub Actions and AWS OIDC  
**Cloud provider:** Amazon Web Services (AWS)  
**Infrastructure as Code:** Terraform