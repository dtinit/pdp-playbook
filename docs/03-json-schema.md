# How to use JSON Schema to define data formats

Once you've [inventoried your portable data](01-identifying-portable-data.md) and broken it
into sensible data types, each data type needs a schema. JSON and JSON Schema are 
the right tools here: JSON is 
machine-readable (satisfying the letter of GDPR Article 20's "structured, commonly used,
machine-readable format" requirement), JSON Schema has mature validators in every language, and is
self-describing enough that a 3rd-party tool can consume your data without extra docs.

## One schema per data type, not one giant schema

Match your schemas to the groupings from the inventory step, not to your database tables. A
few small, focused schemas beat one sprawling one:

- **Content objects** the user created or owns — a playlist, a note, a saved route.
  If the content object is a blob, the JSON file can be alongside it with metadata that's
  not part of the file format, though collection listings would be preferable if that applies.
- **Activity/history records** — a listen event, a watch event, a chat message.
- **Metadata attached to an object** — like count, play count — modeled as properties on the
  object schema it belongs to, per the "keep metadata with objects" principle from the
  inventory step, not as a separate free-floating export.
- **Collections or listings** - when content or data objects are organized into collections of
  some kind, like photos in albums, some kind of index file is needed to describe the 
  contents of each collection.  If there's per-item metadata that doesn't fit in the blob
  file format, the collection information is an even better place to provide that metadata.

This also keeps pagination sane later — you'll paginate a history feed very differently from
a one-off content export (see [job 8](08-pagination.md)).

## Anatomy of a schema

Keep every schema to this shape:

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://schemas.pub/example.com/playlist/v1.json",
  "title": "Playlist",
  "type": "object",
  "properties": {
    "id":        { "type": "string" },
    "name":      { "type": "string" },
    "createdAt": { "type": "string", "format": "date-time" },
    "tracks": {
      "type": "array",
      "items": { "$ref": "https://schemas.pub/example.com/track-ref/v1.json" }
    }
  },
  "required": ["id", "name", "createdAt", "tracks"],
  "additionalProperties": false
}
```

Notes on each part:

- **`$schema`** — pin a real draft (2020-12 is a safe current default) so validators behave
  consistently. Don't leave it out.
- **`$id`** — a stable, dereferenceable URL, not just a label. Third-party tools will use this
  to identify and cache the schema.
- **`additionalProperties: false`** — be strict by default. It's easier to loosen a schema
  later than to discover a client silently depended on an undocumented field.
- **`$ref`** — compose schemas rather than duplicating structure. A track reference inside a
  playlist should point at the same schema used elsewhere, not redefine "track" inline.

## Versioning is a last resort — but be prepared if it comes to that

JSON is extensible by nature, so the default move when your data model changes is to add
optional properties to the *existing* schema in place, not cut a new version. Old clients
that ignore unknown fields keep working; new clients get the new field. Reach for a version
bump only for genuine breaking changes — renaming or removing a required property, changing
a type, anything an old client can't safely ignore.

Even so, set up the versioning scheme now, while it's cheap, so you're ready if a breaking
change does become necessary. Put the version in the `$id` path (`/schemas/playlist/v1.json`)
rather than a separate field, so old links and cached copies keep resolving unchanged once
there's a `v2` to resolve alongside them. This is the contract that
[job 15's schema evolution strategy](15-schema-evolution.md) builds on — decide the scheme
now, because retrofitting it later breaks anyone who's already integrated.

## Where schemas live

Serve schemas at the stable URLs used in their `$id`s, and list them somewhere discoverable —
this feeds directly into [job 14, API discovery](14-api-discovery.md), where clients need to
find both your endpoints and the schemas those endpoints return.

## Output of this step

One JSON Schema file per data type from your inventory, checked into version control and
served at a stable URL that's versioned in name only until you actually need a `v2`. This is
the contract [job 4](04-hooking-storage-into-api.md) serializes your storage layer into, and
what [job 14's discovery document](14-api-discovery.md) points clients at.
