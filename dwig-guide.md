# Dwig — Complete Guide: How Templates Work

Dwig is the template language of DevelopmentBucket themes. It is Twig syntax in `.dwig` files, rendered by the theme view loader: every template is looked up first in the active theme and then in the bundled defaults, compiled with caching and auto-reload, and executed with a strict set of registered functions plus the standard Twig language underneath.

## 1. How rendering works

```
Active theme Templates/      ← checked first
Bundled default Templates/   ← fallback
        ↓ FilesystemLoader
Twig Environment (cache on, auto_reload on, debug on)
        ↓ render(template, { data: {...} })
HTML
```

1. Templates resolve by relative path (`Layouts/main.dwig`, `Modules/Store/Product/default.dwig`). The first match wins; a missing file raises a "template file not found" error naming the resolved path.
2. Data arrives as named variables — almost always one `data` object whose shape each template's doc defines (`data.posts`, `data.product`, `data.params`, and similar). Invoice and notification emails instead receive top-level names (`order_details`, `products`).
3. Only registered function names resolve (see `functions-dwig.md`). Calling anything else fails.
4. Auto-escaping is off for `.dwig` files, so skins escape manually: `|e` for text, `|e('html_attr')` for attributes, `|raw` only for trusted or sanitized HTML.
5. Compilation is cached; `auto_reload` picks up file changes during development. Template errors print the message with file and line to locate the fault fast.

## 2. File structure basics

A page template extends a document layout and fills blocks:

```twig
{% extends "Layouts/main.dwig" %}

{% block content %}
<main>
    <h1>{{ data.content.title|default('Untitled')|e }}</h1>
</main>
{% endblock %}
```

A module skin is a fragment — no `extends`, no document tags — receiving `data` and optionally composing nested modules:

```twig
{% set items = data.posts|default([]) %}
{% for post in items %}
    <h2>{{ post.title|e }}</h2>
{% else %}
    <p>No posts found.</p>
{% endfor %}
<module type="Store/Products" id="extra-products" template="default.dwig" />
```

The `<module>` tag is DevelopmentBucket markup inside dwig files, not Twig: `type` names the module path-style (`Media/Slider`, `Store/Cart`), `id` scopes its settings, `template` picks its skin filename, and any other attributes become tag params the backend reads.

Layout sections carry a registration header so pickers list them:

```twig
{#  type: layout
    name: Hero style 1
    position: 1
    categories: Content
    screenshot: Assets/Preview/Layouts/hero-style-1.jpeg
#}
```

## 3. Variables, setting, and output

```twig
{# assign once, reuse #}
{% set product = data.product|default({}) %}
{% set title = product.title|default('Untitled') %}

{# output with escaping #}
<h2>{{ title|e }}</h2>
<a href="{{ product.link|default('#')|e('html_attr') }}">View</a>

{# trusted HTML only #}
<div>{{ data.content.content|raw }}</div>

{# debug during development only #}
{{ print_data(data) }}
```

Rules: read every optional value through `|default()`; resolve display values once per scope; escape by destination; `print_data()` never ships to production.

## 4. Branching and loops

```twig
{% if product.offer_price|default(0) %}
    <del>{{ currency_format(product.price) }}</del>
    <strong>{{ currency_format(product.offer_price) }}</strong>
{% elseif product.price|default(0) %}
    <strong>{{ currency_format(product.price) }}</strong>
{% else %}
    <span>Price on request</span>
{% endif %}

{% for picture in data.pictures|default([]) %}
    <img src="{{ thumbnail(picture.filename, 800)|e('html_attr') }}"
         alt="{{ picture.title|default('')|e('html_attr') }}"
         loading="{{ loop.first ? 'eager' : 'lazy' }}">
{% else %}
    <p>No images yet.</p>
{% endfor %}
```

`loop` inside `for` gives `index`, `index0`, `first`, `last`, `length`, `revindex`. The `{% else %}` branch on loops is the empty-state design — use it instead of a separate count check.

## 5. Macros for repeated markup

Recursive structures (menu trees, category trees) render through macros that import themselves:

