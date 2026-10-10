# Store/AddToCart — Buy Buttons and Quantity

The `Store/AddToCart` module renders the purchase action for one product: the add-to-bag button, quantity, price display, and stock state. Its skins live in `Templates/Modules/Store/AddToCart/`. The backend resolves which product the button buys, checks stock, and passes button texts and pricing; the skin renders the button and fires `window.dbEvent.shop.cart.add()`.

## 1. Embedding a buy button

```twig
<module
    type="Store/AddToCart"
    id="product-add-to-cart"
    product_id="{{ data.content.id }}"
    template="default.dwig"
/>
```

- `type` is `Store/AddToCart`: the path-style name that mirrors the skin path `Templates/Modules/Store/AddToCart/`. The bare legacy name still resolves, but every example in these docs uses the path-style form.
- `id` identifies this instance and connects it to its skin selection and settings. Keep it stable and unique per placement.
- `product_id` (or `content_id` / `rel_id`) selects the product the button buys. On a product page it may be omitted: the module resolves the current content ID itself. Inside repeating cards, pass it explicitly.
- `template` selects the skin filename from the add-to-cart directory. Pass only the filename, never a path. A template chosen in Live Edit is saved against the instance and takes precedence over the inline attribute.
- An unavailable file falls back to `default.dwig`, so the directory keeps a working `default.dwig`.

Button text attributes:

| Attribute | Purpose |
|---|---|
| `button_text` | Add-button label when no saved text exists. |
| `wishlisted_button_text` | Label variant once the product is wishlisted. |

## 2. Where skins live and how they resolve

```text
Templates/Modules/Store/AddToCart/
+-- default.dwig     # fallback, keep it working
+-- compact.dwig
+-- ...
```

Resolution order for every skin:

1. The same relative path in the active theme.
2. The bundled default template.
3. `default.dwig` in the same directory.

Keep theme-specific buy buttons in the active theme. Changing the bundled fallback changes the default for every generated theme.

## 3. What data an add-to-cart skin receives

| Value | Contents |
|---|---|
| `data.for_id` | Resolved product ID the button buys: `content_id` / `product_id` / `rel_id` attribute first, `content-id` second, current page content last. `0`/empty when unresolvable. |
| `data.in_stock` | Stock flag derived from quantity (`nolimit` counts as in stock). |
| `data.button_text` | Add-button label override, or empty. |
| `data.wishlisted_button_text` | Wishlisted-state label override, or empty. |
| `data.product` | Product record (title and fields) or `false`. |
| `data.title` | Product title with a safe fallback. |
| `data.content_data` | Product content data, including `qty` and `has_variants`. |
| `data.prices` / `data.prices_data` | Named price map and raw price rows. |
| `data.has_variants` | Whether the product has selectable variants. |
| `data.wishlisted` | Whether the product is in the visitor's wishlist. |
| `data.params` | Tag attributes plus the instance ID (`data.params.id`). |

Resolve the working product ID once at the top, mirroring the backend's own chain:

```twig
{% set product_id = data.for_id|default(data.params.product_id|default(data.params.content_id|default(0))) %}
```

Auto-escaping is off in `.dwig` templates: escape titles with `|e` and IDs and attributes with `|e('html_attr')`. Never use `|raw` in an add-to-cart skin.

## 4. The button pattern

One resolved product, one stock gate, one `data-add` hook, and a disabled fallback:

```twig
<div class="product-add-to-cart">
{% if product_id %}
    {% if data.in_stock|default(true) %}
        <button class="add-btn btn btn-primary btn-lg w-100" type="button"
                data-add="{{ product_id|e('html_attr') }}" data-qty="1">
            <i class="fa-solid fa-cart-plus" aria-hidden="true"></i>
            {{ data.button_text|default('Add to Cart')|e }}
        </button>
    {% else %}
        <button class="btn btn-secondary btn-lg w-100" type="button" disabled>Out of stock</button>
    {% endif %}
{% else %}
    <button class="btn btn-primary btn-lg w-100" type="button" disabled>Product unavailable</button>
{% endif %}
```

Rules:

1. Keep `data-add="{{ product_id }}"` on the buy button. Theme cart scripts collect clicks by this attribute; renaming orphans the button.
2. Gate on `data.in_stock`: live button when true, disabled "Out of stock" when false. Never offer to buy what cannot ship.
3. Cover the unresolvable state (`product_id` empty) with a disabled "Product unavailable" button. A missing product must read as unavailable, not as a broken button.
4. Read the label from `data.button_text` with a fallback. When `data.wishlisted` is true and `wishlisted_button_text` is set, prefer the wishlisted label.
5. Carry quantity in `data-qty` (default `1`). Quantity steppers edit this value; the button reads it at click time.

Fire through `window.dbEvent` from `userfiles/modules/developmentbucket/db_lib/events/common.js`, which loads automatically on normal frontend pages. Never reference the `mw` JS library:

```js
await window.dbEvent.shop.cart.add({ content_id: productId, qty: 1 });
```

Every call uses `try/catch` because failures reject. On success the header cart badge (class `js-shopping-cart-quantity`) refreshes automatically; the skin syncs nothing by hand.

## 5. Variant products with custom fields

