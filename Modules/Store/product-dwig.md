# Store/Product — Single Product Cards and Quick Views

The `Store/Product` module renders one product: its image, title, price, and actions. Its skins live in `Templates/Modules/Store/Product/`. Editors pick the product once per instance; the backend loads its record, media, pricing, and stock and passes them as ready values; the skin decides the card — compact tile, quick-view panel, or feature spotlight.

Do not confuse it with its neighbors. `Store/Products` renders a list of products. The `Store/Product` page template (`Templates/Store/Product/`) is the full product page. This module is the reusable single-product fragment embedded anywhere: homepages, sidebars, modals, landing pages.

## 1. Embedding a product

```twig
<module
    type="Store/Product"
    id="featured-product"
    product_id="42"
    template="default.dwig"
/>
```

- `type` is `Store/Product`: the path-style name that mirrors the skin path `Templates/Modules/Store/Product/`. The bare legacy name still resolves, but every example in these docs uses the path-style form.
- `id` identifies this instance and connects it to its product selection and saved settings. Keep it stable and unique per placement. Changing it orphans the selection saved under the old ID.
- `product_id` selects the product when no saved selection exists. The saved setting wins over the attribute; the attribute is the fallback for hardcoded placements.
- `template` selects the skin filename from the product directory. Pass only the filename, never a path. A template chosen in Live Edit is saved against the instance and takes precedence over the inline attribute.
- An unavailable file falls back to `default.dwig`, so the directory keeps a working `default.dwig`.
- With no product selected and no attribute, the module shows an editor notice instead of a card. The skin needs no missing-product branch, but keep it harmless when `data.item` is empty.

## 2. Where skins live and how they resolve

```text
Templates/Modules/Store/Product/
+-- default.dwig       # fallback, keep it working
+-- quick-view.dwig    # modal/detail panel
+-- ...
```

Resolution order for every skin:

1. The same relative path in the active theme.
2. The bundled default template.
3. `default.dwig` in the same directory.

Keep theme-specific product skins in the active theme. Changing the bundled fallback changes the default for every generated theme.

## 3. What data a product skin receives

| Value | Contents |
|---|---|
| `data.item` (also `data.content`) | Full product record plus resolved `link`, `product_type`, and bundle fields (`bundle_type`, `bundle_product_ids`, `bundle_products`, min/max items). |
| `data.content_id` | The product ID. |
| `data.content_data` | Site-specific extra fields (weight, storage, preparation, and similar). Keys differ per site; always read with `\|default()`. |
| `data.product` | Pricing and stock: `price`, `price_formatted`, `in_stock`, `discount_percentage`, `offer_price`, `custom_fields`, `product_type`, `bundle_data`, `bundle_products`. |
| `data.pictures` | All product pictures (`filename`, `title` each). |
| `data.image` | Primary product image source, or empty. |
| `data.params` | Tag attributes plus the instance ID (`data.params.id`). |

The safe pattern for any product card:

```twig
{% set item = data.item|default({}) %}
{% set product = data.product|default({}) %}

<article class="card h-100">
    {% if data.image|default('') %}
        <a href="{{ item.link|default('#')|e('html_attr') }}" class="d-block">
            <img src="{{ thumbnail(data.image, 800, 800)|e('html_attr') }}"
                 class="card-img-top img-fluid"
                 alt="{{ item.title|default('Product')|e('html_attr') }}">
        </a>
    {% endif %}
    <div class="card-body d-flex flex-column">
        <h3 class="card-title h5">
            <a href="{{ item.link|default('#')|e('html_attr') }}">{{ item.title|default('')|e }}</a>
        </h3>
        <div class="mt-auto d-flex justify-content-between align-items-center gap-3">
            <strong>{{ product.price_formatted|default('')|e }}</strong>
            <a href="{{ item.link|default('#')|e('html_attr') }}" class="btn btn-primary">View product</a>
        </div>
    </div>
</article>
```

Auto-escaping is off in `.dwig` templates: escape titles and attributes (`|e`, `|e('html_attr')`), pass prices through as the preformatted `price_formatted` string, and render `item.description` with `|raw` since it is sanitized body HTML like page content.

## 4. Price, stock, and offers

Read pricing exclusively from `data.product`. Never format `price` by hand; `price_formatted` already carries currency and locale:

