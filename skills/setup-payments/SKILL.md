---
name: setup-payments
description: Sell Kelviq plans and manage subscribers — checkout sessions, subscriptions, trials, the customer portal, upgrades, discount codes, and webhook endpoints — via the kelviq MCP tools. Use when the user wants to charge a customer, set up checkout, manage a subscription, issue a portal link, upgrade/cancel a subscriber, create a discount code, or register a webhook.
---

# Setup payments

Turn a published plan into revenue: checkout for new customers, subscription
management and the customer portal for existing ones, discount codes, and
webhooks that tell the app what happened.

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

`checkout_create_session` required fields: `planIdentifier`, `successUrl`
(an absolute URL — a relative path is rejected), `chargePeriod`. It also accepts `discountCode`, `metadata`, `trialPeriod`
(overrides the plan's trial), `billingPeriodsEnabled` (e.g. `"MONTHLY,YEARLY"`)
and `email` (pre-fill; add `lockEmail` to stop edits). If you pass a
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

## Free plans: enrolling at signup, upgrading later

Free plans are one-time charges (`chargePeriod: ONE_TIME`), not a recurring
subscription, and nothing is charged — so they are the one case where
`kelviq:subscription_create` is the right tool rather than checkout. To put a
new signup on the free plan: `kelviq:customer_create` with an **email**
first, then `kelviq:subscription_create` with the plan identifier and
`chargePeriod: "ONE_TIME"`. Doing it in the other order registers an
email-less customer; the fix is `kelviq:customer_update` with the email. See
`kelviq:docs_read` on `product-catalog/free-and-trials`.

To move a customer from a free plan to a paid one, use
`kelviq:checkout_create_session` — not `kelviq:subscription_update`, which
does not handle this transition.

## Managing an existing subscriber

- **Change plan / charge period / features**: `kelviq:subscription_update`.
  Pass `trialEnd` to end a trial immediately (`"now"`) or move it to a new
  ISO 8601 date. Pass `paymentBehavior: "activate_on_payment"` to keep the
  current subscription active while payment is pending, applying the change
  only once it succeeds. `prorationBehavior` decides when the change is billed:
  `IMMEDIATE_CHARGE`, `PRORATE_NEXT_INVOICE`, or `NO_PRORATION`.
- **List a customer's subscriptions**: `kelviq:subscription_list` filtered
  by `customerId`.
- **Cancel**: `kelviq:subscription_cancel` with `cancellationType`
  (`IMMEDIATE` | `CURRENT_PERIOD_ENDS` | `SPECIFIC_DATE`, the last needing
  `cancellationDate`). When the user gives a reason, record it with
  `cancellationFeedback` and `cancellationComment`. Restate what you're about
  to cancel and get the user's confirmation first — this is a destructive,
  hard-to-walk-back action.
- **Customer portal link**: `kelviq:portal_session_create`, then use the
  response's **`signedPortalUrl`** — not `customerPortalUrl` alone, which
  does not authenticate the customer. The token expires; generate a fresh
  session per request rather than reusing an old link. It returns **400**
  until the customer exists in Kelviq with an email address — an unknown
  `customerId`, or a record created without an email. That's a
  precondition, not a server fault: create the customer
  (`kelviq:customer_create`) or add the email (`kelviq:customer_update`),
  then retry.

## Discount codes

- `kelviq:discount_create` makes a code customers enter at checkout (or that
  you pre-apply with `discountCode`). `PERCENTAGE` needs `percentageOff`;
  `FIXED_AMOUNT` needs `amountOff` and `currency`. `code` is uppercase letters
  and numbers. `duration` is `ONCE`, `REPEATING` (with `durationInMonths`)
  or `FOREVER`; `appliesTo` limits it to a product (identifier or UUID) and
  some of its plans. Merchant of Record organizations only.
- `durationInMonths` counts **billing cycles**, not calendar months: `3` on a
  yearly plan is three years off. A "first 3 months" code belongs scoped to
  the monthly plans via `appliesTo`.
- Only `isActive` can change afterwards (`kelviq:discount_update`). Restate
  the code, amount, duration, limits and scope and get confirmation before
  creating. To change anything else, create a new code and disable the old one.
- `kelviq:discount_archive` makes the code unusable — confirm first.

## Webhooks

- Call `kelviq:webhook_endpoint_list` before `kelviq:webhook_endpoint_create`
  so the same URL isn't registered twice. The URL must be publicly reachable;
  localhost and private addresses are rejected, so a local-only app registers
  its endpoint after deploying (or through a tunnel).
- Signing secrets are hidden in tool results. The app needs the secret in its
  env file, not the conversation, so don't pass `revealSecret: true` just to
  look at it. If a secret has to be stored, write it straight into the env
  file with a command that doesn't print it.
- `kelviq:webhook_endpoint_delete` stops deliveries immediately — confirm first.
- Events not arriving? `kelviq:webhook_log_list` shows delivery attempts, status
  codes and retries from the last 30 days.
- Event semantics worth getting right: `subscription.cancelled` fires when the
  subscription actually ends, not when a cancellation is scheduled (that's a
  `subscription.updated`); `invoice.payment_failed` is a failed renewal.

## Docs worth reading first

`kelviq:docs_read` on `product-catalog/subscriptions` for subscription
mechanics, `checkout/checkout-configuration` for checkout options,
`customer-portal/overview` for portal behavior,
`product-catalog/discounts` for discount codes, and `guides/webhooks` for
events and signature verification. Read before answering a field-level
question — don't enumerate the schema from memory here.
