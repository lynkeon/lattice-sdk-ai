# Choosing REST vs. gRPC

Use this decision tree when the user has not explicitly chosen a protocol.

## Start with the user's shape

Ask what the integration actually needs to do:
- Publish or query entities.
- Run or receive tasks.
- Upload or download files.
- Bridge an external source into Lattice.
- Stream high-rate telemetry.

## Prefer REST when

- The user wants the fastest path from nothing to a working Sandbox integration.
- The user prefers HTTP and JSON over protobuf generation.
- The integration is request/response, lower-rate polling, or simple automation.
- The user wants to test with `curl`, Postman, or straightforward scripts.
- The integration needs the **Objects API**.

## Prefer gRPC when

- The user already has `.proto` contracts or a protobuf-first codebase.
- The integration is stream-oriented or bandwidth-sensitive.
- The user is bridging hardware or telemetry feeds that are already gRPC.
- Generated stubs are a natural fit for the codebase.

## Mixed integrations

Call out a mixed design when it is the clearest answer:
- Use gRPC for high-rate or streaming telemetry.
- Use REST for object upload or download or simple control-plane operations.

Do not propose a mixed design just because it is possible. Use it only when the shape of the
integration genuinely needs both surfaces.

## Language guidance

Keep language guidance secondary to the protocol choice:
- Prefer the user's current language.
- Prefer the repo's current language.
- Prefer an officially documented SDK or generation path over a bespoke workaround.

Do not turn an underspecified request into a long language survey unless the user explicitly asks.
