# Allow efficient async requests for chunks of bulk data with pagination or cursors

[Job 5](05-choosing-endpoints.md) mentioned the possibility of pagination, cursors, or some
other sync key. There's no single right answer here — the choice depends on how big a
collection actually gets, how it's queried, and whether "what's new since last time" matters
for that particular endpoint. This section walks through the real options for the
activity/history endpoints [job 1](01-identifying-portable-data.md) and
[job 3](03-json-schema.md) both called out as unbounded, with what each option costs and buys.

While breaking up large collections, consider how they're ordered.
Ordering by date starting from most recent is often the most useful, because 
the user or their software may be most interested in recent data.  

## Option 1: do nothing

If a collection reliably stays small enough, just return all of it in one response.

- **Pros:** zero design or implementation cost. No cursor encoding, no offset math, no
  concurrent-write edge cases to reason about. The simplest possible client experience — parse
  one response, done.
- **Cons:** it only works as long as the size assumption holds. A long-lived power-user
  account, or a service with no retention limit on history, can eventually blow past whatever
  you assumed, and there's no graceful fallback once a response is too large — you have to
  retrofit pagination under pressure instead of by choice.
- **When it fits:** a collection that's bounded by the product itself (a capped favorites
  list, or capped list of other accounts that can be "followed")
  or by real usage data from [job 1](01-identifying-portable-data.md)'s inventory (such as a
  personal watch history at a service with a few years of typical usage, or a list of forum
  memberships).
  Revisit this as usage data comes in.

## Option 2: simple offset/page pagination

`?page=3` or `?offset=200&limit=20`.

