# Store/Products — Product Lists and Grids

The `Store/Products` module renders a list of products: a shop grid, a category row, related items, bestsellers. Its skins live in `Templates/Modules/Store/Products/`. Editors choose the source (category, page, hand-picked products, related, recently viewed) and page size; the backend queries and passes the list with pagination; the skin renders one card per product.

Do not confuse it with its neighbors. `Store/Product` renders one product chosen per instance. `Store/Filter` renders the filter controls that drive this list. This module is the list itself.

## 1. Embedding a product list

```twig
<module
    type="Store/Products"
    id="shop-products-list"
    template="default.dwig"
/>
```

- `type` is `Store/Products`: the path-style name that mirrors the skin path `Templates/Modules/Store/Products/`. 
- `id` identifies this instance and connects it to its source and display settings. Keep it stable and unique per list. The filter pairing in section 6 addresses this list by ID, so renaming it breaks live filtering.
- `template` selects the skin filename from the products directory. Pass only the filename, never a path. A template chosen in Live Edit is saved against the instance and takes precedence over the inline attribute.
- An unavailable file falls back to `default.dwig`, so the directory keeps a working `default.dwig`.

Source selection (saved settings first, tag attributes as fallback):

| Setting | Purpose |
|---|---|
| Category / page source | List products from chosen categories or a shop page. The common shop and category setup. |
| Hand-picked products | List exactly the selected product IDs, in order. Use for bestsellers, bundles of attention, gift guides. |
| Related products | List products related to the current product. Lives on product pages. |
| Recently viewed | List the visitor's recently seen products. Lives on home and product pages. |
| `limit` | Maximum products per page. |
| `paginate` | Turns paging on so large catalogs split across pages (section 6). |

## 2. Where skins live and how they resolve

```text
Templates/Modules/Store/Products/
+-- default.dwig            # fallback, keep it working
+-- products-style-1.dwig   # tabbed grid with category pills
+-- category-style.dwig     # guarded card grid with offers
+-- ...
```

Resolution order for every skin:

1. The same relative path in the active theme.
2. The bundled default template.
3. `default.dwig` in the same directory.

Keep theme-specific lists in the active theme. Changing the bundled fallback changes the default for every generated theme.

## 3. What data a products skin receives

| Value | Contents |
|---|---|
| `data.posts` | The product list in display order. Each item carries the fields below. Always loop with `\|default([])` and keep an `{% else %}` empty branch. |
| `data.pagination` | Paging info: `pages_count` and `paging_param`. |
| `data.params` | Tag attributes plus the instance ID (`data.params.id`). |

Product item fields used by skins:

| Field | Contents |
|---|---|
| `id` | Product ID. Feeds nested modules, `data-*` hooks, and helper calls. |
| `title` | Product name. Escape with `\|e`. |
| `link` / `url` | Resolved product URL. Prefer `link`, fall back to `url`, then `'#'`: `product.link\|default(product.url\|default('#'))`. Never construct URLs. |
| `tn_image` / `picture` | Listing image sources. Prefer `tn_image`, fall back to `picture`, then render the no-image placeholder. |
| `price`, `original_price` | Regular prices. Prefer `price`, fall back to `original_price`. |
| `offer_price` | Active offer price, or empty. When set, strike through the regular price beside it. |
| `discount_percentage` | Offer badge value, or empty. Render the badge only when truthy. |
| `instock` | Stock flag. Disable or relabel the buy button when `false`. |
| `description` | Sanitized body HTML. Render with `\|raw`; strip tags (`\|striptags`) when shortening into attributes. |

Two helpers enrich each row. `content_data(product.id)` returns site-specific fields (weights, storage notes) — read every key with `|default()`. `content_categories(product.id)` returns the product's categories for labels and client-side grouping.

## 4. The card loop

One loop, one card per product, guarded image, offer-aware price, stock-aware action:

```twig
<div class="row row-cols-2 row-cols-lg-4 g-3">
{% for product in data.posts|default([]) %}
    {% set productData = content_data(product.id) %}
    {% set productUrl = product.link|default(product.url|default('#')) %}
    {% set productImage = product.tn_image|default(product.picture|default('')) %}
    {% set regularPrice = product.price|default(product.original_price|default(0)) %}

    <div class="col">
    <article class="card h-100" data-id="{{ product.id|e('html_attr') }}">
        <div class="position-relative">
            {% if productImage %}
                <a class="ratio ratio-1x1" href="{{ productUrl|e('html_attr') }}">
                    <img class="card-img-top object-fit-cover"
                         src="{{ productImage|e('html_attr') }}"
                         alt="{{ product.title|default('Product')|e('html_attr') }}"
                         width="600" height="600" loading="lazy">
                </a>
            {% else %}
                <a class="ratio ratio-1x1 bg-body-secondary" href="{{ productUrl|e('html_attr') }}">
                    <span>Image unavailable</span>
                </a>
            {% endif %}
            {% if product.discount_percentage|default(0) %}
                <span class="badge position-absolute top-0 end-0 m-3">{{ product.discount_percentage|e }}% off</span>
            {% endif %}
        </div>
        <div class="card-body d-flex flex-column">
            <h3 class="h6"><a href="{{ productUrl|e('html_attr') }}">{{ product.title|default('')|e }}</a></h3>
            <div class="price-row mb-3">
                {% if product.offer_price|default(0) %}
                    <del>{{ currency_format(regularPrice) }}</del>
                    <strong>{{ currency_format(product.offer_price) }}</strong>
                {% else %}
                    <strong>{{ currency_format(regularPrice) }}</strong>
                {% endif %}
            </div>
            <button class="btn btn-primary w-100 mt-auto" type="button"
                    data-add="{{ product.id|e('html_attr') }}"
                    {% if product.instock is same as(false) %}disabled{% endif %}>
                {{ product.instock is same as(false) ? 'Out of stock' : 'Add to Cart' }}
            </button>
        </div>
    </article>
    </div>
{% else %}
    <div class="col-12"><div class="alert alert-info text-center mb-0">No products are available yet.</div></div>
{% endfor %}
</div>
```

