---
name: hni-agent-purchase
description: >-
  Build a cart and complete a purchase of HNI Corporation hearth products through the Hearth & Home
  Technologies UCP/MCP endpoint, with the buyer-approval and irreversibility rules the surface requires.
generated: '2026-09-13'
method: generated
source: mcp/hni-hearthnhome-tools.json (live tools/list, probed 2026-09-13)
api: hni:hearth-home-agent-commerce
endpoint: https://hearthnhome.com/api/ucp/mcp
transport: JSON-RPC 2.0 over HTTP POST
operations:
  - create_cart
  - get_cart
  - update_cart
  - cancel_cart
  - create_checkout
  - get_checkout
  - update_checkout
  - complete_checkout
  - cancel_checkout
  - get_order
---

# Purchase from Hearth & Home Technologies

## Read this first — the purchase is irreversible through this API

There is **no refund, void, return or reverse tool** on this surface. Once `complete_checkout` succeeds,
`get_order` can only read the result; nothing here can undo it. `cancel_cart` and `cancel_checkout` work
only *before* completion, and **no cancellation window is documented**. Treat `complete_checkout` as a
one-way door.

`llms.txt` states the rule plainly: *"Checkout requires human approval. Agents must not complete payment
without explicit buyer consent."* If you cannot get contemporaneous buyer approval at the moment of
payment, do not call `complete_checkout`.

## Every call

Send `meta.ucp-agent.profile` — your published, fetchable agent profile URI. Tool execution also
requires a Shopify agent JWT (<https://shopify.dev/docs/agents/get-started/authentication>); without it
the call returns `-32000 AuthenticationRequired`.

## Steps

1. **Find the product** — use the `hni-catalog-search` skill (`search_catalog`, `lookup_catalog`,
   `get_product`).
2. **Create the cart** — `create_cart` with `cart.line_items`, plus `cart.buyer` and `cart.context`
   (country, currency) for correct pricing. Note the returned cart id.
   *No idempotency key is accepted here* — a retried call creates a second cart. Read back with
   `get_cart` before retrying.
3. **Adjust** — `update_cart` with the cart id to change quantities, `cart.fulfillment` or
   `cart.discounts`. `cancel_cart` abandons it.
4. **Create the checkout** — `create_checkout`, referencing the cart via `checkout.cart_id` and
   supplying `checkout.payment`, `checkout.fulfillment` and `checkout.buyer`. Checkout ids are Shopify
   global ids of the form `gid://shopify/Checkout/abc123`.
5. **Confirm totals with the buyer** — `get_checkout` returns line items, totals, discounts and taxes.
   Convert every amount from ISO 4217 minor units before showing it. Fulfillment supports a single
   destination and shipping-only method combinations.
6. **Get explicit approval.** Show the final total and the payment instrument. Do not proceed on
   inferred consent.
7. **Complete** — `complete_checkout` with the checkout id, `checkout.payment` and
   **`meta.idempotency-key`**. This is the *only* tool on the surface that accepts an idempotency key —
   always send one, because it is the only protection against a double charge on a retry. The response
   carries the order id, a Thank You Page URL, or errors.
8. **Confirm** — `get_order` with the order id to read back what was actually purchased.

## Payment instruments

`checkout.payment.instruments[]` entries need `id`, `handler_id` and `type`. Card instruments require a
`credential` with `token` and `type`. Apple Pay is a special case: `type` is `"card"`, the credential
type is `apple_pay_token` with `payment_data`, and `billing_address` is **required**. Addresses use
`address_country` in ISO 3166-1 alpha-2.

## Failure handling

HTTP status is 200 even when the call fails — always read the JSON-RPC `error` member.
`-32001 invalid_profile_url` / `profile_unreachable` mean your agent profile is missing or unfetchable;
`-32000 AuthenticationRequired` means the JWT is missing or invalid. Back off on `429`. Full catalogue in
`errors/hni-problem-types.yml`.

If a `complete_checkout` call times out or errors ambiguously, **retry with the same
`meta.idempotency-key`** rather than starting a new checkout, then verify with `get_order`.
