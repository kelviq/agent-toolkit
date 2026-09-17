---
name: pricing-as-code
description: Manage the Kelviq pricing catalog as version-controlled TypeScript with the Kelviq CLI — pull the live catalog, edit in git, preview a diff, and promote sandbox to production. Use when the user wants to version-control pricing, review pricing changes before they apply, script/automate catalog updates, or promote a tested catalog from sandbox to production.
---

# Pricing as code

The Kelviq CLI (`@kelviq/cli`) manages the catalog as a typed
`kelviq.config.ts` file synced against a live environment: pull it in, edit
it in a branch, preview the diff, apply it, and promote a tested sandbox
catalog to production.

## Setup

Two separate tools, both needed here. The **CLI** edits and syncs the config
file directly against the API — it does not go through MCP:

```bash
npm install -g @kelviq/cli
kelviq login          # paste a sandbox key (default)
kelviq login --prod    # paste a production key, for promote/push --prod
```

`kelviq env` shows what's configured; `kelviq env --check` also tests the
keys against the live API and prints no key material. The CLI loads
`.env.local` then `.env` from the working directory on its own, and takes
`--env-file <path>` for anything else — no `set -a; . ./.env` dance. The
**`kelviq:` MCP tools** are used here only to verify a pushed config against
sandbox (step 5 below) and to read docs (`docs_search`/`docs_read`, keyless).
See the `kelviq` skill's Setup section for the `.mcp.json` snippet if those
tools aren't already available.

## When to reach for the CLI over the dashboard or MCP

Bulk or repeated pricing changes, review-before-apply workflows, version
history in git, and promoting a catalog you've already tested in sandbox to
production. For a one-off plan or a single price, the dashboard (or MCP for
everything except prices) is simpler.

The config file covers products, features and plans. Discount codes and
webhook endpoints are not part of it — create those with `kelviq discount
create` / `kelviq webhook add` (below) or the `kelviq:` discount and webhook
tools (see the `setup-payments` skill), and don't try to express them in
`kelviq.config.ts`.

## The loop

Starting from nothing? `kelviq init --template saas|usage|license` writes a
complete, pushable starter catalog; skip step 1 and go straight to editing.

1. `kelviq pull` — fetch the live catalog into `kelviq.config.ts`
   (`--prod` to pull production instead of the sandbox default).
2. Edit `kelviq.config.ts` in a branch. It's typed TypeScript: `product`,
   `feature`, and `plan` builders, cross-referenced by stable `identifier`
   slugs — never by UUID. UUID resolution is the CLI's job during sync, not
   something you write by hand.
3. `kelviq push --dry-run` — a human-readable diff, zero writes. Read it.
4. If it looks right, `kelviq push` — prompts for confirmation
   interactively; pass `--yes` only in a non-interactive context (e.g. CI)
   where a human has already reviewed the dry-run diff.
5. Test the result: with `KELVIQ_ENV=sandbox`, the `kelviq:` MCP tools (see
   the `usage-based-billing` skill for entitlement checks) can verify what
   the pushed config actually produced against the sandbox environment.
6. `kelviq promote --dry-run` then `kelviq promote` — reconciles sandbox
   into production the same way `push` reconciles a config file, with no
   local config file involved.

New plans land as drafts and stay unpublished unless you pass `--publish` —
`kelviq status` (`--json` for scripts) shows each plan's draft/published
state per environment. A plan's `prices` array replaces its full price list
rather than patching individual entries, but within a price the CLI compares
only the fields the config sets: fields you omit are left to the server and
do not show up as a diff on every push.

## Config shapes worth knowing

- A `METER` feature must set `featureDetails: { featureSubType:
  'PRE_AGGREGATED_USAGE', units: { singular, plural } }`. `kelviq push` refuses to create a METER
  without it, before writing anything, because the API rejects it.
- Free plan price: `{ priceType: 'FREE', currency, freeTrial: false,
  trialPeriod: 0, enabled: true, taxBehavior }` with no charges.
- Paid subscription price: one `chargeCatalogPrice` entry with `feature:
  null, priceModel: 'FLAT', paymentType: 'ADVANCE_COMMITMENT', reset:
  'NEVER'` and one charge per period, e.g. `{ chargePeriod: 'MONTHLY',
  priceData: { amount: 29 } }`. Amounts are in normal currency units, not
  cents.
- Usage price on a METER: `priceModel: 'TIERED', reset: 'EVERY_MONTH',
  paymentType: 'PAY_AS_YOU_GO'` with `tiers: [{ upTo, unitAmount,
  flatAmount }]`, `upTo: 'inf'` on the last tier.
- License keys on a plan: `license: { enabled: true, activationLimitEnabled:
  true, activationLimit: 3, hasExpiry: false }`.

If Kelviq rejects a price, show the user the error rather than retrying
variations.

## Other CLI commands the agent should use instead of raw HTTP

- `kelviq webhook add <url> --events a,b --write-secret .env.local` registers
  an endpoint (reusing an existing one with the same URL), writes
  `KELVIQ_WEBHOOK_SECRET` into the file and never prints it.
  `kelviq webhook list` shows what's registered.
- `kelviq discount create ...` and `kelviq discount list` — the product is
  given by identifier.

## Config values are open, not closed

Fields like `taxCode`, feature `type`, `priceType`, and `reset` are open
enums in the config format: values the CLI doesn't recognize (including
org-gated ones not everyone has enabled) pass through unchanged, and the
server validates them on `push`. Don't reject a config value just because
it's unfamiliar — let the server be the judge.

## Agent guidance: dry-run is yours, applying is not

An agent may run `push --dry-run` or `promote --dry-run` and show the human
the diff — that's read-only and safe to do freely, as are `env --check`,
`status`, `webhook list` and `discount list`. Applying a real change
(`push` or `promote` without `--dry-run`, including with `--yes`, and
likewise `--publish`, `webhook add` and `discount create`) requires explicit
human approval for that specific change. Never chain a dry-run straight into
an unattended `--yes` apply on your own initiative — the human needs to see
the diff first, every time.

## Docs worth reading first

`kelviq:docs_read` on `cli/overview` for the workflow, `cli/kelviq-config`
for the config file format, `cli/commands` for the full flag reference, and
`cli/environments` for how sandbox vs. production targeting and keys work.
Read before answering a schema or flag question — don't enumerate the
config format from memory here.
