# Choose appropriate API endpoints

With schemas ([job 3](03-json-schema.md)) and mappers that produce data objects from real storage
([job 4](04-hooking-storage-into-api.md)) already in place, this job decides the URL surface clients actually
hit. The goal is one endpoint — or endpoint pair — per data type from the
[job 1](01-identifying-portable-data.md) inventory, not a single do-everything endpoint.

## One endpoint set per schema, not per table

Map endpoints 1:1 to the schemas from job 3, following the same split job 1's "break down
types" principle asked for: content and activity history are separate resources, not merged
into one omnibus endpoint.

- **Collection endpoint** — `GET /playlists`, `GET /listen-history`. Every data type gets one
  of these; for activity/history types, this is usually the *only* endpoint you need.
- **Item endpoint** — `GET /playlists/{id}`. Add this where individual items are meaningfully
  addressed on their own, not for every type reflexively.

## Scope every endpoint to the authenticated caller

Portability data is inherently personal, so put that in the URL structure itself:
`GET /me/playlists`, not `GET /playlists?user_id=...`. This makes
[job 6's access control](06-access-control.md) a structural property of the endpoint rather
than something every handler has to remember to check, and it means there's no user ID in a
URL for anyone to guess or enumerate. This is still usable by third parties: a third party
holding an OAuth token issued by a particular account can access that same `/me/playlists`
on the account owner's behalf — see [job 10](10-oauth.md).

> [!WARNING]
> Don't add an admin or support override to these endpoints — no `?user_id=` that only staff
> may use, no role check that lets an admin's token see past `/me`. Every such exception is a
> path around the structural guarantee above, and one bug or stolen token away from exposing
> every user's data. If staff need to read user data, give them completely separate endpoints,
> ideally on a separate internal host, with their own authentication, unreachable by any
> third-party OAuth token, and logged by [job 11](11-logging-access.md).

## Collections, items, and blobs

The index file job 3 said collections need (albums, playlists, anything with per-item
metadata) *is* the collection endpoint's response body — you're not building a second
representation. For content objects backed by a blob (a photo, a video), you usually
don't proxy the raw bytes through this JSON endpoint unless the content is reliably small
like a profile thumbnail.
The default approach is to return the metadata/index and let it point at the blob
directly, ideally via a short-lived signed URL. That keeps every endpoint here uniformly JSON
and leaves you free to change blob storage or add a CDN later without touching the API shape.

Watch for references to other users showing up in what looks like a single-user resource — a
shared playlist or a comment thread. [Job 7](07-references-to-other-users.md) covers how to
handle that without just filtering it out silently.

Collections *may* need some kind of pagination, see [job 8](08-pagination.md), but many collections can
be returned in one GET.

## Signed endpoints need hardening

A signed URL is a bearer credential for one file: anyone holding it can download that file,
and the request never touches [job 6](06-access-control.md)'s identity check. This is super useful for 
blobs that are already in network-attached storage and can greatly improve performance
of export operations.  But since access control is done when the signed URL is issued,
everything job 6 enforces has to happen before the URL is issued, and its lifetime is the only control
left afterwards.

- **Sign only what a scoped query returned.** Generate the URL from the storage key on a
  metadata row fetched by job 6's owner-scoped query — never from a path, key, or file ID the
  client supplied. A `/files/{id}` or "get download link" endpoint that signs whatever it's
  asked for is the usual way this bypasses ownership entirely.
- **Keep the bucket private.** No public-read objects and no permanent public URLs. Use
  unguessable storage keys, not `users/1234/photo-1.jpg`, so a mistake elsewhere doesn't
  leave the files enumerable.
- **Sign one object, read-only.** No wildcard or prefix policies (a signed cookie covering a
  user's whole folder, say), and no upload methods signed from a read endpoint.
- **Keep expiry short — minutes, not days.** Revoking a [job 10](10-oauth.md) grant doesn't
  reach URLs already handed out, so expiry is their only revocation. For a very large download
  that might outlast it, let the client re-fetch the item's metadata for a fresh URL rather
  than lengthening the expiry for everyone.
- **Mind pagination.** Signing a URL for every item on a large [job 8](08-pagination.md) page
  means many expire before a client gets to them. Either sign on the item endpoint instead, or
  set expiry to cover the time it takes to walk one page.
- **Make the CDN check signatures.** If a CDN caches the object without taking the signature
  into account, it can serve the file to requests that never had a valid one. Have the CDN
  validate signatures itself, or don't cache these objects.

Signed URLs also leak through logs; [job 11](11-logging-access.md) covers keeping them out.
Extend job 6's cross-user test to files: as user A, ask for a file belonging to user B and
assert no URL comes back, and assert that an expired URL is rejected.

## Blobs of unusual shape or size

Consider whether blobs in rare formats should be transformed on their way out.  A song
and voice recording service might store in a format suitable for indexing and streaming, but
song and voice note objects could be transformed into a more common format for exporting
and compatibility.

Very large content blobs like video exports can take long enough to download that a dropped
connection is routine, not exceptional. Make sure whatever URL you point the client at
supports HTTP resumable downloads — honoring `Range` request headers and returning
`206 Partial Content` with `Accept-Ranges: bytes` — so an interrupted transfer can pick up
where it left off instead of restarting a multi-gigabyte file from byte zero. Most object
storage and CDN URLs support range requests by default; if you're proxying the bytes through
your own server instead of a signed storage URL, confirm your server does too.

## Endpoint paths aren't where schema versions live

Job 3 was deliberate about keeping schema versioning in the schema's `$id`, not by changing
the whole API out from under the user by moving from `/api/v1/playlists` to `/api/v2/playlists`.
Keep the endpoint simply `playlists` for now and extend the schema in place whenever possible.

If you make a real mistake defining endpoints
or the situation changes massively in the future, it's always possible to add a new endpoint
and migrate away from the old one - or keep both and explain when to use each one.  For example
if your original playlist endpoint and data model merged listen counts and likes from too many
other tables to operate well at scale, your new playlist endpoint could be `playlists-quick`
and rate limit the full/slow one to align user motivations.


## Output of this step

A list of endpoints, one collection (and optionally item) endpoint per job 3 schema, each
scoped to the authenticated caller. This is what [job 6](06-access-control.md) enforces access
on, what [job 8](08-pagination.md) chooses a pagination approach for, and what
[job 15's discovery document](15-api-discovery.md) will eventually publish.
