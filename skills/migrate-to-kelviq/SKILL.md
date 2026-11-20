---
name: migrate-to-kelviq
description: Move an app and its store from another billing platform to Kelviq — inventory the old platform's calls, dry-run the server-side store import, run it with the user's approval, then add Kelviq checkout, webhook, license and customer portal code beside the old platform's. Use when the user wants to migrate to Kelviq, switch billing platforms, import an existing store, or move an app off Lemon Squeezy (e.g. "migrate from Lemon Squeezy to Kelviq").
---

# Migrate to Kelviq

A move to Kelviq is two jobs, in this order: bring the store's data across
with a server-side **import**, then point the app's **code** at Kelviq. The
import copies the catalog, customers and sales history; it does not move live
billing. Existing subscribers keep renewing on the old platform, for as long
as that platform still bills them, while new sales start on Kelviq. The
source file gives the date that ends; never describe it as open-ended.

Everything particular to one platform lives in a reference file under
`sources/`. Open the file for the user's platform before touching code.

## Setup

Needs the `kelviq:` MCP tools (`@kelviq/mcp-server`, `KELVIQ_SERVER_API_KEY`)
or the Kelviq CLI (`@kelviq/cli`). See the `kelviq` skill's Setup section for
the `.mcp.json` snippet and sandbox testing with `KELVIQ_ENV=sandbox`, and
the `pricing-as-code` skill's Setup section for installing the CLI and
`kelviq login`.

An import takes two keys, and neither belongs in the conversation:

- **The Kelviq key** must be a Server API Key created by the organization
  **owner**. A server key created by any other team member is refused with
  `403`, as is a Client API Key. The import lands in the environment that key
  belongs to: sandbox unless `KELVIQ_ENV=production` (MCP) or `--prod` (CLI).
  Sandbox and production are separate, so an import into one puts nothing in
  the other.
- **The source platform's key** is read from an environment variable the
  source file names, never from a tool input or a flag. The MCP server reads
  it from its own environment, so the user sets it in the same place as
  `KELVIQ_SERVER_API_KEY` — their shell profile, or the `env` block of the
  MCP client's config — and restarts the client. The CLI reads it from the
  shell or from `.env.local` / `.env` in the working directory; in a
  non-interactive shell with no key set it stops and names the missing
  variable.

## Sources

| Moving from | Source id | Open first |
|---|---|---|
| Lemon Squeezy | `lemon-squeezy` | `sources/lemon-squeezy.md` |

The source file lists what to search for in the app, how each call maps to
Kelviq, the source's env vars, the import's steps and its source-only
options. A platform with no file here has no import — say so instead of
improvising one.

## The loop

1. **Inventory.** Search the app for every use of the old platform, using the
   patterns in the source file: SDK imports, checkout links, the webhook
   handler, license calls, portal links, env vars. Show the user the list —
   file, line, what it does, what it becomes on Kelviq — before changing
   anything. Env files hold the old platform's keys: use the source file's
   commands as written, which skip env files in the content search and list
   only their variable names. Never print a line or a value from one.
2. **Dry run.** `kelviq:import_start` with `source`, `dryRun: true` and the
   store input the source file names, or `kelviq import <source> --dry-run`.
   It runs the whole import, saves nothing, publishes nothing, and emails a
   report. Tell the user which environment it ran in and where the report is
   going, then wait. Read it with them: the count per record type, the
   totals, and every warning. A dry run previews only the environment it
   ran in.
3. **Real import.** Only after the user has read the dry-run report for
   that environment. Before asking for the go-ahead, tell the user what a
   real run does by default: it publishes the imported plans and switches on
   the imported discount codes, so they are live in that environment as soon
   as it finishes. If they would rather review first, offer
   `publishPlan: false` / `--no-publish-plans` (plans are imported as
   drafts) and `publishDiscount: false` / `--no-publish-discounts` (codes
   stay switched off). Then get approval for this specific run: the
   environment, the store, the steps, and whether plans and codes go live.
   `kelviq:import_start` with `dryRun: false`, or `kelviq import <source>`
   (it restates the run and asks; `--yes` only once the user has approved).
   Rehearse in sandbox first. A production import needs its own dry run in
   production and its own approval; the sandbox report does not cover it.
