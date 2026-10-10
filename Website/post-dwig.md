# Website/Post — Single Blog Articles

The `Templates/Website/Post/` templates render individual blog posts: title header, publication info, hero image, article body, tags, share controls, related posts, and comments. `default.dwig` is the fallback every post resolves to unless the post selects another layout.

## 1. Post template shape

```twig
{% extends "Layouts/main.dwig" %}

{% block content %}
<main class="container py-5">
    <article class="mx-auto" style="max-width: 860px;">
        <header class="text-center mb-5">
            <h1>{{ data.content.title|e }}</h1>
            {% if data.content.created_at|default('') %}
                <time class="text-body-secondary">{{ data.content.created_at|e }}</time>
            {% endif %}
        </header>

        {% if data.content.picture|default('') %}
            <img class="img-fluid rounded-4 mb-5 w-100"
                 src="{{ thumbnail(data.content.picture, 1200, 700)|e('html_attr') }}"
                 alt="{{ data.content.title|default('')|e('html_attr') }}">
        {% endif %}

        {% if data.content.content|default('') %}
            <div class="post-content fs-5">{{ data.content.content|raw }}</div>
        {% elseif data.content.description|default('') %}
            <div class="post-content fs-5">{{ data.content.description|raw }}</div>
        {% endif %}
    </article>
</main>
{% endblock %}
```

- Extends `Layouts/main.dwig` and fills only the `content` block. Page chrome comes from the layout; this file owns the article.
- Constrains the measure (`max-width: 860px`, centered). Body copy wider than ~70 characters exhausts readers.

## 2. Where post templates live and how they resolve

```text
Templates/Website/Post/
+-- default.dwig     # fallback, keep it working
+-- with-sidebar.dwig
+-- ...
```

Resolution: the filename saved as the post's `layout_file` wins; empty, `inherit`, invalid, or unavailable values fall back to `default.dwig`. Keep theme-specific post layouts in the active theme.

## 3. What data the post receives

| Value | Contents |
|---|---|
| `data.content` | Current post record: `id`, `title`, `content` and `description` (sanitized HTML, render with `\|raw`), `picture`, `created_at`. |
| `data.content_data` | Extra post fields by key. Read every key with `\|default()` — custom keys differ per site. |
| `data.post` | Enriched post: `content_data` (plus `custom_data` alias), pictures, custom fields, tags, categories, author details, and frontend link. Prefer `data.post.*` for enriched values, `data.content.*` for the raw record. |

Custom values saved from the post editor read identically from all three paths:

```twig
{{ data.content_data.subtitle|default('')|e }}
{{ data.post.content_data.subtitle|default('')|e }}
{{ data.post.custom_data.subtitle|default('')|e }}
```

## 4. Article anatomy

Header, hero, body, footer modules — each with its guard:

```twig
<header>
    <h1>{{ data.content.title|default('Untitled')|e }}</h1>
    {% if data.content.created_at|default('') %}
        <time datetime="{{ data.content.created_at|e('html_attr') }}">{{ data.content.created_at|e }}</time>
    {% endif %}
    {# author line from data.post author details where available #}
</header>

{% if data.content.picture|default('') %}
    <img src="{{ thumbnail(data.content.picture, 1200, 700)|e('html_attr') }}"
         alt="{{ data.content.title|default('')|e('html_attr') }}" loading="eager">
{% endif %}

{% if data.content.content|default('') %}
    <div class="post-content">{{ data.content.content|raw }}</div>
{% elseif data.content.description|default('') %}
    <div class="post-content">{{ data.content.description|raw }}</div>
{% else %}
    <p class="text-body-secondary">Article body coming soon.</p>
{% endif %}

<footer>
    <module type="Content/Tags" id="post-tags" content_id="{{ data.content.id }}" template="default.dwig" />
    <module type="Social/SocialSharer" id="post-share" enable="all" template="default.dwig" />
    <module type="Website/BlogComments" id="post-comments" template="default.dwig" />
</footer>
```

Rules:

1. Render the body from `content`, then `description`, then a coming-soon line — all with `|raw` as sanitized HTML. Titles, dates, and custom values stay escaped.
2. Pass the hero through `thumbnail()` at display size with `loading="eager"`. The hero is usually the page's largest paint; guard it since text-only posts are normal.
3. Compose footer modules with stable IDs bound to the post: tags with `content_id`, sharer for the current page, comments for the thread. Each follows its own module doc.
4. Date lines render only when `created_at` exists, with matching `datetime` attributes. Undated posts show no date line rather than a broken one.
5. Escape everything except body HTML. `|raw` applies to `content`/`description` only.

## 5. Use cases

**Standard article.** Header, hero, body, tags, share, comments. The default skin pattern.

**Text-only essay.** No hero block, wider measure, reading-time line. Same data, quieter chrome.

**Tutorial with sidebar.** Sticky table of contents beside the body (`with-sidebar.dwig`). Same article, navigable long-form.

**Photo story.** Large hero with caption and minimal text chrome. Same guards, image-forward design.

## 6. Common mistakes

- Escaped body HTML printing tags as text, or `|raw` on titles, dates, and custom values.
- Unthumbnailed hero originals slowing the largest paint.
- Unbound footer modules (tags, comments) showing the wrong post's data.
- Missing coming-soon fallback, leaving body-less posts blank.
- Hardcoded home or section links instead of `site_url()` and resolved values.
- Unescaped `content_data` custom values. Editor extras are untrusted input throughout.
- Skipping `default.dwig`, so posts with unset layouts break instead of falling back.
