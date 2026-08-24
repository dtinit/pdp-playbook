# Handle references to OTHER users in personal data

The [job 1](01-identifying-portable-data.md) inventory already flagged where this shows up —
a shared playlist, a comment thread, a private message — and [job 5](05-choosing-endpoints.md)
and [job 6](06-access-control.md) both punted on what to actually do about it. This is where
that gets decided: not dumped in wholesale, and not silently filtered out either.

## Two different problems, two different fixes

These need different handling:

- **Another person's identifier** inside the requesting user's own
  record — a commenter's name on a note the user owns, a sender's identity on a message the
  user received.
- **Shared Content** — a collaborative playlist, a shared album

## Referencing another person without exporting their profile

Don't export the other person's full user record just because their content is attached to
something the requesting user owns. A small, stable reference — their public ID or handle plus
whatever display value the requesting user actually saw — is normally enough for the export to
stay meaningful.

Concretely, this is a [job 3](03-json-schema.md) schema decision: model a `commenter` field as
a minimal `UserRef { id, displayName }` structure, composed via `$ref` like anything else in
job 3, not as the full internal user schema. The line to hold: include what the requesting
user needs to make sense of their own data — what they actually saw through the product — not
whatever your service internally knows about the other party. Account status, contact info, or
anything not already visible to the requesting user through normal use has no reason to appear
here.

## Shared resources and joint ownership

Truly shared resources are probably pretty rare in personal data.  Regulations that 
offer users the right to delete _their_ personal data may reinforce this, because
now companies need a way to identify exactly what to include in these deletion requests.

Even when resources aren't truly shared-ownership, perhaps users ought to be allowed to 
export something they use or contribute to.  If the API doesn't allow that access, users
will lack functionality or hack it through GUI or other means.  Consider these use cases
where the user really ought to be able to export a resource that is technically owned
by the resource creator:

 * My favourite playlists are some that friends created and technically own.  
 * I'm invited to contribute photos of friends and family to event-centric albums.
 * Family calendar created by my spouse

Silently dropping a shared playlist from someone's export because it's "not 100% theirs"
produces an incomplete, confusing result. Default toward including anything the requesting user
actually holds a stake in, handling the other-person parts with the reference pattern above,
rather than leaving the whole thing out reflexively.

Private group chats are probably the most common shared-ownership resource.  The whole group 
chat should be exported, not just the user's own messages devoid of context, even if the 
"owner" of the group chat is not the person asking for a personal data export.

## How far to go?

Consider forum messages.  Compared to group chats, the membership is pretty loose.  
A thread may be started by one person but continued by many 
others.  The thread or topic presentation may center the original posting and present
the rest as comments.  Should a user be able to export... 

 * Only the forum messages that they posted, and only their own comments, out of context
 * All comments, but only on the forum messages they posted
 * Posts and comments on any thread they contributed to

As long as you're building an API around exports, being inclusive here
can be both useful and consistent with your service's terms and Web functionality.
Consider unifying personal data access to meet compliance requirements with useful
API access to what the user currently has Web view access to anyway.  Your users may
be grateful, especially those who need accessible forum reading software or
those who participate on multiple forums and have trouble tracking them all.  

## Frozen or living references

Some edge cases around refering to people are worth deciding on purpose rather than by accident. Consider a five-year-old forum topic or comment thread.

- If a participating user has deleted their account or blocked the requester since they
  originally contributed, an export of an
  old shared thread should generally still show that user's reference as it existed at the
  time — it's part of the requester's own historical record.  If your service doesn't
  freeze the reference, it resolves live instead: a deleted account shows up blank or
  tombstoned, which quietly tells the requester that account is gone. Re-exporting the same
  thread later would reveal exactly when. That may be an acceptable cost, or it may not be —
  but it's worth knowing that's the trade being made, not discovering it by accident.

- On the other hand, users can change their names or avatars, and their current name and
  thumbnail image may be the ones currently visible even looking at an old thread.

Just be mindful of these cases, and note which approach is taken in internal storage
and external views in the rest of your service, so that you can predict how this will 
work in the API. 

## Output of this step

A documented rule per job 1 data type where other-user references show up: what's included,
what reference shape represents the other party, and how the ownership/membership model
scopes it. [Job 3](03-json-schema.md) encodes the reference shape as a shared schema, and
[job 6](06-access-control.md) queries key off the ownership model decided here.
