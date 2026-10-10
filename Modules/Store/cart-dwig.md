# Store/Cart — Cart Table, Drawer, and Live Updates

The `Store/Cart` module renders the shopping cart: line items with quantities, totals, coupon, and the checkout link. Its skins live in `Templates/Modules/Store/Cart/`. The backend loads the visitor's cart lines with prices and totals; the skin renders the table and updates quantities, removals, and coupons through `window.dbEvent.shop.cart` without a page reload.

## 1. Embedding a cart

```twig
<module
    type="Store/Cart"
    id="cart-page"
    template="default.dwig"
/>
```

- `type` is `Store/Cart`: the path-style name that mirrors the skin path `Templates/Modules/Store/Cart/`. The bare legacy name still resolves, but every example in these docs uses the path-style form.
- `id` identifies this instance and connects it to its settings (checkout link, checkout page). Keep it stable and unique per cart surface. Cart lines themselves belong to the visitor, not the instance.
- `template` selects the skin filename from the cart directory. Pass only the filename, never a path. A template chosen in Live Edit is saved against the instance and takes precedence over the inline attribute.
- An unavailable file falls back to `default.dwig`, so the directory keeps a working `default.dwig`.

Checkout-link attributes:

| Attribute | Purpose |
|---|---|
| `checkout-link-enabled` | Set to `"n"` to hide the checkout button (read-only summaries, confirmation views). Any other value shows it. |
| `data-checkout-page` | Checkout page ID. Falls back to the default checkout page. |

## 2. Where skins live and how they resolve

```text
Templates/Modules/Store/Cart/
+-- default.dwig       # fallback: full cart table
+-- cart-summary.dwig  # compact checkout summary with coupon
+-- mini.dwig          # drawer / header mini cart
+-- ...
```

Resolution order for every skin:

1. The same relative path in the active theme.
2. The bundled default template.
3. `default.dwig` in the same directory.

Keep theme-specific cart skins in the active theme. The full table, the drawer, and the checkout summary are different skins over the same data — never one skin with conditionals per surface.

## 3. What data a cart skin receives

| Value | Contents |
|---|---|
| `data.items` | Cart lines. Each carries `id` (line ID for updates), `rel_id` (product ID), `title`, `price`, `qty`, `item_image`, and `custom_fields` (selected variant HTML). Always loop with `\|default([])` and keep an empty-cart branch. |
| `data.total` | Cart total, preformatted for display via `currency_format()`. |
| `data.cart_totals` | Totals breakdown (subtotal, discounts, total values) for summary skins. |
| `data.checkout_page_link` | Resolved checkout URL, or `false` when the link is disabled. |
| `data.checkout_link_enabled` | Whether to render the checkout button. |
| `data.template_css_prefix` | Skin-name CSS scope class. Keep it on the wrapper. |
| `data.params` | Tag attributes plus the instance ID (`data.params.id`). |

The safe pattern for any cart:

```twig
<div class="mw-cart mw-cart-{{ data.params.id|default('cart')|e('html_attr') }} {{ data.template_css_prefix|default('')|e('html_attr') }}">
{% if data.items|default([]) is not empty %}
    {# ...lines and totals... #}
{% else %}
    <div class="alert alert-info mb-0">Your cart is empty.</div>
{% endif %}
</div>
```

Auto-escaping is off in `.dwig` templates: escape titles with `|e` and IDs and attributes with `|e('html_attr')`. `item.custom_fields` is backend-rendered variant HTML and renders with `|raw`; everything else stays escaped.

## 4. The cart table pattern

One row per line: image, title with variant detail, quantity input, line total, remove control:

