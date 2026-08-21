# Use API keys for defensibility and rate-limiting

[Job 6](06-access-control.md) already drew the line this job builds on: an API key answers
"who's calling" — which client application is making the request — not "whose data" is being
accessed. That second question is [job 10](10-oauth.md)'s job. Conflating the two is the most
common mistake here: an API key is not a substitute for authorizing access to a user's data,
and it shouldn't be treated as one.

## What an API key is actually for

Adding API keys early is an important future-proofing step.  You may not need to do more than
simply require a valid API key with requests on initial deploy.  If overuse of the API
_becomes_ a problem, rate limiting by API key can be added without breaking API customers.  
But if you don't require API keys from the start, they're hard to add later.

Once you get API keys they can be used for: 

- **Rate limiting** — cap how hard any one client can hit the API, independent of which user
  account it's acting on behalf of.
- **Defensibility** — know which client sent a request, so a misbehaving or compromised
  integration can be throttled or revoked without touching every other client.
- **Attribution** — see which third-party clients exist and how they're using the API, useful
  for both support and understanding real-world usage patterns.

An API key is not: proof of who the end user is, permission to act on a specific user's data,
or a long-lived secret that should ever appear in a URL, log line, or client-side code where
it's exposed to anyone but the client holding it.

## When you don't need API keys

There are two cases you might not even need to issue, manage and check API keys.

- An API key may not be necessary if OAuth and `client_id` are already used everywhere an API
  key would otherwise be checked — wherever a caller would need one, it already has the
  other. Since `client_id` is normally public, treat it as an identifier rather than a
  credential: the thing actually standing in for the API key's job of proving the caller is
  who it claims to be is the client secret (or PKCE's verifier for public clients) checked
  alongside it, not the `client_id` on its own.

- You only care about defensibility, not attribution/use — edge web layers/services
  can provide API defense.  Note that if you therefore do not require API keys, the problem
  edge services cannot solve is figuring out who is using your API and how much.

## Issuing and storing keys

Generate a random key with enough entropy to not be guessable (a UUID or a sufficiently long
random string is enough). [Job 2](02-libraries-frameworks.md) has the specific picks: on
Node, `generate-api-key` for issuing and `passport-headerapikey` for checking incoming keys;
on Django, `djangorestframework-api-key`, a DRF permission class with hashing and revocation
already built in. Store a hash of it, never the key itself — the same rule as a password. If
the key is ever compromised via a database leak, a hash gives an attacker nothing usable,
while a raw key stored in plaintext hands over exactly what they need.

Show the raw key to the developer exactly once, at creation time, and never again. If they
lose it, the only valid path is issuing a new one — there's no "forgot my API key" recovery
flow, because that would require the server to have a copy of the raw key to hand back, which
defeats the point of hashing it in the first place.

## Transport

Send the key in a request header (`Authorization: Bearer <key>` or a custom header like
`X-API-Key`), never as a query parameter. Query parameters end up in server access logs,
browser history, and referrer headers — all places a secret shouldn't linger. This is the same
reasoning [job 6](06-access-control.md) applied to user identity: don't let a credential ride
somewhere it can be casually captured.

## Rate limiting

Key the rate limit off the API key, not off the underlying user account — a client
integration serving many users should get one limit budget, not one per user it happens to
act on behalf of. Return `429 Too Many Requests` with a `Retry-After` header so a well-behaved
client knows when to try again instead of guessing.

Again, [job 2](02-libraries-frameworks.md) has the specific picks: `express-rate-limit` on
Node (with a shared Redis store once you're running more than one instance), or DRF's
built-in throttling on Django — its stock throttle classes key off the user or the request IP
by default, so this is one of the few places you'll need a small subclass to key off the API
key instead.

Tune limits to the actual job. A one-time full export used by an occasional sync tool can
tolerate a low request rate; a client doing frequent [job 8](08-pagination.md) cursor polling
for near-real-time sync needs a limit generous enough that "continuous" access (the DMA
language the [README](../README.md#meeting-regulatory-requirements) points to) doesn't become
impractical in practice. Don't set a limit so tight that the pagination mechanism you built in
job 8 can't actually be used the way it was designed to be used.

## Revocation and rotation

Revoking a key should take effect immediately — a live lookup or a short cache TTL, not a key
list baked into a long-lived token. Give developers a way to rotate a key themselves (issue a
new one, transition traffic, retire the old one) without needing to file a support request,
since a compromised key that requires human intervention to fix stays live longer than it
should.

## Output of this step

Every request to the API arrives with a client identified by a hashed, revocable API key sent
via header, rate-limited independently of the account whose data is involved.
[Job 6](06-access-control.md) uses this identity for "who's calling," kept separate from
[job 10](10-oauth.md)'s "whose data," and [job 11](11-logging-access.md) logs which client
made which request.
