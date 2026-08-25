# `terraform-aws`

Terraform for AWS estates. Distinct from `stack-tectonic`'s `terraform`
composition, which has no OIDC role-assumption capability and no credential-free
validation lane.

## Parameters

| Parameter | Required | What |
|-----------|----------|------|
| `stackName` | yes | Logical name of the root; used in log context. |
| `terraformDir` | yes | Path to the root, relative to the component directory. |
| `terraformVersion` | yes | Exact version. A range makes the lane non-reproducible. |
| `awsRegion` | no | Surfaced as `AWS_REGION` and `TF_VAR_awsRegion`. |
| `awsRoleArn` | no | Assumed via GitHub OIDC. Owned by `aws-admin`, never created here. |
| `namespacePrefix` | no | Prefix on every named resource so environments cannot collide in one account. |
| `secretOutputs` | no | `KEY=output,…` — published to the environment rung after apply. |

Every parameter is additionally exported as `TF_VAR_<name>`, alongside
`TF_VAR_environment` and `TF_VAR_component`.

## Profiles

| Profile | Reaches AWS | Reaches state | Mutates |
|---------|-------------|---------------|---------|
| `validate` (default) | no | no | no |
| `plan-only` | yes | yes | no |
| `apply` | yes | yes | **yes** |
| `local` | ambient creds | yes | no |

`validate` runs `terraform init -backend=false`, so providers are downloaded and
the configuration is typechecked against their schemas without a backend or a
credential. It is the lane that lets a repo with no AWS account prove its
Terraform compiles.

`apply` consumes the plan file the `plan` step wrote. Applying a freshly
computed plan instead would let the world change between the plan a human read
and the apply that ran.

## State

State lives on the Orun HTTP backend. The runner exports `TF_HTTP_*` per job
(address `…/state/tfstate/{component}/{env}`, run token as password), so
`backend "http"` needs no `-backend-config`, no AWS credentials for state, and
no S3 bucket to bootstrap before the first apply. The environment is *in* the
address — there are no Terraform workspaces.

## Outputs

`secretOutputs: "BUCKET=bucket_name,ROLE_ARN=irsa_role_arn"` queues each named
output on the runner's sink inside the apply step. The runner publishes them
over the run's lease-bound channel onto the project/environment rung after the
apply succeeds. Downstream components read the same keys through `secretEnv`.

This replaces `terraform_remote_state`: no cross-state data sources, no extra
IAM grants to read another root's state, and no ordering puzzle at bootstrap.