4. **Review what was imported.** The report has counts and totals but no
   per-plan prices, so list the plans and check each one's price and
   settings with `kelviq:plan_list` and `kelviq:plan_prices_list` (or
   `kelviq status`) against the report and the same item on the old
   platform, with the user. Then read the report's warnings for anything
   that did not go live. A plan the import could not publish stays a draft,
   which can't be bought, and is named in the warnings: the user publishes
   it in the dashboard, or you publish it with `kelviq:plan_publish` once
   the user confirms that plan (see the `pricing-plans` skill). A discount
   code limited to plans that are not published stays switched off, with a
   warning: once those plans are published, run the import again, with the
   user's approval, to switch the code on. `kelviq:discount_list` shows the
   codes. If the run imported plans as drafts on purpose, only
   `kelviq:plan_publish` the ones the user confirms.
5. **Keep the old path alive.** Decide this before writing any Kelviq code.
   Existing subscribers still bill on the old platform, so its webhook
   handler, its webhook secret, and the portal and update-payment links
   those subscribers use all stay, along with any script those links need.
   The app must record which platform each subscription bills on, so that
   portal links and cancellation go to the right one; add that first if it
   is missing. Remove the old code only when the user confirms nothing bills
   there any more.
6. **Add the Kelviq code beside it**, one surface at a time, using the
   source file's mapping. This is for new sales; it adds to the app and
   replaces nothing from step 5. Checkout for new buyers is
   `client.checkout.createSession` (Python:
   `client.checkout.create_session`) or a static checkout URL. The Kelviq
   webhook handler is a new route verified with `validateEvent` (Python:
   `validate_event`), registered with `kelviq webhook add` or
   `kelviq:webhook_endpoint_create` for every event the app's handler uses.
   License calls are `client.license.activate`, `client.license.validate`
   and `client.license.deactivate` on a backend; an app distributed to
   customers calls the license endpoints with a Client API Key, never a
   server key (see the source file). Portal links for Kelviq subscribers
   are `client.portal.createSession` (Python:
   `client.portal.create_session`).
7. **Test with sandbox payments.** Buy a plan end to end in sandbox with a
   test card, and confirm the app grants access from the Kelviq webhook, not
   from the success redirect. `kelviq:webhook_log_list` shows whether the
   event was delivered; `kelviq:subscription_list` shows what was created.

Sales and refunds keep landing on the old platform while the user switches.
Running the import again creates no duplicates: records already imported
are skipped, new ones are added, and refunds made since are picked up. By
default it also publishes plans an earlier run left as drafts and switches
on imported codes that are still off, so if the user kept them back on
purpose, ask before a catch-up run whether to pass `publishPlan: false` /
`publishDiscount: false` again. The default catch-up is the full import
again; the source file says when a shorter run with `only` / `--only` is
enough. A catch-up run is still a real import: it needs the user's approval
each time, but not another dry run.

## What the import does and doesn't do

