# `helm-release`

Render, validate and roll a Helm release onto an EKS cluster.

## Parameters

| Parameter | Required | What |
|-----------|----------|------|
| `chartDir` | yes | Chart directory, relative to the component. |
| `releaseName` | yes | Helm release name. |
| `namespace` | yes | Target namespace; created on first install. |
| `helmVersion` | yes | Exact Helm version. |
| `valuesFiles` | no | Space-separated, applied in order, lowest precedence first. |
| `imageRepository` | no | Fully-qualified repository. Usually supplied by `container-image`. |
| `clusterName` | no | For `aws eks update-kubeconfig`. Absent in lint lanes. |
| `kubernetesVersion` | no | API version rendered manifests are validated against. Default `1.31.0`. |
| `renderOut` | no | Where rendered manifests are written. Default `rendered/<env>`. |

## Rendering to disk, not to a pipe

`helm template` writes to a directory rather than piping into a validator. The
rendered manifests become an artifact a sibling test component can assert
invariants over — exactly one Ingress in the fleet, no `latest` tag, every
ServiceAccount carrying an IRSA annotation, every container with resource
limits. A pipeline into a validator proves only that the validator was happy.

## `--atomic` is not optional

The upgrade runs `--atomic --wait --timeout`. A release that does not become
ready inside the timeout rolls itself back rather than leaving a half-migrated
fleet for whoever looks next. Unattended delivery without it is a coin flip.

## The image comes from upstream

`helm upgrade` fails loudly when `IMAGE_REF` is unset. The reference is
published by the `container-image` component and consumed here through
`secretEnv` — the chart is never handed a tag interpolated from a workflow
string.

## Profiles

| Profile | Reaches the cluster | Mutates |
|---------|---------------------|---------|
| `lint` (default) | no | no |
| `deploy` | yes | yes |

`deploy` re-runs the same `kubeconform` validation the lint lane runs. A deploy
lane that skipped it would let a manifest no PR ever rendered reach a cluster.
