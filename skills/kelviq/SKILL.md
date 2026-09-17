---
name: kelviq
description: Works with the Kelviq monetization platform (products, plans, checkout, subscriptions, entitlements, usage-based billing, pricing-as-code) via the kelviq MCP tools. Use when the user mentions Kelviq, or asks to set up billing, pricing, checkout, subscriptions, or feature gating, and no more specific Kelviq skill (pricing-plans, setup-payments, usage-based-billing, pricing-as-code) matches.
---

# Kelviq

Kelviq is a monetization platform: model a pricing catalog, sell it through
checkout and subscriptions, gate features with entitlements, and — for teams
that want it — manage the whole catalog as code. The loop is **model → sell →
gate**, plus a fourth axis, **pricing-as-code**, for teams that version their
catalog in git instead of clicking through the dashboard.

This hub carries the ground rules every Kelviq skill shares. It also routes
to four task skills that go deep on one slice of the loop. Read this skill
first; jump to a task skill once you know which slice you're in.

## Setup

The Kelviq MCP server (`@kelviq/mcp-server`) provides the `kelviq:` tools.
On Claude Code with this plugin installed, it's already configured. On any
other agent, or if `kelviq:` tools are unavailable, add it directly:

```json
{
  "mcpServers": {
    "kelviq": {
      "command": "npx",
      "args": ["-y", "@kelviq/mcp-server@latest"],
      "env": {
        "KELVIQ_SERVER_API_KEY": "${KELVIQ_SERVER_API_KEY}",
        "KELVIQ_CLIENT_API_KEY": "${KELVIQ_CLIENT_API_KEY}",
        "KELVIQ_ENV": "sandbox"
      }
    }
  }
}
```

Get keys at `https://app.kelviq.com/settings/api-keys`. `KELVIQ_SERVER_API_KEY`
covers nearly every tool; `KELVIQ_CLIENT_API_KEY` is only needed for
`promotion_get`. The server uses **sandbox** unless `KELVIQ_ENV=production` —
keep it that way while building and testing, and only switch (with production
keys) when the user explicitly wants live data. Sandbox and production are
fully separate: different data, different keys, a key only works against its
own environment. The `docs_search` / `docs_read` tools work with no keys at
all — use them to answer questions before any key is configured.

## Ground rules

- **Tools available.** Prefer the `kelviq:` MCP tools when they exist. When
  only a terminal is available, prefer the Kelviq CLI (`npx -y @kelviq/cli`)
  over hand-written HTTP calls. Fall back to the REST API only for something
  neither covers.
- **Docs first.** For any "how do I" or "what does X mean" question, call
  `kelviq:docs_search` then `kelviq:docs_read` before answering from memory.
  Docs stay current; your training data doesn't.
- **Read errors, don't retry blind.** API tools return
  `{ error: { status, message } }` with `isError: true` on failure. Read the
  message and adapt. `Configuration error` means a required key env var is
  missing. "Not found" on a valid-looking identifier often means the key is
  scoped to the wrong environment (production vs. sandbox) — check
  `KELVIQ_ENV` before assuming the identifier is wrong.
- **Check the connection before guessing.** When auth or the environment
  looks wrong, call `kelviq:connection_check` first: it reports which
  environment the server targets and whether the server key works, without
  printing the key.
- **Confirm before anything hard to undo.** Restate intent and get explicit
  user confirmation before: archiving a product/feature/plan,
  `kelviq:subscription_cancel`, `kelviq:plan_file_delete`, `kelviq:discount_archive`,
  `kelviq:webhook_endpoint_delete`, or a
  `kelviq:plan_publish` call that sets `updateFeatures` or `updatePricing`
  (these migrate existing customers).
- **Never invent a tool.** Every `kelviq:<tool>` reference must exist in the
  live tool list. Prices and media have no write tools in MCP by design —
  see the concept model below.
- **Keep secrets out of the conversation.** Webhook signing secrets are hidden
  in tool results; don't pass `revealSecret: true` unless the user needs the
  value. Apps read secrets from their env file — write them there with a
  command that doesn't print them.

## Concept model

**Identifier vs. UUID.** Products, features, and plans each have a stable
`identifier` slug and a UUID. Tools accept either wherever they reference
another object: `plan_create` takes the parent product as identifier or
UUID, `plan_entitlements_add` takes features the same way, and
`discount_create.appliesTo.product` too. Prefer the identifier — it's what
the user knows and what `kelviq.config.ts` uses.

**Feature types.** `BOOLEAN` is an on/off flag, `CUSTOMIZABLE` is a setting
the plan configures (e.g. 30 days of history retention), `METER` is
usage-tracked consumption (e.g. API calls). Seats customers buy are a `METER`:
only a meter can carry a per-unit price and take a quantity at checkout.
The entitlement `details` shape you pass to `plan_entitlements_add` differs
by type — check `kelviq:docs_read` on `product-catalog/entitlements` for the
exact fields rather than guessing.

**Plan lifecycle.** `plan_create` always lands in DRAFT; nothing is visible
or purchasable until `plan_publish`. `plan_update` on an already-published
plan creates a new draft — publish again to release the change.

**Environments.** `KELVIQ_ENV=sandbox` points every `kelviq:` tool at the
sandbox hosts instead of production. Data, keys, and identifiers do not
cross between the two.

**Prices are read-only here.** `plan_prices_list` reads a plan's prices;
there is no price-write tool in MCP. Price changes happen in the Kelviq
dashboard or through the CLI — see `pricing-as-code`.

## Routing

| If the task is about... | Go to |
|---|---|
| Products, features, plans, entitlements, publishing | `pricing-plans` |
| Checkout, subscriptions, trials, the customer portal, upgrades, discount codes, webhooks | `setup-payments` |
| Runtime feature gating, usage reporting, license keys | `usage-based-billing` |
| Versioning the catalog in git, CI, promoting sandbox → production | `pricing-as-code` |
| A guided, step-by-step build from nothing | prompt `setup_product` |
| A guided plan launch on an existing product | prompt `launch_plan` |
| Debugging why a specific customer's access looks wrong | prompt `diagnose_customer`, or the mini-flow below |

## Diagnose a customer, in one paragraph

Call `kelviq:entitlements_get` with the customer's ID to see what they
currently have access to (per feature: type, access, usage, limit). Call
`kelviq:subscription_list` filtered by that customer to see what plan(s)
they're actually subscribed to. Call `kelviq:plan_entitlements_list` for
that plan to see what it's *supposed* to grant. Compare the three: a
mismatch usually means a stale entitlement, a missed migration on
`plan_publish`, or a bad usage report. Propose a fix (`usage_report` to
correct a counter, `subscription_update` to move them to the right plan) and
confirm with the user before acting. This mirrors the `diagnose_customer`
prompt — use the prompt directly for the full guided version.
