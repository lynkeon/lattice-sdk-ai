# Objects API reference (REST surface, Python SDK example)

The Objects API is Lattice's **blob store** — a resilient content-delivery network (CDN) that
distributes arbitrary binary objects (≤ 1 GiB each) across the Lattice mesh. It's path-addressed
CRUD: you `upload_object` to a path, `get_object` it back, `list_objects`, `delete_object`, and
`get_object_metadata`. The common use is attaching media to entities — a track thumbnail or a vessel
manifest — by uploading the blob and then pointing an entity's `media` component at its path.

The Objects API is a **REST** surface. All snippets below use the Python REST SDK as a compact
example, but the object model and validation rules apply regardless of language.

## Contents

- [Methods](#methods)
- [Upload](#upload)
- [Download](#download)
- [Metadata, list, delete](#metadata-list-delete)
- [Object schema](#object-schema)
- [Attaching an object to an entity (the `media` component)](#attaching-an-object-to-an-entity-the-media-component)
- [Lifecycle & gotchas](#lifecycle--gotchas)

## Methods

| Method | Purpose | Returns |
|---|---|---|
| `client.objects.upload_object(object_path=, request=<file>)` | Upload a blob to a path | `PathMetadata` (`.content_identifier.path`, …) |
| `client.objects.get_object(object_path=)` | Download | an **iterator of byte chunks** |
| `client.objects.get_object_metadata(object_path=)` | HEAD-style metadata | a **dict of header values** |
| `client.objects.list_objects(...)` | List objects | list response with `path_metadatas` |
| `client.objects.delete_object(object_path=)` | Delete before expiry | — |

## Upload

`request=` takes a file-like object opened in **binary** mode. The response's
`content_identifier.path` is the stored path.

```python
file_name = os.path.basename(file_path)
object_path = f"{file_name}"                 # choose a unique path for the object
with open(file_path, "rb") as file:
    response = client.objects.upload_object(
        object_path=object_path,
        request=file,
    )
# The full retrievable path, e.g. for an entity media reference:
print(f"/api/v1/objects/{response.content_identifier.path}")
```

### Upload with a TTL

Objects default to a **90-day** TTL. To override it, send the `time-to-live` header **in
nanoseconds** via per-call `request_options` (this is one of the few places a per-call header is
correct — it's not an auth header):

```python
time_to_live = 1 * 60 * 60 * 1_000_000_000   # 1 hour, in nanoseconds
client.objects.upload_object(
    object_path=object_path,
    request=file,
    request_options={"additional_headers": {"time-to-live": f"{time_to_live}"}},
)
```

## Download

`get_object` streams the blob back as an **iterator of byte chunks** — join them to reconstruct the
object. It does not return the bytes directly:

```python
response = client.objects.get_object(object_path=object_path)
result = b"".join(chunk for chunk in response)
```

## Metadata, list, delete

`get_object_metadata` returns a **plain dict** keyed by response-header names (not a typed model) —
access with `.get(...)`:

```python
meta = client.objects.get_object_metadata(object_path=object_path)
meta.get("Path")
meta.get("Content-Length")   # bytes
meta.get("Checksum")         # SHA-256
meta.get("Last-Modified")
meta.get("Expires")
```

Delete removes an object before its TTL expires:

```python
client.objects.delete_object(object_path=object_path)
```

## Object schema

The store models two types (from the Objects OpenAPI spec):

**`PathMetadata`**
- `content_identifier` (`ContentIdentifier`, required) — identifies the object path
- `size_bytes` (int, required)
- `last_updated_at` (datetime, required)
- `expiry_time` (datetime, optional) — when the object auto-deletes

**`ContentIdentifier`**
- `path` (string, required) — the unique object path in your environment
- `checksum` (string) — SHA-256 of the object, for integrity verification

## Attaching an object to an entity (the `media` component)

Uploading a blob doesn't put it on the map. To surface it on an entity, set the entity's `media`
component to reference the object's path. `media` is an **overridable** field, so you attach it with
`override_entity(field_path="media.media", ...)` rather than a full re-publish.

The types come from the top-level `anduril` package:

```python
from anduril import Lattice, Media, MediaItem, Entity, Provenance
from datetime import datetime, timezone

provenance = Provenance(
    integration_name="your_integration_name",
    data_type="your_data_type",
    source_update_time=datetime.now(timezone.utc),
)
media = Media(
    media=[
        MediaItem(
            relative_path=object_path,      # e.g. "/api/v1/objects/<OBJECT_NAME>"
            type="MEDIA_TYPE_IMAGE",
        )
    ]
)
client.entities.override_entity(
    entity_id=entity_id,
    field_path="media.media",
    entity=Entity(entity_id=entity_id, media=media),
    provenance=provenance,
)
```

- `MediaItem.relative_path` is the object path (relative to the environment base URL).
- `MediaItem.type` is a string-literal enum (same `Union[Literal[...], Any]` trap as everywhere —
  see [enums.md](enums.md)). A common value for object-backed media is `MEDIA_TYPE_IMAGE`.
- **Append vs. replace:** the override *replaces* `media.media` with what you pass. To add to
  existing media without clobbering it, `get_entity` first, append your new `MediaItem` to
  `entity.media.media`, and override with the combined list.

## Lifecycle & gotchas

- **You own the entity's media reference across the object's lifecycle.** When an object **expires**
  (TTL) or you **delete** it, Lattice does *not* clean up the entity's `media` component for you —
  your app must reset it, or the entity will point at a dead object path.
- **TTL is nanoseconds**, and it's a header, not a kwarg. `time-to-live: "3600"` would be 3.6
  microseconds, not an hour — always multiply out to nanoseconds.
- **`get_object` is an iterator**, `get_object_metadata` is a **dict**, `upload_object` returns a
  typed `PathMetadata`. Three different return shapes — don't assume a uniform model.
- **Validate with a round-trip**, as always in this skill: upload, then read the object (or its
  metadata) back from the server, and if you attached it to an entity, `get_entity` and confirm the
  `media.media` path is what you set. A clean-looking upload call proves nothing until the server
  confirms it.
