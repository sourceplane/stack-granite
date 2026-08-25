# Authoring a composition

## Layout

```
compositions/<name>/
  composition.yaml     metadata.name and spec.type MUST both equal <name>
  schema.yaml          the component contract
  README.md            what it does and why it decides what it decides
  jobs/<name>-*.yaml   every possible step, capability-tagged
  profiles/<name>-*.yaml  which capabilities each lane runs
```

`scripts/verify.sh` enforces the naming rule and the presence of
`schema.yaml` and `jobs/`.

## Rules

1. **Default to the safe profile.** `defaultProfile` must be the lane that
   cannot authenticate or mutate.
2. **Credentialed steps get their own capability.** Never guard a mutating step
   with an `if` inside the template — a profile must be able to exclude it
   structurally.
3. **Pin exact tool versions.** A range in `terraformVersion` or `helmVersion`
   makes a green lane unreproducible tomorrow.
4. **Declare every parameter in `schema.yaml`.** Plan-time validation is what
   turns a typo into a failed plan rather than a failed deploy.
5. **Stdlib templating only.** See [`concepts.md`](concepts.md) § Templating.
   No sprig; no nested Go templates.
6. **Fail loud on a missing input.** `helm-release` exits non-zero when
   `IMAGE_REF` is unset rather than deploying a placeholder tag.

## Ported compositions

`turbo-package` and `publish-stack` are ported **byte-identical** from
[`stack-tectonic`](https://github.com/sourceplane/stack-tectonic), which remains
their upstream.

Two catalogs with a shared shape will drift — a fix lands in one and not the
other. Keeping these two byte-identical means divergence has to be a deliberate
edit rather than an accident, and a diff against tectonic is a meaningful check.

If one of them genuinely needs to differ here, record *why* in this file at the
same time.

## Releasing a change

1. Edit the composition.
2. Bump `metadata.version` in `stack.yaml` (semver: a changed contract is at
   least a minor).
3. `./scripts/verify.sh`.
4. Merge, then push a `v<version>` tag.
5. Bump the pin in each consuming repo's `intent.yaml`. That bump is the
   upgrade — consuming repos never float.
