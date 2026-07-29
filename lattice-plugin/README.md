# Lattice Plugin

Helps an AI coding agent build Lattice integrations against a Lattice Sandbox, in
[OpenAI Codex](https://developers.openai.com/codex).

To install, see the [marketplace README](../README.md).

## What it does

The plugin treats a Lattice integration as four decisions:

- Language: any language.
- Protocol: REST or gRPC.
- Auth: `ENVIRONMENT_TOKEN`, or OAuth client credentials.
- Deployment target: Sandbox preferred.

When a request is underspecified, the plugin narrows these choices, recommends a path, and lets the
user confirm. It then builds the integration, verifies it against the Sandbox, and explains the
result in terms of the Entities, Tasks, and Objects APIs.

Codex reads `.codex-plugin/plugin.json`, which points at the shared `skills/` tree.

## Skills

| Skill | What it does |
|---|---|
| `lattice-sdk-integration` | Entry point for Lattice integration work. Narrows language, protocol, auth, and target; recommends a path; verifies live; explains the result in Lattice terms. |
| `lattice-rest-integration` | Builds REST or HTTP+JSON integrations. Covers Entities, Tasks, and the REST-only Objects API. |
| `lattice-grpc-integration` | Builds gRPC or protobuf-first integrations. Covers generated stubs, metadata auth, and streaming and task-agent patterns. |
