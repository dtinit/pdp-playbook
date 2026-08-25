# Use OAuth for authorization of data access, with limited scopes

[Job 6](06-access-control.md) already named this job's role: OAuth is what answers "whose
data," as distinct from [job 9](09-api-keys.md)'s "who's calling." A third party holding a
valid OAuth token can hit the exact same `/me/playlists` endpoint from
[job 5](05-choosing-endpoints.md) as the user themselves — the token is what turns "my own
data" into "data I've explicitly let this app see."

## Which flow

Use the **Authorization Code flow** as the default — it's the standard, widely-supported OAuth
flow for exactly this situation: delegated, user-present consent, where a human is actively
agreeing to let a specific third-party app see specific data. That rules out the Client
Credentials grant (no user in the loop at all) and the Resource Owner Password grant (the
third party would see the user's actual account password, which is exactly the failure mode
OAuth exists to avoid). For a third-party service with its own server — the common case, and
the one every OAuth library supports out of the box — the standard flow with a confidential
client secret is enough.

Add **PKCE** on top when the third party is a public client that can't safely hold a secret —
a desktop sync tool, a mobile app, a CLI. PKCE closes the authorization-code-interception gap
that a client secret would otherwise cover, and it's worth supporting for exactly the kind of
personal-data-portability tools this playbook expects (local sync utilities, migration tools)
even though it's not yet the universal default third-party developers will assume. Most
current OAuth libraries support PKCE alongside the standard flow, so enabling it is additive,
not a separate implementation.

## Local software — including LLM agents running on the user's own device

A locally-run AI agent that a user wants to give access to their own data is not a new case:
it's the same public-client problem as the desktop sync tool and CLI above, and the same
Authorization Code + PKCE flow answers it. The thing worth stating plainly, because it's the
whole reason to insist on this rather than take a shortcut: the user should never hand the
agent their actual account password, and this API has no path that would even accept one if
they tried — [job 6](06-access-control.md) resolves identity from a token, never a password,
so there is no backdoor for a "just log in as me" script to use instead. An agent that wants
access goes through the same consent screen as any other third party and gets the same scoped,
revocable token — nothing more, regardless of how much the user trusts it.

