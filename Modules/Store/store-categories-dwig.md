# Store/StoreCategories — Category Cards and Showcases

The `Store/StoreCategories` module renders shop categories as cards, spotlights, or menu blocks. Its skins live in `Templates/Modules/Store/StoreCategories/`. Editors choose the source categories (or the module follows the current page context); the backend resolves each category's link, picture, item count, and custom fields and passes the ready list; the skin renders one card per category.

## 1. Embedding categories

```twig
<module
    type="Store/StoreCategories"
    id="home-categories"
    template="categories-style-1.dwig"
/>
```

- `type` is `Store/StoreCategories`: the path-style name that mirrors the skin path `Templates/Modules/Store/StoreCategories/`. The bare legacy name still resolves, but every example in these docs uses the path-style form.
- `id` identifies this instance and connects it to its category selection and display settings (source categories, subcategories toggle, header toggle). Keep it stable and unique per placement. Changing it orphans the selection saved under the old ID.
- `template` selects the skin filename from the categories directory. Pass only the filename, never a path. A template chosen in Live Edit is saved against the instance and takes precedence over the inline attribute.
- An unavailable file falls back to `default.dwig`, so the directory keeps a working `default.dwig`.
- With no matching categories, the module shows an editor notice instead of cards. Keep an `{% else %}` or `{% if %}` guard so the live page never renders a broken grid.

## 2. Where skins live and how they resolve

```text
Templates/Modules/Store/StoreCategories/
+-- default.dwig            # fallback, keep it working
+-- categories-style-1.dwig # image card grid
+-- singleCategory.dwig     # one large spotlight card
+-- megaMenu.dwig           # compact menu cards
+-- ...
```

Resolution order for every skin:

1. The same relative path in the active theme.
2. The bundled default template.
3. `default.dwig` in the same directory.

Keep theme-specific category skins in the active theme. Changing the bundled fallback changes the default for every generated theme.

## 3. What data a categories skin receives

| Value | Contents |
|---|---|
| `data.categories` | The category list in position order. Each item carries the fields below. Always loop with `\|default([])`. |
| `data.settings` | Instance scope: `parent` (comma-separated source IDs), `page`, `show_only_for_parent`, `show_category_header`, `show_subcategories`, `hide_pages`. |
| `data.params` | Tag attributes plus the instance ID (`data.params.id`). |

Category item fields:

| Field | Contents |
|---|---|
| `id` | Category ID. Feeds hooks and helper calls. |
| `title` | Category name. Escape with `\|e`. |
| `url` | Resolved category link. Prefer `category.url\|default(site_url())`; never construct links. |
| `picture` | Category image, or a product image fallback the backend resolved. Can still be empty — guard it. |
| `description` | Category copy. Optional; read with `\|default()`. |
| `content_items_count` | Product count for badges and labels. |
| `custom_data` | Site-specific fields (for example `short_desc`). Keys differ per site; always read with `\|default()`. |
| `children` | Nested subcategories where loaded. Guard with `\|default([])` before recursing. |

## 4. The card loop

One loop, one card per category: guarded image with fallback, resolved link, escaped title, optional count and custom blurb:

```twig
<div class="d-grid gap-3">
{% for category in data.categories|default([]) %}
    <a class="category-card card border-0" href="{{ category.url|default(site_url())|e('html_attr') }}">
        <img src="{{ category.picture|default(assets('images/categories/fallback.jpg'))|e('html_attr') }}"
             alt="{{ category.title|default('Category')|e('html_attr') }}"
             loading="lazy">
        <div class="card-img-overlay d-flex flex-column justify-content-end">
            <span>{{ category.custom_data.short_desc|default('Easy favourites')|e|capitalize }}</span>
            <h3 class="card-title">{{ category.title|default('')|e }}</h3>
            {% if category.content_items_count|default(0) %}
                <small>{{ category.content_items_count|e }} products</small>
            {% endif %}
            <b>Explore</b>
        </div>
    </a>
{% else %}
    <p>No categories available.</p>
{% endfor %}
</div>
```

Rules:

1. Resolve display values per card (`url`, `picture`, blurb) with fallbacks and reuse them. Theme-asset fallbacks load through `assets()`.
2. Guard `picture` with `|default()` to a real fallback image. Categories without photos are normal; the grid must hold alignment regardless.
3. Read `custom_data` keys with `|default()` and a sensible label. These keys are per-site configuration, not a stable contract.
4. Show counts only when truthy. A zero count badge advertises an empty shelf.
5. Escape titles, blurs, and attributes. `|raw` has no place in a categories skin.
6. Chunk with `|batch(n)` when the design pairs cards (`categories|batch(2)`), keeping order intact — never re-sort the backend's position order.

## 5. Scope and spotlight patterns

The instance settings scope the list; skins adapt to the scope they receive:

**Grid of siblings.** Default shape: loop every category the instance passes. Source categories come from the saved selection or page context; the skin renders whatever arrives.

**Single spotlight.** Take the first item for a large feature card beside the grid:

```twig
{% if data.categories|default([])|length > 0 %}
    {% set category = data.categories[0] %}
    <a class="category-card category-card--large" href="{{ category.url|default(site_url())|e('html_attr') }}">
        ...
    </a>
{% endif %}
```

**Parent-filtered menu.** Limit output to the configured parents from `data.settings.parent`:

```twig
{% set parent_list = data.settings.parent|default('')|split(',') %}
{% for category in data.categories|default([]) %}
    {% if category.id in parent_list %}
        ...
    {% endif %}
{% endfor %}
```

Rules:

1. Prefer instance settings for scoping over skin conditionals. A skin that second-guesses the selection fights the editor.
2. When two presentations share one section (spotlight plus grid, as in the Layouts composite pattern), use two instances with distinct IDs and skins (`singleCategory.dwig`, `categories-style-1.dwig`) rather than slicing one list awkwardly.
3. Respect `show_subcategories` intent: skins that ignore children must not claim to be navigation; skins that render children recurse with the same guards as section 4.

## 6. Use cases

**Homepage category grid.** Image cards with blurb overlay and explore CTA, chunked in pairs. The catalogue front door.

**Featured-plus-grid showcase.** `singleCategory.dwig` spotlight beside `categories-style-1.dwig` grid in one layout section, each on its own instance ID.

**Mega-menu block.** Compact image-plus-title cards filtered to top parents, no blurb, no overlay. Navigation support, not a shopping surface.

**Category landing header.** Single spotlight card for the current category above its product list, reusing the same instance scope as the page.

**Footer category links.** Text-only skin (title links, no images) for sitemap-style footers. Same data, minimal markup.

**Seasonal collection row.** Hand-scoped instance for a campaign (festive, clearance) with its own skin. Remove the instance when the campaign ends.

## 7. Common mistakes

- Constructing category links instead of using resolved `category.url`.
- Rendering `picture` unguarded, breaking the grid on imageless categories.
- Reading `custom_data.<key>` without `|default()`, breaking the skin where that site field is missing.
- Re-sorting the backend's position order in the skin.
- One shared instance ID for spotlight and grid, merging two selections into one settings scope.
- Overriding the editor's scope with skin-side filtering that hides selected categories.
- Recursing into `children` without `|default([])`.
- Using this module for blog categories (the blog categories module) or product lists (`Store/Products`).
- Forgetting `default.dwig`, so a missing skin selection breaks category surfaces site-wide.
