# Authentication reference for Lattice Sandboxes

This plugin is **sandbox-first**. Keep deployment guidance targeted at Sandboxes and do not drift
into production tenant instructions here.

Two choices matter:

1. **Which credential** you authenticate with — a long-lived **environment token** by default, or
   OAuth2 **client credentials** as an override.
2. **Which protocol** you are using — **REST** or **gRPC**.

For Sandboxes, the sandbox authorization token is **additive**. You send it on top of whichever
primary credential you chose.

## Contents

- [Default credential choice](#default-credential-choice)
- [Sandbox environment variables](#sandbox-environment-variables)
- [Supplying credentials (and the agent-session gotcha)](#supplying-credentials-and-the-agent-session-gotcha)
- [REST sandbox auth](#rest-sandbox-auth)
- [gRPC sandbox auth](#grpc-sandbox-auth)
- [OAuth2 client credentials vs. environment token](#oauth2-client-credentials-vs-environment-token)
- [gRPC auth is manual](#grpc-auth-is-manual)
- [Gotchas](#gotchas)

## Default credential choice

Use **environment token** by default for this plugin. Reach for **OAuth client credentials** only
when the user explicitly asks for it or the existing integration already uses it.

## Sandbox environment variables

**Environment variables are the preferred source** (this matches the examples on
developer.anduril.com). A `.env` file or a `config.yml` are supported alternatives that the
integration loads into the environment. Never hardcode or log credentials.

| Variable | Meaning | Used by |
|---|---|---|
| `LATTICE_ENDPOINT` | Environment hostname, **no scheme** (for example `<environment_id>.env.sandboxes.developer.anduril.com`) | always |
| `ENVIRONMENT_TOKEN` | Long-lived static bearer token | default auth |
| `LATTICE_CLIENT_ID` | OAuth2 client ID | client-credentials auth |
| `LATTICE_CLIENT_SECRET` | OAuth2 client secret | client-credentials auth |
| `SANDBOXES_TOKEN` | Account-level sandbox authorization token | Sandboxes only |

`LATTICE_ENDPOINT` is a **hostname only**. Build the base URL as `https://<hostname>`. Do not put
`https://` in the environment variable.

## Supplying credentials (and the agent-session gotcha)

The integration you generate should read credentials from the environment (`os.environ` or the
language equivalent), optionally loading a `.env` file first (e.g. `python-dotenv`) or a
`config.yml`. File-based config is robust because it survives across process launches.

**Before writing integration code, check whether the credentials are actually available**
(`LATTICE_ENDPOINT`, `ENVIRONMENT_TOKEN`, `SANDBOXES_TOKEN`). If any are missing, **do not guess
or invent values.** Prompt the user to supply them one of two ways:

1. **Create a `.env` (or `config.yml`) the integration loads.** Offer to scaffold a
   `.env.example` with the variable names (never real values). This is the most reliable path.
2. **`export` the variables in the shell before launching the coding-agent session**, so the whole
   session inherits them.

> **Gotcha for coding-agent sessions:** an `export` that the agent runs inside a single Bash tool
> call affects **only that subprocess** and does not persist to later commands or to the running
> integration. So either load a `.env`/`config.yml` from within the program, or have the user
> `export` the variables *before* starting the session — don't rely on the agent exporting them
> mid-session.

## REST sandbox auth

For REST, the sandbox token must be included on every Sandbox request.

### Environment token + Sandbox

This is the default path.

```python
import os
from anduril import Lattice

client = Lattice(
    base_url=f"https://{os.environ['LATTICE_ENDPOINT']}",
    token=lambda: str(os.environ["ENVIRONMENT_TOKEN"]),
    headers={"anduril-sandbox-authorization": f"Bearer {os.environ['SANDBOXES_TOKEN']}"},
)
```

### OAuth client credentials + Sandbox

Use this only when the user asks for OAuth.

```python
import os
from anduril import Lattice

client = Lattice(
    base_url=f"https://{os.environ['LATTICE_ENDPOINT']}",
    client_id=os.environ["LATTICE_CLIENT_ID"],
    client_secret=os.environ["LATTICE_CLIENT_SECRET"],
    headers={"anduril-sandbox-authorization": f"Bearer {os.environ['SANDBOXES_TOKEN']}"},
    timeout=300,
)
```

Note the environment-token form passes `token=` as a **callable** (`lambda: str(...)`), not a bare
string. That lets the client re-read a rotated token.

## gRPC sandbox auth

For gRPC, auth is manual metadata injection. The metadata keys on the wire are lowercase:

- `authorization: Bearer <environment-token-or-access-token>`
- `anduril-sandbox-authorization: Bearer <SANDBOXES_TOKEN>`

Environment token is the simpler default. OAuth is supported, but you must fetch, cache, and
refresh the access token yourself.

## OAuth2 client credentials vs. environment token

| | Client credentials (OAuth2) | Environment token |
|---|---|---|
| What you supply | `client_id` + `client_secret` | one long-lived bearer token |
| Token lifetime | short-lived; SDK auto-refreshes on REST | long-lived; **you** rotate it before expiry |
| REST constructor | `client_id=`, `client_secret=` | `token=lambda: str(...)` |
| Sandbox recommendation in this plugin | supported override | **default** |

Under the hood, client-credentials exchanges the id and secret at `POST /api/v1/oauth/token`
for an `access_token`. On REST, the SDK can manage that exchange for you. On gRPC, you own it.

## gRPC auth is manual

The REST client manages tokens automatically. **gRPC does not**. For OAuth client credentials on
gRPC, fetch, cache, and refresh the token yourself, and attach both auth metadata entries to each
RPC.

## Gotchas

- **The `Bearer ` prefix is mandatory.** The sandbox header value must be
  `f"Bearer {SANDBOXES_TOKEN}"`, not the bare token.
- **Set the sandbox header on the REST client constructor, not per request.** Per-call headers do
  not cover the OAuth token fetch.
- **`LATTICE_ENDPOINT` is a hostname, not a URL.** Adding a scheme inside the env var produces a
  malformed URL.
- **Do not let a gRPC choice hide REST-only needs.** If the integration also needs the Objects API,
  add a REST companion or reconsider the protocol choice.
- **Validate by making a real call.** Client construction without a real request proves nothing.
