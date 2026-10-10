# Media/Logo — Site Brand Mark

The `Media/Logo` module renders the site brand: an uploaded logo image, a text wordmark, or both. Its skins live in `Templates/Modules/Media/Logo/`. The backend loads the brand settings saved for the instance and passes them as a ready-to-use object; the skin decides the HTML — image, text, sizes, and link.

## 1. Embedding a logo

```twig
<module
    type="Media/Logo"
    id="site-logo"
    template="default.dwig"
/>
```

- `type` is `Media/Logo`: the path-style name that mirrors the skin path `Templates/Modules/Media/Logo/`. The bare legacy name still resolves, but every example in these docs uses the path-style form.
- `id` identifies this instance and connects it to its brand settings (uploaded image, text, colors). Header and footer logos use different IDs (`site-logo`, `footer-logo`) so each is configured independently in Live Edit. Changing an ID orphans the brand settings saved under the old one.
- `template` selects the skin filename from the logo directory. Pass only the filename, never a path. A template chosen in Live Edit is saved against the instance and takes precedence over the inline attribute.
- An unavailable file falls back to `default.dwig`, so the directory keeps a working `default.dwig`.

## 2. Where skins live and how they resolve

```text
Templates/Modules/Media/Logo/
+-- default.dwig     # fallback, keep it working
+-- footer.dwig      # simplified placement variant
+-- ...
```

Resolution order for every skin:

1. The same relative path in the active theme.
2. The bundled default template.
3. `default.dwig` in the same directory.

Keep theme-specific logo skins in the active theme. The bundled skins ship as empty stubs, so writing the header and footer logo skins is part of building the theme.

## 3. What data a logo skin receives

| Value | Contents |
|---|---|
| `data.settings` | Brand settings object (see table below). Every key is always present; image keys are empty when nothing is uploaded. |
| `data.params` | Tag attributes plus the instance ID (`data.params.id`), including inline overrides from section 5. |

`data.settings` keys:

| Key | Contents | Empty means |
|---|---|---|
| `logoType` | Display mode. Becomes `text` when no image is available. | — |
| `logoImage` | Uploaded logo image URL, or the inline `image` / `data-defaultlogo` fallback. | No image uploaded and no fallback given. |
| `logoImageInverse` | Alternate image for dark surfaces. | No inverse image; fall back to `logoImage`. |
| `logoText` | Brand text (wordmark). | No text configured. |
| `logoTextColor` | Text color, default `'#000'`. | — |
| `logoFontFamily` | Text font family, or empty for inherited. | Use inherited font. |
| `logoFontSize` | Text size, default `30`. | — |
| `logoSize` | Image size hint, default `200`. | — |

Assign once at the top of every skin:

```twig
{% set settings = data.settings|default({}) %}
```

## 4. Image first, text as fallback

A logo skin handles three states: image available, image missing but text configured, neither configured. Cover all three:

```twig
{% set settings = data.settings|default({}) %}

<a class="brand" href="{{ site_url()|e('html_attr') }}" aria-label="Home">
    {% if settings.logoImage|default('') %}
        <img class="brand__logo"
             src="{{ settings.logoImage|e('html_attr') }}"
             alt="{{ settings.logoText|default('Home')|e('html_attr') }}"
             width="56" height="56">
    {% endif %}
    {% if settings.logoText|default('') %}
        <span class="brand__wordmark"
              style="color: {{ settings.logoTextColor|default('#000')|e('html_attr') }};
                     font-size: {{ settings.logoFontSize|default(30)|e('html_attr') }}px;">{{ settings.logoText|e }}</span>
    {% endif %}
</a>
```

Rules:

1. Test `logoImage` with `{% if %}` before drawing `<img>`. An empty `src` is worse than a missing image.
2. Derive `alt` from `logoText` with a fallback. The logo is a home link, so the `alt` (or `aria-label` when the image is decorative) must say where the link goes, not describe pixels.
3. Always link the brand home with `href="{{ site_url() }}"`. Never hardcode the domain.
4. Escape text with `|e` and every attribute with `|e('html_attr')`. Brand values come from editor input.
5. Treat an empty font family as inherited. Only emit `font-family` when `logoFontFamily` is non-empty.

## 5. Inline overrides via tag attributes

