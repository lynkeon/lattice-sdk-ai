---
name: lattice-grpc-integration
description: >
  Build gRPC and protobuf-first Lattice integrations against Sandboxes. Use when the user
  explicitly wants gRPC, already has .proto contracts, needs streaming or higher-rate telemetry,
  or is building generated stubs in their own language.
metadata:
  version: 0.9.0
---

# Building a gRPC Lattice integration

Use this skill when the protocol choice is **gRPC**. Keep the guidance language-agnostic: prefer
the user's existing language and tooling, generate the stubs you need, and only go deep on
language specifics when the build actually requires it.

## When to choose gRPC

Prefer gRPC when the user has:
- an existing protobuf or `.proto` contract,
- streaming or higher-rate telemetry requirements,
- a generated-stub workflow already in place,
- or a hardware or bridge integration that is naturally proto-first.

If the user also needs the **Objects API**, call out explicitly that Objects is REST-only and
either add a small REST companion or reconsider the protocol choice.

## Workflow

1. **Start from the exact protobuf contracts.**
   - Use your installed/generated stubs as the wire contract (install or generate them first —
     detect with an import/resolve probe, not a filesystem search). For browsing the public Lattice
     services, use the [gRPC reference](https://developer.anduril.com/openapi/grpc.json) or the
     [rendered reference](https://developer.anduril.com/reference).
   - Treat the package name and fully qualified message names as exact — read them from your
     generated stubs, don't guess.

2. **Generate stubs in the user's language.**
   - Prefer the existing repo language and build tooling.
   - Use the documented Buf or language-native generation path.
   - Do not add long language-specific setup advice unless it is required by the chosen toolchain.

3. **Authenticate to a Sandbox.**
   Use [../lattice-sdk-integration/references/auth-and-deployment.md](../lattice-sdk-integration/references/auth-and-deployment.md).
   - Read credentials from the environment (preferred), or a `.env` / `config.yml` the
     integration loads. Default to environment token auth.
   - Support OAuth client credentials when requested.
   - If credentials are missing, prompt the user — don't invent them.
   - Remember that gRPC auth is manual metadata injection, not an automatic REST client concern.

4. **Implement the integration against the exact RPCs.**
   Use [../lattice-sdk-integration/references/entity-fields.md](../lattice-sdk-integration/references/entity-fields.md).
   - Use the proto definitions and generated stubs as the wire contract.
   - Keep source ingestion and Lattice publishing or consumption as separate concerns.
   - For taskable agents, preserve the task lifecycle invariants even if the method names differ by
     language.

5. **Validate live against the Sandbox.**
   - Run the real integration.
   - Read the resulting entities or tasks back with a separate client path.
   - For a taskable agent, dispatch a real task and prove it reaches a terminal state.

## Minimal language guidance

- Prefer the user's existing language and build system.
- Prefer officially documented gRPC generation flows when available.
- If the user is undecided, recommend a language that fits their existing repo or runtime instead
  of turning the plugin into a language chooser.

## gRPC-specific guidance

### Auth

- Attach lowercase metadata keys on the wire.
- Include both the primary auth credential and the sandbox auth token.
- If using OAuth client credentials, fetch, cache, and refresh the access token yourself.

### Exactness

- Do not guess service names, package names, or message names.
- Do not guess the `type.googleapis.com/<fully.qualified.Message>` task specification URL.
- Do not guess enum literals — introspect your installed/generated stubs. The
  [enum snapshot](../lattice-sdk-integration/references/enum-literals.generated.md) is a
  convenience that may lag; see
  [enums.md](../lattice-sdk-integration/references/enums.md). For browsing the surface, use the
  [gRPC reference](https://developer.anduril.com/openapi/grpc.json).

### Streaming and task agents

Use [../lattice-sdk-integration/references/task-lifecycle.md](../lattice-sdk-integration/references/task-lifecycle.md).
- Reconnect long-lived streams on disconnect.
- Keep task execution off the main stream consumer.
- Increase `status_version` strictly on every update.
- Handle cancellation promptly and move tasks to terminal states.

### Bridging

For external gRPC sources:
- Connect to the exact configured source address.
- Confirm the source is really delivering data.
- Then validate that the translated entities or task updates actually land in Lattice.

## Hard rules

- Do not let "gRPC" collapse into one language's codegen story.
- Do not rely on local compilation as proof that the contract is right.
- Do not skip separate live validation.
- Do not omit the post-verification explanation of the Lattice APIs the integration touched.
