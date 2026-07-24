---
name: setup-payments
description: Sell Kelviq plans and manage subscribers — checkout sessions, subscriptions, trials, the customer portal, upgrades, and discounts — via the kelviq MCP tools. Use when the user wants to charge a customer, set up checkout, manage a subscription, issue a portal link, or upgrade/cancel a subscriber.
---

# Setup payments

Turn a published plan into revenue: checkout for new customers, subscription
management and the customer portal for existing ones.

## Setup

Needs the `kelviq:` MCP tools (`@kelviq/mcp-server`, `KELVIQ_SERVER_API_KEY`).
See the `kelviq` skill's Setup section for the `.mcp.json` snippet and
sandbox testing with `KELVIQ_ENV=sandbox`.

## Checkout is the default way to charge someone

`kelviq:checkout_create_session` returns a hosted checkout URL — the
customer enters their own payment details there. This is the default sale
path for a reason: `kelviq:subscription_create` charges the customer
off-session (no checkout page) and is **org-gated** — most organizations get
an explicit "off-session charging is not enabled" error back. Reach for
`subscription_create` only when you know off-session charging is enabled for
this org; otherwise default to checkout and treat `subscription_create` as
the exception, not the norm.

`checkout_create_session` required fields: `planIdentifier`, `successUrl`,
`chargePeriod`. It also accepts `discountCode` and `metadata`. If you pass a
`customerId` but the email you give already belongs to an existing customer,
Kelviq reuses that existing customer rather than erroring — don't assume a
fresh `customerId` guarantees a fresh customer record.

## Golden path: sell a plan to a new customer

1. Have (or collect) the customer's `customerId` — this is caller-defined,
   not something Kelviq generates for you to look up. There is no customer
   list/retrieve API; if you don't have an ID, ask the user for one instead
   of trying to search for it.
2. `kelviq:checkout_create_session` with `planIdentifier`, `successUrl`,
   `chargePeriod`, and optionally `customerId`, `discountCode`, `metadata`.
   Hand the returned URL to the customer.
3. After they complete checkout, `kelviq:subscription_list` filtered by
   `customerId` confirms the subscription now exists.

## Upgrading a customer off a free plan

Free plans are one-time charges (`chargePeriod: ONE_TIME`), not a recurring
subscription. To move a customer from a free plan to a paid one, use
`kelviq:checkout_create_session` — not `kelviq:subscription_update`, which
does not handle this transition.

## Managing an existing subscriber

- **Change plan / charge period / features**: `kelviq:subscription_update`.
  Pass `trialEnd` to end a trial immediately (`"now"`) or move it to a new
  ISO 8601 date. Pass `paymentBehavior: "activate_on_payment"` to keep the
  current subscription active while payment is pending, applying the change
  only once it succeeds — useful for a plan/price change you don't want to
  take effect on a failed charge.
- **List a customer's subscriptions**: `kelviq:subscription_list` filtered
  by `customerId`.
- **Cancel**: `kelviq:subscription_cancel` with `cancellationType`
  (`IMMEDIATE` | `CURRENT_PERIOD_ENDS` | `SPECIFIC_DATE`, the last needing
  `cancellationDate`). Restate what you're about to cancel and get the
  user's confirmation first — this is a destructive, hard-to-walk-back
  action.
- **Customer portal link**: `kelviq:portal_session_create`, then use the
  response's **`signedPortalUrl`** — not `customerPortalUrl` alone, which
  does not authenticate the customer. The token expires; generate a fresh
  session per request rather than reusing an old link.

## Docs worth reading first

`kelviq:docs_read` on `product-catalog/subscriptions` for subscription
mechanics, `checkout/checkout-configuration` for checkout options,
`customer-portal/overview` for portal behavior, and
`product-catalog/discounts` for discount codes. Read before answering a
field-level question — don't enumerate the schema from memory here.
