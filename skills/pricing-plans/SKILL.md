---
name: pricing-plans
description: Design and publish a Kelviq pricing catalog — products, features, plans, and entitlements — via the kelviq MCP tools. Use when the user wants to create or edit a pricing model, catalog, product, plan, feature, or entitlement, or asks to publish a plan.
---

# Pricing plans

Build and publish the Kelviq catalog: products contain plans, plans grant
entitlements to features, and nothing is live until published.

## Setup

Needs the `kelviq:` MCP tools (`@kelviq/mcp-server`, `KELVIQ_SERVER_API_KEY`).
See the `kelviq` skill's Setup section for the `.mcp.json` snippet and
sandbox testing with `KELVIQ_ENV=sandbox`.

## Golden path: build and publish a plan

1. `kelviq:product_create` — required: `name`, `taxCode`. If the tax code is
   unknown, ask the user rather than guessing; don't invent one. Keep the
   returned product **UUID**.
2. `kelviq:feature_create` for each feature this plan gates. Pick
   `featureType`: `BOOLEAN` (flag), `CUSTOMIZABLE` (numeric quantity, e.g.
   seats), or `METER` (usage-tracked, e.g. API calls). Keep each returned
   feature **UUID**.
3. `kelviq:plan_create` — pass the product **UUID** from step 1 as `product`
   (not the identifier). The plan lands in DRAFT: not visible, not
   purchasable, until published. Don't tell the user it's live yet.
4. `kelviq:plan_entitlements_add` — attach entitlements using the feature
   **UUIDs** from step 2, e.g. `[{ feature: <uuid>, details: {...} }]`. The
   `details` shape depends on the feature's type — check
   `kelviq:docs_read` on `product-catalog/entitlements` if unsure rather than
   guessing the shape.
5. Prices. There is no price-write tool in MCP — this is deliberate, not a
   gap. Direct the user to set prices in the Kelviq dashboard, or use the
   `pricing-as-code` skill's `kelviq push` flow if they manage pricing in
   git. `kelviq:plan_prices_list` reads prices back once they exist, to
   confirm.
6. `kelviq:plan_publish` — makes the plan (and its DRAFT-only entitlements
   from step 4) live. Pass `updateFeatures` and/or `updatePricing` only if
   you also want to migrate customers already on a previous version of this
   plan to the new features/pricing — always confirm with the user before
   setting either; unset, existing subscribers keep what they had.
7. Verify: `kelviq:plan_retrieve` (confirm published), `kelviq:plan_prices_list`
   (confirm prices landed), `kelviq:offering_get_product` by product UUID
   (see the public pricing view a customer would see).

For the fully guided version of this flow, use the `setup_product` MCP
prompt (new product from nothing) or `launch_plan` (new plan on an existing
product) — don't re-derive their step lists here, just invoke them.

## Free plans

A free plan uses `chargePeriod: ONE_TIME` rather than a recurring period.
Free plans are a `setup-payments` concern once a customer needs to move off
one — see that skill for the checkout-based upgrade path.

## Updating a published plan

`kelviq:plan_update` on an already-published (`is_latest`) plan does not
change it in place — it creates a new draft. The previous version stays live
until you `kelviq:plan_publish` the new draft. Don't tell the user a
`plan_update` call alone took effect.

## Plan files

`kelviq:plan_file_update` (display fields: name, ordering, enabled),
`kelviq:plan_file_delete`, and `kelviq:plan_file_download` manage a plan's
*existing* file attachments. None of them upload a new file — that happens
outside these tools (media upload endpoint, or the dashboard). If the user
wants to attach a brand-new file, say so plainly instead of trying to force
it through `plan_file_update`.

## Docs worth reading first

`kelviq:docs_read` on `quickstart/core-concepts` for the product/feature/plan
model, `product-catalog/plans` and `product-catalog/products` for field-level
detail, `product-catalog/entitlements` for entitlement shapes, and
`product-catalog/free-and-trials` for free-plan specifics. Read before you
answer a "what field does X take" question — don't enumerate the schema from
memory here.
