---
name: Search the abcoffee catalog and place an order
description: >-
  Find products in the abcoffee online store and take a buyer from catalog search through cart and
  checkout to a completed, buyer-approved order over the store's UCP/MCP endpoint.
api: mcp/abcoffee-mcp.yml
endpoint: https://abcoffee.in/api/ucp/mcp
operations:
  - search_catalog
  - get_product
  - create_cart
  - update_cart
  - create_checkout
  - update_checkout
  - complete_checkout
  - cancel_checkout
generated: '2026-09-05'
method: generated
source: >-
  Grounded in the live tools/list response saved at mcp/abcoffee-mcp-tools.json and the published agent
  instructions at https://abcoffee.in/agents.md. Every operation name above was read from that response.
---

# Search the abcoffee catalog and place an order

abcoffee's store speaks the Universal Commerce Protocol over MCP at
`https://abcoffee.in/api/ucp/mcp`. Discovery lives at `https://abcoffee.in/.well-known/ucp`;
the store's own agent instructions are at `https://abcoffee.in/agents.md`.

## Before you start

- **Every call needs an agent identity.** Pass `meta["ucp-agent"].profile` — a URI resolving to your
  UCP agent profile — on every tool call. Omitting it returns JSON-RPC `-32001`
  (`invalid_profile_url`, HTTP 422), not a validation error about the tool's own arguments.
- **No API key is required** for catalog, cart and checkout. `tools/list` and `initialize` answer
  anonymously.
- **Money is in minor units.** `{"amount": 600, "currency": "INR"}` is ₹6.00. Divide by 100 before
  quoting any price to the buyer.
- **Pass buyer context.** Send `context.address_country` and `context.currency` so pricing and
  availability are correct; this store trades in India.

## Steps

1. **Confirm capabilities.** `GET https://abcoffee.in/.well-known/ucp` and check
   `ucp.version` (currently `2026-08-25`) and that the capabilities you need
   (`dev.ucp.shopping.cart`, `.checkout`, `.catalog.search`) are listed.
2. **Search.** Call `search_catalog` with `catalog.query` and/or `catalog.filters`
   (`categories`, `price.min`/`price.max`, `available`). At least one of query or filters is required.
   Results are paginated — take `pagination.cursor` from the response and send it back to page.
3. **Inspect.** Call `get_product` with `catalog.id` for full detail on a candidate, or
   `lookup_catalog` with `catalog.ids` to resolve several identifiers at once.
4. **Build a cart.** Call `create_cart` with `cart.line_items[]` (`item.id` + `quantity`), the buyer's
   `context`, and `buyer.email` / `buyer.phone_number`. Adjust with `update_cart`; read it back with
   `get_cart`.
5. **Start checkout.** Call `create_checkout`, passing `checkout.cart_id` to carry the cart forward.
6. **Set fulfillment.** Call `update_checkout` with `checkout.fulfillment.methods[]` and any
   `checkout.discounts.codes[]`. Read totals back with `get_checkout` before showing the buyer a price.
7. **Get approval, then commit.** Present the final total to the buyer and obtain explicit consent.
   Call `complete_checkout` with the payment instrument and — **required** —
   `meta["idempotency-key"]`. Reuse the same key on any retry of the same completion.
8. **Confirm.** `get_order` returns the order, but it requires a Shopify agent JWT; without one it
   returns JSON-RPC `-32000` `AuthenticationRequired` (HTTP 403). See
   https://shopify.dev/docs/agents/get-started/authentication.

## Rules you must not break

- **Never complete payment without contemporaneous buyer approval.** The store's own instructions say
  so explicitly. If you cannot get approval at the moment of payment, route the purchase through
  Shop Pay via the Shop skill (`https://shop.app/SKILL.md`) instead of calling `complete_checkout`.
- **Only `complete_checkout` is idempotent.** It is the one tool declaring
  `meta["idempotency-key"]` as required. `create_cart`, `update_cart`, `create_checkout` and
  `update_checkout` declare no idempotency key — a retried call there may create a duplicate.
- **Back off on 429.** The endpoint is rate limited per IP and publishes no numeric limit and no
  `Retry-After` header; the status code is your only signal.

## Undoing things

- Before completion: `cancel_checkout` (checkout id) and `cancel_cart` (cart id) reverse the flow
  cleanly at any point.
- After completion: there is **no MCP tool to cancel an order**. abcoffee's published refund policy
  gives the buyer roughly **one minute** after placing an order before the store may charge a 100%
  cancellation fee — https://abcoffee.in/policies/refund-policy. Tell the buyer this before you
  commit, not after.
