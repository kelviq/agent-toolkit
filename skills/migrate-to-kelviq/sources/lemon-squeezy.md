# Source: Lemon Squeezy

Reference for the `migrate-to-kelviq` skill when the app is moving off Lemon
Squeezy. Source id: `lemon-squeezy`. Follow the loop in `../SKILL.md`; this
file supplies the Lemon Squeezy specifics for each step.

**Lemon Squeezy stops processing payments, including subscription renewals,
on March 31, 2027.** Until then, imported subscriptions keep renewing on
Lemon Squeezy. After that date an imported subscriber is not billed anywhere
unless they have subscribed again on Kelviq or Kelviq has moved them. Say
this whenever you tell the user that subscribers "keep billing on Lemon
Squeezy": it has an end date, and it is never open-ended.
`kelviq:docs_read` on `guides/migrate-from-lemon-squeezy` has the key dates.

Moving active subscriptions and saved cards is a separate, assisted process
the user arranges with Kelviq (hi@kelviq.com). The import and this skill do
not do it.

## Running the import

| What | Where it comes from |
|---|---|
| Lemon Squeezy API key | `LEMONSQUEEZY_API_KEY` in the environment. Never a tool input, never a flag. |
| Store id | `storeId` (MCP) or `--store <id>` (CLI; `LEMONSQUEEZY_STORE_ID` when omitted). Digits only: the number next to the store in Lemon Squeezy under Settings → Stores. |

Never ask the user to paste the Lemon Squeezy key. If `kelviq:import_start`
says the key is missing, or a `400` says Lemon Squeezy rejected it, tell the
user to set `LEMONSQUEEZY_API_KEY` in the same place as
`KELVIQ_SERVER_API_KEY` — their shell profile, or the `env` block of the MCP
client's config — and to restart the client.

One store is imported per run. A key can reach several stores, so repeat the
run for each one.

```bash
kelviq import lemon-squeezy --store <id> --dry-run     # saves nothing, emails a report
kelviq import lemon-squeezy --store <id>               # real run, only after approval: publishes plans and codes
```

With MCP: `kelviq:import_start` with `source: "lemon-squeezy"`, `storeId` and
`dryRun` (required — choose it deliberately, `true` first).

| Option | MCP input | CLI flag |
|---|---|---|
| Run only some steps | `only` | `--only <steps>` |
| Leave steps out | `skip` | `--skip <steps>` |
| All sales, one-time purchases only, or subscriptions only | `orderType` | `--orders <all\|one-time\|subscription>` |
| Import plans as drafts instead of publishing them | `publishPlan: false` | `--no-publish-plans` |
| Leave imported discount codes switched off | `publishDiscount: false` | `--no-publish-discounts` |
| Send the report somewhere other than the owner | `reportEmail` | `--report-email <address>` |
| Overwrite the organization name and the business profile fields that are already filled in | `overwriteProfile` | `--overwrite-profile` |
| Leave the imported sales out of revenue analytics | `skipAnalytics` | `--skip-analytics` |
| Import into production | `KELVIQ_ENV=production` | `--prod` |

- **`overwriteProfile` replaces what the user has already entered**: the
  organization name and every business profile field that is filled in.
  Never set it unless the user asks for exactly that. Without it only blank
  fields are filled.
- **Leave `reportEmail` unset** unless the user names an address. The report
  carries revenue totals, and by default it goes to the organization owner.
- **A real run publishes plans and switches on codes unless it opts out.**
  By default each imported plan is published, so it can be bought as soon
  as the app points at it, and imported discount codes work at checkout.
  `publishPlan: false` imports plans as drafts, and `publishDiscount: false`
  leaves the codes switched off. Tell the user what the default does and
  let them choose; never set either option on your own. A dry run publishes
  nothing and saves nothing.
- **Running the import again publishes what an earlier run held back.**
  Plans an earlier run left as drafts are published, and imported codes
  that are still off are switched on, unless the new run opts out too.
  Archived plans, plans that already have a published version, and codes
  limited to plans that are not published are left alone.
- **The report's warnings name what did not go live.** A plan the import
  could not publish stays a draft: publish it in the dashboard. A code
  limited to plans that are not published stays off: publish those plans,
  then run the import again. `kelviq:discount_list` shows the codes.
