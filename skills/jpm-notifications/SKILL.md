---
name: jpm-notifications
description: Receive and verify J.P. Morgan Payments webhook events (notifications) in a merchant project. Use when the merchant needs to consume server-to-server event callbacks from JPM — e.g., dispute opened, card updated, payment captured, refund processed — instead of polling REST endpoints. Covers webhook endpoint setup, JPM signing-key fetch, signature verification (EC default, RSA optional), optional mTLS, and CAT smoke test. Companion to `jpm-merchant-integrations`; assumes OAuth is already in place via `jpm-oauth`.
metadata:
  version: 1.0.0
---

# JPM Payments Notifications (Webhooks)

Guide a merchant through receiving JPM webhook events. This is a sibling to `jpm-merchant-integrations` — it handles the inbound webhook path, not request/response REST. APIs that *send* notifications (Disputes, Account Updater card-registration mode, Online Payments capture/refund events, etc.) live in `jpm-merchant-integrations`; this skill handles *receiving* them.

## Step 1 — Confirm prerequisites

The merchant should already have:
- Validated credentials and a working `getAccessToken()` (same auth module as `jpm-merchant-integrations`).
- A way to host a **public HTTPS endpoint** that JPM can POST events to. This is the hard prerequisite — JPM webhooks cannot reach a localhost dev server. Options: a deployed staging environment, a reverse proxy (ngrok / Cloudflare Tunnel) for development, or a dedicated webhook receiver in CAT infra.
- TLS at the endpoint with a publicly trusted certificate (self-signed will not work).

If auth isn't set up yet, point the merchant at `jpm-onboarding-intake` → `jpm-oauth` first and exit.

If invoked standalone, ask:
- Question: "Do you have a public HTTPS endpoint ready to receive webhooks (or a tunnel set up)?"
- Header: "Endpoint ready?"
- Options:
  - "Yes — I have a public HTTPS URL"
  - "Yes — I'll use a tunnel (ngrok / Cloudflare Tunnel) for dev"
  - "No — I need to provision an endpoint first"

If **No**: explain that webhook delivery requires a public endpoint and offer to come back once one exists. Exit.

## Step 2 — Pick options

Ask:
- Question: "Which webhook setup do you want?"
- Header: "Setup"
- Options:
  - "Basic — signature verification only (EC P-256, JPM default)" — recommended starting point.
  - "Signature + RSA" — use RSA-SHA256 instead of EC. Choose if your stack doesn't have an ergonomic EC verifier.
  - "Signature + mTLS" — additionally require JPM to present a client cert. Adds a second layer of authentication. Recommended for high-value event streams (disputes, account updates).
  - "Signature + OAuth-protected webhook" — JPM authenticates to *your* endpoint with a bearer token. Use when you want to reject any inbound POST that doesn't carry a known token.

Carry the choice into Step 3.

Ask which event families the merchant cares about (free text). JPM organizes events into camelCase notification families with subscription-type subtypes — not dotted strings like `payment.captured`. The published families:

- **`paymentUpdateNotification`** (Online Payments) — `PaymentApproved`, `PaymentDeclined`, `PaymentErrored`, `PaymentVoid`, `PaymentClosed`
- **`tokenLifecycleNotification`** (Tokenization) — `CardDetailsUpdate`, `TokenStateChange`, `TokenProvisionUpdate`, `BulkTokenUpdate`
- **`accountUpdateNotification`** (Account Updater) — `AccountUpdaterStatus` (plus Pay-by-Bank subtypes)
- **`consumerProfileNotification`** — `Created`, `PaymentMethodCreated`, `PaymentMethodDeleted`, `BulkConsumerProfileUpdateNotification`
- **`recurringProgramNotification`** — `PlanUpdated`, `PaymentApplied`, `PaymentNotApplied`, etc.
- **`entityOnboardingNotification`**, **`merchantStatusNotification`**, **`payoutNotification`** (onboarding + payouts)

Use `["All"]` to subscribe to every subtype in a family. Full catalog and per-event payload shapes live in `references/webhooks.md`.

**Notes on what this API does NOT cover:**
- **Disputes does not publish events here.** Dispute state changes are discovered by polling `POST /disputes` and `POST /disputes/status-query` in the Disputes API. If the merchant's goal is dispute notifications, redirect them to `jpm-merchant-integrations` → `references/disputes.md`.
- **3-D Secure has no webhook events.** It's a synchronous request/response API.

## Step 3 — Implement the receiver

Follow `references/webhooks.md`. Cross-cutting principles:

- **Reuse the existing auth module.** Some webhook setup calls (registering the endpoint with JPM, fetching public keys) need an OAuth bearer token — get it from the merchant's `getAccessToken()`, not a fresh OAuth client.
- **Fetch JPM's public signing key on startup, cache, refresh on rotation.** Hard-coding the key is brittle; fetching on every request is wasteful. Cache in memory and refresh on a key-id miss.
- **Verify signature before parsing the body.** Read the `Signature`, `Key-ID`, and `Signing algorithm` headers; verify the signature against the raw request bytes using SHA256withECDSA (EC default) or RSA-SHA256 (alternate). Reject unsigned / invalid-signature requests with 401 before any business logic runs.
- **Acknowledge with 2xx within JPM's timeout.** Process asynchronously — push the event to a queue, return 200 immediately. JPM retries on non-2xx; slow handlers cause duplicate deliveries.
- **De-duplicate by `notificationId`.** Retries are normal. Keep a short-TTL store of recently-seen `notificationId`s and skip duplicates. The JPM portal does not document a timestamp header or a recommended replay-skew window — do not invent timestamp-based rejection.
- **Confirm output paths before writing.** Default to a sibling folder of the auth module (`src/webhooks/`) but offer alternatives, same way `jpm-oauth` did.

## Step 4 — CAT smoke test

The full smoke test recipe lives in `references/webhooks.md` under "CAT smoke test." High-level pattern:
1. Register a CAT subscription via `POST /v1/subscriptions` with your `callbackURL`.
2. Fetch and cache the public signing key via `GET /v1/publicKeys`.
3. Trigger an upstream event (e.g., a CAT Online Payments authorization fires `paymentUpdateNotification.PaymentApproved`).
4. Confirm: 2xx response, signature verified, event handed off to your async queue.
5. Force a verification failure (flip a byte in the cached key) and confirm you reject with 401.
6. Re-deliver the same event and confirm de-duplication by `notificationId`.

## Step 5 — Offer to add another

Once one event family is flowing in CAT, ask if the merchant wants to wire additional families or move on to integrating an API that produces them — Online Payments, Tokenization, Account Updater, or Consumer Profile Management — via `jpm-merchant-integrations`. (Note: Disputes does not publish events through this API; dispute state is polled via the Disputes endpoints.)

## Rules

- Never disable signature verification "just to get unblocked." A webhook endpoint without signature verification is an unauthenticated public RCE risk surface. If signature verification is failing, fix the verification — don't bypass it.
- Verify signature **before** parsing the JSON body. The signature is over raw bytes; reparsing and re-serializing changes whitespace and breaks verification.
- Do not block inside the handler. JPM retries on slow responses and you'll get duplicate deliveries. Return 200 fast, process async.
- Do not write CAT webhook URLs into PROD config or vice versa. Endpoint registration is per-environment.
