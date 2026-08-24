# Plan for cloud deployment/ops

Most of what needs deploying here is just more of your existing application — but this
playbook has quietly introduced a handful of new operational surfaces along the way:
[job 10](10-oauth.md)'s authorization server, [job 9](09-api-keys.md)'s rate limiter, and [job 11](11-logging-access.md)'s two
log stores. None of them need a parallel infrastructure stack, but all of them need to
actually be accounted for in how this gets run — the
[README's "real-time" interpretation](../README.md#meeting-regulatory-requirements) only holds
up in practice if the thing actually stays up and responsive.

## Reuse your existing ops stack

Deploy this alongside your main application's existing pipeline, scaling groups, secrets
manager, and observability stack — the same instinct [job 2](02-libraries-frameworks.md)
applied to choosing libraries and [job 11](11-logging-access.md) applied to logging
infrastructure applies here too. The new pieces below need to be *accounted for*, not
*separately reinvented*.

## The new operational surfaces this playbook introduced

- **The OAuth authorization server** ([job 10](10-oauth.md)) is now a critical-path
  dependency for every third-party integration, even though it's separate from your main
  application logic. If it's down, every OAuth-authenticated client is down, regardless of
  whether the rest of the API is healthy. Monitor and alert on it either as its own service
  or as endpoints within the same service that has your data API.
- **The rate limiter** ([job 9](09-api-keys.md)) needs a shared store (Redis or similar) once
  you're running more than one instance — a rate limit enforced per-instance instead of
  globally quietly stops meaning what its number says.
- **The two log stores** ([job 11](11-logging-access.md)) need their retention policy actually
  implemented as a real rotation/deletion job, not just documented as a policy. The low-volume
  grant log is worth backing up like any other compliance-relevant record; the high-volume
  access log generally isn't.

## Capacity planning for bursty, read-heavy traffic

This API's traffic shape is different from typical product traffic: third-party clients doing
[job 8](08-pagination.md)'s cursor-based incremental sync show up periodically and pull
whatever changed, and a first full sync or a
[job 13](13-gui-design.md) export walks a user's entire history in one burst. Plan capacity
for that pattern specifically — occasional large read bursts, not steady per-second traffic —
and treat [job 9](09-api-keys.md)'s rate limits as the actual backstop that keeps one client's
burst from degrading service for everyone else, not just a documented number nobody enforces
under load.

## Secrets and key management

Beyond the individual API keys and client secrets [job 9](09-api-keys.md) and
[job 10](10-oauth.md) already cover, the infrastructure underneath them needs its own secrets:
whatever signs OAuth access tokens, and whatever salt or pepper is used when hashing API keys.
Store these in a real secrets manager or KMS, not application config, and have an actual
rotation plan for the signing key itself — a compromised signing key is a different, larger
incident than a single compromised API key, since it can affect every token the server has
ever issued.

## An incident response runbook that actually uses what you built

[Job 11](11-logging-access.md)'s two logs exist specifically to make this possible: write down
the actual steps for responding to a compromised API key or OAuth client, so nobody has to
improvise it during an incident. At minimum: revoke the credential immediately
([job 9](09-api-keys.md)/[job 10](10-oauth.md)), pull the access log for the exposure window
to see what that credential actually touched, and notify affected users if what it touched
warrants it. A runbook that names which log answers which question during an incident is worth
more than either log existing in the abstract.

## Output of this step

A deployment plan that folds this playbook's new pieces — the authorization server, rate
limiter, and log stores — into your existing infrastructure and monitoring, sized for bursty
read traffic rather than steady-state, with real secrets management and a
written incident-response runbook that uses [job 11](11-logging-access.md)'s logs and
[job 9](09-api-keys.md)/[job 10](10-oauth.md)'s revocation on purpose rather than by
improvisation.
