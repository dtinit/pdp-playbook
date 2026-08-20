# Implement access control keyed to the user

[Job 5](05-choosing-endpoints.md) scoped every endpoint to the caller — `/me/playlists`, not
`/playlists?user_id=...`. This job is what actually enforces that scoping: resolving exactly
one account for every request, and making every storage read prove it's fetching that
account's data rather than just returning whatever it's asked for.

This is structurally private, not just logically private!  An attacker trying to gain
access to a private account simply cannot construct a URL to name that account's data
and hope there's a gap in its protection.  

## Resolve identity once, trust it everywhere

Resolve the caller's identity in one place — middleware, or a framework-level dependency —
from the validated credential itself, once per request. Never take a user ID from a query
param, request body field, or client-supplied header as authoritative; if it's not derived
from a verified token or session, it's not identity, it's just a claim.

Keep two things distinct, since they answer different questions:

- **Who's calling** — the client/app making the request, identified by an API key
  ([job 9](09-api-keys.md)). This is about rate limiting and defensibility, not ownership.
- **Whose data** — the account whose consent an OAuth token represents
  ([job 10](10-oauth.md)). This is the identity every query in this job scopes to. A
  third-party app can be "who's calling" while a user remains "whose data," and access
  control cares about the latter.

One pattern I like for resolving identity once is to put all the user authentication,
session and token management in **decorators** (when using python).  For example, all
the `playlist` and `listen-history` endpoints in a music service can be decorated
with @auth_user, and it's easy to see they're all protected the same way. Each decorator
below wraps a view function and injects `current_user` as an argument — the query
example in the next section is what the body of a function like `get_playlists` does
with it.

```python
@auth_user
def get_playlists(request, current_user):
  # code that uses the current_user.id in queries for playlists

@auth_user
def get_listen_history(request, current_user):
  # code that uses the current_user.id in queries for listen activity
```

## Filter at the query, not after the fetch

Resist the natural-looking pattern of fetching a resource by ID and then checking ownership:

```
playlist = Playlist.objects.get(id=playlist_id)
if playlist.owner_id != current_user.id:
    raise PermissionDenied
```

Scope the query itself instead:

```
playlist = Playlist.objects.get(id=playlist_id, owner=current_user)
```

A mismatched ID now simply doesn't exist from the query's perspective — return `404`, not
`403`, for someone else's resource ID. That's not just tidier code, it avoids confirming to a
prober that a given ID is valid and just happens to belong to someone else.

This belongs in the mapper/data-access layer from [job 4](04-hooking-storage-into-api.md), as
one scoped query helper per resource type reused everywhere that resource is fetched — not
reimplemented ad hoc in every handler, where it's one missed copy-paste away from a real bug.

## Ownership isn't always one row, one owner

Shared or joint content — a shared playlist, a comment thread — doesn't fit a single
`owner_id` column. [Job 7](07-references-to-other-users.md) covers how to model those
references; access control has to key off whatever that model defines as "belongs to this
user" for a given type, not assume a naive foreign key everywhere.

## Fail closed

If identity can't be resolved — missing, expired, or invalid credential — the request is
denied outright, not defaulted to open. This applies to collection endpoints too; "it's just
a list" is not a reason to skip the identity check, since the list itself is the personal data
being protected.

## Test the failure case on purpose

Every access-controlled endpoint should have a test that authenticates as user A, requests a
resource ID known to belong to user B, and asserts it's denied. This single class of test (an
IDOR check, in OWASP terms) catches the most common access-control bug there is: an endpoint
that quietly forgot its owner filter.

## Output of this step

Every [job 5](05-choosing-endpoints.md) endpoint provably scoped to a server-resolved
identity, with cross-user access tests covering each one.
[Job 9](09-api-keys.md) and [job 10](10-oauth.md) supply the credentials identity is resolved
from here, and [job 11](11-logging-access.md) logs what that resolved identity was allowed to
see.
