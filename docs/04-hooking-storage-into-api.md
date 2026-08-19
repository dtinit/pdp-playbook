# Hook your service's data storage into an API

With your data inventory ([job 1](01-identifying-portable-data.md)), your schemas
([job 3](03-json-schema.md)), and a validator/mapper library picked out
([job 2](02-libraries-frameworks.md)), this job wires them together: for each schema, a read
path that pulls from your real storage and produces exactly what the schema promises.

## Don't serialize your internal objects directly

The tempting shortcut is to hand your ORM row (or equivalent internal object) straight to a
JSON serializer and call it the API response. Resist this. It couples your public, versioned
contract to your internal storage shape — rename a column or refactor a table, and you've
silently broken every third-party integration reading that endpoint, with no schema version
bump to signal it. It can also leak fields you never meant to expose, including exactly the
inferred/derived data [job 1](01-identifying-portable-data.md) told you to keep out.

The fix is an explicit mapping/templating layer between storage and wire format: something
that takes your internal representation and produces exactly the shape your job 3 schema
promises, field by field, on purpose. This is also what makes "extend in place, don't
version" from job 3 actually safe — your internals can change all day as long as the mapping
layer keeps producing the same external shape.

The good news is that having written your external-facing data schemas already, AI coding
assistants are likely to do a very good job of writing the transformers or templates needed.

## One mapper per schema

- Build one mapping function or serializer per schema from job 3, not a generic
  "convert any model to JSON" helper. A `Playlist` maps differently than a `ListenEvent`;
  let them.
- For collections — albums, playlists, anything job 3 said needs an index file — the mapper
  is where you join the underlying blob listing with whatever per-item metadata table holds
  the fields the blob format itself can't carry. It's also where the "canonical source"
  decision you made during the job 1 inventory, for facts that were duplicated across tables,
  actually gets resolved into a single value.
- Keep fetch logic and shaping logic as separate steps: fetch → map → respond. Tangling a
  database query with JSON-shaping logic makes both harder to change independently later.

## Validate the mapper's output, not just its inputs

Add tests that run real (or representative) records through the mapper and check the result
against the job 3 schema, using the validator you picked in job 2. This is what catches drift
between an internal change and your public contract before a third party does — treat a
failing schema validation in CI the same as a failing type check.

## Output of this step

Working read endpoints, each backed by a mapper that produces its job 3 schema's shape from
real storage and is tested against it. [Job 8](08-pagination.md) builds pagination on top of
these reads, and [job 14](14-api-discovery.md) points clients at them.