- **Pros:** nothing new to store or encode — it's stateless, and a client can construct,
  bookmark, or jump to a specific page directly. Every ORM and framework supports it with no
  extra code. And despite its reputation, it isn't always a full table scan: whether an offset
  query is cheap or expensive depends on the database and the index behind it, and for
  collections that are bounded (just larger than Option 1's comfort zone), the cost may never
  actually get large enough to matter. Measure before ruling it out.
- **Cons:** in most common setups, cost does grow with the offset, so it's worth actually
  testing against realistic data volumes rather than assuming either way. It's also unstable
  under concurrent writes — an insert or delete between two page fetches shifts what "page 4"
  means, and a client can silently skip a row or see one twice with no way to detect it. And it
  doesn't naturally support "give me what's new since I last checked," since page numbers
  aren't tied to when something was added.
- **When it fits:** collections large enough that "do nothing" doesn't apply, small enough
  (or indexed well enough) that offset cost stays flat in practice, and where clients aren't
  expected to do incremental "what's changed" polling.

## Option 3: cursor (keyset) pagination

- **Pros:** query cost stays flat regardless of how deep a client pages — a keyset lookup, not
  a growing scan — and it's stable under concurrent writes, since inserts elsewhere don't shift
  anything already returned. It's also the natural fit for "what's new since last time": a
  saved cursor is a valid starting point on a later, independent request, which is exactly the
  mechanism the DMA's "continuous" access requirement needs without any streaming protocol.
- **Cons:** it's genuinely new work. You're building and maintaining an opaque token format,
  encode/decode logic, and a correctly-chosen sort key (immutable, and unique enough to break
  ties — get either wrong and cursors misbehave in ways that are easy to miss in testing). A
  cursor issued today also has to keep meaning the same thing whenever it's used later, which
  constrains what you can change about the underlying sort key or storage without breaking
  cursors already handed out.
- **What it costs to build:** less than it sounds, most of the time. [Job 2](02-libraries-frameworks.md)
  covers the specific picks — Django REST Framework's built-in `CursorPagination` needs no new
  dependency, Prisma or Knex's keyset support on Node gets most of the way there in a few
  lines — and a hand-rolled version is a legitimate default too, not a compromise. The core of
  it is a keyset comparison:

  ```sql
  SELECT * FROM events
  WHERE (created_at, id) > (:cursor_created_at, :cursor_id)
  ORDER BY created_at, id
  LIMIT :limit + 1  -- fetch one extra to know if there's a next page
  ```

  wrapped in a token like `base64({"createdAt": "...", "id": "evt_492"})`, opaque to the
  client on both ends. Wrap every response the same way, defined as its own
  [job 3](03-json-schema.md) schema:

  ```json
  {
    "items": [ ... ],
    "nextCursor": "eyJjcmVhdGVkQXQiOiIyMDI2LTA4LTIwVDEyOjAwOjAwWiIsImlkIjoiZXZ0XzQ5MiJ9"
  }
  ```

- **When it fits:** large or fast-growing collections, especially ones where third-party
  clients need reliable incremental sync rather than a one-time export.

## Option 4: partition by a natural facet of the data

Instead of an opaque cursor or an arbitrary count, split a collection along some real
attribute of the data itself — a **facet**. Two common shapes, though not the only ones:

- **Range facets** — a continuous, orderable attribute. Date is the obvious case for
  chronological data: `GET /listen-history?year=2024`, defaulting to the current year, or
  month if a year is still too much. A numeric range (file size, an ID range) works the same
  way.
- **List facets** — a discrete, natural category. A user's favorites on a knitting site span
  yarns, patterns, other users' projects, and other users' posts — types that usually don't
  need to be downloaded together. `GET /me/favorites?type=pattern` is a natural, bookmarkable
  page, and sorting the whole list by date favorited would be less useful here than splitting
  by type first.

Look for whatever facet is genuinely native to the data — geography, status, or whatever
dimension users already use to think about their own collection — rather than inventing one
just for pagination's sake.

- **Pros:** natural, human-legible boundaries with no cursor encoding — bookmarkable,
  guessable, and often cheap to index, since the facet is usually already a real column.
- **Cons:** range and list facets fail differently. A range facet like date usually has a
  sensible default (current year) and a natural walk order (newest to oldest), but page size
  still isn't bounded — an unusually active year can still be too large for one response, and
  a quiet one returns almost nothing. A list facet like type has no inherent default or order
  at all — there's no "first" category — so the client needs to know the valid facet values
  up front, which means publishing them somewhere discoverable
  ([job 15](15-api-discovery.md)). Either way, this is a coarse first split, not a full
  replacement for pagination within one facet value, and it doesn't generalize to collections
  with no natural facet to split on.
- **When it fits:** any collection where a real attribute of the data already gives it
  meaningful, bookmarkable chunks — activity/history data by date is the most common case,
  but not the only one. Combine with Option 2 or 3 for pagination within a facet value once
  one exceeds a safe size.

## Option 5: WebDAV sync or JMAP changes (deletion-aware sync)

For the specific problem of telling a returning client everything that changed since last
time — including deletions, not just additions — two existing standards already solve this,
one in XML and one natively in JSON.

The **WebDAV Sync extension** (RFC 6578, originally built for CalDAV/CardDAV) defines a
`REPORT` request carrying a `sync-token`, answered with every member added, changed, or
removed since that token, plus a fresh token for next time.

**JMAP** (RFC 8620) solves the same problem natively in JSON, and maps more directly onto a
collection endpoint. Its `Foo/changes` method takes a `sinceState` (the opaque `state` string
returned by a previous call) and returns `created`, `updated`, and `destroyed` id arrays plus
a `newState` to store for next time — if there are more changes than the server wants to
return at once, it returns an intermediate state and expects the client to keep calling until
caught up, which is pagination and sync-token combined. Its `Foo/queryChanges` method goes
further and diffs a specific filtered/sorted *view* — closer to what a paginated
`GET /listen-history` collection actually is — returning which items were added or removed
from that view and at what positions.

- **Pros:** both are real, standardized answers to incremental sync, including the deletion
  case that keyset cursors don't handle cleanly — a cursor pass has no way to tell a client
  "item X used to exist and is now gone," and these do. JMAP's version has the added benefit
  of being JSON already, with no XML translation layer needed.
- **Cons:** both are genuinely heavier than most collection API endpoints, just in different ways.
  WebDAV's `REPORT` bodies are XML by
  default. JMAP isn't REST at all — it's JSON-RPC-style, with method calls (optionally
  batched) posted to a single endpoint, so adopting it means a different API architecture,
  at least for the query portion of your API. Either one, or
  reimplementing just the sync-token/state semantics on a plain REST endpoint (which, at that
  point, isn't really WebDAV or JMAP — just borrowing the idea), cuts against this playbook's
  [stated basic approach](../README.md#basic-approach) of plain HTTP+JSON, chosen specifically
  for familiarity.
- **When it fits:** mainly if you're already running a WebDAV-based service (calendars,
  contacts, file sync) or already speak JMAP, where clients already understand the protocol.
  Less compelling as something to bolt onto a from-scratch JSON API purely for this one
  feature — though if deletion-aware sync matters and you don't want either full protocol,
  the shared underlying *model* (an opaque state token, with created/updated/destroyed
  returned alongside a fresh token) is worth copying into a plain JSON response, rather than
  reinventing that shape from scratch.

## Choosing

No single approach has to win for every endpoint, even within one API:

- Bounded, small collections → do nothing.
- Data with a genuinely natural facet (date, type, region) → partition by that facet,
  combined with pagination within one facet value once needed.
- Large or fast-growing collections, or clients that need reliable "what's new" → cursors.
- Deletions need to be communicated as part of sync, not just additions → the WebDAV sync or
  JMAP changes model, whether or not you adopt either protocol wholesale.
- Everything else, especially where offset already performs fine at your actual scale →
  simple pagination, revisited if growth outpaces it.

Whatever's chosen per endpoint, document it consistently —
[job 15's discovery document](15-api-discovery.md) is where a client finds out which
mechanism a given collection uses.

## Output of this step

A pagination approach chosen deliberately per [job 5](05-choosing-endpoints.md) collection
endpoint, based on that endpoint's actual data shape and growth — not defaulted to cursors, or
to nothing, without checking whether the assumptions behind that choice hold.
[Job 15's discovery document](15-api-discovery.md) describes which mechanism each collection
endpoint uses.
