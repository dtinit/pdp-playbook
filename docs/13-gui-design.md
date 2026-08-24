# Design GUI elements

Several earlier jobs already left work for this one: [job 10](10-oauth.md) specified a consent
screen and revocation, but not who builds it; [job 11](11-logging-access.md) built a grant log
specifically so a screen here could read from it. [Job 9](09-api-keys.md) assumed a developer could rotate a key "without needing
to file a support request," which requires some UI to exist for that to be true.

## Two audiences, two surfaces

- **End users** — a consent screen, a connected-apps/manage-access page, and (per the note
  below) self-service export.
- **Third-party developers** — a place to register a client, get its first
  [API key](09-api-keys.md) or [OAuth](10-oauth.md) client credentials, and rotate or revoke
  them later.

## Build "download my data" as a client of your own API, not a second pipeline

Now is the time to capitalize on some good preparation.  Many of the decisions that
go into a good personal data export that your users can load into other software
or even use on their own have already been made.  Some of the code can even be 
re-used.

"Download my data" buttons are required in some jurisdictions, and everything
that button needs already exists: the schemas
([job 3](03-json-schema.md)), the mapper that produces them from real storage
([job 4](04-hooking-storage-into-api.md)), and the endpoints that serve them
([job 5](05-choosing-endpoints.md)). The self-service export should be a client of that same
machinery, not a fresh implementation of it.

Two legitimate ways to do this:

- Genuinely call your own public endpoints, authenticated as the logged-in user via a
  first-party OAuth token or an equivalent internal credential. This is the stronger option —
  it dogfoods the real API on every export, so if the export ever breaks, so does something a
  third-party integrator would also hit.
- Invoke the same underlying mapper/service functions in-process, skipping the network hop,
  if that's more efficient for a first-party path. Either way, there's exactly one code path
  that decides what a user's exported data looks like, not two that can quietly drift apart.

A couple of things follow from taking this seriously:

- **Keep one file per data type, not one omnibus file.** Match the same per-type split
  [job 3](03-json-schema.md) already made — a user's playlists and listen history shouldn't
  land in one concatenated document. Whether that means several files inside one archive or
  several separate downloads is a choice made below, not something to decide here.
- **Reuse the real pagination, don't special-case it.** Walk each collection with whatever
  [job 8](08-pagination.md) mechanism that endpoint actually uses, the same way any other
  client would, instead of adding a "give me everything at once" mode that only the export
  screen uses and only the export screen tests.  Each page of results should be a separate
  JSON file although the page size might default to being rather large, such that only exports
  larger than 1MB actually break up into pages.  Exporting pages of < 1MB rather than a single
  5MB personal data archive may be much better performance.  Figure out your own Goldilocks
  size.  
- **Keep JSON formatting.** Don't underestimate users, or the software that may be helping
  them! Machine-readable data is valuable. If an HTML version of the export is also desired,
  it can be achieved with different templates applied to the same data from the same object
  models, or produced by transforming the JSON data as a last step.

## Consider responding synchronously

Some services collect user data, save and compress it, then send the user a link.  While this
is common practice, it's definitely not "real-time". It requires job management and
monitoring, secure and private blob storage, ways to contact the user, and policies and jobs
for how to keep around or expire the exports. It's not quite as friendly to users and may not
actually help you scale and tune this feature.

Consider sending the data directly in response to the request instead — never touching storage
at all. Two ways to keep the one-file-per-data-type structure above while still doing that:

- **Stream an archive on the fly.** A streaming zip library — one that writes each entry to
  the response as it's produced, rather than buffering the whole archive first — can build the
  same one-file-per-data-type structure a pre-generated archive would, without ever
  materializing it in blob storage or on disk. The user still gets one downloadable file; the
  server just never holds all of it in memory or storage at once.
- **Skip the archive; make separate requests.** Since [job 8](08-pagination.md)'s pagination
  already makes each collection endpoint efficient to walk on its own, the export UI can just
  call each [job 5](05-choosing-endpoints.md) endpoint directly — one live request per data
  type — and hand the user several separate downloads instead of bundling anything server-side.
  Simpler than streaming a zip, at the cost of the user getting multiple files instead of one.

Either way, browser support for compressed, chunked responses is good:

```
Content-Encoding: gzip
Transfer-Encoding: chunked
```

[Job 8](08-pagination.md)'s pagination is also what makes doing this live safe in the first
place: each page is a small, already-tested query, not one enormous query trying to
materialize an entire history at once.

## Consent, connected apps, and revocation

The consent screen's content was already specified in [job 10](10-oauth.md): plain-language
scope descriptions, the requesting app's identity, partial-scope approval where feasible. This
is where it actually gets built.  If privacy is a worry, consider adding filters that
the user can choose, such as "only data from the last month/year" or "filter out document
scans from personal photos."

The connected-apps / manage-access page reads directly from [job 11](11-logging-access.md)'s
grant log — which client, which scopes, since when — and its revoke action has to trigger the
same real revocation job 10 already specified: the refresh token invalidated immediately, not
just the row disappearing from this list while the token quietly keeps working.

An access-log view — showing a user actual request-level activity from job 11's access log,
not just grants — is optional. It's a legitimate transparency feature for a product that wants
it, but not every product needs to expose that level of detail, and job 11 already made access
log retention itself configurable per operator judgment; whether to surface it in a GUI is the
same kind of call.

## A minimal developer console

Job 9's promise that a developer can rotate a key themselves, and job 10's consent screen
showing "whatever the app registered as its display name," both assume a registration flow
exists somewhere. This doesn't need to be elaborate: a form to register an app (name, and a
redirect URI for job 10's flow), a page that shows its first API key or client credentials
exactly once at creation, and a way to rotate or revoke them later. This is the piece that
makes jobs 9 and 10's self-service claims actually true rather than aspirational.

## Output of this step

Consent, connected-apps/revocation, and self-service export screens for end users, and a
minimal registration/rotation console for developers. The export screen in particular is built
as a client of the same [job 3](03-json-schema.md)/[job 5](05-choosing-endpoints.md)/
[job 8](08-pagination.md) machinery the rest of this playbook produced, not a second
implementation of it — the same "don't build a parallel system" instinct
[job 2](02-libraries-frameworks.md) applied to libraries, applied here to your own API.
