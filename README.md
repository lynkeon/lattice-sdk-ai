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

  # Lattice SDK Skills

Official skills for building [Lattice](https://www.anduril.com/lattice/) integrations. These teach
coding agents how to create integrations using the latest best practices.

</div>

## Installation

### Codex

Install the [plugin](lattice-plugin/README.md), which includes the skills and updates automatically:

```bash
codex plugin marketplace add anduril/lattice-sdk-ai
codex plugin add lattice-plugin@lattice-sdk-ai
```

### Manual installation

> This route does not update itself. Re-run `npx skills update -y` when you want newer versions of
> the skills.

Run this command in your project:

```bash
npx skills add anduril/lattice-sdk-ai/lattice-plugin
```

## License

[Anduril Lattice SDK License Agreement](LICENSE.md).
