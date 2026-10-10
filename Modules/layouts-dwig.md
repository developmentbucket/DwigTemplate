# Modules/Layouts — Page Sections You Can Drop, Style, and Edit

The `Layouts` module renders one page section per skin: a hero, a features grid, a testimonials band, a product showcase, a trust strip. Each `.dwig` file in `Templates/Modules/Layouts/` is a complete, self-contained section that editors drop onto a page and then edit in place. The backend prepares the section's data and settings; the skin decides the HTML.

## 1. Embedding a layout section

Use a `<module>` tag with the friendly type name `Layouts`:

```twig
<module
    type="Layouts"
    id="home-hero"
    template="hero-style-1.dwig"
/>
```

- `type` is always `Layouts`.
- `id` identifies this placement and connects it to saved settings (background, template choice, custom settings). Keep it stable and unique per placement. Changing it orphans the saved settings for that section.
- `template` selects the skin filename from `Templates/Modules/Layouts/`. Pass only the filename, never a path. A template chosen in Live Edit is saved against the instance and takes precedence over the inline attribute.
- An unavailable file falls back to `default.dwig`, so every layouts directory keeps a working `default.dwig`.

Layouts do not appear as a toolbar module. Sections are picked from the layout picker, which lists skins by the registration header described in section 3.

## 2. Where skins live and how they resolve

```text
Templates/Modules/Layouts/
+-- default.dwig                 # fallback, keep it working
+-- hero-style-1.dwig
+-- features-style-1.dwig
+-- testimonials-style-1.dwig
+-- slider-style-1.dwig
+-- ...
```

Resolution order for every skin:

1. The same relative path in the active theme.
2. The bundled default template.
3. `default.dwig` in the same directory.

Keep theme-specific sections in the active theme. Changing the bundled fallback changes the default for every generated theme.

## 3. Registering a skin in the layout picker

Every layout skin opens with a header comment that names it for the picker:

```twig
{#  type: layout
    name: Layout style 1
    position: 1
    categories: Content
    img: testimonial-style-1.jpeg
    screenshot: Assets/Preview/Layouts/testimonial-style-1.jpeg
#}
```

What each key does:

| Key | Purpose |
|---|---|
| `type` | Must be `layout`. This is what makes the file appear as a layout section rather than a plain file. |
| `name` | Display name in the picker, for example `Hero style 1`. |
| `position` | Sort order within its category. Lower appears first. |
| `categories` | Picker grouping, for example `Content`. |
| `img` / `screenshot` | Preview image shown in the live editor. Point it at a real preview asset so editors can recognize the section. |

Copy this header into every new skin and give each skin a distinct `name`. Two skins with the same name are indistinguishable in the picker.

## 4. What data a layout skin receives

A layout skin receives exactly three values under `data`. There is nothing else; never copy a variable from an unrelated module and assume it exists.

| Value | Contents |
|---|---|
| `data.params` | Tag attributes plus the instance ID (`data.params.id`). Saved background options are also mirrored here as `data-background-color`, `data-background-image`, `data-background-image-mobile`, `data-background-video`, `data-background-video-mobile`. |
| `data.background` | Grouped background settings: `color`, `image.desktop`, `image.mobile`, `video.desktop`, `video.mobile`. Each is an empty string when the editor set nothing (or `none`). |
| `data.custom_settings` | Per-skin editable values (text, image, color, video fields) stored against the instance. Always read with a `|default()` fallback. |

The safe pattern for any optional value:

```twig
{{ data.custom_settings.heading|default('Our story')|e }}
{% for item in data.items|default([]) %}
    ...
{% endfor %}
```

Auto-escaping is off in `.dwig` templates, so escape untrusted text with `|e` and attribute values with `|e('html_attr')`. Use `|raw` only for trusted or sanitized HTML, such as page body content.

## 5. Pattern A — static editable section

Most layout skins are static markup with one editable region. The editable region needs three things on the same element: the `edit` class, a `field` name, and `rel="module"`:

