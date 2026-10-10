# Media/Slider — Hero Carousels and Banner Sliders

The `Media/Slider` module renders a slide carousel: hero banners, promo slides, feature stories. Its skins live in `Templates/Modules/Media/Slider/`. Editors build the slides (images, titles, descriptions, buttons) in Live Edit; the backend passes them as a plain list and the skin decides the HTML — full-width carousel, banner strip, or story slider.

## 1. Embedding a slider

```twig
<module
    type="Media/Slider"
    id="homepage-hero-slider"
    template="default.dwig"
/>
```

- `type` is `Media/Slider`: the path-style name that mirrors the skin path `Templates/Modules/Media/Slider/`. The bare legacy name still resolves, but every example in these docs uses the path-style form.
- `id` identifies this instance and connects it to its slides and settings. Slides are stored against this ID, so keep it stable and unique per slider. Changing it orphans the slides built under the old ID.
- `template` selects the skin filename from the slider directory. Pass only the filename, never a path. A template chosen in Live Edit is saved against the instance and takes precedence over the inline attribute.
- An unavailable file falls back to `default.dwig`, so the directory keeps a working `default.dwig`.

Sliders take no source attributes. Unlike galleries, a slider never rebinds to a content item: its slides always belong to its own instance ID.

## 2. Where skins live and how they resolve

```text
Templates/Modules/Media/Slider/
+-- default.dwig     # fallback, keep it working
+-- hero.dwig
+-- banner.dwig
+-- ...
```

Resolution order for every skin:

1. The same relative path in the active theme.
2. The bundled default template.
3. `default.dwig` in the same directory.

Keep theme-specific sliders in the active theme. Changing the bundled fallback changes the default for every generated theme.

## 3. What data a slider skin receives

| Value | Contents |
|---|---|
| `data.slides` | The slide list in editor order. Each item carries the fields in the table below. Any field can be empty, so read each with `|default()`. |
| `data.params` | Tag attributes plus the instance ID (`data.params.id`). |

Slide fields:

| Field | Contents |
|---|---|
| `image` | Desktop slide image URL. |
| `mobile_image` | Alternate image for small screens. Empty when the editor set none. |
| `title`, `showTitle` | Heading text and its visibility flag. Render only when both are truthy. |
| `description`, `showDescription` | Body text and its visibility flag. Render only when both are truthy. |
| `buttonText`, `showButton`, `url` | Button label, its visibility flag, and the link target. Render only when all are truthy. |
| `titleColor`, `titleFontSize`, `titleFontFamily` | Heading styling. |
| `descriptionColor`, `descriptionFontSize`, `descriptionFontFamily` | Body styling. |
| `buttonColor`, `buttonTextColor`, `buttonFontSize` | Button styling. |
| `imageBackgroundColor`, `imageBackgroundOpacity` | Overlay behind caption text for readability. |
| `bannerUrl`, `addLinkToImage` | Optional link wrapping the whole slide image. |

The safe pattern for any slider:

```twig
{% set slides = data.slides|default([]) %}

{% for slide in slides %}
    ...
{% else %}
    <p>No slides are available.</p>
{% endfor %}
```

Auto-escaping is off in `.dwig` templates, so escape caption text with `|e` and every attribute (including `src`, `href`, and style values) with `|e('html_attr')`. Never render a slide value with `|raw`.

## 4. Slide anatomy — image, caption, button

Each slide has three independent layers, and each layer is gated by its own visibility flag. Never render a layer on the field alone; always test the flag first:

```twig
{% for slide in slides %}
<div class="carousel-item{% if loop.first %} active{% endif %}">
    <picture>
        {% if slide.mobile_image|default('') %}
            <source media="(max-width: 760px)" srcset="{{ slide.mobile_image|e('html_attr') }}">
        {% endif %}
        <img class="d-block w-100 h-auto"
             src="{{ slide.image|e('html_attr') }}"
             alt="{{ slide.title|default('Featured slide')|e('html_attr') }}"
             width="1920" height="750"
             loading="{{ loop.first ? 'eager' : 'lazy' }}"
             fetchpriority="{{ loop.first ? 'high' : 'auto' }}">
    </picture>

    {% if slide.showTitle or slide.showDescription or slide.showButton %}
    <div class="carousel-caption">
        {% if slide.showTitle and slide.title|default('') %}
            <h2 style="color: {{ slide.titleColor|default('#ffffff')|e('html_attr') }};">{{ slide.title|e }}</h2>
        {% endif %}
        {% if slide.showDescription and slide.description|default('') %}
            <p style="color: {{ slide.descriptionColor|default('#ffffff')|e('html_attr') }};">{{ slide.description|e }}</p>
        {% endif %}
        {% if slide.showButton and slide.buttonText|default('') %}
            <a class="btn btn-light btn-lg"
               href="{{ slide.url|default('#')|e('html_attr') }}">{{ slide.buttonText|e }}</a>
        {% endif %}
    </div>
    {% endif %}
</div>
{% endfor %}
```

