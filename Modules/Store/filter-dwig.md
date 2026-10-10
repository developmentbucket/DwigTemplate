# Store/Filter — Product Filter Controls

The `Store/Filter` module renders the controls that narrow a product list: category checkboxes, attribute groups, sort order, reset. Its skins live in `Templates/Modules/Store/Filter/`. The backend passes the shop category tree and the product attribute tree; the skin renders the controls and reloads a `Store/Products` list through `window.dbEvent.shop.filter()` without a page refresh.

Do not confuse it with its neighbors. `Store/Products` renders the list this module drives. `Store/Search` finds products by keywords. This module narrows the visible set by category, attribute, stock, and sort.

## 1. Embedding a filter

```twig
<module
    type="Store/Filter"
    id="shop-filters"
    target="#shop-products-list"
    template="default.dwig"
/>
```

- `type` is `Store/Filter`: the path-style name that mirrors the skin path `Templates/Modules/Store/Filter/`. The bare legacy name still resolves, but every example in these docs uses the path-style form.
- `id` identifies this instance and connects it to its settings (visible categories, match mode, templates). Keep it stable and unique per filter panel.
- `target` is the CSS selector of the `Store/Products` list this filter drives. The filter and the list pair by this address, so both IDs must stay stable and matching.
- `template` selects the skin filename from the filter directory. Pass only the filename, never a path. A template chosen in Live Edit is saved against the instance and takes precedence over the inline attribute.
- An unavailable file falls back to `default.dwig`, so the directory keeps a working `default.dwig`.

Supporting attributes:

| Attribute | Purpose |
|---|---|
| `target` / `data-target` | Selector of the driven products list. Default `#shop-products-list`. |
| `product-template` / `data-product-template` | Skin filename used to render reloaded list cards. Default `default.dwig`. Match it to the list's own skin family so reloaded cards look identical. |

## 2. Where skins live and how they resolve

```text
Templates/Modules/Store/Filter/
+-- default.dwig     # fallback, keep it working
+-- compact.dwig
+-- ...
```

Resolution order for every skin:

1. The same relative path in the active theme.
2. The bundled default template.
3. `default.dwig` in the same directory.

Keep theme-specific filters in the active theme. Changing the bundled fallback changes the default for every generated theme.

## 3. What data a filter skin receives

| Value | Contents |
|---|---|
| `data.categories` | Shop category tree. Each node carries `id`, `title`, and `children` (same shape, recursively). Already limited to the categories the instance allows. |
| `data.attributes` | Product attribute tree for attribute groups (color, size, brand, and similar). Same nested shape as categories. |
| `data.params` | Tag attributes plus the instance ID (`data.params.id`), including `target`, `product-template`, and `match_mode`. |

The safe pattern for any filter:

```twig
{% set params = data.params|default({}) %}
{% set categories = data.categories|default([]) %}
{% set target = params.target|default(params['data-target']|default('#shop-products-list')) %}
{% set product_template = params['product-template']|default(params['data-product-template']|default('default.dwig')) %}
{% set filter_id = params.id|default('shop-category-filter') %}
```

Auto-escaping is off in `.dwig` templates: escape titles with `|e` and IDs, values, and selectors with `|e('html_attr')`.

## 4. Rendering the category tree

Categories nest, so skins render them with a recursive macro. Prefix every input ID with the filter ID so two filter panels on one page never share an input identity:

```twig
{% macro category_options(categories, level, prefix) %}
    {% import _self as filter %}
    {% for category in categories %}
        <li class="mb-2" style="padding-inline-start: {{ (level * 16)|e('html_attr') }}px">
            <div class="form-check">
                <input class="form-check-input" type="checkbox" name="categories"
                       value="{{ category.id|e('html_attr') }}"
                       id="{{ prefix|e('html_attr') }}-category-{{ category.id|e('html_attr') }}">
                <label class="form-check-label"
                       for="{{ prefix|e('html_attr') }}-category-{{ category.id|e('html_attr') }}">
                    {{ category.title|default('Category')|e }}
                </label>
            </div>
            {% if category.children|default([]) %}
                <ul class="list-unstyled mt-2 mb-0">
                    {{ filter.category_options(category.children, level + 1, prefix) }}
                </ul>
            {% endif %}
        </li>
    {% endfor %}
{% endmacro %}

{% import _self as filter %}
<form data-filter-form>
    <ul class="list-unstyled mb-0">
        {{ filter.category_options(categories, 0, filter_id) }}
    </ul>
</form>
```

