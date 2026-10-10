# Media/PictureGallery — Image Galleries and Product Carousels

The `Media/PictureGallery` module renders an image collection: a product carousel, a portfolio grid, a masonry wall, a slider. Its skins live in `Templates/Modules/Media/PictureGallery/`. The backend collects the pictures bound to the instance (or to a content item) and passes them as a plain list; the skin decides the HTML — grid, carousel, slider, or wall.

## 1. Embedding a gallery

```twig
<module
    type="Media/PictureGallery"
    id="home-gallery"
    template="default.dwig"
/>
```

- `type` is `Media/PictureGallery`: the path-style name that mirrors the skin path `Templates/Modules/Media/PictureGallery/`. The bare legacy name still resolves, but every example in these docs uses the path-style form.
- `id` identifies this instance and connects it to its pictures and saved settings. By default a gallery shows the pictures attached to its own ID, so keep the ID stable and unique per gallery. Changing it orphans the pictures uploaded to the old ID.
- `template` selects the skin filename from the gallery directory. Pass only the filename, never a path. A template chosen in Live Edit is saved against the instance and takes precedence over the inline attribute.
- An unavailable file falls back to `default.dwig`, so the directory keeps a working `default.dwig`.

Tag attributes that select which pictures render:

| Attribute | Purpose |
|---|---|
| `content-id` | Show pictures attached to that content item (product, post, page) instead of the instance's own pictures. Accepts an ID value, for example `content-id="{{ product_id }}"`. |
| `rel` / `rel_type` | Same rebinding by relation type (`content`, `page`, `post`). |
| `for` / `for-id` | Explicit relation source. Rarely needed in templates; prefer `content-id`. |
| `images` | Comma-separated image URLs rendered directly, for example `images="url1,url2"`. Useful for hardcoded demo or decorative strips. |
| `handle_empty` | Set to `"true"` when the skin draws its own empty state (placeholder box). Without it, an empty gallery renders nothing outside Live Edit. |
| Other attributes | Custom values the skin reads via `data.params`, for example `product-title` for accessible labels (see section 7). |

## 2. Where skins live and how they resolve

```text
Templates/Modules/Media/PictureGallery/
+-- default.dwig            # fallback, keep it working
+-- product-gallery.dwig    # product carousel with thumbnails
+-- grid.dwig
+-- masonry.dwig
+-- ...
```

Resolution order for every skin:

1. The same relative path in the active theme.
2. The bundled default template.
3. `default.dwig` in the same directory.

Keep theme-specific galleries in the active theme. Changing the bundled fallback changes the default for every generated theme.

## 3. What data a gallery skin receives

| Value | Contents |
|---|---|
| `data.pictures` | The picture list. Each item carries `filename` (image source) and `title` (caption/alt text). Either can be empty, so read both with `|default()`. |
| `data.params` | Tag attributes plus the instance ID (`data.params.id`), including custom attributes such as `product-title`. |

The safe pattern for any gallery:

```twig
{% set pictures = data.pictures|default([]) %}

{% for picture in pictures %}
    <img src="{{ thumbnail(picture.filename, 800)|e('html_attr') }}"
         alt="{{ picture.title|default('Gallery image')|e('html_attr') }}"
         loading="lazy">
{% else %}
    <p>No images available.</p>
{% endfor %}
```

Auto-escaping is off in `.dwig` templates, so escape `filename` going into `src` with `|e('html_attr')` and `title` going into `alt` or captions with `|e`. Never render a picture value with `|raw`.

## 4. Which pictures render — instance, content, or inline list

A gallery shows pictures from exactly one source, chosen by the tag:

**Instance pictures (default).** No source attribute. Editors upload pictures to this gallery instance in Live Edit. Use this for standalone page sections — heroes, portfolios, lookbooks:

```twig
<module type="Media/PictureGallery" id="home-lookbook" template="grid.dwig" />
```

**Content pictures.** `content-id` rebinds the gallery to a product, post, or page, so it follows that record's media wherever it is placed. Use this on product and post templates, and suffix the instance ID so placements for different records stay independent:

```twig
<module
    type="Media/PictureGallery"
    id="product-picture-gallery-{{ product_id }}"
    content-id="{{ product_id }}"
    product-title="{{ data.content.title|e('html_attr') }}"
    handle_empty="true"
    template="product-gallery.dwig"
/>
```

**Inline list.** `images` renders a fixed URL list with no uploads and no settings. Use it for decorative strips and demos only; editors cannot change these images without editing the template.

## 5. Rendering images with thumbnail()

Never output `picture.filename` at original size. Pass every image through `thumbnail()` with the exact size the design needs:

```twig
{# Main image: width 1000, proportional height #}
{{ thumbnail(picture.filename, 1000)|e('html_attr') }}

{# Square thumbnail: 120x120, cropped #}
{{ thumbnail(picture.filename, 120, 120, true)|e('html_attr') }}
```

Rules:

