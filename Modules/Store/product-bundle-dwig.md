# Shop/Product/Bundle — Bundle Product Sets

The `Shop/Product/Bundle` module renders a bundle parent with its children: fixed sets or build-your-own picks with minimum and maximum counts. Its skins live in `Templates/Modules/Store/Product/Bundle/`. The backend resolves the parent, loads children with prices, stock, variants, and selections, and passes the configured set; the skin renders the list and adds it through one form submit.

## 1. Embedding a bundle

```twig
<module
    type="Shop/Product/Bundle"
    id="product-bundle"
    content_id="{{ product_id }}"
    template="default.dwig"
/>
```

- `type` is `Shop/Product/Bundle` (the `Store/Product/Bundle` path form resolves the same module). Either spelling works; keep one per project.
- `id` identifies this instance. Keep it stable and unique per placement.
- `content_id` (or `product_id`, `rel_id`, `content-id`) selects the bundle parent. A saved `product_id` setting wins over attributes; the current page content is the last fallback.
- `template` selects the skin filename from the bundle directory. Pass only the filename, never a path. A template chosen in Live Edit is saved against the instance and takes precedence over the inline attribute.
- An unavailable file falls back to `default.dwig`, so the directory keeps a working `default.dwig`.
- With no bundle product selected, the module shows an editor notice instead of the set. Keep an `{% if %}` guard so the live page never renders a broken card.

## 2. Where skins live and how they resolve

```text
Templates/Modules/Store/Product/Bundle/
+-- default.dwig     # fallback, keep it working
+-- compact.dwig
+-- ...
```

Resolution order for every skin:

1. The same relative path in the active theme.
2. The bundled default template.
3. `default.dwig` in the same directory.

Keep theme-specific bundle skins in the active theme. Changing the bundled fallback changes the default for every generated theme.

## 3. What data a bundle skin receives

| Value | Contents |
|---|---|
| `data.content_id` | Resolved bundle parent ID. Reuse it for hidden form fields. |
| `data.item` / `data.content` | Parent record with `link`, `product_type`, bundle fields (`bundle_type`, `bundle_product_ids`, `bundle_min_items`, `bundle_max_items`, `bundle_products`, `bundle_data`), and `variants`. |
| `data.content_data` | Parent content data. Read keys with `\|default()`. |
| `data.product` | Parent pricing and stock: `price`, `price_formatted`, `in_stock`, `discount_percentage`, `offer_price`, `custom_fields`, plus `variants`, `bundle_data`, `bundle_products`. |
| `data.pictures` / `data.image` | Parent media. Guard both; bundle parents without photos are normal. |
| `data.variants` | Parent variant rows (price, qty, sku, offer, images, stock each) where the parent itself has variants. |
| `data.bundle` | Bundle config: `product_type`, `bundle_type`, `bundle_pricing`, `bundle_additional_fee`, `bundle_product_ids`, `bundle_min_items`, `bundle_max_items`, `bundle_products`. |
| `data.bundle_products` | Child list. Each carries the fields below. |
| `data.params` | Tag attributes plus the instance ID (`data.params.id`). |

Child fields:

| Field | Contents |
|---|---|
| `id` | Purchase ID submitted with the set. For exact-variant children this is the variant ID. |
| `parent_id` / `configured_id` | Owning product and configured row IDs. |
| `is_exact_variant` | True when the child is one preselected variant. Such children need no further input. |
| `title` | Parent title, or parent plus variant title for exact variants. Escape with `\|e`. |
| `link` / `image` / `pictures` | Resolved link and media with parent fallbacks. Guard all three. |
| `price` / `price_formatted` / `offer_price` / `discount_percentage` / `in_stock` | Per-child pricing and stock. Display through `currency_format()` or the preformatted string. |
| `custom_fields` | Option groups of the owning product for variant selection. |
| `variants` | Selectable variants of the owning product (empty for exact-variant children). |
| `requires_variant_selection` | True when the child needs a variant choice before joining the set. |
| `selected_options` | Preselected options for exact-variant children. |