```twig
<section class="story reveal py-5 edit" field="layout-story-{{ data.params.id }}" rel="module">
    <div class="container row g-5 align-items-center">
        <div class="col-lg-6">
            <span class="eyebrow">Our roots</span>
            <h2>Food that tastes like <em>someone cared.</em></h2>
            <p class="lead">Editable copy lives here.</p>
        </div>
        <div class="col-lg-6">
            <img src="{{ assets('images/story.jpg')|e('html_attr') }}" alt="Our kitchen" loading="lazy">
        </div>
    </div>
</section>
```

Rules for the editable region:

1. Suffix the `field` name with `{{ data.params.id }}` so two placements of the same skin on one page get independent editable content. Without the suffix, both placements share one content scope.
2. Give each skin its own `field` prefix (`layout-story-`, `layout-certs-`). Reusing one prefix across skins makes unrelated sections share content.
3. Keep `rel="module"` on the editable element. It scopes the edited content to this module instance.
4. Load theme files with `assets()`, which resolves against the active theme. Never hardcode theme asset URLs.
5. Optimize images with `thumbnail()` when the source is an uploaded file: `{{ thumbnail(image, 1000, 1000, true) }}`.

## 6. Pattern B — composite section that embeds modules

A layout skin can compose live behavior by embedding other `<module>` tags: a category grid, a product row, a testimonials band, a slider. Each nested module renders with its own skin and settings:

```twig
<section class="section categories reveal py-5 edit" field="layout-categories-{{ data.params.id }}" rel="module">
    <div class="container">
        <div class="section-head d-flex justify-content-between align-items-end mb-4">
            <h2>Shop by category</h2>
            <a class="text-link" href="{{ site_url('shop') }}">View all categories</a>
        </div>
        <div class="category-grid d-flex flex-column flex-lg-row gap-3">
            <div class="category-grid-featured">
                <module
                    type="Store/StoreCategories"
                    id="store-category-single-home"
                    template="singleCategory.dwig" />
            </div>
            <div class="category-grid-secondary">
                <module
                    type="Store/StoreCategories"
                    id="store-category-home-style-1"
                    template="categories-style-1.dwig" />
            </div>
        </div>
    </div>
</section>
```

Choosing nested module IDs:

- Use a stable global ID (`store-category-single-home`) when the fragment should be configured once for the whole site. Every placement with that ID shares one settings scope.
- Suffix with the layout instance ID when each layout placement needs independent configuration:

```twig
<module type="Content/Testimonials" id="testimonial-home-{{ data.params.id }}" template="default.dwig" />
```

Without the suffix, two placements of the layout share the nested module's settings.

Rules for composite skins:

1. Write each skin against the nested module's own data contract. Check that module's doc before styling its output.
2. Never nest a `Layouts` module inside a layout skin. Layouts compose other modules, never themselves.
3. Keep every class, `data-*` attribute, element ID, and form name the nested module's JavaScript consumes. A skin can look correct while breaking interaction when those hooks are removed.
4. Never extend a full document layout such as `Layouts/main.dwig` from a layout skin. Layouts render inside a page that already supplies the document.
5. A slider wrapper can be as thin as one module tag:

```twig
<div class="slider-section">
    <module type="Media/Slider" id="slider-style-1" template="default.dwig" />
</div>
```

## 7. Applying the editor-controlled background

Editors set a layout's background (color, image, video, with mobile variants) in Live Edit. The skin applies it by reading `data.background`:

```twig
{% set bg = data.background %}
<section class="hero py-5 edit"
         field="layout-hero-{{ data.params.id }}" rel="module"
         {% if bg.color %}style="background-color: {{ bg.color|e('html_attr') }};"{% endif %}>
    {% if bg.image.desktop %}
        <img class="section-bg d-none d-md-block"
             src="{{ bg.image.desktop|e('html_attr') }}" alt="" loading="lazy">
    {% endif %}
    {% if bg.image.mobile %}
        <img class="section-bg d-md-none"
             src="{{ bg.image.mobile|e('html_attr') }}" alt="" loading="lazy">
    {% endif %}
    <div class="container">
        ...
    </div>
</section>
```

Rules:

