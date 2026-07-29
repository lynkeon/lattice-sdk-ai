---
name: lattice-rest-integration
description: >
  Build REST and HTTP+JSON Lattice integrations against Sandboxes. Use when the user explicitly
  wants REST, needs the Objects API, wants the fastest sandbox path, or is building around HTTP
  requests and official REST SDKs.
metadata:
  version: 0.9.0
---

# Building a REST Lattice integration

Use this skill when the protocol choice is **REST**. The guidance here is intentionally
language-light: keep the user's chosen language when possible, use the documented REST SDK for
that language when available, and otherwise talk directly to the HTTP APIs.

## When to choose REST

Prefer REST when the user wants:
- the fastest sandbox integration path,
- HTTP and JSON over generated protobuf stubs,
- manual testing with `curl`, Postman, or simple scripts,
- lower-rate polling or request/response workflows,
- or the **Objects API**.

If the integration must upload, download, or attach files, call out that **Objects is a REST-only
surface** and either choose REST outright or propose a mixed REST + gRPC architecture.

## Workflow

1. **Pick the language, then stay out of the way.**
   - Use the repo's existing language or the language the user asked for.
   - If the language has a documented REST SDK, prefer it.
   - If not, use raw HTTP against the OpenAPI surface instead of forcing a language change.

2. **Authenticate to a Sandbox.**
   Use [../lattice-sdk-integration/references/auth-and-deployment.md](../lattice-sdk-integration/references/auth-and-deployment.md).
   - Read credentials from environment variables (preferred), or from a `.env` / `config.yml`
     the integration loads.
   - Default auth method is the **environment token** (`ENVIRONMENT_TOKEN`, a long-lived bearer
     token); support OAuth client credentials when requested.
   - Always include the sandbox authorization token on Sandbox requests.
   - If credentials are missing, prompt the user — don't invent them.

3. **Build against the exact REST surfaces.**
   Get exact values from the right source: **field names and enum literals** from the installed SDK
   (introspect it; install it first if absent — use an import probe, not a filesystem search), and
   **REST URL paths** from the [REST OpenAPI spec](https://developer.anduril.com/openapi/rest.json).
   Paths are not in the SDK — the generated client hides them behind methods — so fetch the spec.
   - **Entities**: publish, get, poll or stream, and overrides live under the Entities API.
   - **Tasks**: create, get, update status, cancel, query, and stream through the Tasks API.
   - **Objects**: upload, get, list, and metadata live under the Objects API.

4. **Map the source data completely.**
   Use [../lattice-sdk-integration/references/entity-fields.md](../lattice-sdk-integration/references/entity-fields.md).
   - Inventory every source field before publishing.
   - Populate every field the source genuinely gives signal for.
   - Do not fabricate values to satisfy a checklist.

5. **Treat exact values as exact.**
   Use [../lattice-sdk-integration/references/enums.md](../lattice-sdk-integration/references/enums.md).
   - Do not guess enum literals, field names, or task specification URLs.
   - Introspect the installed SDK for the exact literals and field names (install the latest first;
     detect it with an import probe, not a filesystem search); the
     [enum snapshot](../lattice-sdk-integration/references/enum-literals.generated.md) is a
     convenience that may lag. For REST URL paths, fetch the
     [REST OpenAPI spec](https://developer.anduril.com/openapi/rest.json) — they aren't in the SDK.

6. **Validate live against the Sandbox.**
   - Publish or execute the real integration.
   - Read entities, tasks, or objects back in a separate validation step.
   - Do not stop at syntax checks or local logs.

## Language guidance

Keep this minimal and practical:
- If the repo already has a language and build system, stay there.
- If the user asks for a specific language, use it unless the combination is genuinely unsupported.
- If the user is undecided, prefer an officially documented REST SDK language before inventing a
  raw-HTTP implementation from scratch.

Do not produce a long language comparison unless the user explicitly asks for one.

## REST-specific guidance

### Entities

- Publish via the Entities REST surface and validate with a separate read-back.
- Treat operational completeness and server acceptance as equally important.
- Use a stable `entity_id` and a heartbeat or re-publish strategy for live entities.

### Tasks

- Use exact task types and exact task URLs.
- For taskable agents, advertise the right task specification URL and verify that a dispatched task
  reaches a terminal state.
- Keep the task lifecycle rules from
  [../lattice-sdk-integration/references/task-lifecycle.md](../lattice-sdk-integration/references/task-lifecycle.md),
  even if the implementation language differs from the example.

### Objects

Use [../lattice-sdk-integration/references/objects-api.md](../lattice-sdk-integration/references/objects-api.md).
- Objects are path-addressed blobs.
- Upload alone is not enough; verify by reading the object or metadata back.
- If attaching media to an entity, verify both the object round-trip and the entity reference.

## Hard rules

- Do not let "REST" collapse into "Python" automatically.
- Do not treat a 200-level local wrapper response as the definition of done.
- Do not use production deployment guidance in this plugin.
- Do not skip the post-verification explanation of the Lattice APIs the integration used.
