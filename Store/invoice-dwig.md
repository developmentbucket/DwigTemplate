# Store/Invoice Page Template — Printable Order Invoices

The `Templates/Store/Invoice/` template renders the printable order invoice: seller header, bill-to block, line items, totals, and optional tax sections. It is a standalone document selected by the order flow, not an embedded module: no `<module>` tag places it, and it renders separately from the website frontend for screen, print, and HTML-to-PDF output.

## 1. Template shape and selection

```twig
<!doctype html>
<html>
<head>
    <meta charset="utf-8">
    <title>Invoice #{{ order_details.order_invoice_id|default(order_details.id) }}</title>
    <style>
        body { color: #222; font-family: DejaVu Sans, sans-serif; font-size: 12px; line-height: 1.45; }
        ...
    </style>
</head>
<body>
    {# ...header, bill-to, items, totals... #}
</body>
</html>
```

- Provide the full document (`doctype`, `html`, `head`, `body`). Never extend a site layout — invoice rendering happens outside the frontend, where no layout exists.
- Keep CSS self-contained in one `<style>` block with print-friendly rules (`@page` size and margins, `border-collapse` tables). External stylesheets and web fonts do not survive PDF rendering; DejaVu Sans is the safe base font.
- Invoice selection order: the filename saved in the `invoice_template` shop option first, `default.dwig` second, the first available invoice template third, legacy stored template last. Keep a working `default.dwig` so selection never fails into the error state.

## 2. Where invoice templates live and how they resolve

```text
Templates/Store/Invoice/
+-- default.dwig     # fallback, keep it working
+-- tax-invoice.dwig # tax/GST variant
+-- ...
```

Resolution is theme-first, bundled-default-second per filename. Keep theme-specific invoices in the active theme.

## 3. What data the invoice receives

Top-level variables (not nested under `data.*`):

| Value | Contents |
|---|---|
| `order_details` | Order record: `id`, `order_invoice_id`, `created_at`, `first_name`, `last_name`, `email`, `address` (plus city/state/zip/phone per installation), `payment_status`, `amount`, `currency`. Guard each with `\|default()`. |
| `products` | Order lines: `title` (or `name`), `qty`, `price`, plus `subtotal_without_tax`, `tax_amount`, `subtotal_with_tax` where tax accounting applies. |

Helpers used on invoices:

| Helper | Purpose |
|---|---|
| `currency_format(value, currency)` | Money with the order's own currency: `currency_format(order_details.amount, order_details.currency)`. Never hand-format amounts. |
| `get_option('website_title', 'website')` | Seller name line. Never hardcode the shop name. |
| `barcode(value, type, color, height, width)` | Code 128 barcode as a base64 data URI (for example the SKU or invoice ID) for scanning on print. |

## 4. Document anatomy

Seller header, bill-to block, line-item table, totals, tax section where applicable:

```twig
<table class="header">
    <tr>
        <td>
            <h1>Invoice</h1>
            <div>#{{ order_details.order_invoice_id|default(order_details.id) }}</div>
        </td>
        <td class="text-right">
            <strong>{{ get_option('website_title', 'website')|e }}</strong><br>
            {{ order_details.created_at|default('')|e }}
        </td>
    </tr>
</table>

<table class="summary">
    <tr>
        <td>
            <strong>Bill to</strong><br>
            {{ order_details.first_name|default('')|e }} {{ order_details.last_name|default('')|e }}<br>
            {{ order_details.email|default('')|e }}<br>
            {{ order_details.address|default('')|e }}
        </td>
        <td class="text-right">
            <strong>Order #{{ order_details.id|e }}</strong><br>
            {{ order_details.payment_status|default('')|e }}
        </td>
    </tr>
</table>

<table class="items">
    <thead><tr><th>Item</th><th>Quantity</th><th>Price</th><th>Total</th></tr></thead>
    <tbody>
    {% for product in products|default([]) %}
        <tr>
            <td>{{ product.title|default(product.name|default('Item'))|e }}</td>
            <td>{{ product.qty|default(1) }}</td>
            <td>{{ currency_format(product.price|default(0), order_details.currency|default(false)) }}</td>
            <td>{{ currency_format(product.price|default(0) * product.qty|default(1), order_details.currency|default(false)) }}</td>
        </tr>
    {% endfor %}
    </tbody>
</table>

<div class="total">Total: {{ currency_format(order_details.amount|default(0), order_details.currency|default(false)) }}</div>
```

Rules:

1. Identify the document by invoice number with `id` fallback, and show the order ID plus payment status in the summary. An invoice without reference numbers is unusable for support.
2. Render buyer identity (name, email, address) guarded per field. Missing fields degrade to blank lines, never to broken rows.
3. Title lines from `title` with `name` fallback and an `'Item'` last resort. Never leave a row untitled.
4. Take the grand total from `order_details.amount` with the order currency — never sum lines in the skin for the payable total. Line math may show per-row totals; the payable figure is the order's own.
5. Escape names, addresses, and titles. Order data is customer and merchant input; `|raw` has no place in an invoice.
6. Tax variants (CGST/SGST versus IGST and similar) compute from per-line `subtotal_without_tax`, `tax_amount`, and `subtotal_with_tax` with guarded defaults. Tax logic follows the installation's accounting rules; the skin presents the computed values, it does not define rates.

## 5. Use cases

**Standard invoice.** Header, bill-to, items, total. The default skin for order confirmation and records.

**Tax invoice.** Company registration block, per-line tax columns, jurisdiction-branched tax rows, barcode of the invoice ID. Separate skin, same data plus tax fields.

**Packing slip.** Items with quantities and buyer address only — no prices, no payment status. Fulfillment surface, not a bill.

**Proforma.** Same document marked clearly as estimate (not a tax invoice) with validity note. Wording carries the legal difference.

## 6. Common mistakes

- Extending the site layout or depending on theme CSS inside an invoice. Invoices render standalone for print and PDF.
- Summing line totals in the skin for the payable figure instead of `order_details.amount`.
- Hand-formatted money or missing currency instead of `currency_format()` with the order currency.
- Untitled rows, missing reference numbers, or unguarded buyer fields.
- Unescaped customer and merchant values.
- Web fonts or external assets that vanish in PDF rendering.
- Defining tax rates in the skin instead of presenting the installation's computed tax values.
- No working `default.dwig`, so invoice selection fails when the configured file is missing.
