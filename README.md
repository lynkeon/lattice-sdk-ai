<br/>

<div align="center">
  <a href="https://www.anduril.com/lattice/">
    <picture>
      <source media="(prefers-color-scheme: dark)" srcset="assets/anduril-logo-white.svg">
      <source media="(prefers-color-scheme: light)" srcset="assets/anduril-logo-primary.svg">
      <img alt="Anduril" src="assets/anduril-logo-primary.svg" height="40" align="center">
    </picture>
  </a>

<br/>

  # Lattice SDK AI

Official plugins for building [Lattice](https://www.anduril.com/lattice/) integrations with
[OpenAI Codex](https://developers.openai.com/codex). They cover REST and gRPC, and verify the
integration against a live Lattice Sandbox.

</div>

## Install

```bash
codex plugin marketplace add anduril/lattice-sdk-ai
codex
```

Then inside Codex run `/plugins`, open the `lattice-sdk-ai` marketplace, and install
`Lattice Plugin`.

## Plugins

| Plugin | Use for |
|--------|---------|
| [`lattice-plugin`](lattice-plugin/README.md) | Building Lattice integrations in any language: protocol choice (REST or gRPC), authentication, the Entities, Tasks, and Objects APIs, and live verification against a Lattice Sandbox. |

To confirm the install worked, ask the agent:

```text
Help me build a Lattice integration for my system.
```

## License

[Anduril Lattice SDK License Agreement](LICENSE.md).