1. Read from `data.background`, not from `data.params` directly. The grouped object already normalizes `none` to an empty string.
2. Guard every value with `{% if %}`. Any of them can be empty, and an empty `src` or `style` is worse than a missing one.
3. Prefer the mobile variant below the `md` breakpoint and the desktop variant above it, so editors get the responsive behavior they configured.
4. Escape every background value going into an attribute. These values come from editor input.

## 8. Reading per-skin custom settings

When a skin defines custom settings fields, their stored values arrive in `data.custom_settings`. Always read with a fallback so the section renders before anything is configured:

```twig
{% set settings = data.custom_settings|default({}) %}

<section class="features py-5 edit" field="layout-features-{{ data.params.id }}" rel="module">
    <div class="container text-center">
        <h2>{{ settings.heading|default('Why shop with us')|e }}</h2>
        <p>{{ settings.subheading|default('')|e }}</p>
    </div>
</section>
```

1. Assign `data.custom_settings|default({})` to a `settings` variable once at the top, then read `settings.*` with per-key `|default()` fallbacks.
2. Never assume a key exists. An unconfigured instance passes an empty object.
3. Escape text and color values. Render image values into `src` through `|e('html_attr')`.

## 9. Use cases

**Hero band.** Static pattern: eyebrow, headline, copy, CTA buttons, one image. One editable region over the whole section. Background color or image from `data.background`.

**Features or highlights grid.** Static pattern: three or four columns with icon, title, and one-line copy. Use Bootstrap grid (`row-cols-1 row-cols-md-3 g-4`) so the columns collapse on mobile without extra CSS.

**Category showcase.** Composite pattern: heading row plus two `Store/StoreCategories` modules in different skins (one featured, one grid), exactly like the example in section 6. Link the heading to `site_url('shop')`, never a hardcoded path.

**Product showcase.** Composite pattern: heading row plus a `Store/Products` module in a card-grid skin. Keep the nested module's cart and wishlist hooks intact so add-to-bag keeps working inside the section.

**Testimonials band.** Composite pattern: a single `Content/Testimonials` module with a per-instance ID (`testimonial-home-{{ data.params.id }}`), so each page's testimonials are configured independently.

**Slider section.** Composite pattern: a thin wrapper around one `Media/Slider` module. All slide content stays in the slider's settings; the layout skin only supplies spacing and the section shell.

**Trust or certification strip.** Static pattern: one row of badges or certificates (`row-cols-2 row-cols-lg-5`). Fully static markup with one editable region, no nested modules.

**Brand story block.** Static pattern: image plus copy in two columns, with a stat callout. Theme image via `assets()`, editable copy via the section's `field`.

**Newsletter CTA.** Static pattern or composite with a form module: dark band, headline, and an embedded contact-form module for the newsletter signup.

**Campaign or seasonal section.** Same authoring as any static skin with its own `name` and `screenshot` in the header. Editors add it for the campaign and remove the instance when it ends, without touching other sections.

**Checkout-adjacent reassurance strip.** Static pattern with no navigation and no nested modules: payment icons, delivery promise, return promise. Keep it dependency-free so it renders identically on cart, checkout, and confirmation placements.

## 10. Common mistakes

- Reusing one `field` name across skins or omitting the `{{ data.params.id }}` suffix, which makes unrelated sections share editable content.
- Dropping `rel="module"` from the editable element, which breaks content scoping for the instance.
- Giving two different nested modules the same ID, which makes them share settings.
- Giving a global fragment (site-wide promo, site-wide slider) a per-instance-suffixed ID, which forks its settings every time the layout is placed.
- Reading `data.params['data-background-image']` directly instead of `data.background.image.desktop`, which skips the `none`-to-empty normalization.
- Rendering `data.background` or `data.custom_settings` values without `{% if %}` guards or `|default()` fallbacks, which breaks the section before editors configure it.
- Removing the nested module's JS hooks or `data-*` attributes while skinning, which silently disables cart, wishlist, or slider behavior.
- Nesting a `Layouts` module inside a layout skin, or extending `Layouts/main.dwig` from one.
- Hardcoding asset URLs, shop links, or currency instead of using `assets()`, `site_url()`, resolved `link` values, and `currency_format()`.
- Saving a full path in `template` instead of the filename, or forgetting the `type: layout` header so the skin never appears in the picker.
