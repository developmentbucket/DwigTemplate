# Store/StoreCategory Page Template — Category Landing Pages

The `Templates/Store/StoreCategory/` page template renders a category landing page: header with the category title, product grid, and optional filters. It extends the site layout and fills the content block; the current category and its products arrive as page data, and the template loops them into cards.

## 1. Page template shape

```twig
{% extends "Layouts/main.dwig" %}

{% block content %}
<main>
    <div class="edit main-content" data-layout-container
         rel="categories" rel-id="{{ data.category.id }}" field="content">
        {# ...header, grid... #}
    </div>
</main>
{% endblock %}
```

- Extends `Layouts/main.dwig` and fills only the `content` block. Page chrome comes from the layout; this file owns the category story.
- Wraps the page in the editable container with `rel="categories"`, `rel-id` set to the category ID, and `field="content"`, so category-level editing keeps working.

## 2. Where page templates live and how they resolve

```text
Templates/Store/StoreCategory/
+-- default.dwig     # fallback, keep it working
+-- wide.dwig
+-- ...
```

Resolution is theme-first, bundled-default-second per filename, with `default.dwig` as the fallback. Keep theme-specific category pages in the active theme.

## 3. What data the page receives

| Value | Contents |
|---|---|
| `data.category` | Current category: `id`, `title`, `description` (sanitized HTML, render with `\|raw`). Guard each with `\|default()`. |
| `data.products` | Category products in display order: `id`, `title`, `link`/`url`, `picture`/`pictures`, `price`, `offer_price`, `discount_percentage`, `instock`, `description`, `categories`, `content_data`. Always loop with `\|default([])` and keep an `{% else %}` empty branch. |

## 4. Page anatomy

Header from the category, card grid from the products, same card discipline as list skins:

```twig
<header class="text-center mx-auto mb-5">
    <h1>{{ data.category.title|default('Shop')|e }}</h1>
    {% if data.category.description|default('') %}
        <div class="lead text-body-secondary">{{ data.category.description|raw }}</div>
    {% endif %}
</header>

<div class="row row-cols-2 row-cols-lg-4 g-3" id="productGrid">
{% for product in data.products|default([]) %}
    {% set product_url = product.link|default(product.url|default('#')) %}
    {% set product_image = product.picture|default('') %}
    {% if not product_image and product.pictures|default([])|length %}
        {% set product_image = product.pictures[0].filename %}
    {% endif %}
    <div class="col">
    <article class="card h-100" data-id="{{ product.id|e('html_attr') }}">
        {% if product_image %}
            <a class="ratio ratio-1x1" href="{{ product_url|e('html_attr') }}">
                <img src="{{ thumbnail(product_image, 600, 600, true)|e('html_attr') }}"
                     alt="{{ product.title|default('')|e('html_attr') }}"
                     width="600" height="600" loading="lazy">
            </a>
        {% else %}
            <a class="ratio ratio-1x1 bg-body-secondary" href="{{ product_url|e('html_attr') }}">
                <span>Image unavailable</span>
            </a>
        {% endif %}
        <div class="card-body d-flex flex-column">
            <h2 class="fs-6"><a href="{{ product_url|e('html_attr') }}">{{ product.title|default('')|e }}</a></h2>
            <module type="Store/ProductReviews" id="category-reviews-{{ product.id }}"
                    product_id="{{ product.id }}" template="review-style-1.dwig" />
            <div class="price-row mb-3">
                {% if product.offer_price|default(0) %}
                    <del>{{ currency_format(product.price|default(0)) }}</del>
                    <strong>{{ currency_format(product.offer_price) }}</strong>
                {% else %}
                    <strong>{{ currency_format(product.price|default(0)) }}</strong>
                {% endif %}
            </div>
            <button class="btn btn-primary w-100 mt-auto" type="button" data-add="{{ product.id|e('html_attr') }}"
                    {% if product.instock is same as(false) %}disabled{% endif %}>
                {{ product.instock is same as(false) ? 'Out of stock' : 'Add to Cart' }}
            </button>
        </div>
    </article>
    </div>
{% else %}
    <div class="col-12"><div class="alert alert-info text-center mb-0">No products are available in this category yet.</div></div>
{% endfor %}
</div>
```

Rules:

1. Render everything from `data.category` and `data.products`. Hardcoded titles, images, prices, and links (static demo markup with stock-photo URLs and `product.html` hrefs) freeze the page for every category — the static variant is the top category-page bug to avoid.
2. Resolve images as `picture` first, first `pictures[].filename` second, placeholder third. Never output originals; always `thumbnail()` at the displayed size.
3. Format money only with `currency_format()` with the offer branch; gate buying on `instock is same as(false)`; keep `data-add` hooks per product.
4. Suffix nested review IDs per product and pass `product_id` explicitly.
5. Breadcrumb Home plus the category title (`aria-current="page"`), with `site_url()` for home — never a hardcoded index path.

## 5. Filters on category pages

Pair the grid with a `Store/Filter` module targeting the grid's list ID (see the filter module doc): subcategory checkboxes, keyword, price range, and sort, with mobile off-canvas and desktop aside sharing one wiring. Keep filter input IDs unique per surface; one renamed ID orphans that control while the rest keep working.

## 6. Use cases

**Standard category page.** Header, responsive card grid, empty-state alert. The default skin pattern.

**Hero-led category.** Banner image plus collection copy above the grid for flagship categories. Same data, editorial header.

**Filtered catalogue.** Grid plus filter aside and sort control for large categories. Same cards, narrowed sets.

**Subcategory landing.** Child-category cards above the product grid for parent categories. Compose `Store/StoreCategories` scoped to the current category, then the grid.

## 7. Common mistakes

- Static demo markup (hardcoded titles, stock URLs, fake prices, static hrefs) instead of looping page data. Every category then shows the same frozen page.
- Constructed product or home links instead of resolved values and `site_url()`.
- Unthumbnailed originals, hand-formatted money, live buttons on out-of-stock products.
- Shared nested review IDs across cards, or missing `product_id` on them.
- Missing `{% else %}` branch, breaking new categories with zero products.
- Dropping the `rel="categories"` edit container, breaking category-level editing.
