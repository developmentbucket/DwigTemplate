# Website/BlogCategory — Blog Category Listing Pages

The `Templates/Website/BlogCategory/` templates render a blog category page: the category header plus the grid of its posts. `default.dwig` is the fallback every blog category resolves to unless the category selects another layout.

## 1. Category page shape

```twig
{% extends "Layouts/main.dwig" %}

{% block content %}
<main class="container py-5">
    <header class="text-center mb-5">
        <h1>{{ data.category.title|default('Stories')|e }}</h1>
        {% if data.category.description|default('') %}
            <div class="text-body-secondary">{{ data.category.description|raw }}</div>
        {% endif %}
    </header>

    <div class="row row-cols-1 row-cols-md-2 row-cols-lg-3 g-4">
    {% for post in data.posts|default([]) %}
        {% set post_url = post.link|default('#') %}
        <div class="col">
            <article class="card h-100 border-0 shadow-sm overflow-hidden">
                {% if post.picture|default('') %}
                    <a class="ratio ratio-16x9" href="{{ post_url|e('html_attr') }}">
                        <img class="card-img-top object-fit-cover"
                             src="{{ thumbnail(post.picture, 720, 405)|e('html_attr') }}"
                             alt="{{ post.title|default('')|e('html_attr') }}" loading="lazy">
                    </a>
                {% endif %}
                <div class="card-body d-flex flex-column">
                    <h2 class="card-title h4">
                        <a class="text-decoration-none stretched-link" href="{{ post_url|e('html_attr') }}">{{ post.title|default('Untitled')|e }}</a>
                    </h2>
                    {% if post.description|default('') %}
                        <p class="card-text text-body-secondary">{{ post.description|slice(0, 140) }}{% if post.description|length > 140 %}...{% endif %}</p>
                    {% endif %}
                </div>
            </article>
        </div>
    {% else %}
        <div class="col-12"><div class="alert alert-info text-center">No posts found.</div></div>
    {% endfor %}
    </div>
</main>
{% endblock %}
```

- Extends `Layouts/main.dwig` and fills only the `content` block. Page chrome comes from the layout; this file owns the category story.
- Header from the category, cards from its posts, one shared empty state for the whole grid.

## 2. Where category templates live and how they resolve

```text
Templates/Website/BlogCategory/
+-- default.dwig     # fallback, keep it working
+-- featured.dwig
+-- ...
```

Resolution: the filename saved as the category's layout wins; empty, `inherit`, invalid, or unavailable values fall back to `default.dwig`. Keep theme-specific category layouts in the active theme.

Naming caution: the template registry maps the category page type to `Website/PostCategory`, while the default tree ships `Website/BlogCategory`. Until those names align, the category selector and renderer look in `Website/PostCategory` — keep both names in mind when a category layout is not selected as expected, and keep `BlogCategory/default.dwig` working regardless.

## 3. What data the category page receives

| Value | Contents |
|---|---|
| `data.content` | Page/content record associated with the category. |
| `data.category` | Current category: `title`, `description` (sanitized HTML, render with `\|raw`), and `is_store` (`false` for blog categories — never render shop chrome here). |
| `data.posts` | Enriched posts in the category. Each item can carry `link`, `picture`, `pictures`, `content_data`, `custom_fields`, `tags`, `categories`, and a limited `author` record. Always loop with `\|default([])`. |

## 4. Rules

1. Render the header from `data.category` with escaped title and raw-guarded description. An untitled category shows the fallback heading, never a blank hero.
2. Resolve each card's URL once (`post.link` with `'#'` fallback) and reuse it for image and title links. Never construct post URLs.
3. Guard card images with `{% if %}` and pass them through `thumbnail()` at the displayed ratio. Imageless posts hold grid alignment like any other card.
4. Shorten descriptions by slice with an ellipsis guard, rendering sanitized HTML only. Never escape body excerpts into visible tags, and never `|raw` titles.
5. Keep one shared `{% else %}` empty state for the grid. Per-card fallbacks inside an empty list never render; the branch is the design.
6. Escape titles and attributes throughout. Category and post text is editor input.

## 5. Use cases

**Standard category page.** Header plus responsive card grid with empty state. The default skin pattern.

**Featured category.** Lead article plus grid for flagship topics. Same data, first-item spotlight like the blog feed.

**Text-only index.** Title-and-date rows without images for archive-style categories. Same loop, minimal markup.

**Series landing.** Scoped category with an intro block describing the series. Header copy carries the context.

## 6. Common mistakes

- Looping `data.posts` without `|default([])` or an `{% else %}` branch, breaking new categories with zero posts.
- Constructed post URLs instead of resolved `link` values.
- Escaped body excerpts printing tags as text, or `|raw` on titles.
- Unthumbnailed originals in card images.
- Shop chrome (prices, carts) on a blog category where `is_store` is false.
- Chasing the `PostCategory` versus `BlogCategory` mismatch by duplicating layouts instead of keeping one working `default.dwig`.
