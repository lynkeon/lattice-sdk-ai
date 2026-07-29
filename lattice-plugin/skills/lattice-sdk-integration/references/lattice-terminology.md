# Explaining a finished integration in Lattice terms

After the integration works, explain it in concise first-principles language.

## How to explain it

- Start with what the integration does in the user's domain.
- Then map that behavior onto the Lattice API surfaces it uses.
- Only explain the APIs that the integration actually touched.
- Keep the explanation beginner-friendly and concrete.

## Preferred framing

### Entities

Use this shape:

> The **Entities API** is how your integration puts things into Lattice's shared operational view.
> If your code publishes or updates an entity, it is telling Lattice "this thing exists, here is
> where it is, and here is what we know about it right now."

### Tasks

Use this shape:

> The **Tasks API** is how work gets assigned and tracked. If your integration acts like an agent,
> it advertises what work it can do, receives tasks, updates status as it works, and finishes in a
> terminal state so operators know whether the work succeeded.

### Objects

Use this shape:

> The **Objects API** is how binary files move through Lattice. If your integration uploads an
> image, manifest, or other blob, Lattice stores it at a path, and entities or other workflows can
> refer to that stored object.

## Example structure

1. State what the integration does in plain language.
2. Name the Lattice APIs involved.
3. Describe the request/response or publish/consume loop briefly.
4. End with the verification result in plain language.

Example:

> This integration reads telemetry from your source system, turns each update into a Lattice
> entity, and publishes it to the common operational picture. In Lattice terms, it is mainly an
> **Entities API** integration. Because it also uploads thumbnails and attaches them to those
> entities, it also uses the **Objects API**. We verified it by publishing to the Sandbox and
> reading the entity and object reference back from Lattice in a separate step.
