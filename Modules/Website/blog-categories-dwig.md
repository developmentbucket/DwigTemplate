# Website/BlogCategories — Blog Category Cards and Indexes

The `Website/BlogCategories` module renders blog categories as cards, lists, or menu blocks. Its skins live in `Templates/Modules/Website/BlogCategories/`. Editors choose the source categories (or the module follows the current page context); the backend resolves each category's link, picture, and article count and passes the ready list; the skin renders one entry per category.

Do not confuse it with its neighbors. `Store/StoreCategories` renders shop categories with prices, offers, and stock states. `Navigation/Menu` renders hand-arranged link trees. This module renders the blog's category structure with article counts and reading-oriented cards.

## 1. Embedding blog categories

```twig
<module
    type="Website/BlogCategories"
    id="blog-categories"
    template="default.dwig"
/>
```

- `type` is `Website/BlogCategories`: the path-style name that mirrors the skin path `Templates/Modules/Website/BlogCategories/`. The bare legacy name still resolves, but every example in these docs uses the path-style form.
- `id` identifies this instance and connects it to its category selection and display settings (source categories, subcategories toggle, header toggle). Keep it stable and unique per placement. Changing it orphans the selection saved under the old ID.
- `template` selects the skin filename from the blog-categories directory. Pass only the filename, never a path. A template chosen in Live Edit is saved against the instance and takes precedence over the inline attribute.
- An unavailable file falls back to `default.dwig`, so the directory keeps a working `default.dwig`.
- With no matching categories, the module shows an editor notice instead of cards. Keep an `{% else %}` or `{% if %}` guard so the live page never renders a broken grid.

## 2. Where skins live and how they resolve

```text
Templates/Modules/Website/BlogCategories/
+-- default.dwig     # fallback, keep it working
+-- cards.dwig
+-- list.dwig
+-- ...
```

Resolution order for every skin:

1. The same relative path in the active theme.
2. The bundled default template.
3. `default.dwig` in the same directory.

Keep theme-specific category skins in the active theme. The bundled skins ship as empty stubs, so writing the blog-category skins is part of building the theme — follow the card pattern in section 4 so the first skin you write already behaves like the module's native output.

## 3. What data a blog-categories skin receives

| Value | Contents |
|---|---|
| `data.categories` | Categories in position order. Each item carries the fields below. Always loop with `\|default([])`. |
| `data.settings` | Instance scope: `parent` (source IDs), `page`, `show_only_for_parent`, `show_category_header`, `show_subcategories`, `hide_pages`. |
| `data.params` | Tag attributes plus the instance ID (`data.params.id`). |

Category item fields:

| Field | Contents |
|---|---|
| `id` | Category ID. Feeds hooks and helper calls. |
| `title` | Category name. Escape with `\|e`. |
| `url` | Resolved category link. Prefer `category.url\|default(site_url())`; never construct links. |
| `picture` | Category image, or a resolved fallback. Can still be empty — guard it. |
| `content_items_count` | Article count for labels and badges. |
| `children` | Nested subcategories where loaded. Guard with `\|default([])` before recursing. |
| `is_page` | True for page entries mixed into the list. Honor `hide_pages` intent: page entries render only where pages belong. |

## 4. The category card loop

One card per category: guarded image, resolved link, escaped title, conditional article count:

```twig
<div class="row row-cols-1 row-cols-md-3 g-4">
{% for category in data.categories|default([]) %}
    <div class="col">
    <a class="category-card card h-100 border-0" href="{{ category.url|default(site_url())|e('html_attr') }}">
        {% if category.picture|default('') %}
            <img src="{{ category.picture|e('html_attr') }}"
                 alt="{{ category.title|default('Category')|e('html_attr') }}"
                 loading="lazy">
        {% endif %}
        <div class="card-body">
            <h3 class="card-title h5">{{ category.title|default('')|e }}</h3>
            {% if category.content_items_count|default(0) %}
                <small class="text-body-secondary">{{ category.content_items_count|e }} articles</small>
            {% endif %}
        </div>
    </a>
    </div>
{% else %}
    <div class="col-12"><p class="text-muted">No categories available.</p></div>
{% endfor %}
</div>
```

Rules:

1. Resolve `url` per card with the site-root fallback and reuse it. Category links change with slugs and locales; constructed links rot.
2. Guard `picture` with `{% if %}`. Imageless categories are normal; the grid must hold alignment regardless.
3. Show counts only when truthy, labeled as articles. A zero-count badge advertises an empty shelf.
4. Escape titles and attributes. Category names are editor input; `|raw` has no place in these skins.
5. Never re-sort the backend's position order. Editorial ordering lives in the category manager, not the skin.
6. Recurse into `children` only with `|default([])` guards, and only in skins designed as trees. Card grids stay flat.

## 5. Use cases

**Blog index topics.** Card grid of top-level categories with images and article counts above the feed. The discovery front door.

**Sidebar topic list.** Compact title-plus-count rows in blog sidebars. Same data, text-first markup, no images.

**Footer topics.** Flat link columns for sitemap-style footers. Minimal skin, depth one, no counts.

**Series or collection landing.** Scoped instance showing one parent's children as chapter cards. Scope comes from the instance settings.

**Empty-state guidance.** The `{% else %}` branch linking to the full archive instead of dead-ending new blogs.

## 6. Common mistakes

- Constructing category links instead of using resolved `category.url`.
- Rendering `picture` unguarded, breaking the grid on imageless categories.
- Showing zero-count badges on empty categories.
- Re-sorting the backend's position order in the skin.
- Recursing into `children` without `|default([])`, or tree markup in a flat card grid.
- Mixing page entries into category surfaces where `hide_pages` says otherwise.
- Using this module for shop categories (`Store/StoreCategories` prices and stock don't exist here) or hand-arranged links (`Navigation/Menu`).
- Forgetting `default.dwig`, so a missing skin selection breaks category surfaces site-wide.
