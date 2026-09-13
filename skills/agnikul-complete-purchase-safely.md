---
name: agnikul-complete-purchase-safely
description: Complete payment at Agnikul Cosmos Store with buyer approval and an idempotency key, then confirm the order — and know what cannot be undone.
api: Agnikul Cosmos Store Commerce MCP API
endpoint: https://shop.agnikul.in/api/ucp/mcp
operations:
  - update_checkout
  - complete_checkout
  - cancel_checkout
  - get_order
write: true
---

# Complete a purchase safely at Agnikul Cosmos Store

This is the irreversible step. Read the whole file before calling `complete_checkout`.

## The buyer-approval rule is not advisory

The store states it twice, in `/agents.md` and again inside its own `robots.txt`:

> Checkouts are for humans. Do NOT complete checkout, payment, or order placement automatically — no
> scripted form fills, browser automation, or end-to-end agent flows that finalize payment without an
> explicit, contemporaneous human approval step.

"Contemporaneous" is the operative word. Approval collected earlier in the session, or a standing
instruction to "buy whatever you think is best", does not satisfy it. Get a yes on this specific
total, at this moment, then call.

If you cannot get that, the store's own instructions say to route the purchase through Shopify's Shop
skill (`https://shop.app/SKILL.md`) instead. Note the inconsistency, though: the store's UCP profile
declares only two payment handlers — `com.google.pay` and `dev.shopify.card` — and does **not**
advertise `dev.shopify.shop_pay`.

## Use the idempotency key

`complete_checkout` is the only tool in this contract whose `meta.required` is
`["ucp-agent","idempotency-key"]`. Generate one key per purchase intent, reuse it on any retry of
*that same intent*, and never reuse it across intents.

This is the one write operation with replay protection. `create_cart`, `create_checkout`,
`update_cart` and `update_checkout` have none.

## Success is not the transport status

`complete_checkout` "returns details about the completed checkout, including order ID, Thank You Page
URL, **or any errors encountered**." A 200 with a well-formed JSON-RPC result can still carry a
failure. Inspect the result payload before telling the buyer anything. The specific error vocabulary
is not published, so check for the presence of an order ID rather than for the absence of an error.

Then call `get_order` with the returned `gid://shopify/Order/{id}` to confirm what was actually
placed.

## There is no undo — and no published return policy

`cancel_checkout` works only **before** completion. After `complete_checkout` succeeds there is no
refund, void, reverse or cancel tool anywhere in the contract.

And Cosmos Store publishes **no policies at all**. As of 2026-09-12,
`/policies/refund-policy`, `/policies/shipping-policy`, `/policies/terms-of-service` and
`/policies/privacy-policy` every one returns HTTP 404. Neither `agents.md` nor `llms.txt` states a
return window.

So:

- **Never tell a buyer a completed purchase can be cancelled, returned or refunded.** You have no
  basis for it.
- **Never invent a return window.** There isn't one published to quote.
- If the buyer asks about returns before buying, tell them the store publishes no return policy and
  point them at `curious@agnikul.in`. There is no store-specific support address.

## Trace and pacing

Every response carries `x-request-id` (`<uuid>-<epoch>`). Keep it; quote it if something goes wrong —
while knowing there is no developer support channel to quote it to.

The endpoint returns `shopify-complexity-score` (a cost signal) and **no** budget or remaining-quota
header of any kind. `agents.md` says only: "The MCP endpoint is rate-limited per IP. Back off on 429
responses." No `Retry-After` is documented or observed, so back off exponentially and blind.