Rules:

1. Resolve display values once per row (`productUrl`, `productImage`, `regularPrice`) and reuse them. Scattered fallbacks drift apart.
2. Guard the image with `{% if %}` and draw the same-ratio placeholder in `{% else %}`. Cards without photos must hold grid alignment.
3. Format money only with `currency_format()`. Never concatenate symbols or decimals by hand, and never print raw `price` as display text.
4. Gate the buy action on `instock is same as(false)`. Disabled plus relabeled beats hidden: the grid keeps its shape and the reason is visible.
5. Keep the per-card hooks the theme scripts consume (`data-add`, `data-wish`, `data-quick` with the product ID). A card can look right while its buttons do nothing.
6. Escape titles and attributes; `|raw` applies only to full `description` HTML, and `|striptags` before shortening descriptions into attributes.

## 5. Composing behavior — reviews and quick view

Rows compose sibling modules per product. Suffix nested IDs with the product ID so repeated cards keep independent settings:

```twig
<module type="Store/ProductReviews"
        id="shop-page-product-reviews-{{ product.id }}"
        product_id="{{ product.id }}"
        template="review-style-1.dwig" />
```

1. Pass `product_id` explicitly to every nested product module. Cards do not inherit the row's product.
2. Keep quick-view triggers (`data-quick`, `showQuickView(product.id)`) wired to the modal the theme provides. If the theme has no quick-view modal, drop the button rather than shipping a dead one.
3. Never nest a `Store/Products` list inside its own card. Lists compose cards and single-product modules, never themselves.

## 6. Pagination and live filtering

Paged lists read `data.pagination`. Filter controls in a `Store/Filter` module drive this list by its instance ID:

```twig
{# Filter aside targets the list by matching ID #}
<module type="Store/Filter" id="shop-filters" target="#shop-products-list" template="default.dwig" />
<module type="Store/Products" id="shop-products-list" paginate="true" limit="12" template="category-style.dwig" />
```

1. Keep the list ID stable: filters, pagination, and AJAX refresh all address it. Renaming the ID breaks every wiring at once.
2. Render paging controls only when `data.pagination.pages_count > 1`.
3. The same list reloads via `window.dbEvent.module.load()` for AJAX paging and filtering (see the module introduction). After any reload, per-card hooks must still resolve — keep them attribute-based (`data-add`, `data-wish`) rather than bound once at page load.
4. JSON mode (`output: "json"`) returns `response.data.posts` and `response.data.pagination` for fully custom frontends. Do not mix JSON fetching with server-rendered cards in one skin.

## 7. Use cases

**Shop grid.** Paged `category-style.dwig` cards with offers, stock states, and reviews. The catalog workhorse, paired with the filter aside.

**Category showcase row.** Unpaged grid of the first N products with a "view all" link to the category page. Limit, don't paginate.

**Tabbed catalogue.** Category pills above the grid (from `get_shop_categories`) grouping cards client-side by `data-category`. Same list data, tabbed presentation.

**Related products.** Source mode on the product page: same card skin, narrower grid, no pagination. Contextual upsell without configuration per product.

**Hand-picked gift guide.** Editors order exact IDs for a curated row. Order-sensitive skins must preserve list order, never re-sort.

**Recently viewed.** Visitor-history source on home and product pages. Always keep the `{% else %}` branch: new visitors have no history.

## 8. Common mistakes

- Looping `data.posts` without `|default([])` or an `{% else %}` branch, breaking new categories and new visitors.
- Constructing product URLs or image paths instead of using resolved `link`, `tn_image`, and helpers.
- Printing raw `price` or hand-concatenated currency instead of `currency_format()`.
- Live buy buttons on `instock is same as(false)` products.
- One shared nested-module ID across all cards, merging every card's reviews into one settings scope.
- Missing `product_id` on nested review and cart modules.
- Pagination controls rendered for a single page, or a renamed list ID that orphans filters and AJAX reloads.
- Using this list module for a single known product (`Store/Product`) or rebuilding filter controls inside the list instead of using `Store/Filter`.
- Forgetting `default.dwig`, so a missing skin selection breaks every product list site-wide.
