---
name: boAt Lifestyle agent commerce — search, cart, checkout
description: >-
  Buy a boAt Lifestyle product on a buyer's behalf over the store's Universal
  Commerce Protocol MCP endpoint: discover capabilities, search the catalog,
  build a cart, open a checkout, and complete it only after explicit human
  approval of payment.
api: mcp/boat-lifestyle-mcp.yml
transport: MCP over HTTP JSON-RPC 2.0
endpoint: https://www.boat-lifestyle.com/api/ucp/mcp
operations:
  - search_catalog
  - lookup_catalog
  - get_product
  - create_cart
  - update_cart
  - get_cart
  - cancel_cart
  - create_checkout
  - update_checkout
  - get_checkout
  - complete_checkout
  - cancel_checkout
  - get_order
generated: '2026-08-08'
method: generated
source: mcp/boat-lifestyle-tools-list.json + https://www.boat-lifestyle.com/agents.md
---

# boAt Lifestyle agent commerce

boAt Lifestyle's storefront serves a Universal Commerce Protocol (UCP `2026-04-08`)
shopping service over MCP. Every tool name below was read from a live anonymous
`tools/list` against `https://www.boat-lifestyle.com/api/ucp/mcp`; the verbatim
manifest with full input schemas is in `mcp/boat-lifestyle-tools-list.json`.

## Before you call anything

1. `GET https://www.boat-lifestyle.com/.well-known/ucp` and confirm the store still
   advertises `dev.ucp.shopping` at protocol version `2026-04-08`. Capabilities carry a
   `requires.protocol.min` floor — do not assume a capability that is not listed.
2. Read `https://www.boat-lifestyle.com/agents.md`. It is the provider's own
   instruction document and it is authoritative over this skill.

## Invariants — these are not optional

- **Every tool call requires `meta.ucp-agent.profile`**, a URI identifying the agent
  profile you are acting under. It is in `required` on all 13 tools. A call without it
  is malformed.
- **`complete_checkout` requires `meta.idempotency-key`.** It is the only mutating
  operation on the surface that can double-charge a buyer. Generate one key per
  logical purchase attempt and reuse it verbatim on every retry.
- **Checkout is for humans.** `robots.txt` and `agents.md` both state that payment and
  order placement must not be finalized without an explicit, contemporaneous human
  approval step. Do not script an end-to-end purchase. If you cannot obtain approval
  at the moment of payment, stop and hand the buyer the Shop Pay route instead.
- **Back off on `429`.** The endpoint is rate-limited per IP and publishes no numeric
  quota.

## Flow

### 1. Find the product

Call `search_catalog` with `catalog.query` and a `catalog.context` carrying
`address_country` (ISO 3166-1 alpha-2), `currency` (ISO 4217) and `language`
(BCP 47). Without context the store cannot price or gate availability correctly.

Narrow with `catalog.filters.categories`, `catalog.filters.price` and
`catalog.filters.available`. Page with `catalog.pagination.cursor` and
`catalog.pagination.limit` — the cursor is opaque, pass it back unmodified.

Use `lookup_catalog` when you already hold identifiers and need several products
resolved at once, and `get_product` for full detail on one product, passing
`catalog.selected[]` name/label pairs to pin a variant.

### 2. Build the cart

`create_cart` with `cart.line_items[]` (`item` plus `quantity`) and the same
`cart.context`. Add `cart.buyer.email` / `cart.buyer.phone_number` when the buyer has
supplied them. Apply promotions through `cart.discounts.codes[]` — codes are
case-insensitive and each submission **replaces** the previous set, so always send the
full list you want applied.

`update_cart` and `get_cart` take the cart `id` (a Shopify GID, e.g.
`gid://shopify/Cart/...`). `cancel_cart` discards it.

### 3. Open the checkout

`create_checkout`, passing `checkout.cart_id` to carry the cart over rather than
re-listing items. Set `checkout.fulfillment.methods[]` — this store allows shipping
only, to a single destination, with no method combinations, per the
`dev.ucp.shopping.fulfillment` config in `/.well-known/ucp`.

Set `checkout.payment.instruments[]` against a payment handler the store actually
declares: Google Pay (`gpay`, VISA/Mastercard/Amex/Discover) or the Shopify card
handler (`shopify.card`, which additionally accepts Diners Club).

Populate `checkout.attribution` (`utm_*`, `referring_domain`, click ids) when the
session carries it — attribution is a first-class field on this contract.

Re-read totals, taxes and discounts with `get_checkout` before showing the buyer a
price. Revise with `update_checkout`.

### 4. Complete — with the human in the loop

Present the final `get_checkout` totals to the buyer and obtain explicit approval.
Only then call `complete_checkout` with the checkout `id` and
`meta.idempotency-key`. On a timeout or transport error, retry with the **same** key;
never mint a new one for the same purchase.

`cancel_checkout` abandons an open checkout. `get_order` retrieves the resulting order
by id.

## What this surface does not give you

There is no OpenAPI, no webhook or event surface, and no published error-code
registry — failures come back as JSON-RPC 2.0 error objects. Buyer-scoped history
beyond `get_order` requires an authenticated token from Shopify Customer Accounts
(`customer-account-api:full` / `customer-account-mcp-api:full`); see
`authentication/boat-lifestyle-authentication.yml` and
`scopes/boat-lifestyle-scopes.yml`.
