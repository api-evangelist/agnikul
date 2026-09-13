---
name: agnikul-build-a-merch-cart
description: Assemble a cart at Agnikul Cosmos Store, set the shipping destination, and convert it to a priced checkout.
api: Agnikul Cosmos Store Commerce MCP API
endpoint: https://shop.agnikul.in/api/ucp/mcp
operations:
  - create_cart
  - update_cart
  - get_cart
  - cancel_cart
  - create_checkout
  - get_checkout
  - update_checkout
write: true
---

# Build a cart and a checkout at Agnikul Cosmos Store

This skill covers everything up to but **not including** payment. Completing a purchase is a separate
skill (`agnikul-complete-purchase-safely`) because it is the one irreversible step.

## Create the cart

`create_cart` takes `cart.line_items[]`, where each entry requires `item.id` (a **Product Variant**
GID, not a product GID) and `quantity`. Optional blocks: `buyer`, `context`, `attribution`,
`fulfillment`, `discounts`.

The returned cart identifier looks like `gid://shopify/Cart/{id}?key={secret}`. **That query
parameter is a capability secret — treat the whole string as a bearer credential.** Do not log it,
do not put it in a URL you show a user, and do not hand it to another agent.

## Not idempotent — this matters

`create_cart` and `create_checkout` accept **no idempotency key**. `meta.required` on both is
`["ucp-agent"]` and nothing more. Only `complete_checkout` carries `idempotency-key`.

So: if a `create_cart` or `create_checkout` call times out or errors ambiguously, **do not blind-retry
it.** You will create a second object. Recover instead by cancelling the orphan with `cancel_cart` or
`cancel_checkout`.

## Update

`update_cart` addresses line items by the line item's own `id`. Two semantics worth knowing:

- `discounts.codes[]` is **full replacement** — a submitted array replaces whatever was there. Send
  an empty array to clear all codes.
- Re-issuing an update with the prior values is the reversal for an update. There is no undo tool.

## Fulfillment — one destination, shipping only

The store's UCP profile declares `method_combinations: [["shipping"]]` and an empty
`multi_destination`. That means: shipping only (no pickup, no local delivery) and **one destination
per order**. If your buyer wants two addresses, that is two orders.

## Convert to a checkout

`create_checkout` accepts either `checkout.cart_id` (convert an existing cart) or
`checkout.line_items[]` directly — a cart is not required. It carries `buyer`, `context`,
`attribution`, `fulfillment`, `discounts` and `payment`.

Use `update_checkout` to set the shipping address and method, then `get_checkout` to read back line
items, totals, discounts and taxes before you show the buyer a number. Convert minor units to major
units first.

## Everything here is reversible

`cancel_cart` reverses `create_cart`; `cancel_checkout` reverses `create_checkout`. Neither has a
stated expiry window. A cart and a checkout are real objects with real totals that move no money —
they are the closest thing this store has to a dry run, because there is no test mode at all.

That stops being true the moment you call `complete_checkout`.
