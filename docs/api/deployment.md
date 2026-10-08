# Storefront API deployment and route troubleshooting

This guide covers deploying the Laravel storefront API under:

```text
https://astroemerging.com/gehna/api/v1
```

## What the current source already contains

The API routes are registered in `routes/api.php` under the `v1` prefix. The
file currently defines the documented storefront routes, including:

- Eight authenticated checkout routes: summary, coupon and coin actions, order
  placement, Razorpay verification, and payment status.
- Authenticated cart, order, and wishlist routes.
- `POST /api/v1/cart/combo`, protected by Sanctum.
- Public `POST /api/v1/webhooks/razorpay`.

The webhook handler validates the `X-Razorpay-Signature` header against the raw
request body using the configured Razorpay webhook secret. Keep that endpoint
public so Razorpay can call it, but do not remove its signature validation.

The root storefront route and `/home` route previously shared the name `home`,
which prevented Laravel from creating a route cache. `/home` now has the
distinct name `home.alias` in `routes/web.php`; the root route retains the
`home` name.

## Deploy the application code

The route definitions and the route-name correction must be deployed to the
same Laravel application that serves `/gehna`. Clearing a route cache on a
developer computer does not update production.

Use the project's normal deployment process (Git deployment, release
deployment, or hosting file manager). Ensure these files are part of the
deployed release:

- `routes/api.php`
- `routes/web.php`
- `app/Http/Controllers/Api/CheckoutController.php`
- `app/Http/Controllers/Api/CartController.php`
- `app/Http/Controllers/Frontend/FrontendCheckoutController.php`
- `bootstrap/app.php`

Do not overwrite the production `.env`, customer uploads, or other persistent
production data when deploying.

## Clear and rebuild Laravel caches on production

Connect to production using SSH or the hosting provider's Terminal. Change to
the Laravel application directory for the `/gehna` site—the directory that
contains the `artisan` file—and run:

```bash
pwd
php artisan optimize:clear
php artisan route:cache
```

`route:cache` must report that the routes were cached successfully. If it
reports a duplicate route name such as `home`, the corrected `routes/web.php`
has not been deployed to this app directory, or another route still has a
duplicate name. Do not treat a failed cache command as a successful rollout.

## Verify the production route table

Still in the same production application directory, run:

```bash
php artisan route:list --path=api/v1
php artisan route:list --path=api/v1/checkout -v
php artisan route:list --path=api/v1/cart/combo -v
php artisan route:list --path=api/v1/webhooks/razorpay -v
```

The full API listing should show all 52 documented routes. Specifically:

- All eight `api.checkout.*` routes must be present and have
  `Authenticate:sanctum` middleware.
- `api.cart.combo.store` must be `POST api/v1/cart/combo` and have
  `Authenticate:sanctum` middleware.
- `api.webhooks.razorpay` must be `POST api/v1/webhooks/razorpay`. It must not
  require a customer bearer token; webhook authenticity is checked by its
  Razorpay signature.

If these routes are missing after `optimize:clear`, verify that the deployed
`routes/api.php` is current and that Artisan is being run from the right
application directory.

## Verify the live URLs

Run these checks from any machine with `curl`:

```bash
curl -i -H "Accept: application/json" \
  https://astroemerging.com/gehna/api/v1/products

curl -i -H "Accept: application/json" \
  https://astroemerging.com/gehna/api/v1/checkout/summary

curl -i -X POST -H "Accept: application/json" \
  -H "Content-Type: application/json" -d '{}' \
  https://astroemerging.com/gehna/api/v1/cart/combo

curl -i -X POST -H "Accept: application/json" \
  -H "Content-Type: application/json" -d '{}' \
  https://astroemerging.com/gehna/api/v1/webhooks/razorpay
```

Expected results:

| Request | Expected without credentials |
| --- | --- |
| `GET /products` | `200` with the public product-list response |
| `GET /checkout/summary` | `401 Unauthenticated.` rather than `404` |
| `POST /cart/combo` | `401 Unauthenticated.` rather than `405` |
| `POST /webhooks/razorpay` | Rejected by the webhook handler for missing/invalid signature; not `404` |

A webhook request with an invalid or absent signature should not be accepted.
Do not use a real valid payment event just to test route registration.

## If the server route list is correct but live URLs still fail

If production's `route:list` includes the routes with the expected methods, but
the public URLs still return `404` or `405`, the web server may be forwarding
requests to a different application directory or release.

Ask the hosting administrator to confirm:

1. The `astroemerging.com/gehna` virtual host or subdirectory points at the
   deployed Laravel application's `public` directory.
2. Requests under `/gehna/api/v1/...` reach that Laravel app's front
   controller (`public/index.php`), rather than an older installation or a
   static-file handler.
3. Rewrite rules preserve the `/gehna` mount path and pass the Authorization
   header through to PHP.
4. PHP-FPM/opcache or any reverse proxy is not serving stale application code;
   reload the relevant service/cache if required by the host.

Do not change the frontend API base URL simply because checkout is missing:
if another route on that same base, such as `/products`, works, the base path
is already reaching an API application.

## Checkout authentication and request summary

Send a valid Sanctum token on protected requests:

```http
Authorization: Bearer <sanctum-token>
Accept: application/json
Content-Type: application/json
```

`POST /checkout/place-order` requires `payment_method` (`cod` or `razorpay`)
and these billing fields: `billing_name`, `billing_email`, `billing_phone`,
`billing_line1`, `billing_city`, `billing_state`, `billing_zip`, and
`billing_country`. Use the `checkout_token` returned by
`GET /checkout/summary` when retry safety is needed. Optional fields include
`coupon_code`, `coins`, `shipping_same_as_billing`, and the `shipping_*`
address fields.

Example request body:

```json
{
  "payment_method": "cod",
  "checkout_token": "9eb39c4c-28a2-49d0-9cec-2217fc185221",
  "billing_name": "Jane Doe",
  "billing_email": "jane@example.com",
  "billing_phone": "9876543210",
  "billing_line1": "12 Museum Road",
  "billing_city": "Bengaluru",
  "billing_state": "Karnataka",
  "billing_zip": "560001",
  "billing_country": "India",
  "shipping_same_as_billing": true
}
```

A successful order placement returns `201` with the created order. COD orders
are unpaid at placement (`payment_status: "pending"`). Razorpay orders return
a `data.razorpay` object for the payment popup. After the popup, send
`order_id`, `razorpay_order_id`, `razorpay_payment_id`, and
`razorpay_signature` to `POST /checkout/verify-razorpay`. A valid signature
returns `200` with `payment_status: "paid"`; a mismatch returns `422`.

For complete response examples and all endpoint schemas, see
[`API.md`](../../API.md).

## End-to-end production checkout

Only after route registration and smoke checks pass, test the complete checkout
using an authorized test customer and an approved payment test setup:

1. Authenticate and retain the returned Sanctum token.
2. Add an available product to that account's cart.
3. Fetch `/checkout/summary` and retain its `checkout_token`.
4. Place a COD test order, or create a Razorpay test order and complete the
   provider's test payment flow.
5. For Razorpay, verify the returned popup signature at
   `/checkout/verify-razorpay`.
6. Check `/checkout/payment-status/{order}` with the same token.
7. Confirm the order belongs to the test account and has the expected payment
   state. Avoid real customer data and real charges during testing.
