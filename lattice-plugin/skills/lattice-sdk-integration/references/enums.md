# Lattice SDK enums — how to get exact values

**Purpose:** how to obtain and verify exact enum literals. The literal values themselves live in
[enum-literals.generated.md](enum-literals.generated.md) (a snapshot that may lag). The
**authoritative source is the installed `anduril` SDK** — introspect it.

## Ground truth: introspect the installed SDK

1. **Detect, then install — don't search the filesystem.** Check whether the SDK is installed with
   a one-shot import/resolve probe (Python: `python -c "import anduril, inspect, os;
   print(os.path.dirname(inspect.getfile(anduril)))"`, or `pip show anduril-lattice-sdk`;
   `node -p "require.resolve('@anduril-industries/lattice-sdk')"`; `go list -m
   github.com/anduril/lattice-sdk-go`). **If it isn't installed, install the latest** — installing
   is the correct first step, not a fallback. **Never `find`/`grep` the filesystem hunting for a
   `Lattice` class or SDK files.** Enum sets change between versions, so pin or install deliberately.
2. **Introspect the real literals** rather than trusting memory, docs, or the snapshot:

   ```bash
   # Python
   python -c "import anduril; print(anduril.Sensor.model_fields['sensor_type'].annotation)"
   python -c "import anduril; print(anduril.MilView.model_fields['nationality'].annotation)"
   ```

   Other languages expose the same values through their generated types (Go constants / struct
   tags, TypeScript union types, Java generated enums). Read them from the installed package.
3. The snapshot in [enum-literals.generated.md](enum-literals.generated.md) is a convenience for
   the common values; the docs are the fallback when you can't introspect — explore them from
   <https://developer.anduril.com/llms.txt> (the AI-agent site map). Precedence for literals/fields:
   **installed SDK → snapshot → docs.** (REST URL *paths* are different — they aren't in the SDK;
   fetch them from <https://developer.anduril.com/openapi/rest.json>.)

SDK "enums" are `typing.Union[typing.Literal[...], typing.Any]`, not Python `enum.Enum`. **Pass
the string literal directly** (e.g. `status="STATUS_EXECUTING"`); never iterate or
attribute-access them.

> ## ⚠️ The `Any`-typed enum trap — why you must validate against the live server
>
> Because every enum is typed `Union[Literal[...], Any]`, the `Any` arm means **the model
> accepts any string you pass** without complaint. A wrong value like `sensor_type="RADAR"`
> (instead of `"SENSOR_TYPE_RADAR"`) constructs cleanly, imports fine, and passes every local
> check — then the **server** rejects it on publish with `UNMARSHAL_ERROR: invalid value for enum
> field`. The entity silently never appears.
>
> This is the single most common reason an integration that "runs without crashing" actually
> publishes nothing. You cannot catch it by importing, compiling, or constructing objects — only
> by **publishing to the live sandbox and reading the entity back**. Treat a live round-trip as
> the definition of done, not an optional extra (see SKILL.md).
>
> The same trap hides **wrong field names**: models allow extra fields, so
> `ComponentHealth(message="x")` (the field is `messages=[...]`) is accepted locally and dropped
> silently. When in doubt, inspect `<Type>.model_fields` rather than trusting memory or docs.

## Where each enum lives

The [snapshot](enum-literals.generated.md) lists the literals for ontology template, disposition,
environment, nationality, alternate-ID type, health/connection status, classification level,
sensor type, and task/delivery status. For any value not there — or to confirm one that is —
introspect the installed SDK.

## Choosing values

The snapshot is literals only; here is how to pick among them:

- **`Ontology.template`** — `TEMPLATE_TRACK` for a detected object (vessel, aircraft, contact),
  `TEMPLATE_ASSET` for a taskable friendly-controlled asset, `TEMPLATE_SENSOR_POINT_OF_INTEREST`
  for a sensor-derived point, `TEMPLATE_GEO` for a shape/region, `TEMPLATE_SIGNAL_OF_INTEREST`
  for a signal detection. `TEMPLATE_INVALID` is a sentinel — never use it. Required components
  per template: see [entity-fields.md](entity-fields.md).
- **`MilView.disposition`** — pick `DISPOSITION_UNKNOWN` or `DISPOSITION_NEUTRAL` when the source
  doesn't tell you allegiance.
- **`MilView.environment`** — `ENVIRONMENT_SURFACE` is on the water, `ENVIRONMENT_SUB_SURFACE` is
  under the water.
- **`TaskStatus.status`** — see [task-lifecycle.md](task-lifecycle.md) for which transitions are valid.

For field-level guidance on alternate IDs, nationality, health mapping, and sensors, see
[entity-fields.md](entity-fields.md).
