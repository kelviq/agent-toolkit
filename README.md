# Kelviq Agent Toolkit

[Kelviq](https://kelviq.com) is a monetization platform: model pricing,
sell plans through checkout and subscriptions, gate features with
entitlements, and manage pricing as code. This repo ships the Kelviq MCP
server bundle plus the Kelviq skills — Claude Code guidance for the
monetization loop (model → sell → gate) and pricing-as-code.

## Install — Claude Code

Add the marketplace and install the bundled plugin (MCP server + skills):

```
/plugin marketplace add kelviq/agent-toolkit
/plugin install kelviq@kelviq
```

`kelviq/agent-toolkit` refers to the GitHub repo `owner/repo` once this repo
is published there.

## Install — other agents

For any of the 70+ agents supported by the open skills ecosystem, install
the skills only:

```
npx skills add kelviq/agent-toolkit
```

Add `--skill <name>` to install a single skill instead of the full set. This
path installs skills only — it does not configure an MCP server for you.
Each skill's own Setup section covers connecting the Kelviq MCP server on
your agent.

## The skills

| Skill | Job | Install (single skill) |
|---|---|---|
| `kelviq` | Hub: connect any agent, ground rules, concept model, routing | `npx skills add kelviq/agent-toolkit --skill kelviq` |
| `pricing-plans` | Design the catalog: products, features, plans, entitlements, publish | `npx skills add kelviq/agent-toolkit --skill pricing-plans` |
| `setup-payments` | Checkout, subscriptions, trials, customer portal, upgrades | `npx skills add kelviq/agent-toolkit --skill setup-payments` |
| `usage-based-billing` | Entitlement checks, usage reporting, license keys | `npx skills add kelviq/agent-toolkit --skill usage-based-billing` |
| `pricing-as-code` | The Kelviq CLI: pricing in git, pull → diff → promote | `npx skills add kelviq/agent-toolkit --skill pricing-as-code` |

On Claude Code, `/plugin install kelviq@kelviq` (above) installs all five at
once, bundled with the MCP server.

## Environment variables

The bundled MCP server (`@kelviq/mcp-server`, run via `npx -y
@kelviq/mcp-server`) reads these from your environment:

| Variable | Purpose |
|---|---|
| `KELVIQ_SERVER_API_KEY` | Required for server-scoped API tools (products, plans, customers, checkout, etc.) |
| `KELVIQ_CLIENT_API_KEY` | Required only for `promotion_get` |
| `KELVIQ_ENV` | `production` (default) or `sandbox`. Leave unset for production; set to `sandbox` for testing |

Get keys at
[app.kelviq.com/settings/api-keys](https://app.kelviq.com/settings/api-keys).

**Degraded no-key mode**: with no keys set, the docs tools (`docs_search`,
`docs_read`) still work — Claude can answer Kelviq questions from the docs
without any credentials. API tools return a clear `Configuration error`
until a key is set.

## Local development

Test this plugin locally before it's published:

```
claude --plugin-dir .
```

## Related repos

The Kelviq MCP server, CLI, and SDKs live in their own repos:
`@kelviq/mcp-server` and `@kelviq/cli` on npm, plus per-language SDKs.