- **`skipAnalytics` is the user's choice.** Imported sales are added to
  revenue analytics unless it is set; don't set it on your own.

### Catching up later

Sales and refunds keep landing on Lemon Squeezy while the user switches. The
default catch-up is the full import again, with no `only` or `skip`: it adds
only what is new, including license keys issued since the last run. It also
publishes plans an earlier run left as drafts and switches on codes that are
still off, so if the user held them back on purpose, ask whether to pass
`publishPlan: false` / `publishDiscount: false` again.

```bash
kelviq import lemon-squeezy --store <id>                          # full catch-up
kelviq import lemon-squeezy --store <id> --only orders,renewals   # quicker, for a store that sells no license keys
```

With MCP the quicker variant is `only: ["orders", "renewals"]`. It leaves
out the `licenses` step, so don't use it for a store that issues license
keys. A catch-up run is still a real import: it needs the user's approval
each time, in the environment it runs in. It does not need another dry run.

### The eight steps

They run in this order. `only` and `skip` take these names.

| Step | What arrives in Kelviq |
|---|---|
| `store` | The organization name, and the website, logo and country on the business profile. Fields already filled in are left alone. |
| `products` | Products, and one plan per Lemon Squeezy variant with its price, trial, license settings and links. |
| `customers` | Customers. One whose email already exists in the organization is linked, not duplicated. |
| `files` | Each variant's downloadable files, attached to its plan. |
| `orders` | Orders with their invoices, and the subscription record behind each paid order. |
| `renewals` | Subscription renewal payments, as renewal orders with invoices. |
| `discounts` | Discount codes with their limits and redemption counts. |
| `licenses` | License keys and their activations. |

`orders` needs `products`; `renewals` and `licenses` need `orders`. Those may
have run in an earlier import. A license key is imported only if its order
was, so limiting the order type also limits the keys. Affiliate programs,
email subscriber lists and storefront pages are not imported.

### The report

The report's subject is "Lemon Squeezy import finished", "… finished with
failures" (some records could not be imported; they are listed, and retried
on the next run) or "… failed" (it names the step and the error).

It contains the store and the organization it went into, a count per record
type, the totals charged, tax and refunds from Lemon Squeezy, up to 50
warnings for anything skipped or imported differently, and the start time,
duration and steps that ran. Have the user compare the totals with what they
expect before approving a real run, and read every warning with them.

It has no per-plan prices. To check a plan, compare its price, trial and
license settings in Kelviq (`kelviq:plan_retrieve`, `kelviq:plan_prices_list`)
with the variant in Lemon Squeezy, which the user opens in their Lemon
Squeezy dashboard.

No report is sent in two cases: a second import was started while one was
running (only the first one reports), or the import could not start within
24 hours. If 24 hours pass with no email, run it again — with the user's
approval if it is a real run. If that returns `409` and still nothing
arrives, the user should write to hi@kelviq.com with the `taskId` from the
tool result; support can clear a run that stopped.

### A `403` from the import

Read the `detail` text (the MCP tool shows it in the error `message`)
before deciding what is wrong:

| `detail` | What to do |
|---|---|
| "Use a server key created by the organization owner." | The key works but the owner did not create it. The organization owner creates a Server API Key and the user sets that one. Don't work around it. |
| "Invalid Bearer Token" or "Authentication credentials were not provided." | The Kelviq key was not accepted: it is missing, wrong, a Client API Key, or from the other environment. Run `kelviq:connection_check` and have the user fix the key or `KELVIQ_ENV`. This is not the owner rule, so don't send them to the owner. |

## Concepts

| In Lemon Squeezy | In Kelviq |
|---|---|
| Store | Organization |
| Product | Product |
| Variant | Plan — a variant id in the app becomes a plan `identifier` |
| Discount code | Discount code |
| Custom checkout data (`custom` / `meta.custom_data`) | `customerId`, plus `metadata` on the checkout session |

The import brings plans with price, trial, license settings and links. It
does not create features or entitlements. An app that gates on its own
subscription state from webhooks keeps doing that; moving to entitlement
checks is a separate piece of work (`pricing-plans`, `usage-based-billing`).

