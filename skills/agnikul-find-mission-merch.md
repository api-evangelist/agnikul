---
name: agnikul-find-mission-merch
description: Search the Agnikul Cosmos Store catalog and present accurate, correctly-denominated options to a buyer.
api: Agnikul Cosmos Store Commerce MCP API
endpoint: https://shop.agnikul.in/api/ucp/mcp
operations:
  - search_catalog
  - lookup_catalog
  - get_product
write: false
---

# Find Agnikul Cosmos Store merchandise

Read this first: **this store sells merchandise, not launches.** Agnikul Cosmos builds orbital
rockets, and none of that business is reachable through any API. If your user is asking about
launching a satellite, stop here and point them at `https://agnikul.in/book/` and
`payloadpeople@agnikul.in`. This skill covers t-shirts, hoodies, caps, mugs, keychains and mission
patches — eleven products, priced in **INR** from 99 to 1,499.

## Connect

POST JSON-RPC 2.0 to `https://shop.agnikul.in/api/ucp/mcp`. No credential is required. Send
`Content-Type: application/json` and `Accept: application/json, text/event-stream`.

Every tool requires a `meta` object, and `meta.required` is `["ucp-agent"]` — you must identify
yourself with a UCP agent profile URI. A call without `meta.ucp-agent` does not satisfy the server's
own schema.

## Search

Call `search_catalog`. At least one of `catalog.query` or `catalog.filters` must be provided — there
is no "return everything" mode. Available filters are categories, a price range (`min`/`max`, in
minor units) and `available` (defaults to true, meaning sale-ready items only).

Pass `catalog.context.address_country` and `catalog.context.currency` when you know them. The store's
own instructions say to: "Pass `context.address_country` and `context.currency` for accurate pricing
and availability."

Results are cursor-paginated with a default limit of 10. Take `pagination.cursor` from the response
and send it back as `catalog.pagination.cursor` for the next page. With eleven products in the whole
catalog you will rarely need a second page.

## Resolve and detail

- `lookup_catalog` resolves a batch of product or variant GIDs in one request. Product IDs come back
  marked `featured`; variant IDs come back as exact matches.
- `get_product` returns one product with its variants, exact pricing and real-time availability, and
  supports interactive option selection via `selected` and `preferences` — use it when the buyer is
  choosing a size or colour.

## Quote prices correctly

Every price in this contract is an **integer in ISO 4217 minor units paired with a currency code**:
`{"amount": 69900, "currency": "INR"}` is ₹699.00. Divide by 100 before you say a number to a buyer.
Never quote the raw integer.

## Two key spaces — do not mix them

The MCP tools address products by Shopify GID (`gid://shopify/Product/...`,
`gid://shopify/ProductVariant/...`). The public storefront JSON routes
(`https://shop.agnikul.in/products/{handle}.json`) address them by URL handle. Nothing Agnikul
publishes maps one to the other. Stay inside whichever space you started in.

## What is not here

No response schemas are published for any entity, so validate defensively. There is no test mode or
sandbox — every call is against the live store.
