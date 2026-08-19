# Identifying what counts as portable data

Before you write any API code, you need an inventory: which parts of your data model are
"portable data" under the regulations this playbook targets, and which aren't. Get this step
wrong and you'll either over-expose data you shouldn't, or under-deliver and miss the compliance
bar. This is the foundation the [JSON Schema](03-json-schema.md) work and API design build on.


## The three compliance categories

Regulators (GDPR Article 20, the DMA) draw a rough line between data the user is responsible
for existing, and data your service computed about them. In practice, sort every field or table
in your data model into one of three compliance categories:

1. **Provided data** — the user typed it, uploaded it, or configured it directly. A playlist
   name, a note's contents, a saved route. Always portable.
2. **Observed data** — generated as a side effect of the user doing something on your service.
   Listen history, watch history, chat history, ratings. Usually portable — this is the category
   people most often forget, because it wasn't "entered" anywhere.
3. **Inferred/derived data** — your service computed it *about* the user. Recommendation
   scores, embeddings, fraud/risk flags, internal quality rankings. Generally **not** required
   to be portable, and often shouldn't be exposed even if requested — it can reveal proprietary
   models or internal security signals.

If you're unsure which category something falls in, ask: "did this exist because the user acted,
or because we analyzed the user?"  Data from analyzing the user goes into the inferred/derived
data category.
s
## Examples

| Service type | Provided | Observed | Inferred (exclude) |
| --- | --- | --- | --- |
| Music streaming | Playlists | Listen history | "Taste vector" embeddings |
| Video streaming | Watch lists | Watch history | Recommendation scores |
| Note-taking | Notes, folders | Edit timestamps | Auto-generated tags/summaries* |
| AI chat | — | Chat history | Internal moderation flags |
| Maps | Saved favourites | Route history | Predicted-destination models |

\* Some derived fields are borderline — a summary the user actually sees and relies on may be
worth including even though your service generated it. When in doubt, lean toward including
anything the user would recognize as "their data."

## Doing the inventory

Walk your schema (database tables and/or ORM models) and produce a simple table:
field/table name → bucket → notes. A few things to watch for:

- **Data about other users mixed into your data.** A shared playlist or a comment thread
  contains data from multiple people. This needs separate handling — see
  [job 7](07-references-to-other-users.md) — don't just dump it in wholesale.
- **Soft-deleted or archived records.** If a user can still see it (or could restore it),
  it's still their data.
- **Denormalized/duplicated fields.** Pick the canonical source so you don't export the same
  fact from three tables with three names.
- **Aggregate-only data with no per-user link.** Not personal data at all — out of scope
  entirely, not just excluded from the export.
- **Blob data in cloud storage buckets.** This can be overlooked because blobs don't typically
  link back to user account records.  Instead, you might find bucket paths or URLs in tables
  that also hold the user name or account ID.

## Break down types

Users don't always want all their personal data in one export.  Your personal data access
features will be significantly more useful (and might even become a source of partnerships
or user retention value) if data is organized usefully for access.  It's hard to say a lot
about this without knowing your data domain, but here's some principles:

* Keep activity history separate from content matter.  Uploaded videos should be separately
  accessible from video watch history.
* Keep metadata with objects.  Save "# of likes" as metadata with photos (see JSON formats for
  specific advice about how to do this.


## Output of this step

You should end up with a short document or spreadsheet: one row per data type, its bucket, and
which underlying table(s) it comes from. That inventory is the direct input to defining your
[JSON Schema formats](03-json-schema.md) and deciding how to
[hook your storage into the API](04-hooking-storage-into-api.md).

