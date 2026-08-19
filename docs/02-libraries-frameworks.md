# Suggest some libraries and frameworks for the project

This assumes you already have a web framework in place — Express or Fastify on Node, Django
in Python, whatever the equivalent is elsewhere — and just need to add personal data 
portability. This job is just
about which libraries to reach for when implementing the steps to come.

## Survey what's already there before adding anything new

Before reaching for any library below, check what your project — and any closely related
projects in your org — already has in place. It's common to find a JSON validator or an OAuth
integration already wired up for an unrelated reason (an internal admin API, a webhook
receiver, a partner integration) that's perfectly reusable here instead of a second dependency
doing the same job. This applies across everything this playbook will need a library for.

In a multi-repo org, a quick search across sibling repos is worth it too — a platform team may
already maintain a shared API-key or OAuth service you're meant to plug into rather than
reimplement. Treat the picks below as defaults for when this survey comes up empty, not a
mandate to add them regardless of what's already there.

- **JSON Schema validation** — check `package.json`/`requirements.txt` (or your framework's
  equivalent) for `ajv`, `jsonschema`, `fastjsonschema`, or similar already in the dependency
  tree, and grep for existing `validate(...)` or validator-class usage.
- **Templating/transform** — search for existing serializer/DTO/presenter layers (DRF
  `Serializer` subclasses, GraphQL resolvers, an internal "view model" pattern) that may
  already do the field-by-field mapping [job 4](04-hooking-storage-into-api.md) describes.  
- **API key management** — many services already issue API keys for something (internal
  tooling, a partner integration, rate limiting on a public endpoint). Check for an existing
  keys table or key-hashing/lookup middleware before building a second key system for
  [job 9](09-api-keys.md).
- **OAuth** — if the service already supports "Sign in with X," check whether the OAuth
  *server* side is also present or partially present, so [job 10](10-oauth.md) can extend it
  rather than stand up a second OAuth stack.
