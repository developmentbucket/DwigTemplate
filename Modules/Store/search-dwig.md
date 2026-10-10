# Store/Search — Search Box and Live Results

The `Store/Search` module renders product search: a search box plus live results for products and categories. Its skins span two directories. `Templates/Modules/Store/Search/` holds the module widget skins embedded with `<module>` tags. `Templates/Store/Search/` holds the results fragments the search API renders per keystroke. The backend searches products and categories and passes grouped matches; skins render the box, the rows, and the empty state.

## 1. Embedding a search box

```twig
<module
    type="Store/Search"
    id="header-search"
    template="default.dwig"
/>
```

- `type` is `Store/Search`: the path-style name that mirrors the module skin path `Templates/Modules/Store/Search/`. The bare legacy name still resolves, but every example in these docs uses the path-style form.
- `id` identifies this instance and connects it to its settings. Header, overlay, and mobile boxes use different IDs (`header-search`, `overlay-search`) so each keeps its own configuration. Changing an ID orphans the settings saved under the old one.
- `template` selects the skin filename from the module search directory. Pass only the filename, never a path. A template chosen in Live Edit is saved against the instance and takes precedence over the inline attribute.
- An unavailable file falls back to `default.dwig`, so the directory keeps a working `default.dwig`.

## 2. The two skin locations

```text
Templates/Modules/Store/Search/
+-- default.dwig     # the search box widget (input + results holder)

Templates/Store/Search/
+-- default.dwig     # results fragment rendered per search request
```

Resolution is theme-first, bundled-default-second in both directories, with `default.dwig` as the fallback.

Never extend a document layout from either file. The box renders inside pages; results fragments render inside the box's results holder over AJAX. Neither is a full page.

## 3. The search box pattern

The box is an input plus an empty results holder, both with IDs unique per instance:

```twig
{% set box_id = data.params.id|default('site-search') %}

<div class="search-box w-100" id="search-box-{{ box_id|e('html_attr') }}">
    <input type="text"
           id="search-field-{{ box_id|e('html_attr') }}"
           class="form-control w-100"
           placeholder="Search products"
           autocomplete="off"
           aria-label="Search products"
           aria-expanded="false"
           aria-controls="search-results-{{ box_id|e('html_attr') }}">
    <div class="search-results" id="search-results-{{ box_id|e('html_attr') }}" hidden></div>
</div>
```

Rules:

1. Derive both IDs from the instance ID. Two boxes on one page (header plus overlay) must never share an input or holder identity.
2. Pair the input with its holder through `aria-controls`, and flip `aria-expanded` with visibility. Announce result counts through an `aria-live` region so the dropdown is operable beyond the mouse.
3. Keep `autocomplete="off"`. Browser history dropdowns fight the results panel for the same space.
4. The holder starts empty and hidden. Results arrive only from typed keywords (section 4); never pre-fill it server-side.

## 4. Searching through window.dbEvent

Keystrokes call `window.dbEvent.shop.search` from `userfiles/modules/developmentbucket/db_lib/events/common.js`, which loads automatically on normal frontend pages. Never reference the `mw` JS library:

```js
window.dbEvent.shop.search({
    keyword: 'wireless headphones',
    limit: 10,
    template: 'Store/Search/default.dwig'
});
```

- `keyword` is required and trimmed. Skip the call on empty input and hide the holder instead.
- `limit` defaults to `10`, clamped between `1` and `50`.
- `template` optionally names the results fragment the server renders (a `Templates/`-rooted path such as `Store/Search/default.dwig`). The rendered HTML arrives as `response.html` (also `response.data.html`).
- Without `template`, the response carries raw groups instead: `response.data.products` and `response.data.categories`.

Rules:

1. Debounce input (~200–300 ms) so fast typing fires one request per pause, not one per keystroke.
2. Guard against out-of-order responses: number each request and render only the latest, so a slow early keystroke never overwrites fresher results.
3. Every call uses `try/catch` because failures reject. On error show a plain "Search is temporarily unavailable" message, never raw error internals.
4. Clear and hide the holder on empty input and on Escape; return focus to the input.

