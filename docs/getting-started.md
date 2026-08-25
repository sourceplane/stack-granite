# Getting started

## Consume the catalog

Pin a version in the consuming repo's `intent.yaml`:

```yaml
compositions:
  sources:
    - name: stack-granite
      kind: oci
      ref: oci://ghcr.io/sourceplane/stack-granite:0.1.0
  resolution:
    precedence: [stack-granite]
    bindings:
      terraform-aws: stack-granite
      container-image: stack-granite
      helm-release: stack-granite
      node-service-turbo: stack-granite
      turbo-package: stack-granite
```

Then lock and validate:

```bash
orun compositions lock --intent intent.yaml
orun validate --intent intent.yaml
```

## Declare a component

A component says *what*, never *how*. A Terraform root:

```yaml
apiVersion: sourceplane.io/v1
kind: Component
metadata:
  name: vpc
spec:
  type: terraform-aws
  parameters:
    stackName: vpc
    terraformDir: terraform
    terraformVersion: "1.15.3"
    awsRegion: ap-southeast-1
    secretOutputs: "VPC_ID=vpc_id,PRIVATE_SUBNET_IDS=private_subnet_ids"
  subscribe:
    environments:
      - name: dev
        profile: validate
      - name: stage
        profile: plan-only
        profileRules:
          - profile: apply
            when: { triggerRef: github-push-main }
```

The `profileRules` line is the whole delivery policy: read-only on pull
requests, applied on merge. Nothing in `.github/workflows/` needs to know.

## The three-component service chain

A deployable service is three components wired by `dependsOn`:

```
<service>            node-service-turbo   typecheck, lint, test, build
<service>-image      container-image      build, scan, push by digest
<service>-release    helm-release         render, validate, roll
```

The image component publishes `IMAGE_REF`; the release component reads it
through `secretEnv`. Add the service's Terraform root as a fourth, upstream of
the image, and the plan DAG orders the whole thing without a line of
orchestration.

## Verify locally

```bash
./scripts/verify.sh
```
