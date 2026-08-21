# Log grants and data access requests

This is where the threads from [job 6](06-access-control.md), [job 9](09-api-keys.md), and
[job 10](10-oauth.md) land: each of those jobs resolves an identity — which client, which
user, which OAuth grant — and this job is what actually records that those resolutions
happened. There are two distinct logs here, not one, and they serve different purposes.

## Two logs, not one

- **Grant log** — an append-only record of consent lifecycle events: a user granted a
  specific client a specific set of [job 10](10-oauth.md) scopes, at what time; a grant was
  later revoked, by the user or an admin, at what time. This is the audit trail for "who did I
  say could see my data, and when did that change."
- **Access log** — a per-request record of actual data access: which [job 9](09-api-keys.md)
  client and which [job 6](06-access-control.md)-resolved user identity hit which endpoint,
  when. This is the audit trail for "who actually looked, not just who was allowed to."

Both matter, and they answer different questions during an incident. If a user asks "did app
X ever look at my listen history," the grant log tells you whether it was ever allowed to; the
access log tells you whether it actually did, and how often.

## What to log, and what not to

Log identifiers, not payloads: client ID, user ID, endpoint or scope, timestamp, response
status. Don't log full response bodies. A request log that includes the actual personal data
returned just creates a second copy of that data sitting in a system with different access
controls, different retention, and different assumptions than the primary store — exactly the
kind of uncontrolled copy [job 1](01-identifying-portable-data.md)'s inventory was meant to
prevent.

Never log a raw API key or OAuth token, for the same reason [job 9](09-api-keys.md) said not
to put one in a URL — a log line is one of the easiest places for a secret to leak into a
place with looser access control than the system it authenticates into. Log the key's
non-secret identifier (its ID, or a hash) instead, the same value used to look it up, not the
value that proves possession of it.

## The access log is itself personal data

A log of who accessed a given user's data is, itself, information about that user — and about
whichever other user's client made the request. Apply the same access-control thinking from
[job 6](06-access-control.md) to the log store itself: not everyone internally should be able
to query "show me every request user X's data was involved in," and access to the log store
should be its own audited, restricted thing, not an incidental side effect of having write
access to a logging system built for unrelated purposes.

## Retention

Keep grant-log entries for as long as the grant matters for dispute resolution or compliance
review.  This is a small, low-volume log, so there's little cost to keeping it well past when
an access log entry would be discarded. Keep access-log entries only as long as they're useful
for security investigation and abuse detection, then age them out. The access log grows with
every single request, and indefinite retention of "who looked at what, when" is itself a
privacy liability, not just a storage cost. Make the access-log retention window configurable
rather than hard-coded — how much access history is actually worth keeping varies a lot by
company and by how the log ends up getting used, and that's a call worth leaving to whoever
operates the service rather than presuming here.

## Reuse what already exists

This doesn't need a new logging system. Whatever structured logging or observability stack
already exists in the project is almost certainly enough: write
grant and access events as structured entries (one event per line, consistent field names) into
it, rather than standing up a separate audit database for this alone. The bar to clear is that
the fields above are captured and the access-control and retention points above are honored,
not that the mechanism is novel.

## Output of this step

Two structured, access-controlled logs: a low-volume grant log recording every consent and
revocation event from [job 10](10-oauth.md), and a request-scoped access log recording every
identified access from [job 6](06-access-control.md) and [job 9](09-api-keys.md), both
retained deliberately rather than by default. The grant log is also the data source
[job 13](13-gui-design.md) needs to show a user what's currently connected to their account.
