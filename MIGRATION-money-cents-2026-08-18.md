# Money Conventions Migration — 2026-08-18

**Status:** migrated on pilot (`10.0.2.7`), shipped on branch `fix/codebase-cleanup` (API repo) — staging/prod via CI on merge.

## TL;DR

Every money value in the API is an **integer in minor units (cents)**. JSON keys are named `*_cents`. There are no dollar floats on the wire anymore. The server does all arithmetic (fees, taxes, totals); the client only displays.

Before this migration the API mixed four conventions: dollar floats (via a cast), decimal-dollar columns, raw cents under dollar-named keys, and formatted strings used as data. All of that is gone.

## The five contract rules

1. **Naming** — every money JSON field ends in `_cents` (`price_cents`, `total_amount_cents`, `amount_cents`). Percentage fields are the only money-adjacent exceptions and never carry `_cents`: `fee_percentage`, `platform_fee_percentage`, `refund_percentage`, `priceChargedAs`-style POI fields.
2. **Type** — money is always `type: integer` (JSON number, 64-bit safe: max ~9.2e18). No floats, no strings.
3. **Math is server-side** — the client never computes fees or totals. The single fee implementation is `PriceCalculatorService::calculatePlatformFee()` (platform fee default 15%, `PLATFORM_FEE_PERCENTAGE`).
4. **Display strings** — `*_formatted` fields (e.g. `total_formatted: "1234.56"`) are **leaf display strings** for emails/PDFs/pre-rendered UIs. Never parse them, never send them back, never format from them.
5. **Stripe boundary** — the API passes integer cents straight to Stripe. Checkout/payment-intent payloads you see (`{clientSecret, amount_cents, currency}`) already are Stripe-ready.

### Displaying money in the frontend

```ts
new Intl.NumberFormat(locale, { style: "currency", currency: "USD" })
  .format(price_cents / 100)
```

The division by 100 lives **only in the view layer** — it is formatting, not data.

## Affected endpoints

Legend — **R** = request field change, **S** = response shape change, **B** = behavior change.

### Catalog & pricing (creator tools)

| Endpoint | Change |
|---|---|
| `POST /api/influencer/masterclasses` | **R** `price_cents: integer ≥ 0` (was `price`, float dollars) |
| `PUT /api/influencer/masterclasses/{id}` | **R** `price_cents: integer ≥ 0`, nullable |
| `POST /api/influencer/media-assets` | **R** `price_cents: integer ≥ 0` (required) |
| `PUT /api/influencer/media-assets/{id}` | **R** `price_cents: integer ≥ 0` (required) |
| `POST /api/influencer/blog` | **R** `price_cents: integer ≥ 0`, nullable |
| `PUT /api/influencer/blog/{id}` | **R** `price_cents: integer ≥ 0`, nullable. Setting `is_premium: false` clears price server-side |
| `POST /api/influencer/podcast` | **R** `price_cents: integer ≥ 0`, nullable |
| `PUT /api/influencer/podcast/{id}` | **R** `price_cents: integer ≥ 0`, nullable |
| `PUT /api/influencer/consultation` | **R** `price_cents: integer ≥ 0` (required) |
| `POST /api/maps` | **R** `price_cents: integer ≥ 0`, nullable (int when `is_for_sale`) |
| `PUT /api/maps/{map}` | **R** `price_cents: integer ≥ 0`, nullable |

**B** — all money inputs above are now validated as `integer`. Sending `49.99` (float) or `"49.99"` (string) is a **422**, previously it would round/truncate.

### Public reads (catalog)

Response `price` field renamed to `price_cents` (integer) on:

- `GET /api/influencers/{username}/masterclasses` / `…/masterclasses/{slug}`
- `GET /api/influencers/{username}/media-assets` / `…/media-assets/{id}`
- `GET /api/influencers/{username}/blog` / `…/blog/{slug}`
- `GET /api/influencers/{username}/podcast` / `…/podcast/{slug}`
- `GET /api/influencers/{username}/consultation` (+ `GET /api/influencers/{username}` profile)
- `GET /api/maps`, `GET /api/maps/{map}`, `GET /api/influencers/{username}/maps`

Schemas: `MasterclassPublic`, `MasterclassFull`, `MediaAsset`, `BlogPostPublic`, `BlogPostFull`, `PodcastEpisodePublic`, `PodcastEpisodeFull`, `Consultation`, `MapResource`.

### Cart

| Endpoint | Change |
|---|---|
| `GET /api/cart` | **S** `total_cents` (int) + `total_formatted` (display string) |
| `GET /api/cart/items`, `POST /api/cart/items`, `PUT /api/cart/items/{id}` | **S** `price_cents` (int), `price_formatted`, `total_price_cents` (int), `total_price_formatted`; nested `item.price_cents` (int) |

### Checkout & orders