## 5. The results fragment

Results fragments receive grouped matches and render product rows, category rows, and one shared empty state:

```twig
<div class="shop-search-results">
{% if data.products|default([]) %}
    <section aria-label="Products">
    {% for product in data.products %}
        <a href="{{ product.url|default('#')|e('html_attr') }}" class="search-row">
            {% if product.thumbnail|default('') %}
                <img src="{{ product.thumbnail|e('html_attr') }}"
                     alt="{{ product.title|default('')|e('html_attr') }}"
                     width="64" height="64" loading="lazy">
            {% endif %}
            <span>
                <b>{{ product.title|default('')|e }}</b>
                <small>{{ product.category|default('Product')|e }}</small>
            </span>
            <strong>
            {% if product.has_offer|default(false) %}
                {{ currency_format(product.offer_price) }}
            {% else %}
                {{ currency_format(product.price|default(0)) }}
            {% endif %}
            </strong>
        </a>
    {% endfor %}
    </section>
{% endif %}

{% if data.categories|default([]) %}
    <section aria-label="Categories">
    {% for category in data.categories %}
        <a href="{{ category.url|default('#')|e('html_attr') }}" class="search-row">
            <b>{{ category.title|default('')|e }}</b>
            <small>Category</small>
        </a>
    {% endfor %}
    </section>
{% endif %}

{% if not data.products|default([]) and not data.categories|default([]) %}
    <p class="text-muted">No products or categories found.</p>
{% endif %}
</div>
```

Row data used by skins:

| Group | Fields |
|---|---|
| `data.products` items | `id`, `title`, `url`, `thumbnail`, `category`, `price`, `offer_price`, `has_offer` |
| `data.categories` items | `title`, `url`, `thumbnail` (often empty — guard it) |

Rules:

1. Guard `thumbnail` with `{% if %}` in both groups. Categories frequently have no image; a broken-image icon in results looks like a bug.
2. Format money only with `currency_format()`, choosing `offer_price` under `has_offer` and `price` otherwise. Never concatenate symbols by hand.
3. Link rows to resolved `url` values, never constructed paths.
4. Keep exactly one empty state for both groups together. Separate "no products" plus "no categories" messages double the noise for one empty result.
5. Escape titles and attributes. `|raw` has no place in a search skin: keywords echo back through these rows, and unescaped echoes are reflected markup injection.

## 6. Use cases

**Header search.** Compact box in the header partial on its own `header-search` ID, dropdown results under the input, Escape to dismiss.

**Search overlay.** Full-width panel with a large input reusing the same results fragment. Different box ID, same fragment path.

**Mobile search.** Box inside the mobile menu drawer. Same call shape and fragment; narrower CSS only.

**Category-scoped search.** Box on a category landing page whose calls add the category constraint. Same fragment, pre-filtered groups.

**Empty-query browse.** Some shops show bestsellers in the open-but-empty holder. Explicit design choice, not a default: the holder stays empty unless the theme implements it.

## 7. Common mistakes

- Sharing input or holder IDs between two boxes, so typing in one fills the other.
- Firing one request per keystroke with no debounce, hammering the endpoint.
- Rendering out-of-order responses, letting stale results overwrite fresh ones.
- Requesting `template` with a `Modules/`-rooted path. Result fragments resolve from the `Templates/` root (`Store/Search/default.dwig`).
- Rendering `thumbnail` unguarded, showing broken images for imageless categories.
- Hand-formatted prices instead of `currency_format()` with the `has_offer` branch.
- Two empty states (one per group) instead of one shared message.
- Escaping gaps where the typed keyword echoes into rows. Escape every reflected value.
- Rebuilding keyword search inside `Store/Filter`. Filters narrow; this module finds.
- Forgetting `default.dwig` in either directory, so a missing skin breaks the box or every keystroke site-wide.
