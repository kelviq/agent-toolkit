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

`kelviq env` shows what's configured, with no live validation. The **`kelviq:`
MCP tools** are used here only to verify a pushed config against sandbox
(step 5 below) and to read docs (`docs_search`/`docs_read`, keyless). See the
`kelviq` skill's Setup section for the `.mcp.json` snippet if those tools
aren't already available.

## When to reach for the CLI over the dashboard or MCP

Bulk or repeated pricing changes, review-before-apply workflows, version
history in git, and promoting a catalog you've already tested in sandbox to
production. For a one-off plan or a single price, the dashboard (or MCP for
everything except prices) is simpler.

The config file covers products, features and plans. Discount codes and
webhook endpoints are not part of it — create those with the `kelviq:`
discount and webhook tools (see the `setup-payments` skill), and don't try to
express them in `kelviq.config.ts`.

## The loop

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

New plans land as drafts and stay unpublished unless you pass `--publish`;
price changes replace a plan's full price list rather than patching
individual entries.

## Config values are open, not closed

Fields like `taxCode`, feature `type`, `priceType`, and `reset` are open
enums in the config format: values the CLI doesn't recognize (including
org-gated ones not everyone has enabled) pass through unchanged, and the
server validates them on `push`. Don't reject a config value just because
it's unfamiliar — let the server be the judge.

## Agent guidance: dry-run is yours, applying is not

An agent may run `push --dry-run` or `promote --dry-run` and show the human
the diff — that's read-only and safe to do freely. Applying a real change
(`push` or `promote` without `--dry-run`, including with `--yes`) requires
explicit human approval for that specific change. Never chain a dry-run
straight into an unattended `--yes` apply on your own initiative — the
human needs to see the diff first, every time.

## Docs worth reading first

`kelviq:docs_read` on `cli/overview` for the workflow, `cli/kelviq-config`
for the config file format, `cli/commands` for the full flag reference, and
`cli/environments` for how sandbox vs. production targeting and keys work.
Read before answering a schema or flag question — don't enumerate the
config format from memory here.