## Inventory: what to search for

Two searches. The first reads source and config and skips env files, because
those hold the Lemon Squeezy API key and the webhook secret. The second
lists the variable names in env files and nothing else.

```bash
# Source, config and deployment files. Env files are excluded on purpose.
grep -rnEi 'lemon[ ._-]?squeezy|lmsqueezy|lemon\.js|x-signature|checkout/buy/|event_name|custom_data|subscription_payment_|\bLEMON_|\bLS_[A-Z]' . \
  --exclude='.env*' \
  --exclude-dir=node_modules --exclude-dir=.git --exclude-dir=dist --exclude-dir=build \
  --exclude-dir=.next --exclude-dir=out --exclude-dir=coverage

# Env files, at any depth: file and variable NAME only, never the value.
find . -type f -name '.env*' -not -path '*/node_modules/*' -not -path '*/.git/*' \
  -exec grep -HoE '^(export )?[A-Z0-9_]*(LEMON|LS_)[A-Z0-9_]*' {} +
```

- **Never print a matching line or a value from an env file.** Don't `cat`
  or open `.env*` files, don't run any other search over them, and don't
  drop the `--exclude='.env*'`. The names are all the inventory needs.
- If a hit in any other file (deployment config, a CI workflow, a script)
  turns out to hold a key or secret, don't repeat the value: give the user
  the file and line, and tell them to move it to an env var and rotate it.
- `event_name`, `custom_data` and `LS_` also match code that has nothing to
  do with Lemon Squeezy. Open each hit and drop the ones that don't belong.
- Add the app's other build output directories to the exclusions if it has
  any.

| Surface | What it looks like in the app |
|---|---|
| SDK | `from '@lemonsqueezy/lemonsqueezy.js'` (some package names and URLs spell it `lmsqueezy`): `lemonSqueezySetup`, `createCheckout`, `getSubscription`, `updateSubscription`, `cancelSubscription`, `activateLicense`, `validateLicense`, `deactivateLicense` |
| REST calls | Requests to `api.lemonsqueezy.com/v1/...` (`checkouts`, `subscriptions`, `orders`, `customers`, `licenses`) |
| Checkout links | `https://<store>.lemonsqueezy.com/checkout/buy/<variant>` or `/buy/<variant>`, or the same `/checkout/buy/<variant>` path on the store's custom domain, in buttons, emails and pricing pages, often with `checkout[email]`, `checkout[custom][...]` or `checkout[discount_code]` query parameters |
| Checkout overlay | The `lemon.js` script tag, the `lemonsqueezy-button` class, `createLemonSqueezy()`, `LemonSqueezy.Url.Open(...)`, `LemonSqueezy.Setup({ eventHandler })`, `embed=1` on a link |
| Webhook handler | A route that reads the `X-Signature` header and compares it with an HMAC-SHA256 hex digest of the raw body, then switches on `meta.event_name` (or the `X-Event-Name` header). Some handlers never name Lemon Squeezy: look for `event_name`, `custom_data` and event names such as `subscription_payment_success` |
| License API | `api.lemonsqueezy.com/v1/licenses/activate`, `/validate`, `/deactivate`, sent with `license_key`, `instance_name` or `instance_id` |
| Customer portal | `urls.customer_portal`, `urls.update_payment_method` or `urls.customer_portal_update_subscription` read from a subscription, or a link to `https://<store>.lemonsqueezy.com/billing` |
| Env vars | `LEMONSQUEEZY_API_KEY`, `LEMONSQUEEZY_STORE_ID`, `LEMONSQUEEZY_WEBHOOK_SECRET`, variant id variables, and the `LEMON_SQUEEZY_*`, `LEMON_*` and `LS_*` spellings |

## Before the mappings: add, don't replace

Every mapping below is for **new sales**. Subscribers who bought on Lemon
Squeezy still bill there until March 31, 2027, so their paths stay in the
app: the Lemon Squeezy webhook handler and its secret, the Lemon Squeezy
portal and update-payment links, and the calls that change or cancel those
subscriptions. The app adds Kelviq checkout, webhook, license and portal
code beside them, and records which platform each subscription bills on
(see "What stays on Lemon Squeezy" below). Settle that record first; the
portal and cancellation code branches on it.