```twig
{% macro menu_tree(items, level) %}
    {% import _self as nav %}
    <ul>
    {% for item in items|default([]) %}
        <li>
            <a href="{{ item.url|default('#')|e('html_attr') }}">{{ item.title|e }}</a>
            {% if item.children|default([]) %}
                {{ nav.menu_tree(item.children, level + 1) }}
            {% endif %}
        </li>
    {% endfor %}
    </ul>
{% endmacro %}

{% import _self as nav %}
{{ nav.menu_tree(data.menu_items, 0) }}
```

Guard recursion inputs with `|default([])` and cap depth for unbounded trees.

## 6. Filters used across shipped skins

All standard Twig filters work; these carry the most weight in real skins:

| Filter | Use |
|---|---|
| `\|default(x)` | Fallback for missing values. The most-used filter in every skin. |
| `\|length` | Counts and emptiness tests. |
| `\|slice(a, b)` | Excerpts with an ellipsis guard on length. |
| `\|split(',')`, `\|join(', ')` | Comma lists to arrays and back. |
| `\|batch(n)` | Chunked grids and paired cards. |
| `\|filter(v => ...)` / `\|map(...)` | Selecting and reshaping without loops. |
| `\|striptags` | Tag-stripping before trimming HTML copy. |
| `\|url_encode` | Query and path values built from editor text. |
| `\|number_format(n)` | Ratings and counts. |
| `\|trim`, `\|lower`, `\|capitalize`, `\|first` | Normalizing and comparing text. |
| `\|json_encode` | Server data into scripts (with context escaping). |
| `\|raw` / `\|e` / `\|e('html_attr')` | Trusted HTML, text, attributes. |

## 7. Functions at a glance

The full reference with signatures and examples lives in `functions-dwig.md`. The daily dozen:

```twig
{{ thumbnail(image, 800, 800, true) }}      {# sized/cropped image URL #}
{{ currency_format(price) }}                {# locale money #}
{{ assets('css/app.css') }}                 {# versioned theme asset URL #}
{{ site_url('shop') }}                      {# site link, never hardcoded #}
{{ url_current() }}                         {# this page's URL #}
{{ get_picture(data.content.id) }}          {# primary image with guard #}
{{ content_data(data.content.id, 'sku') }}  {# extra field by key #}
{{ get_option('website_title', 'website') }} {# stored setting #}
{{ is_login() }}                            {# authenticated or guest #}
{{ get_products({ 'limit': 8 }) }}          {# product query #}
{{ get_shop_categories() }}                 {# shop category list #}
{{ barcode(sku, 'png', '#000', 50, 2) }}    {# scannable data URI #}
```

## 8. Editable regions and settings

Editable page regions pair three attributes on one element so Live Edit scopes content per instance:

```twig
<section class="edit" field="layout-hero-{{ data.params.id }}" rel="module">
```

Per-skin editor settings come from `settings.json` beside the skins and arrive as `data.custom_settings.<key>` — always read with `|default()` fallbacks. Both mechanisms are specified in the module introduction (`Modules/introduction.md`, sections 5 and 9).

## 9. Failure behavior

- Missing template: "template file not found" naming the resolved path. Keep `default.dwig` fallbacks so selection never lands here.
- Template error: message with template line, file, and line number. Read the named line first; most faults are unclosed tags, unknown names, or missing filters.
- Empty data: skins degrade through `|default()` and `{% else %}` branches. A blank render for a reachable state is a skin bug, not an empty database.
- Unknown function: resolution fails. If the name is not in the reference, it does not exist — find the registered equivalent or the prepared data value.

## 10. Checklist for every new skin

1. Resolve display values once per scope with `|default()` fallbacks.
2. Escape text and attributes; `|raw` only for trusted/sanitized HTML.
3. Guard images, links, prices, and lists with `{% if %}` / `{% else %}`.
4. Unique IDs derived from the instance for widgets, inputs, and scripts.
5. `thumbnail()` at display size; `currency_format()` for money; `assets()` for theme files; `site_url()` and resolved links for navigation.
6. Keep module JavaScript hooks (`data-*`, classes, IDs, field names) exactly as their docs specify.
7. One `default.dwig` fallback per directory, real content before configuration, and no backend internals in markup.
