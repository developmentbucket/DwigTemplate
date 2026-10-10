# Layouts/main.dwig — The Document Shell

`Templates/Layouts/main.dwig` is the shared HTML document every full page extends. It owns the `<!doctype>`, `<html>`, `<head>`, and `<body>` tags plus the global blocks. Page templates such as `Website/Page/default.dwig` or `Store/Product/default.dwig` fill those blocks. Module skins and search fragments never extend it, because they render inside a page that already supplies the document.

## 1. The bundled file, line by line

The bundled layout is intentionally tiny (11 lines):

```twig
<!doctype html>
<html lang="en">
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <title>{{ data.content.title|default(get_option('website_title', 'website')) }}</title>
</head>
<body class="{{ helper_body_classes() }}">
{% block content %}{% endblock %}
</body>
</html>
```

What each part does:

- `<!doctype html>` plus `<html lang="en">` opens a standards-mode document. Change `lang` dynamically for multilingual sites (see use cases below).
- The two `<meta>` tags set UTF-8 and the mobile viewport. Keep both in every custom shell.
- `<title>` prefers the current page title and falls back to the site title via `get_option('website_title', 'website')`. This is the pattern to copy: specific value first, site-wide option as fallback.
- `<body class="{{ helper_body_classes() }}">` lets DevelopmentBucket inject contextual classes (page type, edit mode, theme state). Keep this call so CSS and Live Edit keep working.
- `{% block content %}{% endblock %}` is the single insertion point. Every full-page template overrides it:

```twig
{% extends "Layouts/main.dwig" %}

{% block content %}
<main class="container py-5">
    <h1>{{ data.content.title|e }}</h1>
</main>
{% endblock %}
```

## 2. How to use it

1. Extend it from any full-page template: `Website/Page`, `Website/Post`, `Website/Service`, `Store/Product`, `Store/StoreCategory`.
2. Put only the page body in `{% block content %}`. Never repeat `<html>`, `<head>`, or `<body>` there.
3. Do not extend it from `Templates/Modules/*`, `Templates/Store/Search/*`, invoices, or emails. Invoices are standalone documents with their own shell and print-friendly inline CSS. Search templates are AJAX fragments.
4. Override it per theme by providing the same relative path, `Templates/Layouts/main.dwig`, in the active theme. The bundled file is the fallback.

## 3. SEO tags in `<head>` — use `data.seo`

You do not hand-write the title, description, or social tags per page. DevelopmentBucket builds a ready-to-use `data.seo` array for every page and passes it to all `.dwig` templates, so your job in `main.dwig` is to place the tags once and every page inherits them.

What `data.seo` contains:

| Key | Value | Fallback chain |
|---|---|---|
| `title` | Page meta title | Content `content_meta_title`, then record `title`, then `website_title` setting. Category pages use `category_meta_title`. |
| `description` | Page meta description | Content `content_meta_description` or record `description`, then `website_description` setting. Category pages use `category_meta_description`. |
| `keywords` | Page meta keywords | Content `content_meta_keywords`, or `category_meta_keywords` on category pages. No website fallback, so it can be empty. |
| `favicon` | Favicon URL | `favicon_image` website option, then the brand favicon. |
| `head_tags` | Custom head HTML for this page | The page's Head Tags editor box (`content_head_tags`), already HTML-decoded. Always empty on category pages. |
| `type`, `id` | Source record info | Content type or `category`, plus the record ID. |

Where editors supply the values: page or product editor, SEO tab. Meta Title, Meta Description, and Meta Keywords fill the matching keys, and the Head Tags box accepts raw `<meta>`, `<link>`, or custom tags for that page only.

Place this block in your `main.dwig` head. It is the standard pattern:

```twig
<title>{{ data.seo.title|e }}</title>
<meta name="description" content="{{ data.seo.description|e('html_attr') }}">
<meta name="keywords" content="{{ data.seo.keywords|e('html_attr') }}">
{% if data.seo.favicon %}
<link rel="icon" href="{{ data.seo.favicon|e('html_attr') }}">
{% endif %}
{{ data.seo.head_tags|raw }}
```

Rules for this block:

1. Read from `data.seo`, never from `data.content` directly. The platform already resolved the content-versus-category choice and the website-setting fallbacks for you.
2. Escape `title`, `description`, `keywords`, and `favicon`. These values come from editor input. Only `head_tags` renders raw, because it is intentional HTML from the Head Tags box.
3. Guard `favicon` and `keywords` with `{% if %}`. Either can be empty, and an empty `href` or `content` attribute is worse than a missing tag.
4. Keep `{{ data.seo.head_tags|raw }}` last in the block so per-page tags can override earlier ones.
5. Leave a `{% block head_extra %}{% endblock %}` after the supplied tags so individual page templates can add per-page markup, such as article or product structured data and Open Graph image tags, without forking the shell.
6. Do not duplicate what the platform injects automatically before `</head>`: search-engine verification tags, Google Analytics, and Facebook Pixel. Those arrive on their own, so adding them in the shell loads them twice.

