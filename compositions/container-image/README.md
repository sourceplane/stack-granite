# `container-image`

Build, scan and publish a service image to ECR.

## Parameters

| Parameter | Required | What |
|-----------|----------|------|
| `imageName` | yes | Repository name within the registry, without host or tag. |
| `dockerfile` | yes | Path to the Dockerfile. |
| `buildContext` | yes | Build context. Usually the workspace root for a monorepo build. |
| `registry` | no | ECR host. Resolved from the service's Terraform outputs at deploy time, not committed — an account id in a component file is an instantiation blocker. |
| `buildArgs` | no | Space-separated `KEY=VALUE` pairs. |
| `scanSeverity` | no | Severities that fail the build. Default `CRITICAL,HIGH`. |
| `imageOutputs` | no | `KEY=field,…` where field is `digest`, `tag` or `ref`. |

## Tagging

Tags are `<environment>-<12-char-commit-sha>` and images are pushed by digest.
`latest` is never written. A rendered manifest that says `latest` cannot tell
you what is running, and a mutable tag makes a rollback a guess.

## Scan before push

The Trivy gate runs *before* the push, not after. An image that reached the
registry is already a supply-chain artifact even if nothing deploys it.

## Profiles

| Profile | Authenticates | Pushes |
|---------|---------------|--------|
| `verify` (default) | no | no |
| `publish` | yes (OIDC → ECR) | yes, by digest |

`verify` builds to the local daemon so the scan has something to read, then
stops. The layer cache is shared with `publish`, so the publish lane does not
pay for the verify lane's work twice.