```twig
{% for item in data.items %}
<tr class="mw-cart-item mw-cart-item-{{ item.id|e('html_attr') }}">
    <td>
        {% set item_image = item.item_image|default('') ?: get_picture(item.rel_id|default(0)) %}
        {% if item_image %}
            <img src="{{ thumbnail(item_image, 80, 80)|e('html_attr') }}" alt="{{ item.title|default('')|e('html_attr') }}">
        {% endif %}
        <strong>{{ item.title|default('')|e }}</strong>
        {% if item.custom_fields|default('') %}
            <div class="small text-muted">{{ item.custom_fields|raw }}</div>
        {% endif %}
    </td>
    <td>
        <input type="number" min="1" value="{{ item.qty|default(1) }}"
               aria-label="Quantity for {{ item.title|default('')|e('html_attr') }}"
               data-cart-qty data-item-id="{{ item.id|e('html_attr') }}">
    </td>
    <td>{{ currency_format(item.price|default(0) * item.qty|default(1)) }}</td>
    <td><button type="button" data-cart-remove data-item-id="{{ item.id|e('html_attr') }}"
                aria-label="Remove {{ item.title|default('')|e('html_attr') }}">Remove</button></td>
</tr>
{% endfor %}
<strong>Total: {{ currency_format(data.total|default(0)) }}</strong>
{% if data.checkout_link_enabled and data.checkout_page_link %}
    <a class="btn btn-primary" href="{{ data.checkout_page_link|e('html_attr') }}">Checkout</a>
{% endif %}
```

Rules:

1. Address lines by `item.id` (the cart line ID), never by product ID. Two lines can share a product with different variants; the line ID is the only correct update key.
2. Resolve images as `item.item_image` first, `get_picture(item.rel_id)` second, guarded. Pass through `thumbnail()` at the displayed size.
3. Format money only with `currency_format()`. Never concatenate symbols or multiply strings by hand outside the documented line-total pattern.
4. Render the checkout button only when `checkout_link_enabled` is true and `checkout_page_link` is non-empty, linking the resolved URL — never a hardcoded checkout path.

## 5. Updating the cart with AJAX via common.js

All cart mutations go through `window.dbEvent.shop.cart` from `userfiles/modules/developmentbucket/db_lib/events/common.js`, which loads automatically on normal frontend pages. Never reference the `mw` JS library. One delegated handler covers quantity changes and removals; totals refresh from the totals API; the header badge updates itself:

```html
<script>
(function () {
    const root = document.getElementById('cart-page-controls');
    if (!root || !window.dbEvent || !window.dbEvent.shop) {
        return;
    }
    if (root.dataset.initialized === 'true') {
        return;
    }
    root.dataset.initialized = 'true';

    const message = root.querySelector('[data-cart-message]');

    async function refreshTotals() {
        const response = await window.dbEvent.shop.cart.totals();
        const totals = (response && response.data && response.data.totals) || {};
        if (totals.total) {
            const totalElement = root.querySelector('[data-cart-total]');
            if (totalElement) {
                totalElement.textContent = totals.total.formatted || totals.total.value;
            }
        }
    }

    async function setQuantity(itemId, qty, row) {
        qty = Math.max(1, Number(qty) || 1);
        row.classList.add('is-updating');
        try {
            await window.dbEvent.shop.cart.update({ id: itemId, qty: qty });
            const input = row.querySelector('[data-cart-qty]');
            if (input) {
                input.value = qty;
            }
            const lineTotal = row.querySelector('[data-cart-line-total]');
            const unitPrice = Number(row.getAttribute('data-item-price')) || 0;
            if (lineTotal) {
                lineTotal.textContent = new Intl.NumberFormat('en-IN', {
                    style: 'currency', currency: 'INR'
                }).format(unitPrice * qty);
            }
            await refreshTotals();
        } catch (error) {
            message.textContent = (error && error.message) || 'Unable to update your cart. Please try again.';
        } finally {
            row.classList.remove('is-updating');
        }
    }

    async function removeItem(itemId, row) {
        row.classList.add('is-updating');
        try {
            await window.dbEvent.shop.cart.remove({ id: itemId });
            row.remove();
            if (!root.querySelector('[data-cart-row]')) {
                window.dbEvent.module.load({
                    module: 'shop/cart',
                    id: root.getAttribute('data-module-id'),
                    template: 'default.dwig',
                    target: '#' + root.id,
                    replace: false
                }).catch(function () { return null; });
            }
            await refreshTotals();
        } catch (error) {
            message.textContent = (error && error.message) || 'Unable to remove this item. Please try again.';
        } finally {
            const live = root.querySelector('.is-updating');
            if (live) {
                live.classList.remove('is-updating');
            }
        }
    }

    root.addEventListener('change', function (event) {
        const input = event.target.closest('[data-cart-qty]');
        if (!input || !root.contains(input)) {
            return;
        }
        const row = input.closest('[data-cart-row]');
        setQuantity(input.getAttribute('data-item-id'), input.value, row);
    });

    root.addEventListener('click', function (event) {
        const button = event.target.closest('[data-cart-remove]');
        if (!button || !root.contains(button)) {
            return;
        }
        event.preventDefault();
        const row = button.closest('[data-cart-row]');
        removeItem(button.getAttribute('data-item-id'), row);
    });
})();
</script>
```