Rules:

1. Test `category.children|default([])` before recursing. Leaf nodes carry no children; unguarded recursion breaks them.
2. Pair every `id` with its `for`. Duplicated or missing pairs toggle the wrong checkbox.
3. Keep the checkbox name exactly `categories`. The reload script collects `input[name="categories"]:checked`; renaming breaks selection reading while the boxes look fine.
4. Render attribute groups from `data.attributes` with the same macro shape and one name per group, so selections stay distinguishable per attribute.

## 5. Wiring the panel to the list

The wrapper carries the pairing as `data-*` attributes, plus reset and a live status region:

```twig
<div class="shop-category-filter"
     id="{{ filter_id|e('html_attr') }}-controls"
     data-target="{{ target|e('html_attr') }}"
     data-product-template="{{ product_template|e('html_attr') }}">
    <div class="d-flex align-items-center justify-content-between mb-3">
        <h2 class="h5 mb-0">Categories</h2>
        <button class="btn btn-link btn-sm" type="button" data-filter-reset>Clear</button>
    </div>
    <form data-filter-form><!-- ...tree... --></form>
    <div class="small text-muted mt-3" data-filter-status aria-live="polite"></div>
</div>
```

Reload through `window.dbEvent` from `userfiles/modules/developmentbucket/db_lib/events/common.js`, which loads automatically on normal frontend pages. Never reference the `mw` JS library:

```js
window.dbEvent.shop.filter({
    target: target,                       // '#shop-products-list'
    categories: selectedCategories(),     // [12, 18]
    page: 1,
    limit: 24,
    sort: 'date_desc',
    template: productTemplate             // card skin for reloaded rows
}).then(function (response) {
    var total = response && response.data ? response.data.total : 0;
    status.textContent = total + (total === 1 ? ' product' : ' products');
}).catch(function () {
    status.textContent = 'Products could not be loaded. Please try again.';
});
```

Call defaults added by the client: `page` 1, `limit` 20, `sort` `date_desc`, `stock_status` `all`, `template` `default.dwig`, `categories` `[]`.

Rules:

1. Reload on form `change` (resetting to page 1) and on reset (clear first, then reload). Every call uses `then/catch`: update the count from `response.data.total` on success and show a retry message on failure.
2. Guard the script with an initialized flag (`root.dataset.initialized`) looked up by the unique wrapper ID, so AJAX re-renders never bind handlers twice.
3. Keep the `data-filter-form`, `data-filter-reset`, and `data-filter-status` hooks spelled exactly so. The script addresses the panel through them.
4. Announce results through the `aria-live` status region ("N products", loading, errors). Silent reloads strand screen-reader and keyboard users.
5. Keep `product-template` matched to the driven list's skin family. A mismatched card skin makes filtered results look like a different shop.

## 6. Use cases

**Shop sidebar.** Full category tree plus attribute groups in an `<aside>`, driving the paged grid beside it. The standard catalogue pairing from the products doc.

**Mobile filter drawer.** Same panel inside an off-canvas drawer, same target. One instance drives both desktop and mobile when the markup is shared; two instances need two IDs and must not fight over the list.

**Category landing.** Pre-scoped tree (instance category visibility) above a curated row. Editors bound the tree; the skin renders whatever arrives.

**Compact top bar.** Sort control plus a single-level category row above the grid for small catalogs. Same data, horizontal CSS, no children recursion visible.

**Stock and offer narrowing.** `stock_status` and offer-aware sorts (`price_asc` and siblings) for sale and clearance placements. Same call shape, different arguments.

## 7. Common mistakes

- Pointing `target` at a renamed or missing list ID, so filtering silently updates nothing.
- Reloading cards with a mismatched `product-template`, so filtered results change appearance.
- Renaming the `categories` checkbox name or the `data-filter-*` hooks, breaking selection reading while controls look fine.
- Recursing into `children` without `|default([])`, breaking leaf categories.
- Duplicated input IDs across two filter panels, toggling the wrong boxes.
- Forgetting the initialized guard, stacking duplicate change handlers after every re-render.
- Silent reloads with no status region or error message.
- Rebuilding search inside the filter instead of using `Store/Search` for keywords.
- Forgetting `default.dwig`, so a missing skin selection breaks filtering site-wide.
