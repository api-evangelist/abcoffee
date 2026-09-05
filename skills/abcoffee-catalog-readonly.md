---
name: Read the abcoffee catalog without transacting
description: >-
  Browse abcoffee product, collection and price data using only anonymous, read-only calls — the MCP
  catalog tools and the storefront JSON endpoints — with no cart, checkout or credential involved.
api: mcp/abcoffee-mcp.yml
endpoint: https://abcoffee.in/api/ucp/mcp
operations:
  - search_catalog
  - lookup_catalog
  - get_product
generated: '2026-09-05'
method: generated
source: >-
  Grounded in the live tools/list response saved at mcp/abcoffee-mcp-tools.json and the read-only
  browsing section of https://abcoffee.in/llms.txt.
---

# Read the abcoffee catalog without transacting

Use this when the buyer only wants to know what abcoffee sells and what it costs.

## MCP route (structured, UCP-shaped)

`POST https://abcoffee.in/api/ucp/mcp`, `Content-Type: application/json`,
`Accept: application/json, text/event-stream`. Include `meta["ucp-agent"].profile` on every call.

- `search_catalog` — `catalog.query` (natural language) and/or `catalog.filters`
  (`categories[]`, `price.min`/`price.max`, `available`). Page with `pagination.cursor`.
- `lookup_catalog` — `catalog.ids[]` to resolve known product or variant identifiers in one call.
- `get_product` — `catalog.id` for the full product record, with `catalog.selected[]`
  (`name` + `label`) to pin a variant.

Pass `catalog.context` (`address_country`, `currency`, `language`, `postal_code`) so prices and
availability match the buyer's market. Prices come back as integers in ISO 4217 minor units.

## Plain-HTTP route (no MCP client needed)

The store's own agent instructions publish these unauthenticated paths:

- `GET https://abcoffee.in/products.json` — all products
- `GET https://abcoffee.in/products/{handle}.json` — one product
- `GET https://abcoffee.in/collections/{handle}/products.json` — a collection
- `GET https://abcoffee.in/search?q={query}&type=product`
- `GET https://abcoffee.in/sitemap.xml`

## Do not

- Do not call `create_cart`, `create_checkout` or `complete_checkout` from this skill. Reading the
  catalog requires none of them.
- Do not call `get_order` — it requires a Shopify agent JWT and returns `-32000`
  `AuthenticationRequired` without one.
- Back off on `429`; the endpoint is rate limited per IP.