1. Request the displayed size, not the original. A 120px thumb must call `thumbnail(..., 120, 120, true)`, never the full file.
2. Pass `true` as the last argument only for cropped squares (thumbs, avatars). Omit it for proportional main images.
3. Set `width` and `height` attributes matching the requested size to avoid layout shift.
4. Mark the first visible image `loading="eager"` and the rest `loading="lazy"`:
   `loading="{{ loop.first ? 'eager' : 'lazy' }}"`.
5. Derive `alt` from `picture.title` with a fallback. Decorative duplicates (thumbnail buttons mirroring a labelled main image) use `alt=""`.

## 6. Drawing the empty state

When a gallery can legitimately have no pictures — a product without photos — set `handle_empty="true"` on the tag and draw a placeholder in the skin:

```twig
{% if pictures is not empty %}
    {# ...carousel or grid... #}
{% else %}
    <div class="ratio ratio-1x1 bg-body-secondary rounded-4 text-body-secondary">
        <div class="d-flex align-items-center justify-content-center">
            <span>Product image unavailable</span>
        </div>
    </div>
{% endif %}
```

Without `handle_empty="true"`, an empty gallery renders nothing on the live page and the placeholder never appears. The placeholder keeps the page layout stable (same `ratio` box as a real image) instead of collapsing the section.

## 7. Unique IDs for scripted widgets

Carousels, sliders, and lightboxes address their widget by element ID. Derive the ID from `data.params.id` so two galleries on one page never share it:

```twig
{% set gallery_id = data.params.id|default('picture-gallery') ~ '-carousel' %}

<div id="{{ gallery_id|e('html_attr') }}" class="carousel slide" data-bs-ride="carousel">
    ...
    <button type="button" data-bs-target="#{{ gallery_id|e('html_attr') }}" data-bs-slide="prev">
```

Rules:

1. Never hardcode a widget ID. Two placements of the same skin would then control each other.
2. Point every control and indicator at the derived ID (`data-bs-target="#{{ gallery_id }}"`, `data-bs-slide-to="{{ loop.index0 }}"`).
3. Render prev/next controls and indicators only when there is more than one picture (`{% if pictures|length > 1 %}`). A single-image gallery with dead controls looks broken.
4. Custom tag attributes reach the skin through `data.params` — `data.params['product-title']` with a fallback makes accessible labels without extra settings:
   `{% set product_title = data.params['product-title']|default('Product image') %}`.

## 8. Editable text around a gallery

Gallery skins have no text settings of their own. When a section needs an editable heading above the pictures, put the heading in the surrounding layout's editable region (the `edit` class with `field` and `rel="module"`) and keep the gallery tag purely for pictures. Never hardcode gallery headings per placement; that forces a template edit for a copy change.

## 9. Use cases

**Product carousel with thumbnails.** The `product-gallery.dwig` pattern: Bootstrap carousel of large images plus thumbnail indicator buttons, controls only when `pictures|length > 1`, labelled with `product-title`, empty-state placeholder. Embed with `content-id` and `handle_empty="true"` as in section 4.

**Simple grid.** Fixed grid (`row-cols-2 row-cols-lg-4 g-3`) of `thumbnail(picture.filename, 600)` images with `title` captions. Instance pictures. The everyday portfolio, team photos, or press wall.

**Masonry wall.** Same data, CSS-columns or masonry layout with proportional images (`thumbnail(picture.filename, 600)` without cropping) and optional `title` overlay. Best for varied aspect ratios where cropping would ruin the photo.

**Slider.** One visible image at a time with autoplay (`data-bs-ride="carousel"`, `data-bs-interval`), no thumbnails. Hero-adjacent storytelling or before/after-style showcases. Keep slide markup identical so height stays stable.

**Lightbox gallery.** Grid of thumbnails where each links to the full image (`thumbnail(picture.filename, 1200)`) opened in a lightbox. Preserve the `title` as the lightbox caption and keep the lightbox hooks the theme script consumes.

**Banner strip.** Inline `images` list in a full-width strip with fixed height and cropped thumbs. Decorative brand or partner logos. No uploads, no empty state — the list is the content.

**Content-bound sidebar gallery.** `content-id` gallery in a narrow `aside` column showing the current post's or product's pictures as small thumbs. Suffix the instance ID per record so each page keeps its own pictures.

## 10. Common mistakes

- Omitting `handle_empty="true"` while drawing an `{% else %}` placeholder, so the placeholder never renders on the live page.
- Hardcoding a carousel or slider ID, so two galleries on one page control each other.
- Rendering controls and indicators for a single picture.
- Outputting `picture.filename` without `thumbnail()`, shipping full-size originals to every visitor.
- Reading `picture.title` without `|default()`, leaving empty `alt` attributes on meaningful images — or, conversely, repeating the main image's label on decorative thumbnails instead of `alt=""`.
- Binding a product gallery to the instance ID instead of `content-id`, so every product page shows the same uploaded pictures.
- Reusing one instance ID for two galleries that need different pictures; shared IDs share pictures and settings.
- Passing a path in `template` instead of the filename, or forgetting `default.dwig` so a missing skin breaks the section.
