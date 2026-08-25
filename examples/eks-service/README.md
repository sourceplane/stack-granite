# Example: one service on EKS

The minimum wiring for a deployable service: a Terraform root for its AWS
dependencies, the code lane, the image lane, and the release — four components
whose `dependsOn` edges are the entire deployment order.

There is no orchestration file. `orun plan` derives the DAG:

```
documents-deps ──▶ documents-image ──▶ documents-release
       │                                     ▲
       └── (bucket, redis, ECR, IRSA role) ──┘
documents ─────────────────────────────────▶ (code must pass first)
```

`orun plan --view dag --intent intent.yaml` prints it.