| Endpoint | Change |
|---|---|
| `POST /api/orders`, `GET /api/orders`, `GET /api/orders/purchased`, `GET /api/orders/sold`, `GET /api/orders/{id}`, `POST /api/orders/{id}/cancel` | **S** `subtotal_cents`, `tax_amount_cents`, `discount_amount_cents`, `total_amount_cents`, `platform_fee_amount_cents`, `influencer_payout_amount_cents` — all integers — plus matching `*_formatted` strings; line items `unit_price_cents` / `total_price_cents` |
| `POST /api/orders/{id}/refund` | **R** partial refund field is `amount_cents` (integer, `min: 1`); omit for full refund. **B** sending the legacy `amount` field is now **422** (`prohibited`). **S** response `data.amount_refunded_cents` (integer) — was float dollars |
| `POST /api/payments/confirm` | **S** order + payment in cents (schemas `Order`, `Payment`) |
| `POST /api/payments/intent` | unchanged (returns `redirect_url` + `checkout_session_id` only) |

### Invoices

| Endpoint | Change |
|---|---|
| `GET /api/invoices`, `…/paid`, `…/received`, `GET /api/invoices/{id}` | **S** schema `Invoice`: `subtotal_cents`, `tax_amount_cents`, `discount_amount_cents`, `total_amount_cents`, `platform_fee_amount_cents`, `influencer_payout_amount_cents` (ints) + `*_formatted` |
| `GET /api/invoices/summary` | **S** restructured: `paid: {count, total_cents, total_formatted}`, `received: {count, total_cents, total_formatted}`, `status_breakdown: [{status, count, total_cents, total_formatted}]` |
| `GET /api/admin/invoices`, `GET /api/admin/invoices/{id}` | **S** list rows use admin wire keys `subtotal_cents`, `tax_cents`, `total_cents`, `amount_paid_cents`; detail adds `amount_due_cents`; line items `unit_price_cents` / `total_cents` |
| `GET /api/admin/invoices/export` | **B** CSV headers renamed: `tax_cents` → `tax_amount_cents`, `total_cents` → `total_amount_cents` |

### Purchases (single-item checkout)

| Endpoint | Change |
|---|---|
| `POST /api/media-assets/{id}/purchase` | **S** `data.paymentIntent: {clientSecret, amount_cents (int), currency}`; on completed purchase `data.purchase` (schema `MediaAssetPurchase`, no money fields) |
| `GET /api/media-assets/my-purchases` | unchanged shape (no money fields on `MediaAssetPurchaseResource`) |
| `POST /api/influencers/{username}/blog/{slug}/purchase` | **S** purchase rows `amount_cents` (int) + `currency` (schema `BlogPostPurchase`) |
| `GET /api/user/blog-purchases` | **S** same schema |
| `POST /api/influencers/{username}/podcast/{slug}/purchase` | **S** `amount_cents` (int) + `currency` (schema `PodcastEpisodePurchase`) |
| `GET /api/user/podcast-purchases` | **S** same schema |
| `POST /api/consultations/{id}/book` | **S** `data.paymentIntent: {clientSecret, amount_cents (int), currency}` |
| `POST /api/consultations/{id}/book-and-pay` | unchanged (redirect flow, no money fields) |
| `POST /api/consultations/bookings/{bookingId}/payment-session` | unchanged (redirect flow) |
| `GET /api/consultations/bookings/{bookingId}/cancellation-policy` | **S** `refund_amount_cents` (int) + `refund_amount_formatted` (display string) |
| `POST /api/consultations/bookings/{bookingId}/cancel` | **S** refund object `amount_cents` (int) + `amount_formatted` |

### Maps (paid)

| Endpoint | Change |
|---|---|
| purchases list/detail (via `MapPurchase` schema) | **S** `unitPrice` / `totalPrice` keys kept for compatibility but are now **integer cents** (docs previously said float dollars); nested `map.price_cents` (int, was string `"29.99"`) |

### Payouts & earnings

| Endpoint | Change |
|---|---|
| `GET /api/influencer/payouts`, `GET /api/influencer/payouts/{id}` | **S** `amount_cents` (int) — was float dollars |
| `GET /api/influencer/earnings/summary` | **S** keys renamed/typed: `total_earnings_cents`, `total_payouts_cents`, `pending_payouts_cents`, `platform_fees_paid_cents` (all ints) |
| `GET /api/admin/payouts`, `GET /api/admin/payouts/{id}` | **S** `amount_cents` (int). **B** sort param: `sort=amount` → `sort=amount_cents` |
| `GET /api/admin/platform/earnings`, `…/summary` | **S** `amount_cents` (int) + `amount_formatted`; summary `total_earnings_cents` / `reversed_earnings_cents` / `net_earnings_cents` (ints) + `*_formatted` |
| `GET /api/admin/stats` | **S** `revenue.this_month/last_month/all_time.{total_cents, platform_fees_cents, influencer_payouts_cents}` (ints). The `revenue` section key itself is unchanged |
| `GET /api/admin/stats/trends` | **S** `revenue[].amount_cents` (int) |
| `GET /api/admin/orders`, `GET /api/admin/orders/{id}` | **S** admin wire keys (pre-existing names, values now guaranteed integer cents): `total_cents`, `platform_fee_cents`, `influencer_payout_cents`, items `unit_price_cents`/`total_cents` |
| `GET /api/admin/payments`, `GET /api/admin/payments/{id}` | **S** `amount_cents`, `platform_fee_cents`, `influencer_payout_cents` (ints) |

