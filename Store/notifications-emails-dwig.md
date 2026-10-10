# Store/Notifications/Emails — Order Notification Emails

The `Templates/Store/Notifications/Emails/` directory holds the order notification emails: new order, paid, updated, cancelled, shipped, delivered, and the default fallback. These templates are chosen in the shop auto-respond mail settings and render per order event with the order's data. They are standalone email documents, not embedded modules: no `<module>` tag places them.

## 1. The email files and their events

```text
Templates/Store/Notifications/Emails/
+-- default.dwig         # fallback for any event
+-- newOrder.dwig        # new_order
+-- orderPaid.dwig       # paid
+-- orderUpdated.dwig   # updated
+-- orderCancelled.dwig  # status_canceled
+-- orderShipped.dwig    # status_shipped, status_delivered
```

Event-to-template defaults: `new_order` renders `newOrder.dwig`, `updated` renders `orderUpdated.dwig`, `paid` renders `orderPaid.dwig`, `status_canceled` renders `orderCancelled.dwig`, `status_shipped` and `status_delivered` render `orderShipped.dwig`, anything else renders `default.dwig`. Each event can be repointed to a different file and switched off in the auto-respond settings; only enabled events send.

Template discovery reads two directories: the legacy `Store/Notfication/Email` path and the active theme's `Store/Notifications/Emails`. Empty placeholder files in the theme folder are ignored so working templates remain selectable. A non-empty themed file wins over same-named files elsewhere. Keep a working `default.dwig` so every event has a fallback.

## 2. What data each email receives

Top-level variables (not nested under `data.*`):

| Value | Contents |
|---|---|
| Order fields, flattened | The order record as individual variables: `id`, `order_invoice_id`, `created_at`, `first_name`, `last_name`, `email`, `address` (plus city/state/zip/phone per installation), `payment_status`, `amount`, `shipping`, `taxes_amount`, `currency`. Money fields arrive preformatted to two decimals. Guard each with `\|default()`. |
| `order` | The same order array under one name for grouped reads (`order.id`, `order.amount`). |
| `order_id` | Order ID shorthand. |
| `event` | Event key that fired this email (`new_order`, `paid`, `status_shipped`, and similar). Branch copy on it when one file serves several events. |
| `status_label` | Human-readable order status for the current event. |
| `cart_items` | Order lines with `price` and `item_total` preformatted. Always loop with `\|default([])`. |

Recipients: the customer address for valid emails on every event, plus all admin addresses for new orders per the send setting. Subjects are set by the sender ("New order #id", "Payment received for order #id", and status variants) — skins own the body only.

## 3. Email document rules

Standalone table documents with inline styles, absolute URLs, and no scripts — the same discipline as user account mail:

```twig
<table role="presentation" width="100%" cellpadding="0" cellspacing="0">
<tr><td align="center">
<table role="presentation" width="600" cellpadding="24" cellspacing="0">
    <tr><td>
        <h1>Order #{{ order_id|default('')|e }} confirmed</h1>
        <p>Hi {{ first_name|default('there')|e }}, your order is {{ status_label|default('received')|e }}.</p>
        <table role="presentation" width="100%">
        {% for item in cart_items|default([]) %}
            <tr>
                <td>{{ item.title|default(item.name|default('Item'))|e }} × {{ item.qty|default(1) }}</td>
                <td align="right">{{ currency_format(item.item_total|default(0), currency|default(false)) }}</td>
            </tr>
        {% endfor %}
        </table>
        <p><strong>Total: {{ currency_format(amount|default(0), currency|default(false)) }}</strong></p>
    </td></tr>
</table>
</td></tr>
</table>
```

1. Never extend a layout or depend on theme CSS. Email clients render this file alone; keep one `<style>` block with inline-friendly rules and a safe font stack.
2. Absolute URLs everywhere through `site_url()` and resolved links. Relative paths break outside the browser.
3. No JavaScript, no forms. The email carries information and at most an account/order link.
4. Take the payable total from the order `amount`, never a skin-side sum of lines. Line math may display per-row totals; the charged figure is the order's own.
5. Escape names, addresses, titles, and statuses. Order data is customer and merchant input; `|raw` has no place in notification mail.
6. An empty render sends nothing. A template that outputs blank for an event silently drops that notification — keep every file rendering real content for its events.

## 4. Per-event copy

| Event | Subject pattern | Body leads with |
|---|---|---|
| `new_order` | New order #id | Confirmation, what happens next, account link. |
| `paid` | Payment received for order #id | Receipt: amount charged, invoice reference. |
| `updated` | Order #id updated | What changed, in plain words. |
| `status_canceled` | Order #id: Cancelled | Cancellation confirmation plus refund note where applicable. |
| `status_shipped` / `status_delivered` | Order #id: Shipped / Delivered | Tracking or delivery confirmation with address check. |

Branch shared files on `event` when one skin serves several statuses; prefer one file per status when copy genuinely differs.

## 5. Common mistakes

- Extending the site layout or depending on theme CSS inside email skins.
- Summing line totals in the skin for the charged figure instead of the order `amount`.
- Hand-formatted money or missing currency instead of `currency_format()` with the order currency.
- Relative links and images that die outside the browser.
- Scripts, forms, or interactive widgets that clients strip.
- Unescaped customer and merchant values.
- Blank output for an event, silently dropping its notification.
- Disabling an event in settings while its template is the only copy of that message, with no fallback plan.
- No working `default.dwig`, so unmapped events fail instead of falling back.
