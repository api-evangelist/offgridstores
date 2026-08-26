---
name: offgridstores-build-cart
description: Build and revise an Off Grid Stores cart, apply codes, and undo any cart-level change — anonymously.
api: Off Grid Stores Storefront MCP Server
generated: '2026-08-26'
method: generated
source: >-
  Grounded in mcp/offgridstores-storefront-mcp-tools.json, mcp/offgridstores-ucp-mcp-tools.json and
  graphql/offgridstores-storefront.graphql, all captured live on 2026-08-26.
operations:
  mcp:
    endpoint: https://offgridstores.com/api/mcp
    tools: [get_cart, update_cart]
  ucp_mcp:
    endpoint: https://offgridstores.com/api/ucp/mcp
    tools: [create_cart, get_cart, update_cart, cancel_cart]
  graphql:
    endpoint: https://offgridstores.com/api/2025-10/graphql.json
    fields: [cart, cartCreate, cartLinesAdd, cartLinesUpdate, cartLinesRemove, cartBuyerIdentityUpdate,
      cartDeliveryAddressesAdd, cartDeliveryAddressesReplace, cartDeliveryAddressesUpdate,
      cartSelectedDeliveryOptionsUpdate, cartDiscountCodesUpdate, cartGiftCardCodesAdd,
      cartGiftCardCodesRemove, cartNoteUpdate, cartPrepareForCompletion]
---

# Build a cart at Off Grid Stores

Two routes reach the same cart object.

- `https://offgridstores.com/api/mcp` — **anonymous end to end**, tools/list and tools/call both.
  This is the route to use unless you need the UCP checkout flow.
- `https://offgridstores.com/api/ucp/mcp` — needs `meta.ucp-agent.profile` on every call (see the
  checkout skill).

## One tool does everything

`update_cart` is a composite. Its schema exposes, in a single call: `add_items`, `update_items`,
`remove_line_ids`, `buyer_identity`, `delivery_addresses_to_add`, `delivery_addresses_to_replace`,
`selected_delivery_options`, `discount_codes`, `gift_card_codes` and `note`. On the GraphQL side
that same call fans out to a dozen separate `cart*` mutations.

```json
{"jsonrpc":"2.0","id":2,"method":"tools/call",
 "params":{"name":"update_cart","arguments":{
   "cart_id":"gid://shopify/Cart/<id>",
   "add_items":[{"product_variant_id":"gid://shopify/ProductVariant/<id>","quantity":1}],
   "discount_codes":["<code>"]}}}
```

## Undoing things

Cart writes are all reversible while the cart is open:

| Change | Undo |
|---|---|
| Added a line | `update_cart` with `remove_line_ids` (GraphQL: `cartLinesRemove`) |
| Changed a quantity | `update_cart` with `update_items` (GraphQL: `cartLinesUpdate`) |
| Applied a discount code | `update_cart` with an empty `discount_codes` (GraphQL: `cartDiscountCodesUpdate`) |
| Applied a gift card | GraphQL `cartGiftCardCodesRemove` |
| The whole cart | `cancel_cart` on the UCP server |

There is **no** cart-cancel mutation in GraphQL — `cancel_cart` is MCP-only. No window is published
for any of these; treat them as available while the cart exists.

## Errors

Cart mutations return typed `userErrors[]` drawn from `CartErrorCode` (55 values). The ones you will
actually hit on this catalogue are delivery-shaped, because much of the inventory is heavy freight:
`INVALID_DELIVERY_OPTION`, `PENDING_DELIVERY_GROUPS`, `INVALID_ZIP_CODE_FOR_COUNTRY`,
`ZIP_CODE_NOT_SUPPORTED`, `PROVINCE_NOT_FOUND`, `MINIMUM_NOT_MET`, `MAXIMUM_EXCEEDED`. Full list in
`errors/offgridstores-problem-types.yml`.

## Handing off to a human

`Cart.checkoutUrl` gives the buyer a normal browser checkout. If you cannot get contemporaneous
approval for payment, this is the correct exit — see the checkout skill.