## Mapping: SDK and client

Add a Kelviq client beside the Lemon Squeezy one, server-side only. The
Lemon Squeezy client stays while the app still reads or changes
subscriptions that bill there:

```typescript
import { Kelviq, environmentFromEnv } from '@kelviq/node-sdk';

const client = new Kelviq({
  accessToken: process.env.KELVIQ_SERVER_API_KEY!,
  environment: environmentFromEnv(), // reads KELVIQ_ENV: sandbox when unset
});
```

Python: `from kelviq_sdk import Kelviq`, then
`Kelviq.create_sync_client(access_token=...)` or
`Kelviq.create_async_client(access_token=...)`.

## Mapping: checkout

| Lemon Squeezy | Kelviq |
|---|---|
| `createCheckout(storeId, variantId, {...})`, or `POST /v1/checkouts`, then redirect to the returned URL | `client.checkout.createSession({ planIdentifier, chargePeriod, successUrl, customerId, metadata })`, then redirect to `checkoutUrl` |
| A fixed `.../buy/<variant>` link | A static checkout URL: `https://www.kelviq.com/buy/{PRODUCT_UUID}/` (sandbox: `https://www.kelviq.com/sandbox/buy/{PRODUCT_UUID}/`), copied from **Product Catalog → Products → Get Link** |
| `checkout[custom][user_id]=...` on a link, `checkoutData.custom` in the SDK | `customer_id` on a static URL; `customerId` and `metadata` on a session |
| `checkout[discount_code]=...` | `discount_code` on a static URL; `discountCode` on a session |
| Redirect URL after purchase | `success_url` on a static URL; `successUrl` on a session (absolute URL) |
| The `lemon.js` overlay and its `Checkout.Success` handler, on buy buttons | Send the customer to the hosted checkout URL, and grant access from the webhook. Take the button class off the buy buttons that now go to Kelviq. Leave the `lemon.js` script in place if the app also uses it to open update-payment links for existing subscribers |

- `planIdentifier`, `chargePeriod` and `successUrl` are required on a
  session. `chargePeriod` is `ONE_TIME` for a one-time purchase and a period
  such as `MONTHLY` or `YEARLY` for a subscription; the plan must support it.
- Static URLs take snake_case query parameters (`plan_identifier`,
  `charge_period`, `enabled`); the SDK takes camelCase. Don't mix them.
- Always pass the app's own user id as `customerId` / `customer_id`.
  Without it Kelviq creates a new customer, and the webhook can't be tied
  back to the app's user.
- A visit to the success URL is not proof of payment, even where the
  overlay's success handler was treated as one. Grant access from the
  webhook.
- Don't assume an imported customer's Kelviq customer id equals the app's
  user id. Open one imported customer in the dashboard, check, and agree the
  mapping with the user before wiring `customerId`.

## Mapping: webhooks

Add a new route for Kelviq. Leave the Lemon Squeezy route as it is (step 5).

| | Lemon Squeezy | Kelviq |
|---|---|---|
| Signature header | `X-Signature` | `webhook-signature`, formatted `v1,<hex>`, with `webhook-id` and `webhook-timestamp` |
| What is signed | The raw body | `{webhook-id}.{webhook-timestamp}.{raw body}` |
| Secret | `LEMONSQUEEZY_WEBHOOK_SECRET` | `KELVIQ_WEBHOOK_SECRET` |
| Event name | `meta.event_name` | `type` |
| Resource | `data.attributes` | `data.object` |
| Custom data | `meta.custom_data` | `metadata` on `checkout.completed`; the embedded `customer.customer_id` |

The signed string differs, so the old verify function can't be reused. Use
the SDK helper, which verifies and parses in one step and throws
`WebhookVerificationError` on a bad signature:

```typescript
import { validateEvent, WebhookVerificationError } from '@kelviq/node-sdk';

export async function POST(request: Request) {
  const body = await request.text(); // raw, unparsed
  try {
    const event = validateEvent(
      body,
      Object.fromEntries(request.headers),
      process.env.KELVIQ_WEBHOOK_SECRET!,
    );
    // switch on event.type
    return new Response(null, { status: 202 });
  } catch (err) {
    if (err instanceof WebhookVerificationError) {
      return new Response(null, { status: 403 });
    }
    throw err;
  }
}
```

