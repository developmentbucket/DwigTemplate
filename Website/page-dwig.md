# Website/Page — General Website Pages

The `Templates/Website/Page/` templates render ordinary website pages: home, landing, contact, story, and content pages. `default.dwig` is the fallback every page resolves to unless the page selects another layout. Page templates extend the site layout, fill the content block, and compose the page from embedded modules — most pages are an editable container holding layout sections.

## 1. Page template shape

```twig
{% extends "Layouts/main.dwig" %}

{% block seo_content %}
   <title>{{ data.seo.title }}</title>
   <meta name="description" content="{{ data.seo.description }}">
   <meta name="keywords" content="{{ data.seo.keywords }}">
   {{ data.seo.head_tags }}
{% endblock %}

{% block content %}
<main>
    <div class="edit main-content" data-layout-container rel="content" field="content">
        <module type="Layouts" id="home-hero" template="hero-style-1.dwig" />
        <module type="Layouts" id="home-features" template="features-style-1.dwig" />
    </div>
</main>
{% endblock %}
```

- Extends `Layouts/main.dwig` and fills the `content` block (plus `seo_content` for meta tags from `data.seo`). Page chrome comes from the layout; this file owns page composition.
- Carries the editable layout container (`edit main-content` with `data-layout-container`, `rel="content"`, `field="content"`) so editors compose the page from sections in Live Edit.
- Seeds the container with one `<module>` per section, each with a stable unique ID. Changing an ID orphans that section's content and settings.

## 2. Where page templates live and how they resolve

```text
Templates/Website/Page/
+-- default.dwig     # fallback for every page type below
+-- shop.dwig
+-- contact.dwig
+-- story.dwig
+-- checkout.dwig
+-- account.dwig
+-- ...
```

Resolution: the filename saved as the page's `layout_file` wins; empty, `inherit`, invalid, or unavailable values fall back to `default.dwig`. Keep filenames relative to the directory — never a full filesystem path. Keep theme-specific pages in the active theme.

## 3. What data the page receives

| Value | Contents |
|---|---|
| `data.content` | Current page record: `id`, `title`, body fields. |
| `data.content_data` | Extra page fields by key. Read every key with `\|default()` — custom keys differ per site. |
| `data.seo` | Meta values: `title`, `description`, `keywords`, `head_tags`. Render in the `seo_content` block. |

## 4. Composing pages from modules

Pages compose; they rarely contain finished markup. One embedded module per section, each documented in its own module doc:

```twig
<module type="Layouts" id="story-hero" template="hero-style-1.dwig" />
<module type="Store/Products" id="home-products" template="default.dwig" />
<module type="Content/Testimonials" id="home-testimonials" template="default.dwig" />
<module type="Media/Slider" id="home-slider" template="default.dwig" />
```

Rules:

1. Give every embedded module a stable unique ID. Shared or shifting IDs merge sections' content and settings.
2. Write each section against its own module contract. The page file arranges modules; module skins own their markup.
3. Keep page-specific one-off markup minimal. Repeated structures graduate into `Layouts` section skins or module skins, not page-template conditionals.
4. Specialized pages (shop, checkout, account, contact) embed their working modules directly instead of leaving everything to the editor: product lists, cart and checkout modules, dashboard, and forms render with explicit IDs and skins.

## 5. Use cases

**Homepage.** Seeded layout sections (slider, categories, products, testimonials) in editorial order. Editors rearrange without touching markup.

**Landing page.** Narrow conversion flow: hero, proof, single CTA. Minimal sections, no catalogue modules.

**Contact page.** Info columns plus the contact form module with an explicit ID. Same container, working modules seeded.

**Story/brand page.** Editorial sections with images and copy, no commerce modules. Content-led composition.

**Account and checkout pages.** Dashboard, cart, and checkout modules seeded with explicit IDs. Functional pages, not editable canvases.

## 6. Common mistakes

- Duplicated or missing module IDs, merging sections' content and settings.
- Finished markup baked into page templates instead of module skins, uneditable and unreusable.
- Missing the editable container, freezing editors out of page composition.
- Hardcoded page URLs instead of `site_url()` and resolved links.
- Unescaped titles or missing `|default()` on site-specific `content_data` keys.
- Skipping `default.dwig`, so pages with unset layouts break instead of falling back.
