# Website/Posts — Blog Feeds and Article Grids

The `Website/Posts` module renders the blog feed: article cards in reverse-chronological order with images, excerpts, dates, and read-more links. Its skins live in `Templates/Modules/Website/Posts/`. Editors choose the source (blog page, category) and page size; the backend queries through the shared content pipeline and passes the list with pagination; the skin renders the editorial feed.

Do not confuse it with its neighbors. `Content/Content` is the generic record list for any content type. `Store/Products` renders products with pricing and stock. This module is the blog feed with editorial chrome: dates, excerpts, featured treatment.

## 1. Embedding a blog feed

```twig
<module
    type="Website/Posts"
    id="blog-feed"
    template="default.dwig"
/>
```

- `type` is `Website/Posts`: the path-style name that mirrors the skin path `Templates/Modules/Website/Posts/`. The bare legacy name still resolves, but every example in these docs uses the path-style form.
- `id` identifies this instance and connects it to its source and display settings. Keep it stable and unique per feed. Renaming it breaks pagination and AJAX reloads that address the list.
- `template` selects the skin filename from the posts directory. Pass only the filename, never a path. A template chosen in Live Edit is saved against the instance and takes precedence over the inline attribute.
- An unavailable file falls back to `default.dwig`, so the directory keeps a working `default.dwig`.

Source and paging settings mirror the shared list behavior: blog page or category source, `limit` per page, and `paginate` for multi-page archives.

## 2. Where skins live and how they resolve

```text
Templates/Modules/Website/Posts/
+-- default.dwig     # fallback, keep it working
+-- featured.dwig
+-- ...
```

Resolution order for every skin:

1. The same relative path in the active theme.
2. The bundled default template.
3. `default.dwig` in the same directory.

Keep theme-specific feeds in the active theme. The bundled skins ship as empty stubs, so writing the blog skins is part of building the theme — follow the feed pattern in section 4 so the first skin you write already behaves like the module's native output.

## 3. What data a posts skin receives

| Value | Contents |
|---|---|
| `data.posts` | Articles in display order (newest first unless configured otherwise). Each item carries the fields below. Always loop with `\|default([])` and keep an `{% else %}` empty branch. |
| `data.pagination` | Paging info: `pages_count` and `paging_param`. |
| `data.params` | Tag attributes plus the instance ID (`data.params.id`). |

Article fields used by skins:

| Field | Contents |
|---|---|
| `id` | Article ID. Feeds hooks and helper calls. |
| `title` | Article title. Escape with `\|e`. |
| `link` / `url` | Resolved article URL. Prefer `link`, fall back to `url`, then `'#'`. Never construct URLs. |
| `description` | Sanitized body excerpt HTML. Render with `\|raw`; shorten with slice, never mid-tag assumptions. |
| `image` | Cover image source where the article carries one. Guard it; text-only posts are normal. |
| `created_at` | Publish timestamp for the date line. Guard it; format once per row. |

## 4. The feed pattern

Featured first article, uniform cards after, date lines, excerpts, conditional pager:

```twig
{% set posts = data.posts|default([]) %}

{% if posts is not empty %}
    {% set featured = posts|first %}
    {% set featured_url = featured.link|default(featured.url|default('#')) %}
    <article class="featured-post card mb-4">
        {% if featured.image|default('') %}
            <a href="{{ featured_url|e('html_attr') }}">
                <img src="{{ thumbnail(featured.image, 1200, 630)|e('html_attr') }}"
                     alt="{{ featured.title|default('')|e('html_attr') }}" loading="eager">
            </a>
        {% endif %}
        <div class="card-body">
            {% if featured.created_at|default('') %}<small>{{ featured.created_at|e }}</small>{% endif %}
            <h2><a href="{{ featured_url|e('html_attr') }}">{{ featured.title|default('Untitled')|e }}</a></h2>
            {% if featured.description|default('') %}
                <p>{{ featured.description|slice(0, 200) }}{% if featured.description|length > 200 %}...{% endif %}</p>
            {% endif %}
        </div>
    </article>

    <div class="row row-cols-1 row-cols-md-2 g-4">
    {% for post in posts|slice(1) %}
        {% set post_url = post.link|default(post.url|default('#')) %}
        <div class="col">
        <article class="card h-100">
            {% if post.image|default('') %}
                <a href="{{ post_url|e('html_attr') }}">
                    <img src="{{ thumbnail(post.image, 800, 500)|e('html_attr') }}"
                         alt="{{ post.title|default('')|e('html_attr') }}" loading="lazy">
                </a>
            {% endif %}
            <div class="card-body d-flex flex-column">
                {% if post.created_at|default('') %}<small>{{ post.created_at|e }}</small>{% endif %}
                <h3 class="h5"><a href="{{ post_url|e('html_attr') }}">{{ post.title|default('Untitled')|e }}</a></h3>
                <a class="mt-auto" href="{{ post_url|e('html_attr') }}">Read more</a>
            </div>
        </article>
        </div>
    {% endfor %}
    </div>

    {% if data.pagination.pages_count|default(1) > 1 %}
        {# ...pager controls bound to the list ID... #}
    {% endif %}
{% else %}
    <p class="text-muted">No articles yet. Check back soon.</p>
{% endif %}
```

Rules:

1. Treat the first item as featured only when the skin is designed for it. Do not special-case position one in a uniform grid — the exception must be visible in the design, not hidden in code.
2. Resolve each row's URL once and reuse it for image, title, and read-more links.
3. Guard images with `{% if %}`. Text-only posts must hold layout like any other card.
4. Render excerpts with `|raw` only as full sanitized HTML, shortened by slice with an ellipsis guard. Never escape body HTML into visible tags, and never `|raw` titles or dates.
5. Render pager controls only when `pages_count > 1`. Single-page feeds with dead pager buttons look broken.
6. Load the featured image `eager`, the rest `lazy`. The lead image is usually the page's largest paint.

## 5. Use cases

**Blog index.** Featured lead plus paged card grid with dates and excerpts. The editorial front door.

**Category archive.** Same feed scoped to one category, with the category name as the section heading. Scope comes from the instance.

**Related articles.** Unpaged short list under posts, scoped by the current article's categories or tags. Same card skin, compact grid.

**Sidebar latest.** Title-and-date rows without images or excerpts for narrow columns. Same data, minimal markup.

## 6. Common mistakes

- Looping `data.posts` without `|default([])` or an `{% else %}` branch, breaking new blogs with zero posts.
- Constructing article URLs instead of using resolved `link`/`url`.
- Escaping body excerpts into visible tags, or applying `|raw` to titles and dates.
- Featuring position one in a skin designed as a uniform grid (or vice versa).
- Pager controls on single-page feeds, or a renamed list ID orphaning pagination and reloads.
- Using this feed for products with pricing (`Store/Products`) or generic mixed records (`Content/Content`).
- Forgetting `default.dwig`, so a missing skin selection breaks the blog site-wide.
