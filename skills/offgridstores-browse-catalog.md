---
name: offgridstores-browse-catalog
description: Search the Off Grid Stores catalog, resolve a specific variant, and read the store's policies — all anonymously.
api: Off Grid Stores Storefront MCP Server
generated: '2026-08-26'
method: generated
source: >-
  Grounded in mcp/offgridstores-storefront-mcp-tools.json (live tools/list, 2026-08-26),
  graphql/offgridstores-storefront.graphql (anonymous introspection) and
  https://offgridstores.com/llms.txt.
operations:
  mcp:
    endpoint: https://offgridstores.com/api/mcp
    tools: [search_catalog, get_product_details, get_cart, update_cart, search_shop_policies_and_faqs]
  graphql:
    endpoint: https://offgridstores.com/api/2025-10/graphql.json
    fields: [products, search, predictiveSearch, product, productByHandle, collection, collectionByHandle, collections, shop, paymentSettings]
  json:
    base: https://offgridstores.com
    paths: ['/products.json', '/collections.json', '/products/{handle}.json', '/collections/{handle}/products.json']
---

# Read the Off Grid Stores catalog

Everything in this skill is anonymous. No key, no token, no account. Verified live on 2026-08-26.

## Pick a surface

| You want | Use |
|---|---|
| Natural-language product search | `search_catalog` on `https://offgridstores.com/api/mcp` |
| Exact fields, one round trip | Storefront GraphQL at `https://offgridstores.com/api/2025-10/graphql.json` |
| A bulk dump you can page through | `GET /products.json?limit=250&page=N` |

## MCP search

```json
{"jsonrpc":"2.0","id":1,"method":"tools/call",
 "params":{"name":"search_catalog","arguments":{"catalog":{"query":"3000W inverter"},"meta":{}}}}
```

Returns `gid://shopify/Product/<id>` records. Feed that id to `get_product_details`, passing
`options` to pin a variant and `country`/`language` for localised pricing.

## GraphQL

```graphql
{ products(first: 10, query: "LiFePO4") { edges { node { id title handle
    priceRange { minVariantPrice { amount currencyCode } } } } } }
```

Connections are Relay-style: `first`/`after`/`last`/`before` with `pageInfo` and `edges.cursor`.
Every response carries `extensions.cost.requestedQueryCost` — that is the only budget signal on the
surface; `throttleStatus` is not returned, so you cannot see remaining capacity. Errors come back
with **HTTP 200** and an `errors[]` array. Read the body, never the status code.

## JSON endpoints

`GET /products.json?limit=1&page=2` pages correctly (verified). `/collections/{handle}/products.json`
scopes to a collection. `/sitemap.xml` indexes products, collections, pages and the blog, and
`/sitemap_agentic_discovery.xml` points at `/agents.md`.

## Policies and FAQs

`search_shop_policies_and_faqs` answers from the store's own policy set. The same text is on
`/policies/refund-policy`, `/policies/shipping-policy`, `/policies/terms-of-service`,
`/policies/privacy-policy` and the human FAQ at `/pages/faqs`. Note that `robots.txt` disallows
`/policies/` for crawlers — use the tool or the GraphQL `shop` fields rather than scraping.

## Prices

Integers in ISO 4217 minor units paired with a currency code: `{"amount": 259900, "currency": "USD"}`
is $2,599.00. Divide by 100 before quoting to a buyer. `paymentSettings` reports USD / US.

## Crawl politely

`robots.txt` sets `Crawl-delay: 1` and explicitly **allows** GPTBot, CCBot, Google-Extended and
OAI-SearchBot. It disallows `/admin`, `/cart`, `/checkout`, `/checkouts/`, `/orders`, `/account`,
`/search` and `/policies/`.