Rules for live updates:

1. Call `update({ id, qty })` for quantity and `remove({ id })` for removal, keyed by cart line ID. Clamp quantities to a minimum of 1 client-side; zero-quantity updates are removals — route them to `remove`.
2. Every call uses `try/catch` because failures reject. Show the message in a `role="alert"` region near the cart and restore the row state in `finally`.
3. Mark the acting row `is-updating` (dimmed, pointer-events off) while its request flies, and clear it after. Never lock the whole cart for one row.
4. Refresh totals from `dbEvent.shop.cart.totals()` after every mutation; update line totals locally from the row's unit price. Never recompute tax, discount, or shipping in the skin.
5. When the last row leaves, reload the module (`dbEvent.module.load` with the same instance ID and skin) so the empty-cart branch renders instead of a blank table.
6. Guard the binder with an initialized flag on the unique wrapper ID, so AJAX re-renders never stack duplicate handlers.
7. The header badge (class `js-shopping-cart-quantity`) refreshes automatically after each mutation; the skin syncs nothing by hand.

## 6. Drawer and summary variants

**Mini drawer.** The `mini.dwig` skin reuses `data.items`, `data.total`, and the same `update`/`remove` calls in compact rows. It lives once in the layout shell and opens from any drawer trigger. Keep its hooks identical to the table's so one script shape serves both.

**Checkout summary.** Read-only rows with a coupon form: quantities display as text, removal stays available, totals come from `data.cart_totals` with discount rows revealed only when discounts exist. Coupon calls go through `dbEvent.shop.cart.coupon.add({ coupon_code })` with applying/success/error messaging, and totals refresh after every coupon change.

## 7. Use cases

**Cart page table.** Full `default.dwig` table with quantities, coupon-adjacent totals, and the checkout button. The pre-checkout workhorse.

**Header mini drawer.** Compact rows with quantity display and remove controls, plus subtotal and a view-cart link. Same data, tighter chrome.

**Checkout order summary.** `cart-summary.dwig` with read-mostly rows, coupon form, and discount-aware totals beside the checkout form.

**Empty-cart guidance.** The `{% else %}` branch links onward to the catalogue instead of dead-ending. New and post-purchase visitors land here.

## 8. Common mistakes

- Updating by product ID instead of cart line ID, merging variant lines into one.
- Hand-rolled cart requests or the `mw` JS library instead of `dbEvent.shop.cart`.
- Zero-quantity updates instead of routing empties to `remove`.
- Recomputing totals, tax, or discounts in the skin instead of refreshing from `cart.totals()`.
- Locking the whole cart (or nothing) during a row mutation instead of marking the acting row.
- Missing the initialized guard, stacking duplicate handlers after every re-render.
- Leaving a blank table after the last removal instead of reloading into the empty-cart branch.
- Hardcoding the checkout URL instead of the resolved `checkout_page_link`, or showing checkout when `checkout_link_enabled` is false.
- Unescaped titles or `|raw` on anything but backend-rendered `custom_fields` HTML.
- Forgetting `default.dwig`, so a missing skin selection breaks the cart site-wide.
