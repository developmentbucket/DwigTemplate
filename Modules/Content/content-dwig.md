# Content/Content — Generic Content Lists

The `Content/Content` module renders a list of content records: posts, pages, or any content type the instance selects. Its skins live in `Templates/Modules/Content/Content/`. Editors choose the content type and source; the backend queries through the shared content pipeline and passes the list with pagination; the skin renders one card per record.

Do not confuse it with its neighbors. `Website/Posts` renders the blog feed with blog chrome. `Store/Products` renders products with pricing and stock. This module is the generic list for content records of any selected type.

## 1. Embedding a content list

```twig
<module
    type="Content/Content"
    id="news-list"
    content_type="post"
    template="default.dwig"
/>
```

- `type` is `Content/Content`: the path-style name that mirrors the skin path `Templates/Modules/Content/Content/`. The bare legacy name still resolves, but every example in these docs uses the path-style form.
- `id` identifies this instance and connects it to its source and display settings. Keep it stable and unique per list. Renaming it breaks pagination and AJAX reloads that address the list.
- `content_type` selects the record type (`post` by default; `page` and other types where configured). The saved setting wins over the attribute.
- `template` selects the skin filename from the content directory. Pass only the filename, never a path. A template chosen in Live Edit is saved against the instance and takes precedence over the inline attribute.
- An unavailable file falls back to `default.dwig`, so the directory keeps a working `default.dwig`.

Source and paging settings mirror the shared list behavior: category or page source, `limit` per page, and `paginate` for multi-page records.

## 2. Where skins live and how they resolve

```text
Templates/Modules/Content/Content/
+-- default.dwig     # fallback, keep it working
+-- cards.dwig
+-- ...
```

Resolution order for every skin:

1. The same relative path in the active theme.
2. The bundled default template.
3. `default.dwig` in the same directory.

Keep theme-specific lists in the active theme. The bundled skins ship as empty stubs, so writing the content-list skins is part of building the theme — follow the card loop in section 4 so the first skin you write already behaves like the module's native output.

## 3. What data a content list skin receives

| Value | Contents |
|---|---|
| `data.posts` | Content records in display order. Each item carries the fields below. Always loop with `\|default([])` and keep an `{% else %}` empty branch. |
| `data.pagination` | Paging info: `pages_count` and `paging_param`. |
| `data.params` | Tag attributes plus the instance ID (`data.params.id`). |

Record fields used by skins:

| Field | Contents |
|---|---|
| `id` | Record ID. Feeds hooks and helper calls. |
| `title` | Record title. Escape with `\|e`. |
| `link` / `url` | Resolved record URL. Prefer `link`, fall back to `url`, then `'#'`. Never construct URLs. |
| `description` | Sanitized body excerpt HTML. Render with `\|raw`; shorten with slice, never mid-tag assumptions. |
| `image` | Lead image source where the record carries one. Guard it; records without images are normal. |

## 4. The card loop

One loop, one card per record: guarded image, linked title, excerpt, read-more link, paging-aware empty branch:

```twig
<div class="row row-cols-1 row-cols-md-3 g-4">
{% for post in data.posts|default([]) %}
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
            <h3 class="h5"><a href="{{ post_url|e('html_attr') }}">{{ post.title|default('Untitled')|e }}</a></h3>
            {% if post.description|default('') %}
                <p>{{ post.description|slice(0, 160) }}{% if post.description|length > 160 %}...{% endif %}</p>
            {% endif %}
            <a class="mt-auto" href="{{ post_url|e('html_attr') }}">Read more</a>
        </div>
    </article>
    </div>
{% else %}
    <div class="col-12"><p class="text-muted">No content yet.</p></div>
{% endfor %}
</div>
```

Rules:

1. Resolve `post_url` once per row and reuse it for image, title, and read-more links. Scattered fallbacks drift apart.
2. Guard images with `{% if %}`. Imageless records must hold grid alignment like any other card.
3. Render `description` excerpts with `|raw` only as full sanitized HTML, shortened by character slice with an ellipsis guard. Never escape body HTML into visible tags, and never inject `|raw` on titles.
4. Render paging controls only when `data.pagination.pages_count > 1`. Single-page lists with dead pager buttons look broken.
5. Escape titles and attributes. `|raw` applies to body excerpts only.

## 5. Use cases

**News or journal grid.** Paged post cards with images, excerpts, and read-more links. The editorial workhorse.

**Related reading.** Unpaged short list under articles, scoped by the current record's categories. Same card skin, narrower grid.

**Page directory.** `content_type="page"` list for sitemap-style or index pages. Titles and links dominate; images optional.

**Search-adjacent listing.** Curated record rows on landing pages with hand-picked sources. Limit, don't paginate.

## 6. Common mistakes

- Looping `data.posts` without `|default([])` or an `{% else %}` branch, breaking new or empty sources.
- Constructing record URLs instead of using resolved `link`/`url`.
- Escaping body excerpts into visible tags, or applying `|raw` to titles.
- Pager controls on single-page lists, or a renamed list ID orphaning pagination and reloads.
- Using this generic list where the specialized module fits: blog feeds (`Website/Posts`), products with pricing (`Store/Products`), people (`Content/TeamCard`).
- Forgetting `default.dwig`, so a missing skin selection breaks content lists site-wide.
