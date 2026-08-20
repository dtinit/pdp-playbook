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
and rate limit the full/slow one to align user motiviations.


## Output of this step

A list of endpoints, one collection (and optionally item) endpoint per job 3 schema, each
scoped to the authenticated caller. This is what [job 6](06-access-control.md) enforces access
on, what [job 8](08-pagination.md) chooses a pagination approach for, and what
[job 15's discovery document](15-api-discovery.md) will eventually publish.