Tag attributes fill gaps when no saved setting exists. Precedence for each value is: saved setting first, tag attribute second, built-in default last.

| Attribute | Fills | Notes |
|---|---|---|
| `image` | `logoImage` | Fixed image when no logo is uploaded. Overrides are per tag, not per editor. |
| `data-defaultlogo` | `logoImage` | Theme-level fallback (for example `assets/images/logo.jpeg` resolved by the skin). Preferred over `image` for theme defaults. |
| `text` | `logoText` | Fixed wordmark when no text is configured. |
| `font_family` | `logoFontFamily` | Fixed font for the wordmark. |
| `font_size` | `logoFontSize` | Fixed text size; default `30`. |
| `size` | `logoSize` | Image size hint; default `200`. Apply it to `width` / `max-width` in the skin. |
| `logoimage_inverse` | `logoImageInverse` | Fixed alternate image for dark surfaces. |
| `logo-name` | Settings group | Read settings saved under a different name, so two placements share one configuration. Rarely needed; prefer distinct IDs. |

Example — footer logo with a theme fallback image:

```twig
<module type="Media/Logo" id="footer-logo" template="footer.dwig" />
```

with the skin resolving the image as `settings.logoImage` first and the theme asset second:

```twig
src="{{ (settings.logoImage|default('')) ?: assets('images/logo.jpeg')|e('html_attr') }}"
```

## 6. Placement variants: header, footer, inverse

Logo skins are per placement. The header skin and the footer skin differ in context, not data:

```twig
{# Templates/Layouts/Partials/header.dwig #}
<a class="navbar-brand" href="{{ site_url()|e('html_attr') }}">
    <module type="Media/Logo" id="site-logo" template="default.dwig" />
</a>
```

```twig
{# Templates/Layouts/Partials/footer.dwig #}
<module type="Media/Logo" id="footer-logo" template="footer.dwig" />
```

Rules:

1. Give each placement its own skin (`default.dwig`, `footer.dwig`) and its own ID (`site-logo`, `footer-logo`). Sharing one ID across header and footer forces both placements into one configuration.
2. For dark surfaces, prefer the uploaded inverse image before restyling the main one:
   `{% set brand_image = settings.logoImageInverse|default(settings.logoImage) %}`.
3. Keep footer variants simpler than header ones: smaller dimensions, no extra controls, light-safe text color.
4. Never put navigation, search, or menu buttons inside a logo skin. The logo module renders the brand only; surrounding controls belong to the header partial.

## 7. Use cases

**Header brand.** Image plus wordmark side by side, linked home. The primary `site-logo` instance every page renders.

**Footer brand.** Compact `footer.dwig` skin on the independent `footer-logo` instance: smaller image, light-safe text, same home link.

**Inverse logo for dark bands.** Same markup as the placement skin but reading `logoImageInverse` first, so dark heroes, footers, and newsletter bands get the light artwork automatically.

**Text-only wordmark.** No image uploaded: the skin renders `logoText` with `logoTextColor`, `logoFontFamily`, and `logoFontSize`. Zero artwork needed for a clean launch.

**Compact mobile brand.** Image-only skin (no wordmark) at small dimensions for the mobile bar. Same `site-logo` ID is wrong here if desktop and mobile need different settings; use a dedicated ID when the configurations genuinely differ.

**Login and maintenance brand.** Reuse the `site-logo` instance on standalone pages (login, coming-soon shells) so the brand stays identical without a second configuration.

## 8. Common mistakes

- Drawing `<img>` without testing `logoImage`, emitting an empty `src` before anything is uploaded.
- Hardcoding the home URL instead of `site_url()`.
- Missing or meaningless `alt` on a linked logo. The link goes home; say so.
- Sharing one instance ID between header and footer, so configuring one rewrites the other.
- Putting header controls (menu button, search, cart) inside the logo skin instead of the header partial.
- Building a second configuration for a surface that only needs the inverse image. Read `logoImageInverse` first; add IDs only for genuinely independent brands.
- Hardcoding brand text, colors, or image paths in the skin instead of reading `data.settings` with `|default()` fallbacks, which freezes out editor configuration.
- Forgetting `default.dwig`, so a missing skin selection breaks the brand site-wide.
