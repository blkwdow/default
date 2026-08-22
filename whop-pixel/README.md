# Whop Pixel — conversion events on Shopify

Business ID: `biz_RNOKEq1IycxqCA`

## What this tracks

| Moment | Event fired | Where it fires |
| --- | --- | --- |
| Add to cart | `whop.track("add_to_cart", { value, currency })` | Every successful cart add — drawer, quick-add, product page, legacy form submit |

That is the whole event set. One event, one name, fired once per add.

## What this deliberately does not track

**Completed checkout / purchase.** Whop records purchases and subscriptions
server-side from its own payment data. Adding a client-side checkout event
would double-count every sale and corrupt the attribution the pixel exists to
provide. It is also not technically possible from a Shopify theme: Shopify
serves checkout from its own infrastructure and theme code does not run there.
Nothing is lost by leaving it out — the purchase still lands in Whop.

**Page views**, including cart, thank-you, and order-status pages. The base
pixel already records every page view.

## Files

- `whop-add-to-cart.liquid` — the main snippet. Required.
- `whop-buy-it-now-optional.liquid` — optional add-on for the "Buy it now" /
  Shop Pay express button, which bypasses `/cart/add` entirely.

## Installation

1. Shopify admin → **Online Store** → **Themes**.
2. On your live theme, click **⋯** → **Duplicate**. This is your rollback.
3. On the live theme, click **⋯** → **Edit code**.
4. In the left sidebar open **Layout** → **`theme.liquid`**.
5. Press `Ctrl/Cmd + F` inside the editor and search for `</body>`. It is near
   the bottom of the file.
6. Confirm the Whop pixel snippet (the one defining the global `whop` object)
   is already in this file. Note whether it sits in `<head>` or near `</body>`.
7. Paste the entire contents of `whop-add-to-cart.liquid` on the line
   immediately **before** `</body>`.
   - If the Whop pixel snippet is also near `</body>`, paste this **after** it.
   - The snippet queues events for up to 10 seconds while the pixel loads, so
     ordering will not silently drop events — but correct order is cleaner.
8. If your product pages show a **Buy it now** or Shop Pay button, paste
   `whop-buy-it-now-optional.liquid` directly below the first snippet.
9. Click **Save**.

No app, no `theme.liquid` restructuring, no Custom Pixel needed. Shopify's
Customer Events / Custom Pixels sandbox cannot reach `window.whop`, which is
why this goes in the theme instead.

## How it works

Clicking an add-to-cart button does not guarantee an item was added — the
request can fail, and quick-add and upsell widgets never touch the product
form. So instead of binding to one button, the snippet watches the request
Shopify actually makes and fires on success:

1. **`fetch()` interception** — covers Dawn, Refresh, Craft, and effectively
   every Online Store 2.0 theme, plus cart drawers and quick-add buttons.
2. **`XMLHttpRequest` interception** — covers older jQuery-based themes.
3. **Form submit fallback** — covers legacy themes that reload the page.
   Skips itself if the theme called `preventDefault`, since that means an AJAX
   add is already in flight and path 1 or 2 owns the event.

The `value` comes from the real `/cart/add.js` response (`final_line_price`),
so it reflects the actual variant, quantity, and any line discount, converted
from minor units to a decimal. If the response cannot be parsed, the event
still fires without a value rather than not firing.

## Known trade-offs

- **1 second dedupe window.** Two `add_to_cart` events inside 1s collapse into
  one. This prevents double-counting when a theme both submits the form and
  fires its own AJAX call. The cost: a shopper quick-adding two products in
  under a second registers as one add. Tune `DEDUPE_MS` if that matters.
- **Zero-decimal currencies** (JPY, KRW) are divided by 100 like every other
  currency. Check one event's value in the Whop dashboard if you sell in one.
- **Legacy form-submit path** fires as the page begins unloading. Delivery
  depends on the pixel using `sendBeacon`. Modern themes never hit this path.

## Verification

1. Open your live site in a normal browser tab (not the theme preview — the
   preview can behave differently).
2. Open the Whop pixel verification panel on the page.
3. Add a product to the cart. `add_to_cart` should appear once, with a value
   matching the product price.
4. Repeat for each add path your store offers: product page, collection
   quick-add, cart drawer upsell, and "Buy it now" if installed.
5. Walk the rest of the funnel end to end and complete one real purchase. The
   purchase should appear on its own, recorded by Whop server-side — you
   should not see a duplicate checkout event next to it.
