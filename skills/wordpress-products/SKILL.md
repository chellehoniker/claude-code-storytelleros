---
name: wordpress-products
description: Use when the user wants to manage shop products (WooCommerce or FluentCart) on their connected WordPress site, or look at orders, customers, or coupons. Triggers on "add a product to my shop", "update the price on X", "list my recent orders", "create a 20% off coupon", "show me my customers".
---

# WordPress shop products, orders, customers, coupons

Backed by WooCommerce or FluentCart, whichever is installed on the connected site. The plugin auto-detects and the response carries `engine: 'woocommerce' | 'fluentcart'` per row so the same tools work for both.

**Requires the StorytellerOS WordPress plugin.** Application Password connections do not have access — confirm with `stos_wp_site` first; if `pluginVersion` is null, the user needs to install the plugin.

## Products

```js
stos_wp_products_list({ search, page, per_page })
stos_wp_products_get({ id })
stos_wp_products_create({ fields: { engine, name, description, short_description, sku, price, sale_price, stock, status, featured_media } })
stos_wp_products_update({ id, fields: { ... } })
stos_wp_products_delete({ id, force })
```

`engine` is required on `create` only when both WooCommerce AND FluentCart are active. With only one engine installed, the plugin auto-routes.

## Orders

```js
stos_wp_orders_list({ status, customer_id, page, per_page })
stos_wp_orders_get({ id })
stos_wp_orders_update({ id, fields: { status, note } })
```

Order creation typically happens at checkout, not via API. Use `update` to change status (`completed`, `refunded`, `processing`, etc.) or add a note.

## Customers

```js
stos_wp_customers_list({ search, page, per_page })
stos_wp_customers_get({ id })
stos_wp_customers_create({ fields: { email, first_name, last_name, billing_address, shipping_address } })
stos_wp_customers_update({ id, fields: { ... } })
```

Address fields are nested objects: `billing_address: { first_name, last_name, address_1, city, state, postcode, country }`.

## Coupons

```js
stos_wp_coupons_list()
stos_wp_coupons_create({ fields: { engine, code, discount_type, amount, minimum_amount, expires_at } })
stos_wp_coupons_update({ id, fields: { ... } })
stos_wp_coupons_delete({ id })
```

`discount_type`: `percent` | `fixed_cart` | `fixed_product`.

## Patterns

### Create a launch product

```js
const product = await stos_wp_products_create({
  fields: {
    name: "Demon at Dusk — Limited signed paperback",
    description: "<p>Hand-numbered, signed by the author...</p>",
    sku: "DAD-PB-SIGNED-001",
    price: "29.99",
    stock: 100,
    status: "publish",
  }
});
```

### Drop a 20% off launch coupon

```js
stos_wp_coupons_create({
  fields: {
    code: "LAUNCH20",
    discount_type: "percent",
    amount: 20,
    expires_at: "2026-07-01T23:59:59Z",
  }
});
```

### Mark an order refunded

```js
stos_wp_orders_update({ id: 1234, fields: { status: "refunded", note: "Customer requested refund — book damaged in shipping" } });
```

## Anti-patterns

- Do not delete orders. Use status changes (`refunded`, `cancelled`) — orders are financial records.
- Do not change a customer's email without confirming. Most shops use email as the login identity.
- Do not use the products tool for KDP / IngramSpark / Smashwords listings. Those live in **Sales Studio → Products** (the variation table), not on the WP shop.

## Composes well with

- `wordpress-media` — product cover images
