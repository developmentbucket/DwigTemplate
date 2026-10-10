# Store/Product Page Template — Product Detail Pages

The `Templates/Store/Product/` page template renders the full product detail page: breadcrumb, gallery, purchase card, information tabs, and related products. It extends the site layout and fills the content block; the current product arrives as page data, and the template composes store modules bound to that product.

Do not confuse it with its neighbor. `Modules/Store/Product/` skins render a reusable single-product fragment (cards, quick views) embedded anywhere via `<module>` tags. This page template is the whole product URL: layout extension, page data, and full-page composition.

## 1. Page template shape

```twig
{% extends "Layouts/main.dwig" %}

{% block content %}
{% set product_id = data.content.id %}
{% set product = data.product|default({}) %}
{% set product_data = data.content_data|default({}) %}

<main>
    <div class="edit main-content" data-layout-container
         rel="content" rel-id="{{ product_id }}" field="content">
        {# ...breadcrumb, gallery, purchase card, tabs, related... #}
    </div>
</main>
{% endblock %}
```

- Extends `Layouts/main.dwig` and fills only the `content` block. Page chrome comes from the layout; this file owns the product story.
- Resolve working values once at the top (`product_id`, `product`, `product_data`) and reuse them. Scattered lookups drift apart.
- Wrap the page in the editable content container (`edit main-content` with `data-layout-container`, `rel="content"`, `rel-id`, `field="content"`) so page-level editing keeps working around the modules.

## 2. Where page templates live and how they resolve

```text
Templates/Store/Product/
+-- default.dwig     # fallback, keep it working
+-- Bundle/          # bundle product pages
+-- ...
```

Resolution is theme-first, bundled-default-second per filename, with `default.dwig` as the fallback. Keep theme-specific product pages in the active theme.

## 3. What data the page receives

| Value | Contents |
|---|---|
| `data.content` | Current product record: `id`, `title`, `description` and `content_body` (sanitized HTML, render with `\|raw`), `link`. |
| `data.product` | Pricing and stock: `price`, `offer_price`, `discount_percentage`, `instock`. Display through `currency_format()`; gate buying on stock. |
| `data.content_data` | Site-specific fields (`sku`, `product_weight`, `storage`, and similar). Keys differ per site; always read with `\|default()`. |

Helpers used on product pages:

| Helper | Purpose |
|---|---|
| `content_categories(product_id)` | Category list for the breadcrumb trail. Guard with `\|default([])`. |
| `get_related_products(product_id)` | Related records; filter to the current product type, exclude self, cap the count. |
| `get_picture(id)` / `get_product_price(id)` | Image and price fallbacks for related cards missing direct values. |
| `content_data(id)` | Site-specific fields per related record, read with `\|default()`. |
| `thumbnail()` / `currency_format()` | Sized images and locale prices. Never output originals or hand-formatted money. |

## 4. Page anatomy

Breadcrumb, two-column gallery plus purchase card, tabbed information, related grid:

```twig
<nav aria-label="breadcrumb">
    <ol class="breadcrumb mb-4">
        <li class="breadcrumb-item"><a href="{{ site_url()|e('html_attr') }}">Home</a></li>
        {% if primary_category %}
            <li class="breadcrumb-item">
                <a href="{{ primary_category.url|default('#')|e('html_attr') }}">{{ primary_category.title|e }}</a>
            </li>
        {% endif %}
        <li class="breadcrumb-item active" aria-current="page">{{ data.content.title|e }}</li>
    </ol>
</nav>

<article class="row g-4 align-items-start">
    <div class="col-lg-6">
        <module type="Media/PictureGallery" id="product-picture-gallery-{{ product_id }}"
                content-id="{{ product_id }}" product-title="{{ data.content.title|e('html_attr') }}"
                handle_empty="true" template="product-gallery.dwig" />
    </div>
    <div class="col-lg-6">
        <h1>{{ data.content.title|e }}</h1>
        <module type="Store/ProductReviews" id="product-page-reviews-{{ product_id }}"
                product_id="{{ product_id }}" template="review-style-1.dwig" />
        {# ...offer-aware price, description, stock alert... #}
        {% if in_stock %}
            <module type="Store/AddToCart" id="product-add-to-cart-{{ product_id }}"
                    product_id="{{ product_id }}" content_id="{{ product_id }}" template="default.dwig" />
        {% endif %}
    </div>
</article>
```

Rules:

1. Bind every nested module to `product_id` explicitly (`content-id`, `product_id`) and suffix every nested ID with it. Unbound modules on a product page show the wrong product's gallery, reviews, or cart.
2. Show the buy module only when in stock; otherwise show the unavailable state. Never offer to buy what cannot ship.
3. Render `description` and `content_body` with `|raw` (sanitized body HTML), titles and attributes escaped. Body-with-fallback chains (`content_body`, then `description`, then a coming-soon line) keep new products presentable.
4. Build tab IDs from `product_id` (`product-description-{{ product_id }}`) with explicit `aria-controls`/`aria-labelledby` pairing and exactly one active pane.
5. Cap related products (four is the working default), exclude self, and keep only matching content types. Render the section only when the filtered list is non-empty.
6. Keep per-card hooks in related grids (`data-add`, `data-wish`, `data-quick` with the related ID) and per-product nested review IDs, exactly like list skins.

## 5. Bundle products on the page

Bundle parents sell children together: fixed sets or build-your-own picks with minimum and maximum counts. The bundle module resolves the parent, lists the children with prices and stock, and adds the configured set through one form.

```twig
<module type="Store/Product/Bundle"
        id="product-bundle-{{ product_id }}"
        content_id="{{ product_id }}"
        template="default.dwig" />
```