- **Pagination** — check whether any existing list endpoint already has a cursor or
  offset/limit pattern, and whether your ORM has cursor/keyset support built in (Prisma,
  Django's `QuerySet`, and others do). A pattern that already works elsewhere in the codebase
  is one less thing to design for [job 8](08-pagination.md).

Note that the recommendations below for a template or transform library (your taste) don't
mention serialization libraries, and that's intentional. An automated serialization approach
is a bunch of invisible traps laid on your future path, because the format you'll be mapping
data to will be relied on externally — see [job 4](04-hooking-storage-into-api.md) for why.

It's also worth saying plainly: for API keys and pagination especially, a hand-rolled solution
is often the right call, not a compromise. Both are small enough in scope (a hashed random
string looked up in a table; a cursor built from a sort key and an ID) that a library can add
more indirection than it saves. The picks below are for when you'd rather not write that code
yourself, not a signal that you need a dependency to do this properly.

## JavaScript / Node

- **Mapping: [JSONata](https://jsonata.org/)** — a declarative JSON query/transformation
  language purpose-built for reshaping data from one JSON structure into another, with no
  arbitrary code execution, so it's safe even if a mapping ever needs to be config-driven
  rather than hard-coded. If you'd rather stay in plain JS with no new syntax to learn, a
  hand-written mapper function per resource (`function toPlaylistJSON(row) { ... }`) does the
  same job with more code and less magic — either is a legitimate choice.

- **Validation: [Ajv](https://ajv.js.org/)** — the standard choice, fastest JSON Schema
  validator in the JS ecosystem and the most spec-compliant, supporting draft 2020-12. Plug
  it into middleware to validate outgoing responses against your schemas, or use it in tests
  as described in [job 4](04-hooking-storage-into-api.md).

- **API keys: [generate-api-key](https://www.npmjs.com/package/generate-api-key)** for
  issuing keys plus **[passport-headerapikey](http://www.passportjs.org/packages/passport-headerapikey/)**
  for checking them on incoming requests — store a hash of the key, never the key itself, the
  same way you'd store a password. This is for identifying and rate-limiting a *client*
  ([job 9](09-api-keys.md)), not for authorizing access on behalf of a *user* — that's OAuth's
  job, below.
- **OAuth: [node-oidc-provider](https://github.com/panva/node-oidc-provider)** — the most
  complete and actively maintained option for standing up an OAuth 2.0/OpenID Connect
  authorization server in Node, and OpenID Certified. If you only need bare OAuth2 (no OIDC
  identity layer) and want something lighter, **[@node-oauth/oauth2-server](https://www.npmjs.com/package/@node-oauth/oauth2-server)**
  is framework-agnostic and has an Express adapter. Either feeds [job 10](10-oauth.md).
- **Pagination:** no single dominant library for generic REST cursor pagination — most teams
  build a cursor directly on top of whatever's already querying the database. If you're on
  Prisma or Knex, their built-in keyset/cursor query support gets you most of the way there in
  a few lines. On MongoDB specifically,
  [mongo-cursor-pagination](https://www.npmjs.com/package/mongo-cursor-pagination) is a solid,
  purpose-built option. See [job 8](08-pagination.md).

## Python / Django

- **Validation: [jsonschema](https://python-jsonschema.readthedocs.io/)** — the reference
  implementation and most portable option. If validation throughput becomes a bottleneck,
  swap in [fastjsonschema](https://horejsek.github.io/python-fastjsonschema/) (compiles the
  schema to Python code, roughly two orders of magnitude faster) or
  [jsonschema-rs](https://pypi.org/project/jsonschema-rs/) (Rust-backed, holds up well on
  more complex schemas) — same schemas, drop-in swap.
- **Mapping:** Django REST Framework `Serializer`s — plain `serializers.Serializer` with
  explicit fields, not `ModelSerializer` with `fields = "__all__"`. The explicit version is
  already the mapping layer you want, and it's likely already in your dependencies if you're
  running DRF. On plain Django without DRF,
  [Marshmallow](https://marshmallow.readthedocs.io/) does the same job framework-agnostically.
- **API keys: [djangorestframework-api-key](https://github.com/florimondmanca/djangorestframework-api-key)**
  — issues and checks API keys as a DRF permission class, with revocation built in. It's
  explicitly for identifying and rate-limiting a *client* ([job 9](09-api-keys.md)), not for
  authorizing access on behalf of a *user* — the package's own docs point you at OAuth for
  that, which is job 10 below.
- **OAuth: [django-oauth-toolkit](https://django-oauth-toolkit.readthedocs.io/)** — the
  standard choice for turning a Django project into an OAuth 2.0 authorization server, built
  on `oauthlib`, jazzband-maintained, and integrates directly with DRF. Feeds
  [job 10](10-oauth.md).
- **Pagination:** Django REST Framework's built-in `CursorPagination` — no extra dependency,
  it already ships with DRF. Reach for `LimitOffsetPagination` instead only for small,
  admin-style listings where jumping to an arbitrary offset matters more than performance at
  scale. See [job 8](08-pagination.md).

## Other languages

Follow the same pattern in whatever framework you're already running: a fast, spec-compliant
JSON Schema validator; an explicit field-by-field mapper rather than reflection-based
auto-serialization; a key-issuing/checking library (or hand-rolled equivalent) for client
identification; either an OAuth authorization-server library or toolkit for user-authorized
access; and a cursor pagination approach, likely already available in your ORM or query
builder. The specific package names change; jobs to be done don't.

## Output of this step

A validator, a mapping approach, and your picks — or deliberate hand-rolled approach — for
API keys, OAuth, and pagination, reused from what already exists in your project where the
survey above found something, added fresh where it didn't. The validator and mapper get wired
into your build or test suite now, so any response can be checked against its
[job 3](03-json-schema.md) schema before it ships — which is what
[job 4](04-hooking-storage-into-api.md) will build on to connect these to real storage. The
API-key, OAuth, and pagination picks wait until [job 8](08-pagination.md),
[job 9](09-api-keys.md), and [job 10](10-oauth.md) respectively.
