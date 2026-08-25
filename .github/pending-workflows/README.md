# Pending workflows

These are the finished catalog workflows. GitHub refuses a push that creates
anything under `.github/workflows/` unless the pushing token carries the
`workflow` OAuth scope, and the account this repo was scaffolded from does not
have it. They live here so they are versioned and reviewable.

| File | What it does |
|------|--------------|
| `verify.yml` | Runs `scripts/verify.sh` on every PR and on `main` — the catalog gate. |
| `release.yml` | On a `v*` tag: asserts the tag matches `stack.yaml`'s version, runs the gate, creates the GitHub release, publishes the OCI artifact to `ghcr.io/sourceplane/stack-granite`, and pulls it back to confirm it resolves. |

## Activating them

In an interactive terminal:

```bash
gh auth refresh -h github.com -s workflow -s write:packages
```

Then here:

```bash
mkdir -p .github/workflows
git mv .github/pending-workflows/verify.yml  .github/workflows/verify.yml
git mv .github/pending-workflows/release.yml .github/workflows/release.yml
git rm .github/pending-workflows/README.md
git commit -m "ci: activate the catalog verify and release workflows"
git push
git tag v0.1.0 && git push origin v0.1.0
```

The tag push is what publishes `oci://ghcr.io/sourceplane/stack-granite:0.1.0`,
which [`sourceplane/cumulus`](https://github.com/sourceplane/cumulus) pins in
its `intent.yaml`. Nothing in cumulus's CI can resolve until that artifact
exists.

`write:packages` is needed because `orun publish` pushes to GHCR. In CI the
workflow's own `GITHUB_TOKEN` carries `packages: write`, so once `release.yml`
is live the scope is only needed for a manual local publish.

## Verifying without CI

The gate itself needs no special scope and runs locally today:

```bash
./scripts/verify.sh
```