```twig
<strong>{{ product.price_formatted|default('')|e }}</strong>
{% if product.discount_percentage|default(0) %}
    <span class="badge text-bg-danger">-{{ product.discount_percentage|e }}%</span>
{% endif %}
{% if not product.in_stock|default(true) %}
    <span class="badge text-bg-secondary">Out of stock</span>
{% endif %}
```

Rules:

1. Display `price_formatted`, compute nothing. Raw `price` is for arithmetic the skin never does.
2. Gate stock-dependent actions on `product.in_stock`. An add-to-bag button on an unavailable product must be hidden or disabled, never live.
3. Show `discount_percentage` only when truthy, and prefer `offer_price` handling to stay inside the module's own pricing values rather than inventing math.
4. Never hardcode currency symbols or codes. Prices, currency, and formatting all arrive resolved.

## 5. Media — primary image and picture list

`data.image` is the fast path for the single lead image; `data.pictures` is the full list for galleries and thumb rows:

```twig
{% if data.image|default('') %}
    <img src="{{ thumbnail(data.image, 800, 800)|e('html_attr') }}" alt="{{ item.title|default('')|e }}">
{% endif %}
```

1. Pass every image through `thumbnail()` at the displayed size. Never output the original file.
2. Guard with `{% if %}`. Products without photos are normal; the card must hold its shape regardless (fixed-ratio box or a styled placeholder).
3. For multi-image skins, loop `data.pictures` exactly like a `Media/PictureGallery` skin: `thumbnail(picture.filename, ...)`, `alt` from `picture.title`, eager first and lazy rest.
4. Link images and titles to the resolved `item.link`, never a constructed URL.

## 6. Composing behavior — reviews, cart, gallery

A product skin composes sibling modules for live behavior. Each nested module gets a stable ID strategy matching its scope:

```twig
<module type="Store/ProductReviews" id="reviews-{{ data.content_id }}" template="review-style-1.dwig" />
<module type="Store/AddToCart" id="quick-add-{{ data.content_id }}" product_id="{{ data.content_id }}" template="default.dwig" />
<module type="Media/PictureGallery" id="quick-gallery-{{ data.content_id }}" content-id="{{ data.content_id }}" handle_empty="true" template="product-gallery.dwig" />
```

1. Suffix nested IDs with `data.content_id` so each product's reviews, cart, and gallery keep independent settings when the skin repeats across products.
2. Pass `product_id` explicitly to cart and review modules. They do not inherit the product from the surrounding skin.
3. Keep every hook the nested modules' JavaScript consumes (`data-*` attributes, classes, element IDs). A card can look right while its buttons do nothing.
4. Read site-specific facts from `data.content_data` with fallbacks (`data.content_data.storage|default('')`). These keys are per-site configuration, not a stable contract — never assume one exists.

## 7. Use cases

**Product card.** Image, title linked to `item.link`, `price_formatted`, view button. The default skin for grids, sidebars, and search-adjacent placements.

**Quick-view panel.** Two-column modal content: gallery or lead image one side, title, reviews, price, site-specific facts from `content_data`, add-to-bag the other. Suffix all nested IDs with the product ID.

**Featured product hero.** Large single-product spotlight on the homepage with a hardcoded `product_id` attribute, so merchandising survives content edits.

**Bundle box.** When `product.product_type` is `bundle`, loop `product.bundle_products` (`title`, `link`, `image`, `price` each) with min/max-item notes from the bundle fields.

**Compact list row.** Thumb, title, and price in one line for carts-adjacent upsells, order-history reorder prompts, and dense category sidebars.

**Stock-aware teaser.** Card variant that swaps the action for an out-of-stock badge via `product.in_stock`, used in sale and clearance placements.

## 8. Common mistakes

- Formatting `price` by hand or hardcoding currency instead of displaying `price_formatted`.
- Leaving a live buy button on an out-of-stock product instead of gating on `product.in_stock`.
- Constructing product URLs instead of using the resolved `item.link`.
- Outputting `data.image` without `thumbnail()`, shipping full-size originals.
- Escaping `item.description` with `|e`, printing body HTML as visible text — or applying `|raw` to titles and prices, which must stay escaped.
- Reading `data.content_data.<key>` without `|default()`, breaking the skin on products missing that site-specific field.
- Nesting cart, review, or gallery modules with one shared ID across products, merging their settings.
- Forgetting `product_id` on nested cart and review modules, leaving them unbound.
- Using this module where a list belongs (`Store/Products`) or rebuilding the full product page inside a fragment instead of linking to it.
- Forgetting `default.dwig`, so a missing skin selection breaks product placements site-wide.
