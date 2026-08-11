# Lattice entity field reference (publish_entity)

Fields passed to `client.entities.publish_entity(...)`. Components are the top-level `anduril`
types (`Aliases`, `Location`, `Position`, `MilView`, `Ontology`, `Provenance`, `Health`,
`TaskCatalog`, ...). Enum literals: see the snapshot in
[enum-literals.generated.md](enum-literals.generated.md) and introspect the installed SDK to
confirm ([enums.md](enums.md)). Authoritative model:
<https://developer.anduril.com/guides/entities/overview>.

## Contents

- [Required for a live entity](#required-for-a-live-entity) — the server floor
- [Required by template](#required-by-template) · [Provenance](#provenance-provenance)
- [Production-quality fields](#production-quality-fields) — completeness checklist + **full worked example**
  - [Symbology](#symbology--symbology) · [Data classification](#data-classification--data_classification)
    · [Alternate IDs](#alternate-ids--aliasesalternate_ids) · [Nationality](#nationality--mil_viewnationality)
- [TEMPLATE_TRACK additional fields](#template_track-additional-fields) — track quality, dimensions, relationships
- [TEMPLATE_ASSET additional fields](#template_asset-additional-fields) — health, sensors
- [Heartbeat pattern](#heartbeat-pattern)

## Required for a live entity

| Field | Type | Notes |
|---|---|---|
| `entity_id` | str | Stable, unique, deterministic. Reuse the same id across republishes — it's the upsert key. |
| `is_live` | bool | `True` creates/updates a live entity. |
| `expiry_time` | RFC3339 str | **Future** time, < 30 days out. Use `datetime.now(timezone.utc) + timedelta(...)` then `.isoformat()`. Required unless you use a no-expiry option. |
| `provenance` | `Provenance` | `integration_name` is required; set `data_type` and `source_update_time` too (see below). |

The server rejects a publish missing these with `400 VALIDATION_ERROR` (e.g. `provenance.dataType
must be set`, `aliases.name must be set`, `expiryTime must be set`), so the entity never appears.

## Required by template

For `TEMPLATE_TRACK` / `TEMPLATE_ASSET` (the common cases), also set:

| Field | Type | Notes |
|---|---|---|
| `ontology` | `Ontology` | `template=` + free-text `platform_type=`. |
| `location` | `Location` | `Location(position=Position(latitude_degrees=, longitude_degrees=, altitude_hae_meters=))`. Altitude optional. |
| `mil_view` | `MilView` | `disposition=` + `environment=`. |
| `aliases` | `Aliases` | `name=` must be non-empty (the COP display name). |

`TEMPLATE_GEO` instead needs `geo_details` + `geo_shape` (and `location` if the shape is an ellipse).

## Provenance (`Provenance`)

```python
Provenance(
    integration_name="<your-integration>",   # required
    data_type="<source kind>",               # set it — validation expects it
    source_update_time=now.isoformat(),      # bump on every publish; drives update events
)
```

`source_update_time` should change each publish — Lattice keys update events off it.

## Production-quality fields

Beyond the server-enforced minimum, well-formed entities carry additional components that
operators and downstream systems depend on for display, filtering, and symbology. These are
not optional extras — an entity that omits them is incomplete for operational use. Include
them when your source provides the underlying information; use a sensible default when it
doesn't.

**Completeness checklist** — before you consider an entity done, walk this list and ask "do I
have signal for this, and have I set it?":

- [ ] `symbology.mil_std2525c.sidc` — always (derive from disposition + environment)
- [ ] `data_classification` — always (`CLASSIFICATION_LEVELS_UNCLASSIFIED` if nothing else)
- [ ] `aliases.alternate_ids` — whenever the source has any cross-reference id (MMSI, serial, ICAO…)
- [ ] `mil_view.nationality` — whenever the source indicates a flag/country (omit if truly unknown)
- [ ] **TEMPLATE_TRACK** → `tracked.track_quality_wrapper` (always), `tracked.radar_cross_section`
      + `dimensions.length_m` (if the source has size), `relationships` trackedBy (if you know the sensor)
- [ ] **TEMPLATE_ASSET** → `health.health_status` + `health.connection_status` (always),
      `sensors` (always — declare at least the asset's primary sensing capability)

The "always" items have sensible defaults and should never be skipped; the conditional ones
follow the rule *fill what your source gives you signal for*.

### Full worked example — a complete track entity

This is the target shape (not an aspirational extra). The per-field detail follows below.

```python
from datetime import datetime, timedelta, timezone
from anduril import (
    Aliases, AlternateId, Location, Position, MilView, Ontology, Provenance,
    Symbology, MilStd2525C, Classification, ClassificationInformation,
    Tracked, Dimensions,
)

now = datetime.now(timezone.utc)
client.entities.publish_entity(
    # --- lifecycle (server floor) ---
    entity_id="<stable-unique-id>",                 # deterministic; reused across republishes
    is_live=True,
    expiry_time=(now + timedelta(seconds=30)).isoformat(),   # future; heartbeat re-publishes
    provenance=Provenance(
        data_type="<your-source-kind>", integration_name="<your-integration-name>",
        source_update_time=now.isoformat(),
    ),
    # --- identity & cross-reference ---
    aliases=Aliases(
        name="<display name>",                      # the COP label; non-empty
        alternate_ids=[AlternateId(id="<source id>", type="ALT_ID_TYPE_MMSI_ID")],  # if you have one
    ),
    # --- where it is ---
    location=Location(position=Position(
        latitude_degrees=..., longitude_degrees=..., altitude_hae_meters=...,
    )),
    # --- what it is ---
    ontology=Ontology(template="TEMPLATE_TRACK", platform_type="<free-text platform>"),
    mil_view=MilView(
        disposition="DISPOSITION_UNKNOWN", environment="ENVIRONMENT_SURFACE",
        nationality="NATIONALITY_...",              # set if the source tells you the flag/country
    ),
    # --- how it should render (operators rely on this) ---
    symbology=Symbology(mil_std2525c=MilStd2525C(sidc="<15-char SIDC>")),
    # --- handling ---
    data_classification=Classification(
        default=ClassificationInformation(level="CLASSIFICATION_LEVELS_UNCLASSIFIED")),
    # --- track-specific quality & physical detail ---
    tracked=Tracked(track_quality_wrapper=3, radar_cross_section=...),   # quality required for a well-formed track
    dimensions=Dimensions(length_m=...),            # if the source gives size
)
```

For a `TEMPLATE_ASSET` (incl. a taskable agent), swap the track-specific components for `health`
and `sensors` — see [TEMPLATE_ASSET additional fields](#template_asset-additional-fields).

### Symbology — `symbology`

The MIL-STD-2525C SIDC string controls how operator displays render the entity (symbol shape,
color, fill). Every track and asset should carry one. Derive it from the entity's
disposition/environment rather than hardcoding a single value.

```python
from anduril import Symbology, MilStd2525C   # note the capital C

symbology=Symbology(mil_std2525c=MilStd2525C(sidc="SFGPUC---------"))
```

A SIDC is a 15-character string. The characters that matter most:

| Position | Meaning | Common values |
|---|---|---|
| 2 | Affiliation | `F`=Friendly, `H`=Hostile, `N`=Neutral, `U`=Unknown |
| 3 | Battle dimension | `A`=Air, `S`=Sea surface, `G`=Ground, `U`=Subsurface |
| 4 | Status | `P`=Present (use for live tracks) |
| 5–10 | Function code | `UC----` = track/contact; `------` = unspecified |
| 11–15 | Modifiers | Fill with `-` when unused |

Position 1 is always `S` (standard identity scheme). Map affiliation from `mil_view.disposition`
(friendly→`F`, hostile→`H`, neutral→`N`, unknown/pending→`U`) and battle dimension from
`mil_view.environment` (surface→`S`, air→`A`, land→`G`, subsurface→`U`).

### Data classification — `data_classification`

Marks the entity's handling level. Even unclassified data should be labeled explicitly — omitting
this field signals an incomplete entity to downstream systems.

```python
from anduril import Classification, ClassificationInformation

data_classification=Classification(
    default=ClassificationInformation(level="CLASSIFICATION_LEVELS_UNCLASSIFIED")
)
```

Use `CLASSIFICATION_LEVELS_UNCLASSIFIED` as the default for non-sensitive data; see
[enum-literals.generated.md](enum-literals.generated.md) for other levels.

### Alternate IDs — `aliases.alternate_ids`

Cross-reference IDs let other systems correlate your entity with their own records. Include them
when your source provides a well-known identifier (MMSI for AIS vessels, ICAO for aircraft, etc.).

```python
from anduril import Aliases, AlternateId

aliases=Aliases(
    name="<display name>",
    alternate_ids=[AlternateId(id="<source-id>", type="ALT_ID_TYPE_MMSI_ID")],
)
```

The literal names are not the obvious ones — watch the naming gotchas: it's
`ALT_ID_TYPE_MMSI_ID` (AIS — **not** `ALT_ID_TYPE_MMSI`) and `ALT_ID_TYPE_CALLSIGN` (**not**
`ALT_ID_TYPE_CALL_SIGN`); there is **no** `ALT_ID_TYPE_ICAO`; and avoid `ALT_ID_TYPE_ASSET_ID`
(it conflicts with the entity's own id). The full valid set is in
[enum-literals.generated.md](enum-literals.generated.md). Because the field is typed `Any`, a
wrong literal passes locally and fails on publish — confirm the exact value with
`python -c "import anduril; print(anduril.AlternateId.model_fields['type'].annotation)"`.

### Nationality — `mil_view.nationality`

Set when the source indicates a flag state or country of origin. Omit when unknown rather than
guessing.

```python
mil_view=MilView(
    disposition="DISPOSITION_UNKNOWN",
    environment="ENVIRONMENT_SURFACE",
    nationality="NATIONALITY_UNITED_STATES_OF_AMERICA",   # full literal, NOT an ISO code like "USA"
)
```

`nationality` is a long enum literal of the form `NATIONALITY_<COUNTRY>` (e.g.
`NATIONALITY_UNITED_KINGDOM`, `NATIONALITY_FRANCE`) — **not** an ISO 3166 code like `"USA"`. A
bare ISO code is accepted locally and rejected on publish. Confirm the exact literal with
`python -c "import anduril; print(anduril.MilView.model_fields['nationality'].annotation)"`.

## TEMPLATE_TRACK additional fields

### Track quality — `tracked`

Indicates how well-established the track is. Required for any well-formed track entity; use a
conservative default (`TRACK_QUALITY_2`) when your source provides no quality signal.

```python
from anduril import Tracked

tracked=Tracked(
    track_quality_wrapper=3,   # plain int, 0 (tentative) … higher = firmer
)
```

`track_quality_wrapper` is a plain integer on this SDK — pass `0` for a tentative track up to a
higher value for a firm one; `2`–`3` is a reasonable default. (It is not a wrapper object or a
`TRACK_QUALITY_*` string literal — those don't exist in the SDK.)

Also on `Tracked`:
- `radar_cross_section=<float>` — RCS in m², include when your source provides it.

### Physical dimensions — `dimensions`

```python
from anduril import Dimensions

dimensions=Dimensions(length_m=<float>)   # length in meters
```

Include when your source (AIS, radar, etc.) provides size information.

### Relationships — `relationships`

Links this track to the sensor that detected it. Include when you know the detecting sensor's
entity ID.

```python
from anduril import Relationships
from anduril.types.relationship import Relationship
from anduril.types.relationship_type import RelationshipType
from anduril.types.tracked_by import TrackedBy

relationships=Relationships(relationships=[
    Relationship(
        related_entity_id="<sensor-entity-id>",
        relationship_type=RelationshipType(tracked_by=TrackedBy()),
    ),
])
```

The shape is nested and easy to get wrong: `Relationships` holds a `relationships=` list of
`Relationship` objects; each `Relationship` has a `related_entity_id` plus a `relationship_type`
that is a **`RelationshipType` object** (not a string literal) whose `tracked_by`/`group_parent`/
etc. field selects the kind. There is no `tracked_by=` field on `Relationships` and no
`EntityRelationship` type. This is a suggestion-level enrichment — skip it unless you actually
know the detecting sensor's entity id.

## TEMPLATE_ASSET additional fields

### Health — `health`

Tells operators whether the asset is ready to operate. Always set for assets; map your source's
health state onto Lattice values from [enum-literals.generated.md](enum-literals.generated.md)
(introspect the installed SDK to confirm). Mapping an upstream source's health is a judgment
call — a reasonable default:

| Source state (typical) | Lattice `health_status` |
|---|---|
| Healthy / OK / nominal | `HEALTH_STATUS_HEALTHY` |
| Degraded / warning | `HEALTH_STATUS_WARN` (or `HEALTH_STATUS_NOT_READY`) |
| Faulted / failed | `HEALTH_STATUS_FAIL` (or `HEALTH_STATUS_OFFLINE`) |
| Unknown / unspecified | `HEALTH_STATUS_NOT_READY` (or `HEALTH_STATUS_INVALID`) |

```python
from anduril import Health, ComponentHealth, ComponentMessage

health=Health(
    health_status="HEALTH_STATUS_HEALTHY",
    connection_status="CONNECTION_STATUS_ONLINE",  # reflects this integration's connectivity
    components=[
        ComponentHealth(
            id="main",
            health="HEALTH_STATUS_HEALTHY",
            messages=[ComponentMessage(status="HEALTH_STATUS_HEALTHY", message="Nominal")],
        )
    ]
)
```

Note: `ComponentHealth` carries `messages=` — a **list** of `ComponentMessage` objects, not a
`message=` string. A bare `message="..."` kwarg is silently accepted by the Pydantic model
(extra fields allowed) but never reaches the server correctly.

Set `connection_status` to `CONNECTION_STATUS_ONLINE` while the integration is running,
`CONNECTION_STATUS_OFFLINE` on graceful shutdown.

### Sensors — `sensors`

Declares sensing capabilities.

```python
from anduril import Sensors, Sensor

sensors=Sensors(sensors=[Sensor(sensor_id="<id>", sensor_type="SENSOR_TYPE_RADAR")])
```

`sensor_type` is an enum literal, **not** a free-text label (values in
[enum-literals.generated.md](enum-literals.generated.md); introspect the SDK to confirm). Passing
a bare `"RADAR"` is silently accepted locally (the field is typed `Union[Literal[...], Any]`) but
**rejected by the server** with `UNMARSHAL_ERROR: invalid value for enum field`. See the
enum-trap warning in [enums.md](enums.md).

## Optional but common components

- `task_catalog=TaskCatalog(task_definitions=[TaskDefinition(task_specification_url="type.googleapis.com/anduril.tasks.v2.Investigate")])` — required to be taskable. Use a real built-in type (`anduril.tasks.v2.*`) or your own (`<your-org>.tasks.v1.<YourTaskName>`), never a placeholder.
- `description=...` — free-text description.
- `created_time=...` (RFC3339) — when the entity was first observed.

## Heartbeat pattern

Because `expiry_time` is short, publish on a loop so the entity stays live:

```python
while True:
    now = datetime.now(timezone.utc)
    client.entities.publish_entity(
        entity_id=eid, is_live=True,
        expiry_time=(now + timedelta(seconds=EXPIRY_OFFSET)).isoformat(),
        provenance=Provenance(integration_name=..., data_type=..., source_update_time=now.isoformat()),
        # ...other fields unchanged...
    )
    time.sleep(UPDATE_RATE_SECONDS)
```
