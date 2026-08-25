# `node-service-turbo`

Verify one deployable service inside a pnpm/Turborepo workspace. The pure-code
lane, upstream of `container-image`.

Modeled on `stack-tectonic`'s `cloudflare-worker-turbo` with the Wrangler
capabilities removed and a docker-compose smoke lane added in their place.

## Parameters

| Parameter | Required | What |
|-----------|----------|------|
| `nodeVersion` | yes | |
| `pnpmVersion` | yes | |
| `turboFilter` | yes | Filter selecting this service's workspace package. |
| `installCommand` | no | Default `pnpm install --frozen-lockfile`. |
| `typecheckCommand` / `lintCommand` / `testCommand` / `buildCommand` | no | Default to `turbo run <task> --filter=<turboFilter>`. |
| `smokeComposeFile` | no | Default `docker-compose.yml`. |
| `smokeCommand` | no | Non-empty enables the smoke lane. |

## The structure check earns its place

`verify-structure` fails a service with no `Dockerfile`. A `node-service-turbo`
component exists to be containerized; catching that in the cheapest lane beats
discovering it when the image component runs twenty minutes later.

## The smoke lane

`smoke` composes the service against local dependency doubles — LocalStack,
Redis, Postgres — and runs `smokeCommand` against it. It exercises what unit
tests mock away: that the service can reach its dependencies at all, and that a
*degraded* dependency degrades rather than fails. Logs are dumped and the stack
is torn down on every exit path, success or failure.

It is still credential-free, so it runs on every PR.

## Profiles

| Profile | Runs |
|---------|------|
| `quick-check` | install + structure + typecheck |
| `verify` (default) | + lint, test, build |
| `smoke` | + docker-compose smoke |
