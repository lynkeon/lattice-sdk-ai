---
name: lattice-sdk-integration
description: >-
  Guides building sandbox-first Lattice SDK integrations without assuming a
  language or protocol up front. Use when the user wants to build, modify, or
  review a Lattice integration: publishing entities/tracks, connecting a sensor
  or telemetry source, building a taskable agent, watching or consuming
  entities, uploading media/objects, or bridging an external system into
  Lattice. Classifies the integration, recommends REST or gRPC when
  underspecified, defaults to environment-token auth, enforces live sandbox
  verification, and explains Entities/Tasks/Objects afterward.
metadata:
  version: 0.9.0
---

# Building a Lattice integration

Target **Lattice Sandboxes** for quick `0 → 0.9` integrations. Default auth is the **environment
token**. Don't guess exact values — get them from the right source, which depends on *what kind of
fact* you need.

**Fact precedence — split by fact type:**

- **Enum literals, field names, protobuf message/service names, auth headers** → **install and
  introspect the Lattice SDK** for your language (authoritative — install the latest) →
  [enum-literals.generated.md](references/enum-literals.generated.md) snapshot (convenience, may
  lag) → docs (fallback).
- **REST URL paths** (e.g. `/api/v1/entities`) → the **REST OpenAPI spec** at
  <https://developer.anduril.com/openapi/rest.json>. Paths are **not** in the installed SDK — the
  generated client abstracts them behind methods like `entities.publish_entity(...)`, so
  introspection will never reveal them. Fetch the spec instead.
- **gRPC surface** (RPCs, message shapes) → your installed/generated stubs, and the gRPC reference
  at <https://developer.anduril.com/reference/grpc> for browsing.
- **Anything else in the docs** → start from <https://developer.anduril.com/llms.txt>, the
  AI-agent site map (append `.md` to any docs page URL for clean markdown).

A live round-trip against the sandbox is the definition of done.

## Integration routing

Classify the integration, then read the relevant reference before writing code:

| Building…                                             | Path                              | Details |
| ----------------------------------------------------- | --------------------------------- | ------- |
| Publishing entities / tracks from a source            | Entity publisher                  | <references/entity-fields.md>, <references/enums.md> |
| A taskable agent that receives and executes tasks     | Task agent                        | <references/task-lifecycle.md> |
| Watching / consuming the entity stream                | Watcher / consumer                | <references/entity-fields.md> |
| Uploading or attaching media / files                  | Object / media (REST-only)        | <references/objects-api.md> |
| Bridging an external system into Lattice              | Bridge                            | <references/protocol-selection.md> |
| Unsure whether to use REST or gRPC                    | Protocol decision                 | <references/protocol-selection.md> |
| Authenticating to a Sandbox                           | Auth                              | <references/auth-and-deployment.md> |

**Read the relevant reference file before answering any integration question or writing code.**

## Critical rules

- **Do not guess** enum literals, field names, task-specification URLs, protobuf message/service
  names, or auth headers. Introspect the installed SDK — see [enums.md](references/enums.md). For
  REST URL paths, fetch the OpenAPI spec (see the fact-precedence split above); paths aren't in the
  SDK.
