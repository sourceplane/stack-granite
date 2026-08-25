# Verification

```bash
./scripts/verify.sh
```

Four gates, in order:

1. **Structural.** Every `compositions/<name>/` carries a `composition.yaml`
   whose `metadata.name` and `spec.type` both equal `<name>`, plus a
   `schema.yaml` and a `jobs/` directory.
2. **Publish resolution.** `orun publish --dry-run` resolves the target and the
   version without uploading.
3. **Pack.** The shipped subset (`stack.yaml`, `compositions/`, `examples/`) is
   staged and packed exactly as `orun publish` would ship it — not the whole
   repo, so the artifact under test is the artifact consumers resolve.
4. **Consumer resolution.** The packed archive is resolved back through a
   throwaway intent and every composition must appear in
   `orun compositions list`. This is the gate that catches a composition Orun
   can parse but cannot export.

Gate 2 alone would prove nothing: a dry-run publish resolves a manifest and a
target, not the validity of the package. Gates 3 and 4 exercise the same code
path a consuming repo runs.

## What it does not check

The gates prove the catalog is *well-formed and consumable*. They do not run a
job template — that requires a consuming repo, a runner, and in the credentialed
lanes an AWS account. A composition that packs and exports cleanly can still
have a step that fails at run time.

The first real exercise of these contracts is
[`sourceplane/cumulus`](https://github.com/sourceplane/cumulus), whose CI runs
the `validate`, `verify`, `lint` and `smoke` lanes on every PR.