When `data.has_variants` is true, the buy action needs the visitor's variant choice as custom fields. Each variant group (size, color, pack) is one custom field; each option is one custom-field value. The skin renders one selector per group, collects `{fieldId: valueId}` pairs, and sends them with the cart call.

### 5.1 Reading the variant groups

`data.product.custom_fields` maps field IDs to groups. Each group carries `id`, `type`, `name`, `required`, and `values` keyed by value ID:

```twig
{% set custom_fields = data.product.custom_fields|default({}) %}

{% if data.has_variants|default(false) and custom_fields is not empty %}
<div class="product-variants" data-variant-groups>
    {% for field_id, field in custom_fields %}
        {% if field.is_active|default(true) and field.values|default([]) is not empty %}
        <label class="d-block mb-2">
            {% if field.show_label|default(true) %}<span>{{ field.name|default('Option')|e }}</span>{% endif %}
            <select name="custom_fields[{{ field_id|e('html_attr') }}]"
                    data-variant-field="{{ field_id|e('html_attr') }}"
                    {% if field.required|default(false) %}required{% endif %}>
                <option value="">Choose {{ field.name|default('option')|e }}</option>
                {% for value_id, value in field.values %}
                    <option value="{{ value_id|e('html_attr') }}">{{ value.value|default(value.title|default(''))|e }}</option>
                {% endfor %}
            </select>
        </label>
        {% endif %}
    {% endfor %}
</div>
{% endif %}
```

Rules:

1. Render selectors only when `has_variants` is true and groups exist. Simple products skip this block entirely; a stray empty dropdown blocks buying for no reason.
2. Name selects `custom_fields[<fieldId>]` so a backing `<form>` submits the same shape the object call uses. Keep `data-variant-field` for script collection.
3. Mark `required` groups required. Unchosen required groups fail server-side; the skin should stop them client-side first with a visible message naming the group.
4. Escape group and option names. They are merchant input.

### 5.2 Sending selections with the cart call

Collect the chosen pairs and pass them as `custom_fields` alongside `content_id` and `qty`:

```html
<script>
async function addVariantToCart(rootId, productId) {
    const root = document.getElementById(rootId);
    const customFields = {};
    let missing = null;
    root.querySelectorAll('[data-variant-field]').forEach(function (select) {
        if (select.required && !select.value) {
            missing = select;
        }
        if (select.value) {
            customFields[select.getAttribute('data-variant-field')] = select.value;
        }
    });
    if (missing) {
        missing.focus();
        return;
    }
    const qty = Number(root.querySelector('[data-qty-input]').value) || 1;
    try {
        await window.dbEvent.shop.cart.add({
            content_id: productId,
            qty: qty,
            custom_fields: customFields
        });
    } catch (error) {
        console.error(error.code, error.message);
    }
}
</script>
```

A form-backed equivalent passes `FormData` instead — same fields, same names:

```js
await window.dbEvent.shop.cart.add(new FormData(form));
```

Rules:

1. Send `custom_fields` as a plain object only: `{fieldId: valueId}`. Arrays and `null` values are rejected.
2. Validate required groups before calling. A variant product bought without a selection fails or buys the wrong variant.
3. Scope collection to the skin's own root element (unique wrapper ID per placement). A page with two buy boxes must not mix their selections.
4. Every call uses `try/catch` because failures reject. Surface the message near the button so the visitor can correct the selection.

### 5.3 Refreshing the price for the selection

Variant choices can change the price. Refresh the displayed price through the price API before or alongside adding:

```js
const response = await window.dbEvent.shop.product.get_price({
    product_id: productId,
    custom_fields: customFields
});
```

1. Call `get_price` on selection change and update only the price element. Never recompute prices in the skin.
2. Keep showing the base `price_formatted` until the first selection resolves, so the button area never flashes empty.

### 5.4 Quantity with variants

Keep quantity steppers as plain inputs bound to the call's `qty`. Clamp to available stock where known; never below 1. Quantity multiplies the chosen variant — it never replaces the selection, so always send both together.

## 6. Use cases

**Product page buy box.** Full button with quantity stepper and variant selectors beside the gallery. Explicit or page-resolved product ID.

**Card quick-add.** Compact `data-add` button on `Store/Products` and `Store/Product` cards with `data-qty="1"` and no stepper. Same hook, smaller chrome.

**Quick-view modal action.** Full-width button in the modal reusing the card's product ID. Suffix any wrapper IDs per product when modals repeat.

**Buy-now duet.** Primary instant-checkout action beside add-to-bag where the shop supports it. Keep both hooks intact; never merge two behaviors into one button.

**Stock-aware teaser.** Disabled out-of-stock rendering for clearance and backorder placements, driven by `data.in_stock`.

## 7. Common mistakes

- Renaming `data-add` (or dropping `data-qty`), orphaning the button while it looks fine.
- Offering a live button for `in_stock == false` products.
- Rendering nothing (or a live button) when the product is unresolvable instead of disabled "Product unavailable".
- Hardcoding the product ID in the skin instead of resolving `for_id` / `product_id`, freezing the button to one product.
- Buying variant products without passing the visitor's variant selection.
- Hand-rolling cart requests instead of `dbEvent.shop.cart.add()`, or calling the `mw` JS library.
- Assuming the call succeeded before updating buttons and badges; failures reject and need `catch` handling.
- Forgetting `default.dwig`, so a missing skin selection breaks buying site-wide.