Rules:

1. Gate every layer: `slide.showTitle and slide.title`, `slide.showDescription and slide.description`, `slide.showButton and slide.buttonText`. Editors toggle visibility without deleting content; the skin must honor the flags.
2. Serve `mobile_image` through `<source media="...">` inside `<picture>`, guarded by `{% if %}`. The desktop `image` stays the `<img>` fallback.
3. Mark the first slide `eager` / `fetchpriority="high"` and the rest `lazy`. The hero image is usually the page's largest paint.
4. Set `width` and `height` matching the design ratio to avoid layout shift while slides load.
5. Apply per-slide colors and sizes inline with `|default()` fallbacks, escaped for attributes. When a caption is unreadable over a bright photo, use `imageBackgroundColor` / `imageBackgroundOpacity` as a scrim behind the caption.

## 5. Unique IDs, indicators, and editor hooks

Derive the widget ID from `data.params.id` and keep the Live Edit hooks the skin carries, so two sliders on one page stay independent and editable:

```twig
{% set slider_id = data.params.id|default('hero-slider') %}

<section id="{{ slider_id|e('html_attr') }}"
         class="carousel slide"
         data-id="{{ data.params['data-id']|default(data.params.id)|e('html_attr') }}"
         data-module="{{ data.params['data-module']|default('slider_v2')|e('html_attr') }}"
         data-parent-module-id="{{ data.params['data-parent-module-id']|default('')|e('html_attr') }}"
         data-template="{{ data.params['data-template']|default('default.dwig')|e('html_attr') }}"
         data-bs-ride="carousel"
         data-bs-interval="5000"
         aria-label="Featured products">
```

Rules:

1. Never hardcode the slider ID. Point every indicator and control at the derived ID (`data-bs-target="#{{ slider_id }}"`, `data-bs-slide-to="{{ loop.index0 }}"`).
2. Render indicators and prev/next controls only when `slides|length > 1`. A single slide with dead controls looks broken.
3. Keep the `data-id`, `data-module`, `data-parent-module-id`, and `data-template` attributes. Live Edit uses them to wire the slider back to its settings; removing them breaks in-place editing while the slider looks fine.
4. Mark exactly one item `active` (`{% if loop.first %}`), with matching `aria-current="true"` on its indicator. Zero or two active items both break the carousel.

## 6. Drawing the empty state

A slider with no slides must still render a stable placeholder so the page layout holds and editors see where to add slides:

```twig
{% for slide in slides %}
    ...
{% else %}
    <div class="carousel-item active">
        <div class="d-flex align-items-center justify-content-center bg-body-secondary p-5 text-body-secondary">
            No slides are available.
        </div>
    </div>
{% endfor %}
```

## 7. Use cases

**Homepage hero carousel.** Full-width slides with image, heading, description, and CTA button. First slide eager-loaded, indicators plus controls, 5-second autoplay. The default skin pattern.

**Promo or sale slider.** Same carousel shell with bolder caption styling and button-first captions. Editors swap slides per campaign without touching the template.

**Category showcase slider.** One slide per category with the category image and a button linking to the category page. Reuses the hero skin with different content, not a new skin.

**Brand story slider.** Fewer, taller slides with long descriptions and no buttons. Caption scrim from `imageBackgroundColor` keeps text readable over photography.

**Compact banner strip.** Short fixed-height slides (single heading plus link) for announcements above the footer or below the header. Same data, tighter CSS, no indicators.

**Multi-card row.** Same `data.slides` loop in a grid of cards instead of a carousel: image, title, and link per card. No widget ID or controls needed; the empty state still applies.

## 8. Common mistakes

- Rendering caption layers on field presence instead of the `show*` flags, so hidden titles and buttons reappear.
- Hardcoding the slider ID, so two sliders on one page advance each other.
- Rendering indicators or prev/next controls for a single slide.
- Removing the `data-id` / `data-module` / `data-parent-module-id` / `data-template` hooks, which silently breaks Live Edit wiring.
- Marking zero or multiple items `active`.
- Loading all slides `eager`, or omitting `width` and `height`, causing slow paints and layout shift.
- Serving only the desktop image when `mobile_image` is set, wasting mobile bandwidth.
- Reading slide fields without `|default()`, breaking the skin before editors build slides.
- Reaching for a slider when a gallery fits: single unchanging image sets belong in `Media/PictureGallery`, not in one-slide sliders.