A complete head section looks like this:

```twig
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <title>{{ data.seo.title|e }}</title>
    <meta name="description" content="{{ data.seo.description|e('html_attr') }}">
    <meta name="keywords" content="{{ data.seo.keywords|e('html_attr') }}">
    {% if data.seo.favicon %}
    <link rel="icon" href="{{ data.seo.favicon|e('html_attr') }}">
    {% endif %}
    {{ data.seo.head_tags|raw }}
    <link rel="stylesheet" href="{{ assets('css/main.css')|e('html_attr') }}">
    {% block head_extra %}{% endblock %}
</head>
```

## 4. Growing the shell for a real site

The bundled shell is a starting point. A production theme normally adds the SEO block above plus header, footer, and assets, in this order:

```twig
<!doctype html>
<html lang="{{ get_option('website_language', 'en')|default('en')|e('html_attr') }}">
<head>
    {# ...SEO tags from section 3... #}
    <link rel="stylesheet" href="{{ assets('css/main.css')|e('html_attr') }}">
    {% block head_extra %}{% endblock %}
</head>
<body class="{{ helper_body_classes() }}">
    <module type="Navigation/Menu" id="site-header" template="default.dwig" />
    {% block content %}{% endblock %}
    <module type="Navigation/Menu" id="site-footer" template="footer.dwig" />
    <script src="{{ assets('js/main.js')|e('html_attr') }}"></script>
    {% block scripts %}{% endblock %}
</body>
</html>
```

Notes on this grown version:

- Extra blocks (`head_extra`, `scripts`) let individual pages inject per-page CSS or JS without forking the shell.
- Header and footer are `<module>` tags with stable IDs, so their settings persist and editors can configure them in Live Edit.
- Theme files load through `assets()`, which resolves against the active theme. Module-owned CSS or JS stays with the module.

## 5. Use-case ideas

**Marketing or home page.** Add a hero block above the editable region, then hand control to the editor:

```twig
{% extends "Layouts/main.dwig" %}
{% block content %}
    <section class="hero"><!-- headline, CTA --></section>
    <div class="edit" data-layout-container rel="content" field="content">
        <module type="Layouts" template="default.dwig" />
    </div>
{% endblock %}
```

**Blog post with sidebar.** Split `content` into article plus reusable modules instead of hardcoding related posts:

```twig
{% block content %}
<main class="container py-5"><div class="row">
    <article class="col-lg-8">{{ data.content.content_body|raw }}</article>
    <aside class="col-lg-4">
        <module type="Navigation/Menu" id="blog-sidebar" template="sidebar.dwig" />
    </aside>
</div></main>
{% endblock %}
```

**Shop category with filters.** Follow the bundled `Store/StoreCategory/default.dwig` pattern: filters in an `<aside>`, product list in a `<section>`, wired by matching IDs (`target="#shop-products-list"` with `id="shop-products-list"`).

**Distraction-free landing page.** Some pages should not show the global header or footer. Create a second shell, `Templates/Layouts/landing.dwig`, without the header and footer module tags, and extend that from the landing template instead of adding conditionals to `main.dwig`.

**Coming-soon or campaign shell.** Same technique: a minimal shell with no navigation, one centered `{% block content %}`, and campaign-only CSS. Delete it when the campaign ends without touching `main.dwig`.

**Multilingual and RTL.** Drive `<html lang>` from the site language option and load the RTL stylesheet variant when the locale needs it. Keep one shell with a conditional rather than forking the whole document per language.

**SEO and social previews.** This is covered by section 3: the shell supplies the `data.seo` tags once, and page templates add extras through `head_extra`. A product template, for example, adds Open Graph image and price markup there while the shell keeps the shared tags.

## 6. Common mistakes

- Extending `main.dwig` from a module skin, which nests a full document inside a page.
- Copying the shell per page instead of adding a block. If two shells differ by one section, that section should be a block.
- Dropping `helper_body_classes()`, which silently breaks theme CSS and edit-mode styling.
- Hardcoding the site name, language, or asset URLs instead of using `get_option()` and `assets()`.
- Reading `data.content.content_meta_title` directly instead of `data.seo.title`, which skips the category handling and website fallbacks.
- Rendering `head_tags` escaped, which prints the editor's HTML as visible text instead of tags.
- Pasting Analytics, Pixel, or verification tags into the shell when the platform already injects them, which loads them twice.
- Giving the header and footer modules throwaway IDs per page. Reused IDs share settings across pages; that is usually what a global header wants, but per-page IDs create one settings scope per page by accident.