- **Detect the SDK with an import/resolve probe — never a filesystem search.** Check whether the SDK
  is installed with a one-shot import (Python: `python -c "import anduril, inspect, os;
  print(os.path.dirname(inspect.getfile(anduril)))"`, or `pip show anduril-lattice-sdk`;
  `node -p "require.resolve('@anduril-industries/lattice-sdk')"`; `go list -m
  github.com/anduril/lattice-sdk-go`). **If it isn't installed, install it** (`pip install
  anduril-lattice-sdk`, etc.) — installing is the correct first step, not a fallback. **Do not
  `find`/`grep` the filesystem hunting for a `Lattice` class or SDK files.**
- **A live round-trip is the definition of done.** Running a process cleanly or reading your own
  logs proves nothing — the `Any`-typed enum trap means wrong values pass locally and are
  rejected only by the server. Publish/execute against the sandbox and **read it back in a
  separate step**.
- **Default to environment-token auth; never hardcode or log credentials.** If credentials are
  missing, prompt the user — don't invent them (see
  [auth-and-deployment.md](references/auth-and-deployment.md)).
- **Sandbox-only.** This plugin stops at sandbox verification; don't drift into production-tenant
  guidance.
- **Don't assume Python or REST.** Prefer the repo's existing language and choose the protocol
  deliberately.

## Workflow

1. **Classify** the integration using the routing table above.
2. **Extract the four dimensions** from the user or repo: language, protocol, auth, deployment
   target. Recommend a protocol when underspecified
   ([protocol-selection.md](references/protocol-selection.md)); let the user confirm or override.
3. **Keep language guidance minimal** — prefer the repo's language; for REST prefer a documented
   SDK, for gRPC prefer generated stubs. Don't force a language switch.
4. **Choose auth and target** ([auth-and-deployment.md](references/auth-and-deployment.md)):
   environment token + sandbox token by default; OAuth client credentials as an override.
5. **Build from exact contracts.** Introspect the installed SDK for literals, field names, and
   message names (install it first if absent — use an import probe, not a filesystem search). For
   REST URL paths, fetch the [REST OpenAPI spec](https://developer.anduril.com/openapi/rest.json) —
   they aren't in the SDK. Don't guess.
6. **Map the source completely.** Populate every field the source genuinely gives signal for
   ([entity-fields.md](references/entity-fields.md)); don't fabricate values.
7. **Validate live** against the sandbox and read the data back separately.
8. **Explain the result** in Lattice terms ([lattice-terminology.md](references/lattice-terminology.md)),
   covering only the APIs the integration used.

## Protocol-specific handoff

- REST → follow the `lattice-rest-integration` skill.
- gRPC → follow the `lattice-grpc-integration` skill.

## Reference map

- [protocol-selection.md](references/protocol-selection.md) — REST vs. gRPC decision tree.
- [auth-and-deployment.md](references/auth-and-deployment.md) — sandbox auth: env-var-first defaults, overrides, gotchas.
- [entity-fields.md](references/entity-fields.md) — what makes an entity operationally complete.
- [enums.md](references/enums.md) — how to get exact literals (introspect the SDK) + the open-string-enum trap.
- [enum-literals.generated.md](references/enum-literals.generated.md) — enum literal snapshot (generated; may lag the SDK).
- [task-lifecycle.md](references/task-lifecycle.md) — taskable-agent lifecycle invariants + Python example.
- [objects-api.md](references/objects-api.md) — REST-only object storage and media attachment.
- [lattice-terminology.md](references/lattice-terminology.md) — concise post-verification explanation language.

## Key documentation (API tour)

When a request doesn't fit a single row above, consult the Lattice docs. Know which artifact is which:

- [llms.txt](https://developer.anduril.com/llms.txt) — **start here to explore the docs.** An
  AI-agent site map of every guide, reference, and sample. Append `.md` to any docs page URL for
  clean markdown.
- [REST OpenAPI spec](https://developer.anduril.com/openapi/rest.json) — the fetchable REST contract
  (`/api/v1/...` paths, request/response schemas). **The source for REST URL paths.**
- [gRPC reference](https://developer.anduril.com/reference/grpc) — the gRPC surface for browsing
  (RPC names, message shapes). Reference only — generate stubs from your installed SDK, not this.
- [API reference](https://developer.anduril.com/reference) — the human-rendered HTML reference site.
- [Guides](https://developer.anduril.com/guides) — getting started, entities, tasks, objects.
- [Samples](https://developer.anduril.com/samples/overview) — runnable sample apps.

> Note: `https://developer.anduril.com/openapi.json` is an HTML index page, **not** a spec — use
> `openapi/rest.json` above for the machine-readable REST contract. There is no fetchable gRPC
> spec; the gRPC surface is browsable HTML, so take gRPC contracts from your generated stubs.
