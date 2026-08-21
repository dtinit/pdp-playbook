# Publish documentation

Everything so far — [job 3](03-json-schema.md)'s schemas, [job 5](05-choosing-endpoints.md)'s
endpoints, [job 8](08-pagination.md)'s pagination choices, [job 9](09-api-keys.md)'s keys,
[job 10](10-oauth.md)'s scopes — is internally consistent, but none of it is legible to an
actual third-party developer until it's written down in one place they can read before writing
any code. [Job 15](15-api-discovery.md) will make the shape of the API discoverable by
machines; this job makes it understandable by people.

## What to actually include

- **Getting started** — how to register a client, get an [API key](09-api-keys.md) and/or
  [OAuth](10-oauth.md) client credentials, and make one successful authenticated request.
  Nobody should have to reverse-engineer this from the reference material.
- **Endpoint reference** — one entry per [job 5](05-choosing-endpoints.md) endpoint: method,
  path, auth required, and request/response shapes linked to their
  [job 3](03-json-schema.md) schemas. Say plainly which [job 8](08-pagination.md) mechanism
  each collection endpoint uses — job 8's own discovery metadata says *what* the mechanism is;
  the prose here is what explains *how* to actually use it, cursor value and all. An OpenAPI
  document is a reasonable way to produce this rather than hand-writing it: it can reference
  the same [job 3](03-json-schema.md) schemas directly, and tools like Swagger UI or Redoc
  turn it into a browsable, interactive reference for free.
- **Scopes reference** — the same plain-language descriptions used on
  [job 10](10-oauth.md)'s consent screen, reused verbatim rather than redescribed. If the
  wording drifts between what a user approves and what a developer reads, one of them is
  wrong.
- **Rate limits** — the actual numbers from [job 9](09-api-keys.md), and what a `429` response
  looks like, so a client author budgets for it instead of discovering it in production.
- **Error format** — pick one consistent error response shape and document it, if nothing
  earlier in this playbook already forced the decision. Writing the docs is often exactly when
  you notice every endpoint has been improvising its own ad hoc error body — better to
  standardize it here than let integrators reverse-engineer a different shape per endpoint.

## Tell integrators how to survive your own evolution policy

[Job 3](03-json-schema.md) made two decisions that only work together if the documentation
says so out loud: schemas are strict (`additionalProperties: false`) *and* the default way to
evolve them is adding new optional fields in place, not cutting a new version. That's safe on
the server side, since the schema file and the data it validates change together. It's not
automatically safe on the client side — a third party that fetched the schema once, saved a
copy, and validates incoming responses against that frozen copy will start rejecting perfectly
valid responses the moment a new optional field is added.

Say this explicitly: integrators should treat the schema's `$id` as a live document to
re-fetch, not a frozen contract to copy once, and their own validation should tolerate
unrecognized fields even where the server-side schema doesn't. This is the single most
important thing job 12 can tell a third-party developer that isn't obvious from reading the
schema or the endpoint list alone.

## Docs as code, not a wiki

Keep documentation in the same repository, versioned and reviewed alongside the API code it
describes, rather than in a separate wiki or doc tool that can silently drift out of sync with
what [job 16](16-schema-evolution.md)'s evolution actually does to the schemas. Once
[job 15](15-api-discovery.md)'s machine-readable discovery document exists, generate as much
of the endpoint and scope reference from it as you can, rather than hand-maintaining two
descriptions of the same thing that can quietly disagree — but don't wait for job 15 to write
this documentation now; everything it needs already exists from jobs 3, 5, 8, 9, and 10. An
OpenAPI document is a natural candidate for that eventual single source: it's checked into the
same repository as ordinary text, and the same file can drive both this job's generated
reference and job 15's machine-readable discovery.

## Output of this step

A single published reference — getting started, endpoint reference, scopes, rate limits, and
error format — kept in version control alongside the API, that explicitly tells integrators how
to consume a schema that's designed to grow. [Job 15](15-api-discovery.md) later adds the
machine-readable counterpart to this same information.
