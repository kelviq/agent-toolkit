---
name: usage-based-billing
description: Gate features at runtime, report metered usage, and manage license keys on Kelviq — entitlement checks, usage reporting, and license activation — via the kelviq MCP tools. Use when the user wants to check feature access, report or correct usage, or activate/validate/deactivate a license key.
---

# Usage-based billing

Runtime checks against what a customer is entitled to, safe usage reporting
for metered features, and license key lifecycle.

## Setup

Needs the `kelviq:` MCP tools (`@kelviq/mcp-server`, `KELVIQ_SERVER_API_KEY`).
See the `kelviq` skill's Setup section for the `.mcp.json` snippet and
sandbox testing with `KELVIQ_ENV=sandbox`.

## Feature types, briefly

`BOOLEAN` is a flag, `CUSTOMIZABLE` is a numeric quantity, `METER` is
usage-tracked. Only `METER` features take usage reports; `BOOLEAN` and
`CUSTOMIZABLE` are checked via entitlements alone.

## Gate a feature at runtime

`kelviq:entitlements_get` with `customerId` (and optionally `featureId` to
scope to one feature) is the fast path — it's served from the edge, not the
main API, specifically so runtime access checks are cheap. Each entry
returns `featureType`, `hasAccess`, `usageLimit`, `currentUsage`, and
`resetAt` where applicable. Check `hasAccess` (and, for metered features,
whether `currentUsage` is under `usageLimit`) before letting an action
through — don't infer access from the customer's plan name.

If access looks wrong for a specific customer, don't guess — walk the
`kelviq` hub skill's diagnose-customer flow (or invoke the `diagnose_customer`
prompt) to compare their entitlements against what their plan actually
grants.

## Report usage safely

`kelviq:usage_report` takes a `behaviour` of `SET` or `DELTA`. `SET`
replaces the stored value outright — safe to retry, since retrying just sets
the same value again. `DELTA` increments by the given amount — retrying a
DELTA call blindly (e.g. after a network timeout with an unclear outcome)
double-counts usage. Before retrying a DELTA report, check whether the first
call actually landed (e.g. via `entitlements_get`'s `currentUsage`) rather
than firing it again on faith. When correcting a wrong counter, prefer `SET`
to an absolute value over a compensating `DELTA`.

## License keys

Lifecycle: `kelviq:license_activate` → `kelviq:license_validate` →
`kelviq:license_deactivate`.

- `license_activate` binds a license key to one instance (a device or
  install) and returns an `instanceId`. Store it — `license_deactivate` and
  instance-scoped `license_validate` calls need it later. Losing it means
  you can't cleanly free that activation slot.
- `license_validate` checks a key (optionally scoped to an instance) and
  returns `{ valid, code, detail }` plus the underlying license object.
- `license_deactivate` frees up an activation slot for reuse.

## Docs worth reading first

`kelviq:docs_read` on `product-catalog/usage-based-billing` for metering
concepts, `product-catalog/entitlements` for entitlement fields, and
`product-catalog/license-keys` for license key management. Read before
answering a field-level question — don't enumerate response shapes from
memory here.