Python: `validate_event(payload=<raw body>, headers=request.headers,
secret=<secret>)` from `kelviq_sdk`. In both, pass the raw body — parsing it
as JSON first breaks the check. The headers differ by language: in Node pass
a plain object, so convert a Fetch `Headers` instance with
`Object.fromEntries(request.headers)` as above; in Python `request.headers`
works as it is.

Register the endpoint once the route is deployed (the URL must be public;
localhost is rejected). This writes the signing secret into the env file
without printing it, and needs the user's go-ahead:

```bash
kelviq webhook add https://app.example.com/api/kelviq/webhooks \
  --events <the Kelviq events the handler uses, comma-separated> \
  --write-secret .env.local
```

The `--events` value comes from the app: take every Lemon Squeezy event the
existing handler acts on and map each one with the table below.

### Event mapping

| Lemon Squeezy event | Kelviq event |
| --- | --- |
| `order_created` | `checkout.completed`, `order.created` |
| `order_refunded` | `order.refunded` |
| `subscription_created` | `subscription.created` |
| `subscription_updated` | `subscription.updated`, `subscription.plan_changed` |
| `subscription_cancelled` | `subscription.updated` (the end date is set while the period runs out) |
| `subscription_resumed` | `subscription.updated` (a scheduled cancellation was undone). Not Kelviq's `subscription.resumed`, which is for pauses |
| `subscription_expired` | `subscription.cancelled` |
| `subscription_paused`, `subscription_unpaused` | `subscription.paused`, `subscription.resumed` |
| `subscription_payment_success`, `subscription_payment_recovered` | `invoice.paid` |
| `subscription_payment_failed` | `invoice.payment_failed` |
| `subscription_payment_refunded` | `order.refunded`, `refund.created` |
| `license_key_created` | No separate event. The key is issued with the order (`order.created`) |

Register the Kelviq endpoint for every event the app's handler actually
uses, not a fixed list. For a Lemon Squeezy event that isn't in this table,
read the event list in `guides/webhooks` and agree the choice with the user
rather than guessing.

- `checkout.completed` carries no `subscription_id`. The subscription's id
  arrives in `subscription.created`; tie the two together with the customer
  id or the checkout `metadata`.
- `subscription.cancelled` fires when the subscription actually ends. A
  cancellation scheduled for the period end is a `subscription.updated` with
  `end_date` set and the status still active — don't remove access on it.
- A failed renewal sends `invoice.payment_failed`, and the subscription
  moves to `past_due` through `subscription.updated` while Kelviq retries.
  Access ends on `subscription.cancelled`, unless the app's own policy says
  otherwise.
- `order.refunded` doesn't carry the refunded amount; read it from
  `refund.created` / `refund.updated` when the app needs it.
- Deliveries can repeat. Dedupe on the event `id` (same value as the
  `webhook-id` header).

## Mapping: license keys

| Lemon Squeezy | Kelviq |
|---|---|
| `POST /v1/licenses/activate` with `license_key`, `instance_name` (SDK: `activateLicense`) | `client.license.activate({ licenseKey, instanceName })` — returns `instanceId` |
| `POST /v1/licenses/validate` with `license_key`, optional `instance_id` (SDK: `validateLicense`) | `client.license.validate({ licenseKey, instanceId })` — returns `valid`, `code`, `detail` |
| `POST /v1/licenses/deactivate` with `license_key`, `instance_id` (SDK: `deactivateLicense`) | `client.license.deactivate({ licenseKey, instanceId })` |

- The Lemon Squeezy license endpoints take no API key. Kelviq's take a
  Bearer key, and which key depends on where the call runs. The
  `client.license.*` calls belong on a backend with the Server API Key. An
  app distributed to customers (a desktop app, mobile app, CLI or plugin)
  must call `POST /license/activate/`, `/license/validate/` and
  `/license/deactivate/` with a **Client API Key**, never a server key.
  `kelviq:docs_read` on `product-catalog/license-keys` has the table of
  which key goes where. Find out which case the app is before writing any
  license code.