The bundle skin receives `data.item` (parent with `link`), `data.bundle` (type, pricing, min/max, product IDs), `data.bundle_products` (children), and `data.product` (bundle price). Each child carries `id`, `title`, `link`, `image`, `price`/`price_formatted`, `in_stock`, `requires_variant_selection`, and preselected `selected_options` for exact-variant children:

```twig
<form class="bundle-cart-form">
    <input type="hidden" name="content_id" value="{{ data.content_id|e('html_attr') }}">
    <input type="hidden" name="bundle_items" data-bundle-selected-items value="">
    <ul>
    {% for child in data.bundle_products|default([]) %}
        <li>
            {% if is_build_your_own and child.in_stock|default(true) %}
                <input type="checkbox" value="{{ child.id|e('html_attr') }}" data-bundle-item
                       id="bundle-item-{{ child.id|e('html_attr') }}">
            {% endif %}
            <label for="bundle-item-{{ child.id|e('html_attr') }}">
                <a href="{{ child.link|e('html_attr') }}">{{ child.title|e }}</a>
                {% if not child.in_stock|default(true) %}<span>Out of stock</span>{% endif %}
            </label>
            <span>{{ child.price_formatted|e }}</span>
        </li>
    {% else %}
        <li>No products are configured for this bundle.</li>
    {% endfor %}
    </ul>
    <input type="number" name="qty" value="1" min="1">
    <button type="button" data-bundle-add-button disabled>Add bundle to cart</button>
</form>
```

Rules:

1. Render the bundle module only when the product is a bundle parent (`product_type == 'bundle'` with non-empty `bundle_products`). Simple products skip this section entirely.
2. For build-your-own sets, enable the add button only when the checked count satisfies min/max, and write the checked IDs into the `bundle_items` hidden field. Fixed bundles add the whole configured set with no checkboxes.
3. Never offer out-of-stock children as checkable picks. Fixed bundles containing unavailable children must say so per child.
4. Children needing variant selection (`requires_variant_selection`) cannot join the set until their variant is chosen. Exact-variant children arrive preselected and need no further input.
5. Submit `content_id` plus `bundle_items` plus `qty` in one call. Splitting the set across calls breaks bundle pricing.
6. Self-close the embed tag (`/>`). An unclosed bundle tag swallows the markup that follows it.

## 6. Product options and add to cart via the AddToCart module

Variant products choose options through custom fields rendered on the page, bought through the `Store/AddToCart` module. The page renders the selectors from `data.product.custom_fields`; the cart call carries the chosen `{fieldId: valueId}` pairs; the displayed price refreshes per selection.

```twig
{% set custom_fields = product.custom_fields|default({}) %}

{% if product.has_variants|default(false) and custom_fields is not empty %}
<div class="product-options" data-variant-groups>
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

<module type="Store/AddToCart" id="product-add-to-cart-{{ product_id }}"
        product_id="{{ product_id }}" content_id="{{ product_id }}" template="default.dwig" />
```

```js
async function buyWithOptions(rootId, productId) {
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
    try {
        await window.dbEvent.shop.cart.add({
            content_id: productId,
            qty: Number(root.querySelector('[data-qty-input]').value) || 1,
            custom_fields: customFields
        });
    } catch (error) {
        console.error(error.code, error.message);
    }
}

// Refresh the displayed price when the selection changes.
const price = await window.dbEvent.shop.product.get_price({
    product_id: productId,
    custom_fields: customFields
});
```

Rules:

1. One buying path per page: either the `Store/AddToCart` module button (simple products, quantity only) or the options-plus-call flow above (variant products). Two live paths double-buy on one click.
2. Render selectors only when `has_variants` is true and groups exist. Simple products show no dropdowns.
3. Send `custom_fields` as a plain object only (`{fieldId: valueId}`); arrays and `null` values are rejected. Validate required groups before calling — an unchosen variant fails or buys wrong.
4. Scope selector collection to the page root. Stray selects elsewhere on the page must never join the payload.
5. Refresh the displayed price through `get_price` on selection change; never recompute prices in the skin. Keep the base `price_formatted` until the first selection resolves.
6. Full selector, payload, and validation rules live in the `Store/AddToCart` module doc — this page follows that contract, it does not redefine it.

## 7. Use cases

**Standard product page.** Breadcrumb, gallery, purchase card, description/details/reviews tabs, related grid. The default skin pattern.

**Bundle page.** Bundle box module plus component list in the `Bundle/` variant. Same anatomy, bundle composition in the purchase column.

**Minimal product page.** Gallery, title, price, buy button only — no tabs or related grid. Same data, quieter chrome for single-SKU shops.

**Pre-order page.** Stock-false rendering with notify-me copy instead of the buy module. Same gate as out-of-stock, different message.

## 8. Common mistakes

- Rendering bundle sections for simple products, or fixed-bundle checkboxes that invite invalid sets.
- Splitting bundle contents across calls instead of one `content_id` plus `bundle_items` plus `qty` submit.
- Two live buying paths (module button plus options script) double-buying on one click.
- Buying variant products without their `{fieldId: valueId}` selections, or hand-computed variant prices.

- Unbound nested modules showing another product's gallery, reviews, or cart.
- Live buy buttons on out-of-stock products.
- Related grids including the current product, mixing content types, or rendering uncapped.
- Tab IDs not suffixed per product, or zero/multiple active panes.
- Escaped body HTML printing tags as text, or `|raw` on titles and prices.
- Hand-formatted prices or constructed product links instead of helpers and resolved values.
- Reading `content_data` keys without `|default()`, breaking products missing site-specific fields.
- Dropping the editable content container, breaking page-level editing around the modules.
