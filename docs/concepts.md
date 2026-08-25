# Concepts

## The three layers

```
intent.yaml     repo-level: environments, triggers, discovery roots, the pinned
                composition source, the state backend
component.yaml  per-unit desired state: type, parameters, dependsOn, which
                environments it subscribes to and with what profile
this catalog    execution contracts: what a `terraform-aws` unit actually does
                when it runs — versioned, published, pinned
```

The rule that makes it work: **a component says what, never how.** A service's
`component.yaml` names its chart directory; it never names `helm upgrade`.
Changing how every Helm release in the fleet is rolled is one release here and
one tag bump there.

## Composition, schema, job template, profile

| File | Answers |
|------|---------|
| `composition.yaml` | Which jobs and profiles exist, and which are the defaults. |
| `schema.yaml` | What parameters a component of this type may and must set. Validated at plan time, so a typo fails before a runner starts. |
| `jobs/*.yaml` | Every step that could run, each tagged with a capability. |
| `profiles/*.yaml` | Which capabilities actually run in this lane, and which credentials the lane may resolve. |

A job template is the union of everything possible; a profile is the subset
chosen. That split is why one contract serves both the PR lane and the deploy
lane without a branch in a shell script.

## Capabilities are the safety boundary

A profile does not *skip* the credentialed steps — it excludes their
capabilities, so those steps are not in the plan at all. `terraform-aws`'s
`validate` profile has no `terraform-aws.oidc` capability, which means that lane
cannot authenticate to AWS even if the component supplies a role ARN and even
if someone edits the Terraform to try.

That is the difference between a lane that is *configured* not to touch
production and one that *cannot*.

## Safe defaults

Every composition's `defaultProfile` is its least-privileged lane. A component
that forgets to declare a profile gets validation, not apply.

## Templating

Job templates render through Go's stdlib `text/template` with **no funcmap** —
no sprig, no `splitList`, no `splitLines`. Available: `.parameters`, `.env`,
`.orun.component.name`, `.orun.environment.name`, and the stdlib builtins
(`if`, `range`, `index`, `eq`, …).

Two consequences worth internalising:

- **Split in the shell, not the template.** Space-separated parameters are
  looped over in bash.
- **Never nest a Go template.** `docker inspect --format='{{ index … }}'` and
  `kubectl -o go-template` are themselves templates; the planner would try to
  evaluate them against its own context and fail to render the job. Where a
  literal is genuinely needed, quote it: `` {{`{{.Group}}`}} ``.