### MCP (admin)

`POST|GET /api/mcp` — tool arguments for content creation use `price_cents` (integer ≥ 0); blog/map update tools validate `price_cents`. POI tools keep `estimatedMinPrice` / `estimatedMaxPrice` as decimal local-currency values (see "unchanged").

### Sort query parameters (behavior)

These accepted values were renamed — old values are silently ignored (no sort applied):

| Endpoint | Old | New |
|---|---|---|
| `GET /api/admin/masterclasses` | `sort=price` | `sort=price_cents` |
| `GET /api/admin/media-assets` | `sort=price` | `sort=price_cents` |
| `GET /api/admin/payments` | `sort=amount` | `sort=amount_cents` |
| `GET /api/admin/payouts` | `sort=amount` | `sort=amount_cents` |
| `GET /api/admin/invoices` | `sort=total_amount` | `sort=total_amount_cents` |

## What did NOT change

- **POI pricing** — `PoiPricing.estimatedMinPrice` / `estimatedMaxPrice` remain `number` (float). They are *estimated, local-currency* note values, not platform money. Same for `priceChargedAs`, `priceNotes`.
- **`*_formatted` fields** — display strings, format `10.50`. Present on `Order`, `OrderItem`, `Invoice`, `Payment`, `PlatformEarning`, cart, payouts, consultation cancellation payloads. Treat as opaque.
- **Percentage fields** — `fee_percentage`, `platform_fee_percentage`, `refund_percentage`: `DECIMAL(5,2)` / float in JSON (e.g. `15.00`).
- **Admin short keys** — `total_cents`, `platform_fee_cents`, `influencer_payout_cents`, `tax_cents`, `amount_paid_cents`, `revenue` (section): pre-existing admin-wire names, kept for compatibility. They already end in `_cents` where they are amounts.
- **Cart `total_cents`** — pre-existing name, unchanged.
- **`refund_percentage` in cancellation-policy** — still a percentage number.

## Database migrations (ran on pilot)

| # | Migration | Effect |
|---|---|---|
| 1 | `convert_media_assets_price_to_cents` | `decimal` dollars → `bigint` cents (×100) |
| 2 | `convert_maps_price_to_cents` | nullable `decimal` → nullable `bigint` |
| 3 | `convert_content_purchases_amount_to_cents` | `decimal` → `bigint` |
| 4 | `normalize_masterclass_purchases_purchase_price` | type fix only (values already cents) |
| 5 | `repair_map_purchases_unit_price` | repairs dollar-valued rows from order totals (logs before/after sums) |
| 6 | `rename_money_columns_to_cents` | pure renames across 14 tables (`orders`, `order_items`, `cart_items`, `payments`, `payment_refunds`, `invoices`, `platform_earnings`, `influencer_payouts`, `payouts`, `map_purchases`, `masterclasses`, `consultations`, `blog_posts`, `podcast_episodes`) |
| 7 | `add_amount_cents_to_media_asset_purchases` | records the paid amount on the purchase row |

Migrations are additive/renaming — no destructive table rebuilds. Staging/prod apply them automatically via CI on deploy.

## How to detect a money field (cheat sheet)

| JSON key shape | Example | Kind |
|---|---|---|
| `*_cents` | `price_cents: 4999` | money — integer cents |
| `*_formatted` | `total_formatted: "49.99"` | display string — don't parse |
| `*_percentage` | `platform_fee_percentage: 15.00` | percentage — not money |
| `estimatedMin/MaxPrice` | `estimatedMinPrice: 12.5` | POI local-currency estimate — not platform money |
| `currency` | `"USD"` | ISO-4217 code, travels with money |

## References

- API commits: `fix/codebase-cleanup` — `ba73151` (refactor), `34e4398` (S6 refund revocation)
- OpenAPI commits: `a8096a0` (contract rename), `cd548fc` (alignment fixes)
- Governance rules: `AGENTS.md` → **Money (STRICT)** (API repo)
- Fee formula: `app/Services/PriceCalculatorService.php::calculatePlatformFee()`
- Verification: full feature suite green (money tests assert exact stored cent values via `getRawOriginal()`)
