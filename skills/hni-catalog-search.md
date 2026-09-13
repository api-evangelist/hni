---
name: hni-catalog-search
description: >-
  Search and read the Hearth & Home Technologies (HNI Corporation) product catalog — fireplaces, stoves
  and hearth products — through the storefront's UCP/MCP endpoint.
generated: '2026-09-13'
method: generated
source: mcp/hni-hearthnhome-tools.json (live tools/list, probed 2026-09-13)
api: hni:hearth-home-agent-commerce
endpoint: https://hearthnhome.com/api/ucp/mcp
transport: JSON-RPC 2.0 over HTTP POST
operations:
  - search_catalog
  - lookup_catalog
  - get_product
---

# Search the Hearth & Home Technologies catalog

Read-only catalogue access to HNI Corporation's residential hearth brands (Heatilator, Heat & Glo,
Majestic, Monessen, Quadra-Fire, Harman, SimpliFire, PelPro) on the hearthnhome.com storefront.

## Before you call

Every request carries `meta.ucp-agent.profile` — an absolute https URI pointing at **your** published
UCP agent profile. The server fetches it before dispatching the call. If it is missing you get
`-32001 invalid_profile_url`; if it cannot be fetched you get `-32001 profile_unreachable`.

Tool **discovery** (`tools/list`) is anonymous. Tool **execution** (`tools/call`) additionally requires a
Shopify agent JWT — without one you get `-32000 AuthenticationRequired`. Obtain it per
<https://shopify.dev/docs/agents/get-started/authentication>.

## Steps

1. **Discover** — `GET https://hearthnhome.com/.well-known/ucp` to confirm the store's UCP version and
   which capabilities it serves. Current version is `2026-08-25`.
2. **Search** — call `search_catalog` with `catalog.query` set to the buyer's intent. Pass
   `catalog.context.address_country` (ISO 3166-1 alpha-2) and `catalog.context.currency` (ISO 4217) so
   pricing and availability are correct for the buyer. Narrow with `catalog.filters.categories` (OR
   logic) and `catalog.filters.price.min` / `.max`.
3. **Page** — results are cursor-paginated via `catalog.pagination.cursor` and
   `catalog.pagination.limit` (default 10, minimum 1). No maximum is declared; request conservatively.
4. **Resolve identifiers** — call `lookup_catalog` with `catalog.ids` to fetch several products or
   variants at once. `lookup_catalog` has no pagination block.
5. **Detail** — call `get_product` with `catalog.id`, plus `catalog.selected` to pin a specific variant
   and `catalog.preferences` to express buyer preference.

## Reading prices

Every amount is an **integer in ISO 4217 minor units** paired with a currency code.
`{"amount": 2500, "currency": "USD"}` is **$25.00**. Divide by 100 for two-decimal currencies; do not
divide zero-decimal currencies such as JPY. Never quote the raw integer to a buyer.

## Errors

Transport status stays **HTTP 200 even on failure** — inspect the JSON-RPC `error` member, not the
status code. See `errors/hni-problem-types.yml` for the observed catalogue. `data` is polymorphic: an
object for `-32001`, a bare string for `-32000`.

## Limits

The endpoint is rate-limited per IP; back off on `429`. No limit numbers and no `RateLimit-*` or
`Retry-After` headers are published. Responses do carry an undocumented `shopify-complexity-score`
header you can use to track cumulative cost, and `x-request-id` for correlation when reporting a
problem.
