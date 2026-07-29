# Lattice SDK AI

A plugin marketplace for building [Lattice](https://www.anduril.com/lattice/) integrations with AI
coding agents. It distributes the Lattice plugin to [OpenAI Codex](https://developers.openai.com/codex).

The marketplace hosts one plugin today:

- [`lattice-plugin`](lattice-plugin/README.md): builds Lattice integrations across REST and gRPC,
  and verifies them against a live Lattice Sandbox.

## Install

### OpenAI Codex

From your terminal:

```bash
codex plugin marketplace add anduril/lattice-sdk-ai
codex
```

Then inside Codex run `/plugins`, open the `lattice-sdk-ai` marketplace, and install `Lattice Plugin`.

### Confirm it works

Ask the agent:

```text
Help me build a Lattice integration for my system.
```

## License

[Anduril Lattice SDK License Agreement](LICENSE.md).