## 4. Fixed sets and build-your-own picks

`bundle_type` decides the interaction: `fixed` adds the whole configured set; `build-your-own` lets visitors check children within min/max bounds:

```twig
{% set bundle = data.bundle|default({}) %}
{% set is_build_your_own = bundle.bundle_type|default('fixed') == 'build-your-own' %}
{% set minimum = is_build_your_own ? bundle.bundle_min_items|default(1) : 0 %}
{% set maximum = is_build_your_own ? bundle.bundle_max_items|default(0) : 0 %}
{% set children = data.bundle_products|default([]) %}

<form class="bundle-cart-form">
    <input type="hidden" name="content_id" value="{{ data.content_id|e('html_attr') }}">
    <input type="hidden" name="bundle_items" data-bundle-selected-items value="">
    <ul>
    {% for child in children %}
        <li>
            {% if is_build_your_own and child.in_stock|default(true) %}
                <input type="checkbox" value="{{ child.id|e('html_attr') }}" data-bundle-item
                       id="bundle-item-{{ child.id|e('html_attr') }}">
            {% endif %}
            <label for="bundle-item-{{ child.id|e('html_attr') }}">
                <a href="{{ child.link|default('#')|e('html_attr') }}">{{ child.title|default('')|e }}</a>
                {% if not child.in_stock|default(true) %}<span>Out of stock</span>{% endif %}
            </label>
            <span>{{ child.price_formatted|default('')|e }}</span>
        </li>
    {% else %}
        <li>No products are configured for this bundle.</li>
    {% endfor %}
    </ul>
    <input type="number" name="qty" value="1" min="1">
    <button type="button" data-bundle-add-button
            {% if children|length == 0 or is_build_your_own %}disabled{% endif %}>Add bundle to cart</button>
</form>
```

```js
// Enable the button only when the checked count satisfies min/max,
// write checked IDs into bundle_items, then submit content_id + bundle_items + qty.
```

Rules:

1. Enable the add button only when the selection is valid (fixed: non-empty children; build-your-own: checked count within min/max). Start disabled; validate on every change.
2. Write checked IDs into the `bundle_items` hidden field and submit `content_id` plus `bundle_items` plus `qty` in one call. Splitting the set across calls breaks bundle pricing.
3. Never offer out-of-stock children as checkable picks. Fixed bundles containing unavailable children say so per child.
4. Children with `requires_variant_selection` cannot join until their variant is chosen; exact-variant children (`is_exact_variant`) arrive preselected with `selected_options` and need no input.
5. Guard the script with a ready flag on the unique root ID so re-renders never stack handlers. Scope checkbox collection to the skin's own root.
6. Self-close the embed tag (`/>`). An unclosed bundle tag swallows the markup that follows it.

## 5. Use cases

**Fixed gift set.** Whole configured set with one price and one button. No checkboxes, no counting.

**Build-your-own box.** Checked picks within min/max (for example any 3 of 6) with live count messaging and max-cap disabling.

**Bundle with variants.** Children offering their own variant rows before joining. Selection precedes the set submit.

**Bundle spotlight card.** Parent image, title, price, and child count linking to the full bundle section. Same data, teaser chrome.

## 6. Common mistakes

- Checkboxes on fixed bundles, inviting invalid sets that pricing cannot honor.
- Submitting without `bundle_items`, buying the parent alone at bundle expectations.
- Splitting the set across calls instead of one submit.
- Offering out-of-stock or variant-undecided children as ready picks.
- Unscoped checkbox collection mixing two bundle instances on one page.
- Unclosed embed tag swallowing following markup.
- Rebuilding bundle pricing in the skin instead of presenting configured values.
- Forgetting `default.dwig`, so a missing skin selection breaks bundles site-wide.
