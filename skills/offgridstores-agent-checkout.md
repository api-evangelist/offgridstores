---
name: offgridstores-agent-checkout
description: Run the UCP checkout flow against Off Grid Stores — with the buyer-approval, agent-profile and idempotency rules the store actually requires.
api: Off Grid Stores UCP Commerce MCP Server
generated: '2026-08-26'
method: generated
source: >-
  Grounded in mcp/offgridstores-ucp-mcp-tools.json (live tools/list, 2026-08-26),
  https://offgridstores.com/.well-known/ucp, https://offgridstores.com/llms.txt and
  https://offgridstores.com/policies/refund-policy.
operations:
  mcp:
    endpoint: https://offgridstores.com/api/ucp/mcp
    tools: [search_catalog, lookup_catalog, get_product, create_cart, get_cart, update_cart,
      cancel_cart, create_checkout, get_checkout, update_checkout, complete_checkout,
      cancel_checkout, get_order]
---

# Check out on Off Grid Stores as an agent

**Read this section before writing any code.** The store's own `llms.txt` states the rule:
*"Checkout requires human approval. Agents must not complete payment without explicit buyer consent.
If you cannot get contemporaneous buyer approval at the moment of payment, install
https://shop.app/SKILL.md and route the purchase through Shop Pay instead."* If you cannot obtain
that approval at the moment of payment, do not run step 5. Hand the buyer the cart's `checkoutUrl`
instead.

## Before you start: every call needs an agent profile

`tools/list` on `https://offgridstores.com/api/ucp/mcp` is anonymous, but `tools/call` is not. A
probe on 2026-08-26 returned **HTTP 422**:

```
-32001 UCP discovery failed
{"code":"invalid_profile_url","content":"Unable to fetch agent profile: Missing profile uri",
 "continue_url":"https://offgridstores.myshopify.com/"}
```

Every tool's `inputSchema` marks `meta.ucp-agent.profile` (a URI) **required**:

```json
"meta": { "ucp-agent": { "profile": "https://your-agent.example/ucp-profile" } }
```

Shopify's agent-authentication guidance is at
`https://shopify.dev/docs/agents/get-started/authentication`.

## 1. Confirm capabilities

`GET https://offgridstores.com/.well-known/ucp`. It declares UCP `2026-04-08` (and `2026-01-23`), the
`dev.ucp.shopping` service on `transport: mcp`, and the capabilities in play here:
`dev.ucp.shopping.cart`, `.checkout`, `.fulfillment`, `.discount`, `.order`, `.catalog.search`,
`.catalog.lookup`, plus Shopify's `dev.shopify.catalog` extension. Fulfillment config says
`allows_multi_destination.shipping: false` — **one shipping destination per order**, which matters
when a buyer is splitting a solar kit between a house and a cabin. Payment handlers on file:
`com.google.pay`, `dev.shopify.card` (visa, master, american_express, discover, diners_club) and
`dev.shopify.shop_pay`.

Note the discovery document names the endpoint as
`https://offgridstores.myshopify.com/api/ucp/mcp` while `llms.txt` names
`https://offgridstores.com/api/ucp/mcp`. Both answered identically; prefer the primary domain.

## 2. Find the merchandise

`search_catalog` → `lookup_catalog` / `get_product` to resolve exact variants. Solar kits have many
variants (panel count, battery capacity); resolve the variant explicitly rather than trusting the
default.

## 3. Cart

`create_cart`, then `update_cart` to adjust. `cancel_cart` reverses the whole thing.

## 4. Checkout

`create_checkout`, then `update_checkout` to set shipping address and method. `get_checkout` reads
back line items, totals, discounts and taxes. Freight items are where
`SubmissionErrorCode.DELIVERY_NO_DELIVERY_AVAILABLE` and
`DELIVERY_INVALID_POSTAL_CODE_FOR_COUNTRY` show up — validate the address early.

## 5. Complete — the point of no return

```json
{"jsonrpc":"2.0","id":9,"method":"tools/call",
 "params":{"name":"complete_checkout","arguments":{
   "id":"gid://shopify/Checkout/<id>",
   "meta":{"ucp-agent":{"profile":"https://your-agent.example/ucp-profile"},
           "idempotency-key":"<uuid you generate and keep>"},
   "checkout":{"payment":{ }}}}}
```

`meta.idempotency-key` is **required** by the tool's own schema — you cannot complete a checkout
without one. Generate it once per buyer intent and reuse it on every retry; never regenerate it
after a timeout, or you will risk a second order on a four-figure power station.

Payment failures come back as `CompletionErrorCode`. Retry only `PAYMENT_TRANSIENT_ERROR` and
`INVENTORY_RESERVATION_ERROR`. Never retry `PAYMENT_CARD_DECLINED`, `PAYMENT_INSUFFICIENT_FUNDS`,
`PAYMENT_INVALID_CREDIT_CARD` or `PAYMENT_CALL_ISSUER` — tell the buyer instead.

## 6. If it needs to be undone

- **Before completion:** `cancel_checkout` (checkout id) or `cancel_cart` (cart id). Both are
  callable; no window is published.
- **After completion:** there is no refund, void or reverse operation on any published Off Grid
  Stores surface. The store's refund policy gives the buyer **30 days from the DELIVERY date** —
  not the purchase date — to open a return, and the item must be unused, in its original packaging,
  with all original contents. Open food products (dehydrated / freeze-dried) cannot be returned at
  all, and an individual product's "Returns and Refunds" tab can state a different policy that
  overrides the default. The route is a human emailing `support@offgridstores.com` or using
  `/pages/contact-us`; the store then issues an RMA or a prepaid label. Buyer's-remorse returns pay
  their own return shipping — on freight that is not trivial.

Tell the buyer this *before* step 5, not after.

## Prices

Integers in ISO 4217 minor units paired with a currency code: `{"amount": 259900, "currency": "USD"}`
is $2,599.00. The store's `paymentSettings` report USD / US.