- Field names change from snake_case to camelCase, and the instance id moves
  from `instance.id` in the response to `instanceId`. Store it: validate and
  deactivate need it.
- Imported keys and their activations arrive with the `licenses` step, for
  the orders that were imported. Before relying on that, validate one real
  imported key against Kelviq, and check whether an instance id the app
  stored from Lemon Squeezy is accepted.
- **Don't activate again blindly.** An imported activation may already hold
  one of the key's activation slots, and a fresh `activate` call can take
  another. Validate first. If a stored instance id is not accepted, stop
  and work out with the user how imported instance ids behave — test with
  one key, read the activation count in the response, and ask Kelviq
  (hi@kelviq.com) if it is unclear — before writing code that re-activates
  on every customer's machine.
- Copies of the app already installed keep calling Lemon Squeezy until
  their users update. A server-side change does not reach them; tell the
  user this needs a release.

## Mapping: customer portal

| Lemon Squeezy | Kelviq |
|---|---|
| `urls.customer_portal` and the other `urls.*` links on a subscription, or the store's `/billing` page | `client.portal.createSession({ customerId })`, then send the customer to `${customerPortalUrl}?token=${token}` built from the response |

`customerPortalUrl` alone does not authenticate; the token has to be on the
link. Tokens expire, so create a session per visit rather than storing the
link. Python: `client.portal.create_session(customerId=...)`. To test from
the agent, `kelviq:portal_session_create` returns the same link ready-made
as `signedPortalUrl`. A session can only be created for a customer that
exists in Kelviq with an email address.

Subscribers who still bill on Lemon Squeezy manage their subscription and
card there, so keep showing them the Lemon Squeezy portal and update-payment
links, and keep whatever the app uses to open them.

Calls that change a subscription (`updateSubscription`,
`cancelSubscription`) apply only to subscriptions billed on Lemon Squeezy.
For subscriptions sold on Kelviq, see the `setup-payments` skill.

## Mapping: env vars

| Lemon Squeezy | Kelviq |
|---|---|
| `LEMONSQUEEZY_API_KEY` | `KELVIQ_SERVER_API_KEY` (server only) |
| `LEMONSQUEEZY_WEBHOOK_SECRET` | `KELVIQ_WEBHOOK_SECRET`, written by `kelviq webhook add --write-secret` |
| `LEMONSQUEEZY_STORE_ID` | Nothing: the key identifies the organization |
| Variant id variables | Plan identifiers |
| — | `KELVIQ_ENV` (`sandbox` or `production`); `KELVIQ_CLIENT_API_KEY` only for license calls made from shipped software |

Add the Kelviq variables; don't remove the Lemon Squeezy ones yet. Name the
variables the user has to set in their host's settings — don't ask for the
values.

## What stays on Lemon Squeezy

Until the user confirms no subscription bills there:

- the Lemon Squeezy webhook route, its secret, and whatever it updates;
- the portal and update-payment links shown to subscribers who bill there,
  and the `lemon.js` script if the app uses it to open them;
- the calls that change or cancel those subscriptions;
- `LEMONSQUEEZY_API_KEY`, which the import also needs for catch-up runs.

The app has to know which platform a subscription bills on. If it stores
subscriptions without that, add a field for it (existing rows are Lemon
Squeezy; rows created by the Kelviq handler are Kelviq) and branch on it for
portal links and cancellation. Show the user this change in the inventory,
and make it before the mappings above.

This arrangement ends on March 31, 2027, when Lemon Squeezy stops processing
payments, renewals included. A subscriber still on Lemon Squeezy then is not
billed anywhere unless they have subscribed again on Kelviq or Kelviq has
moved them, so raise the date with the user now. Don't promise that
subscriptions will move: that is the assisted process at the top of this
file, arranged with Kelviq.

## Docs worth reading first

`kelviq:docs_read` on `guides/migrate-from-lemon-squeezy` for the full
walkthrough and key dates, `backend-integration/node-sdk` or
`backend-integration/python-sdk` for creating a checkout session and a
portal session, `checkout/static-checkout-url` for fixed checkout links,
`checkout/checkout-metadata` for carrying the app's ids through checkout,
`guides/webhooks` for payloads and verification, and
`product-catalog/license-keys` for which key each license call takes.