The practical mechanism for an agent that can't embed a browser or receive a normal web
redirect is the one already standardized for exactly this in
[RFC 8252](https://www.rfc-editor.org/rfc/rfc8252.html): the agent opens the user's actual
system browser to your real login/consent page — where the user authenticates directly with
your service, never through the agent — and briefly listens on a loopback address
(`http://127.0.0.1:<port>/callback`) to receive the redirected authorization code once the user
approves. This is the same pattern behind `gh auth login`, `gcloud auth login`, and similar
CLI tools, so it's a familiar shape to implement and to explain to users.

For an agent that can't open a browser or bind a local port at all — running headless, or on a
device with no display — the
[Device Authorization Grant (RFC 8628)](https://www.rfc-editor.org/rfc/rfc8628) is the
fallback: the agent displays a short code and a URL, the user opens that URL on any other
device (their phone, say) to approve access, and the agent polls until the grant completes.

If the OAuth flow for local software is not feasible for your service, a user-specific 
API key is a reasonable hack.  This violates the normal ideal where the API key does not 
actually grant any access, but is very narrow; a *personal API key* grants access 
(probably read-only) to exactly one account.  Then the user can configure that API key 
into their agent or software.

## Scopes, one per data type

Define scopes at the same granularity as the [job 3](03-json-schema.md) schemas and
[job 5](05-choosing-endpoints.md) endpoints — `read:playlists`, `read:listen-history` — not
one all-or-nothing `read:everything` scope. A user exporting their playlists to a
music-migration tool shouldn't have to also hand over their listen history just because the
API only offers one lever.

This isn't just good practice, it's what makes the consent screen honest: a user can only make
a real choice about what they're sharing if the scopes on offer actually correspond to
distinct, meaningful pieces of their data. A scope list that mirrors the job 1 inventory's data
types is usually the right shape.

## The consent screen

Show scopes in plain language, tied to what the user actually recognizes — "your playlists,"
not `read:playlists` — and show which third-party app is asking, using whatever the app
registered as its display name. Let the user approve a subset of the requested scopes where
that's feasible, rather than an all-or-nothing accept; if the app claims it needs a scope, it
should say what it's for, and the user should be able to say no to what it doesn't need.

## Issuing client_id and client_secret

If the user data is reasonably private, then the 3rd party requesting access needs to be
confirmed.  A company called "XYZ Photos" asks for access to a user's photo album.  Are
they a legitimate company?  If they are a legitimate company, what stops an attacker
from pretending to be XYZ Photos? `client_id` and `client_secret` are shared between
an OAuth authorization server and client to solve the latter problem.

The [Data Trust Registry](https://www.dt-reg.org) (DTR) provides a robust solution to the first
problem, by taking on the task of verifying a data portability ecosystem participant once for
the whole ecosystem.  But you still need to be sure you're issuing the `client_id` to the real
company listed in the DTR. Domain verification is the current widely
adopted solution for that.  Also, the DTR ecosystem is tracking and adopting technology
solutions to make this whole `client_id`/`client_secret` and API key sharing go easier.

## Token lifetimes and continuous access

Issue short-lived access tokens with a refresh token behind them. This is also the concrete
mechanism behind the [README's regulatory notes](../README.md#meeting-regulatory-requirements):
the DMA's "continuous" access doesn't require a long-lived credential sitting around
forever, or a streaming connection — a refresh token lets an authorized third party come back
next week, mint a new short-lived access token, and pick up exactly where
[job 8](08-pagination.md)'s cursor left off. Short access-token lifetimes limit the blast
radius if one leaks; the refresh token is the piece that actually needs careful storage and
prompt revocation.

If privacy and leaking tokens are still a major concern,
[DPoP (RFC 9449)](https://www.rfc-editor.org/rfc/rfc9449) is an extension to OAuth that secures
tokens to the real party.


## Revocation

A user has to be able to revoke a grant, and revocation has to actually stop access — not just
hide the grant from a UI while the still-valid access token keeps working until it expires
naturally. Revoke the refresh token immediately, and keep access-token lifetimes short enough
that a leaked or already-issued access token stops mattering quickly on its own.
[Job 11](11-logging-access.md) is where each grant and revocation gets logged; surfacing
revocation as something a user can actually do themselves is a GUI concern this playbook picks
up again in [job 13](13-gui-design.md).

## Libraries

[Job 2](02-libraries-frameworks.md) has the specific picks: `node-oidc-provider` (or the
lighter `@node-oauth/oauth2-server` for bare OAuth2 without an OIDC identity layer) on Node,
and `django-oauth-toolkit` on Django. All three implement the Authorization Code flow, PKCE,
and scope handling out of the box — this job is about how to configure them, not building an
authorization server from scratch.

One configuration detail worth checking specifically for the local-agent case above: RFC 8252
requires an authorization server to treat the port in a loopback redirect URI
(`http://127.0.0.1:<port>/...`) as variable, since a native agent picks an ephemeral port at
launch. A server that naively exact-matches the whole registered redirect URI, port included,
will reject every one of these logins — confirm the library allows loopback redirects to vary
by port before assuming this case is handled.

## Output of this step

An OAuth authorization server issuing short-lived, scoped access tokens via the Authorization
Code flow (with PKCE available for public clients), with per-data-type scopes matching
[job 3](03-json-schema.md)/[job 5](05-choosing-endpoints.md),
a consent screen a user can actually make a decision from, and working revocation.
[Job 11](11-logging-access.md) logs every grant this issues, and
[job 15's discovery document](15-api-discovery.md) publishes the scopes a client can request.
