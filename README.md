# Stack Granite

**The Orun composition catalog for services on AWS EKS.**

A versioned, OCI-hosted set of execution contracts — how a Terraform root is
validated, planned and applied against AWS; how a service image is built,
scanned and pushed to ECR; how a Helm release is rendered, schema-validated and
rolled onto a cluster; how a service and a shared package are verified in a
pnpm/Turborepo workspace.

Consumed by pinning a version in a repo's `intent.yaml`:

```yaml
compositions:
  sources:
    - name: stack-granite
      kind: oci
      ref: oci://ghcr.io/sourceplane/stack-granite:0.1.0
  resolution:
    precedence: [stack-granite]
```

Pin an explicit version. A bare repository reference resolves to `:latest`,
which lets a catalog release change a consuming repo's execution contracts
without a commit there. Bumping the tag is the deliberate upgrade step.

Sibling of [`stack-tectonic`](https://github.com/sourceplane/stack-tectonic)
(Cloudflare, Supabase, monorepo) and
[`stack-basalt`](https://github.com/sourceplane/stack-basalt) (.NET, Azure
Container Apps). Kept separate on purpose: a Helm- or EKS-lane change here
cannot destabilize the Cloudflare products that pin `stack-tectonic`.

## The compositions

| Composition | What it contracts | Profiles |
|-------------|-------------------|----------|
| [`terraform-aws`](compositions/terraform-aws/) | fmt → init → validate → plan → apply for a Terraform root, with AWS credentials assumed via GitHub OIDC and state on the Orun HTTP backend. Declared outputs are lease-published to the environment rung. | `validate` (default), `plan-only`, `apply`, `local` |
| [`container-image`](compositions/container-image/) | BuildKit build with a GHA layer cache, Trivy gate, digest-addressed push to ECR, and publication of the image coordinates downstream. | `verify` (default), `publish` |
| [`helm-release`](compositions/helm-release/) | `helm lint` → `helm template` to disk → `kubeconform` → `helm diff` → `helm upgrade --install --atomic --wait` → rollout check. | `lint` (default), `deploy` |
| [`node-service-turbo`](compositions/node-service-turbo/) | install → typecheck → lint → test → build for one service in a pnpm/Turborepo workspace, plus an optional docker-compose smoke lane. | `quick-check`, `verify` (default), `smoke` |
| [`turbo-package`](compositions/turbo-package/) | The same for a non-deployable workspace package. | `quick-check`, `verify` (default) |
| [`publish-stack`](compositions/publish-stack/) | This catalog's own release lane. | `dry-run` (default), `verify`, `release` |

`turbo-package` and `publish-stack` are ported **byte-identical** from
`stack-tectonic`, which remains their upstream. Divergence must be a deliberate
edit — see [`docs/authoring.md`](docs/authoring.md) § Ported compositions.

## Two properties worth knowing

**Every default profile is the safe one.** `terraform-aws` defaults to
`validate`, `container-image` to `verify`, `helm-release` to `lint`. A component
that forgets to declare a profile gets a lane that cannot authenticate, cannot
push and cannot mutate a cluster. Reaching production is always an explicit act.

**Credential-free lanes are structurally credential-free.** The `validate` and
`verify` profiles do not merely skip the credentialed steps — they exclude the
OIDC capability itself, so those lanes have no path to AWS even if a component
supplies a role ARN. That is what lets a repo with no AWS account at all still
prove its Terraform and its charts are well-formed on every PR.

## Verification

```bash
./scripts/verify.sh
```

Structurally validates every composition contract, resolves the publish target,
packs the shipped layers, and resolves them back through a throwaway consumer
intent — the same path a consuming repo runs. See
[`docs/verification.md`](docs/verification.md).

## Releasing

Bump `metadata.version` in `stack.yaml`, then push a matching `v<version>` tag.
The release workflow asserts the tag and the stack version agree, runs the
catalog gate, creates the GitHub release, publishes the OCI artifact, and pulls
it back to confirm it is resolvable.
