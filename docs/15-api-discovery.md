# Support API discovery (including data schemas and OAuth scopes)

Everything referenced here already exists — [job 3](03-json-schema.md)'s schemas, [job 5](05-choosing-endpoints.md)'s
endpoints, [job 8](08-pagination.md)'s pagination choices, [job 9](09-api-keys.md)'s API key
auth, [job 10](10-oauth.md)'s scopes. This job is where all of it gets published as a document
a client can fetch and act on, rather than something a developer has to read and hand-translate
into code.  API discovery is optional but it can really help with usage, especially
with AI agents.

## One document, two standards — not a bespoke format

Reuse what already exists rather than inventing a custom discovery schema:

- **OpenAPI** describes the REST surface: endpoints, request/response shapes, and API-key
  auth. [Job 3](03-json-schema.md) already noted that OpenAPI 3.1 is fully compatible with
  JSON Schema draft 2020-12 — the same draft this playbook recommends — so job 3's schemas
  plug into `components.schemas` directly instead of being redescribed.
- **OAuth 2.0 Authorization Server Metadata (RFC 8414)** describes the OAuth layer
  specifically, published at the standard well-known location
  `/.well-known/oauth-authorization-server`. If [job 10](10-oauth.md)'s library speaks OpenID
  Connect (`node-oidc-provider` does), it likely already publishes the closely related
  `/.well-known/openid-configuration` document for free — check before hand-rolling this.

Together these are the actual deliverable, not one all-purpose format covering everything.
Existing tools (SDK generators, HTTP clients, generic OAuth libraries) already
know how to consume these resources, which is the entire point of using standards here.

## What goes in the OpenAPI document

- **Paths** — one entry per [job 5](05-choosing-endpoints.md) endpoint.
- **Schemas** — [job 3](03-json-schema.md)'s schemas, referenced via `$ref` rather than
  duplicated inline.
- **Pagination** — OpenAPI has no first-class "pagination mechanism" field, so make it
  discoverable through what's already there: the actual query parameters an endpoint declares
  (`cursor`, `year`, `type`, and so on, per [job 8](08-pagination.md)'s options) already imply
  which mechanism applies. Naming the mechanism explicitly in the endpoint's description, or a
  vendor extension like `x-pagination: cursor`, removes any doubt for a client trying to build
  one generic pagination handler across endpoints that don't all use the same approach.
- **Security schemes** — an `apiKey` scheme for [job 9](09-api-keys.md), and an `oauth2` scheme
  with a `scopes` map for [job 10](10-oauth.md). Pull the scope descriptions from the same
  plain-language wording used on job 10's authorization screen — [job 12](12-documentation.md)
  already said to reuse that wording verbatim in the human docs; this is the third place it
  should appear unchanged, not rewritten again.

## What goes in the OAuth metadata document

The standard RFC 8414 fields: `authorization_endpoint`, `token_endpoint`, `scopes_supported`
(the same scopes as above), and `revocation_endpoint` if [job 10](10-oauth.md)'s library
exposes one. Publishing this at its well-known path means a generic OAuth client library can
configure itself from just your base URL — no developer hand-copying endpoint URLs out of
[job 12](12-documentation.md)'s docs into their code.

## Generate it, don't hand-maintain it

[Job 12](12-documentation.md) already flagged the risk of hand-maintaining two descriptions of
the same API that can quietly disagree — that risk is highest here, since this document is
what other software parses, not a forgiving human reader. Generate it from the same
route/schema definitions [job 2](02-libraries-frameworks.md) already chose libraries for:
`drf-spectacular` derives an OpenAPI document from a Django REST Framework project directly;
on Node, `tsoa` generates one from TypeScript controllers and models.

## Don't let generated documentation replace high-level documentation

Generating this document is not the same job as [job 12](12-documentation.md)'s documentation,
and it's worth being explicit about the difference. A tool that reads your schema and prints
"`Activity` is an `object` with the field `date`, which is a `datetime`" hasn't told an
integrator anything the schema didn't already say — just in a more bloated, less
machine-readable form. If that's genuinely all your documentation adds, integrators are better
off reading the schema directly.  Higher-level documentation exists
for what a schema can't say: how to actually walk a
[job 8](08-pagination.md) cursor, what a [job 10](10-oauth.md) scope means for someone deciding
whether to grant it, what to expect from a `429`. Autogenerating the reference documentation is
good, but don't let the ease
of that stand in for the prose only a person can write.

## Consider offering MCP as a thin layer

[Job 10](10-oauth.md) already described how to authorize AI agents that run either
on the cloud or on the user's device.  This step goes further in supporting AI agent
access to users' personal data, and suggests wrapping your API calls in an MCP layer
that agents can figure out how to use.

Keep that layer thin. Each MCP tool should map to an existing [job 5](05-choosing-endpoints.md)
endpoint and return the same [job 3](03-json-schema.md) schemas, with no new business logic of
its own. In many Web API frameworks, you may be able to wrap each View method or object
for the regular API in an MCP view.  OAuth tokens and scopes (job 10) can be reused, since
MCP's authorization spec is built on OAuth — and the RFC 8414 metadata document this chapter
already publishes is one of the pieces MCP clients look for. 

## List your API in the Data Trust Registry

The [Data Trust Registry](https://dt-reg.org) (DTR) is the ecosystem registry
where data destinations can *discover* your API.  It's the same registry that allows
you to check that third parties accessing your API are trusted.  If onboarding or
supporting your users involves fetching their data from other services in the ecosystem,
you'll have even more reason to join as a destination as well as a source.

To get listed:

1. **Create an account** on the DTR site and start an application from
   [Join the registry](https://dt-reg.org/apply-to-registry/). The Application Guide linked
   from there lists the specific requirements for each step.
2. **Start at the private, test-only level** while you're still building. This lets you test
   transfers with other verified participants without a public listing, and gives you time to
   pull together what a full application needs.
3. **Apply for a trusted listing** when you're ready to go public. Expect to provide an
   organisational identifier (such as an LEI or DUNS number), your country of jurisdiction,
   and attestations about your data security practices and end-user authentication. If your
   service holds sensitive content, the higher trust level also asks for evidence of regular
   security audits, such as SOC 2.
4. **Add your connection information** to your registry entry, so destinations can find this
   job's OpenAPI and OAuth metadata documents.

The DTR reviews listings annually and works with its members to
iterate on requirements and technical choices in both trust and discovery/connection.


## Output of this step

An OpenAPI document covering endpoints, schemas, pagination parameters, and security schemes,
plus an RFC 8414 metadata document for the OAuth layer, both generated from existing code
rather than maintained by hand, both served at stable, well-known locations a client can
discover without being told where to look first. A Data Trust Registry listing lets data
destinations find the API in the first place.
