# Plan for the future with a schema evolution strategy

[Job 3](03-json-schema.md) set the policy: extend schemas in place, treat a version bump as a
last resort. [Job 12](12-documentation.md) told integrators how to survive that policy on
their end. This job is where the policy stops being a stated intention and becomes something
actually enforced — mainly through tests that catch a violation before it ships, not after an
integrator reports one.

## Classify every schema change before it ships

- **Additive** — a new optional field, a new enum value, a new endpoint. This is the default
  path [job 3](03-json-schema.md) already described: ship it in place, no version bump.
- **Transitional** - while preparing to remove or rename fields, or improve their format
  (e.g. transitioning to a time field with timezone instead of one without), it's often
  possible to keep the field that existing clients may depend on.  
- **Breaking** — renaming or removing a field that clients may depend on, or changing its
  format. This is exactly the case [job 3](03-json-schema.md) said to be *prepared* for, not
  to expect often.

Additive and transitional changes can update the schema to keep informing clients what
to expect (and keep checking your own code against your interface contract).

When a breaking change is genuinely preferable, define a new `$id` for a new schema.  
Try to keep the old version around unchanged rather than immediately replaced. List both
listed in [job 15](15-api-discovery.md)'s discovery document, and communicate a
timeline for the old version to be retired.

## Tests that expose gaps

Classifying changes by hand relies on someone remembering to do it correctly, for every change,
forever. The reliable version of this policy is automated:

- **Diff the schema/OpenAPI document in CI, and fail the build on a breaking diff.**
  [oasdiff](https://github.com/oasdiff/oasdiff) checks a wide range of breaking-change
  categories against an OpenAPI document, runs as a CI step, and exits non-zero when it finds
  one — pairing directly with [job 15](15-api-discovery.md)'s generated OpenAPI document, since
  that's exactly the artifact it diffs build over build. This is what actually catches "someone
  quietly removed a required field" before merge, rather than relying on a reviewer noticing.
- **Regression-test the mapper's real output, not just its shape at one point in time.**
  [Job 4](04-hooking-storage-into-api.md) already said to validate the mapper's output against
  its schema; keep a set of representative fixture records around and re-run that validation
  on every change, so a change to the *code* producing the data (not just the schema file) that
  silently alters output shape gets caught the same way a schema edit would.
- **Simulate the naive strict client job 12 warned about.** [Job 12](12-documentation.md)
  described a real failure mode: an integrator who copied the schema and validates
  incoming responses against it strictly. Write that exact test — validate current live
  responses against a *frozen* copy of last release's schema — and treat a failure as a signal
  worth investigating even when the change was intentional, since it tells you exactly what a
  poorly-behaved but common integration pattern will do when this ships.

## Output of this step

A CI pipeline that fails the build on an undocumented breaking change, fixture-based regression
tests protecting the mapper's actual output over time, and a real version-cutover and sunset
process for the rare cases that need it. This is what turns [job 3](03-json-schema.md)'s
policy and [job 12](12-documentation.md)'s guidance to integrators from a stated intention into
something enforced — keeping, over time, the same promise
[job 1](01-identifying-portable-data.md)'s inventory made on day one about what a user's data
would look like when they asked for it.