- **It only queues.** The tool or command returns as soon as the job is
  queued; nothing has been imported at that point. A run takes anything from
  seconds to more than an hour. There is no status to query: the report
  email is the only completion signal. The `kelviq:import_start` result
  carries the `environment` the run was queued in and a `note`: tell the
  user which environment it was, and do what the note says (wait for the
  email; don't call the tool again to check progress).
- **A real run publishes by default.** Each imported plan is published, so
  it can be bought as soon as the app points at it, and imported discount
  codes work at checkout. `publishPlan: false` / `--no-publish-plans`
  imports plans as drafts; `publishDiscount: false` /
  `--no-publish-discounts` leaves codes switched off. Running the import
  again publishes plans an earlier run left as drafts and switches on
  imported codes that are still off. It leaves alone archived plans, plans
  that already have a published version, and codes limited to plans that
  are not published. A dry run publishes nothing and saves nothing.
- **Sandbox and production are separate.** A run touches only the
  environment the Kelviq key belongs to, and a dry run previews only that
  environment.
- **Subscriptions are history.** Imported subscriptions are a record of what
  was sold. They keep billing on the old platform, for as long as it bills
  them, and Kelviq does not charge them. Saved cards are not part of the
  import.
- **Nobody is notified.** Customers are not emailed and no webhooks fire for
  imported records, so the app's new Kelviq handler never sees them.
- **One run at a time, one store per run.** A second request while one is
  running is refused with `409`; its report is already on the way, so wait
  for it instead of retrying.
- **The report goes to the organization owner** unless `reportEmail` /
  `--report-email` says otherwise. Its subject says whether the import
  finished, finished with failures, or failed, and a dry run is marked
  "(dry run, nothing saved)". A failed run keeps what it imported before it
  stopped. It holds a count per record type, the totals charged, tax and
  refunds, up to 50 warnings, and the start time, duration and steps that
  ran. It has no per-plan prices.
- **Sometimes no report comes.** None is sent for a second import started
  while one was already running (only the first reports), or for an import
  that could not start within 24 hours. After 24 hours with no email, run
  it again, with the user's approval if it is a real run. If that returns
  `409` and still nothing arrives, the user should write to hi@kelviq.com
  with the `taskId` from the result; support can clear a run that stopped.
- **Read errors, don't retry blind.** A `400` naming the source platform
  means its key or the store id is wrong. `Configuration error` means the
  source key isn't in the MCP server's environment. For either, tell the
  user where the key is set (see Setup) and to restart; don't ask for the
  value. A `403` has more than one cause, so read its `detail`: "Use a
  server key created by the organization owner." means the owner must
  create the key; "Invalid Bearer Token" or "Authentication credentials
  were not provided." means the Kelviq key is wrong or belongs to the other
  environment, so run `kelviq:connection_check`.

## Agent guidance: a dry run is yours, a real import is the user's

An agent may start a dry run and help read the report: nothing is saved
and nothing is published. Starting a real import — `dryRun: false`, or the
CLI without `--dry-run`, including with `--yes` — is the user's decision
and needs their explicit approval for that specific run. Never chain a dry
run straight into a real import.

- **Say what a real run publishes, and let the user choose.** By default a
  real run makes the imported plans purchasable and the discount codes
  usable the moment it finishes. Tell the user that before asking for
  approval. Never set `publishPlan` / `--no-publish-plans` or
  `publishDiscount` / `--no-publish-discounts` on your own, to `false` or
  to `true`: pass them only as the user decided, and leave them out
  otherwise.

- **Dry run first, per store and per environment.** A store's first import
  into an environment starts with a dry run in that environment. A sandbox
  dry run does not stand in for a production one: a production import needs
  its own dry run and its own approval.
- **A catch-up run is still a real import.** Running the import again later
  needs the user's approval each time. It does not need another dry run.
- **`kelviq:import_start` is annotated as destructive**, so the client may
  ask the user to confirm the call. That prompt does not replace the
  approval above: get it in the conversation first. When the tool returns,
  tell the user which `environment` the run was queued in and follow its
  `note`.
- **The import can't be polled.** After `queued`, stop. Don't loop, sleep,
  or start it again to see whether it finished. Wait for the user to say the
  email arrived, and ask them to paste the counts and warnings.
- **Never overwrite the business profile unprompted.** `overwriteProfile` /
  `--overwrite-profile` replaces the organization name and the business
  profile fields that are already filled in. Never set it unless the user
  asks for exactly that.
- **Leave the report address alone.** Don't set `reportEmail` /
  `--report-email` unless the user names an address: the report carries
  revenue totals, and it goes to the organization owner by default.
- **Keys stay in env files.** Never ask the user to paste the source
  platform's key, or the Kelviq key, into the chat, and don't print an env
  file, or a line from one, to check a key. If the tool says the source key
  is missing or rejected, tell the user where to set it — the same place as
  `KELVIQ_SERVER_API_KEY`: their shell profile or the MCP client's config
  `env` block — and to restart the client. If a key does land in the
  conversation, tell the user to rotate it.
- **Never say subscriptions or saved cards have moved.** After an import the
  honest summary is: history is in Kelviq, existing subscribers still renew
  on the old platform, new sales go through Kelviq once the code is switched.
- **Never present billing on the old platform as open-ended.** Give the
  date from the source file. Lemon Squeezy stops processing payments,
  including renewals, on March 31, 2027; after that, imported subscribers
  are not billed anywhere unless they have subscribed again on Kelviq or
  Kelviq has moved them.
- **Read a `403` before acting on it.** Only "Use a server key created by
  the organization owner." means the owner has to create the key; don't
  work around that one. Any other `403` is a wrong key or the wrong
  environment: run `kelviq:connection_check`.
- **Don't guess at what the import did.** The report and the records in
  Kelviq are the evidence. Where this skill or the source file says to
  check something — an imported customer's id, an imported license key's
  instance id — check it with the user instead of assuming.
- **Deleting old-platform code is step 5's call**, made by the user, not a
  cleanup to do while adding the Kelviq code.

## Docs worth reading first

`kelviq:docs_read` on the migration guide the source file names for what the
import brings over, `cli/commands` for the `kelviq import` flags,
`guides/webhooks` for events and signature verification,
`backend-integration/node-sdk` or `backend-integration/python-sdk` for the
SDK calls, including creating a checkout session and a portal session,
`product-catalog/license-keys` for which key a license call takes, and
`guides/test-payments` for sandbox cards. Read before
answering a field-level question — don't enumerate request or payload shapes
from memory here.
