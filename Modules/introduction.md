# Modules — Introduction and How They Work

A module is behavior plus data. A module `.dwig` template is presentation. The module's backend loads settings and prepares the exact values for one job, such as a menu tree, a cart table, or a product list. Its DWIG skin turns those values into HTML. This separation lets one module ship many visual variants without duplicating its logic.

## 1. The render pipeline

Every module render follows the same five steps:

```text
Page template contains a <module> tag
        |
DevelopmentBucket parses the tag and runs the mapped module
        |
Module Backend loads instance settings and prepares data
        |
Theme Studio resolves the skin file (active theme first, bundled default second)
        |
The skin renders with that data
        |
HTML fragment replaces the <module> tag in the page
```

The module decides what data exists. The skin decides what HTML appears. A skin cannot invent data the module did not pass, and editing a skin never changes saved settings or what the module loads.

## 2. Embedding a module

Use a `<module>` tag inside any page, layout, partial, or another module template:

```twig
<module
    type="Navigation/Menu"
    id="main-navigation" # should be unique for a purpose 
    template="default.dwig"
/>
```

The three attributes that matter on every tag:

- `type` selects the module using its friendly Theme Studio name, such as `Navigation/Menu`, `Media/Slider`, or `Store/AddToCart`. Use the friendly names shown in each module's doc, such as `Navigation/Menu`, `Media/Slider`, or `Store/AddToCart`. Only registered modules appear in the Live Edit template selector.
- `id` identifies this instance and connects it to saved settings. Many module options are stored against this ID, so it must be stable and unique per independently configured instance. Changing it orphans the old settings scope. Reusing one ID for two different purposes makes those instances share settings.
- `template` selects the skin filename from that module's registered template directory, such as `Modules/Navigation/Menu/`. Pass only the filename (`centered.dwig`), never a path. An unavailable file falls back to `default.dwig` in modules that support fallback, so every module directory keeps a working `default.dwig`.

Other attributes are module inputs and differ per module. A module that needs the current product receives it explicitly:

```twig
<module
    type="Store/AddToCart"
    id="product-add-to-cart"
    product_id="{{ data.content.id }}"
    template="default.dwig"
/>
```

A template chosen in Live Edit is saved as that instance's `data-template` option and takes precedence over the inline `template` attribute. When a render ignores the filename in the tag, check the saved setting first, then the filename casing.

## 3. Where skins live and how they resolve

Module skins are grouped by purpose, then by module:

```text
Templates/Modules/
+-- Content/      Accordion, Content, FAQ, Tabs, Tags, TeamCard, Testimonials
+-- Media/        Audio, Logo, PictureGallery, Slider, Video
+-- Navigation/   MegaMenu, Menu, PageMenu
+-- Social/       SocialLinks, SocialSharer
+-- Store/        AddToCart, Cart, Checkout, Filter, Product, Products, Search, ...
+-- Users/        Dashboard, Login, Register
+-- Utilities/    Forms
+-- Website/      BlogComments, BlogCategories, Posts
```

Each module's doc shows its skin directory under `Templates/Modules/`. Resolution order for every skin:

1. The same relative path in the active theme.
2. The bundled default template.
3. `default.dwig` in the same directory, when the module supports fallback.

Keep theme-specific skins in the active theme. Changing the bundled fallback changes the default for every generated theme.

## 4. What data a skin receives

There is no universal data contract across modules. Each module prepares the exact values its skins may use, and most place them under `data`, commonly including `data.params` (tag attributes plus instance ID) and `data.config` (sanitized configuration). A skin must be written against its own module's contract. Never copy a variable from an unrelated module and assume it exists.

The safe pattern for any collection:

```twig
{% for item in data.items|default([]) %}
    <h3>{{ item.title|default('')|e }}</h3>
{% else %}
    <p>No items available.</p>
{% endfor %}
```

Auto-escaping is off in `.dwig` templates, so escape untrusted text with `|e` and attribute values with `|e('html_attr')`. Use `|raw` only for trusted or sanitized HTML. Helper functions such as `thumbnail()`, `get_picture()`, and `currency_format()` are available in all skins and documented in the template helper reference.

## 5. Creating a new skin

1. Copy the module's working `default.dwig` to a new filename in the same directory, for example `Templates/Modules/Navigation/Menu/compact.dwig`.
2. Keep the `.dwig` extension and the registered directory.
3. Preserve every class, `data-*` attribute, element ID, and form name the module's JavaScript or backend consumes. A skin can look correct while breaking interaction when those hooks are removed.
4. Change presentation while continuing to use the data the module passes.
5. Select it with `template="compact.dwig"` or through the module's Live Edit template setting.
6. Drive all frontend behavior through `window.dbEvent` from `userfiles/modules/developmentbucket/db_lib/events/common.js`, never the `mw` JS library. Cart skins use `dbEvent.shop.cart.update()` and `.remove()`, wishlist skins use `dbEvent.shop.wishlist`, and the header cart badge uses the `js-shopping-cart-quantity` class that `common.js` refreshes automatically.

## 6. Nesting and sharing markup

Module templates can contain other `<module>` tags for separate reusable behavior, with each nested instance getting its own stable ID. Never nest a module inside itself without a termination condition.

Share genuinely common markup between skins with `{% include %}`, using paths relative to the `Templates` root:

```twig
{% include "Modules/Navigation/Menu/partials/menu-item.dwig" %}
```

Never extend a full document layout such as `Layouts/main.dwig` from a module skin. A module renders inside a page that already supplies the document, header, and footer.

## 7. Loading a module via AJAX with `window.dbEvent.module`

Use `<module>` tags for the initial server render. Use `window.dbEvent.module.load(config)` when you need to load or re-load a module without a full page reload, for example lazy-loading below-the-fold content, refreshing the mini-cart drawer, or paginating a product list. It POSTs to `<DB_SITE_URL>module/` as URL-encoded form data and returns the same skin HTML the `<module>` tag would have rendered. All frontend JavaScript goes through `window.dbEvent` from `userfiles/modules/developmentbucket/db_lib/events/common.js`. Never reference the `mw` JS library.

The script loads automatically on normal frontend pages with the site URL already set. Do not include it a second time. On a standalone page, define `window.DB_SITE_URL`, `window.DB_API_URL`, and a `csrf-token` meta tag before loading it. The full method reference lives in `userfiles/modules/developmentbucket/db_lib/events/README.md`.

```html
<div id="sidebar-cart-placeholder"></div>
<script>
async function refreshSidebarCart() {
    try {
        await window.dbEvent.module.load({
            module: "Shop/Cart",
            id: "sidebar-cart",
            template: "mini.dwig",
            target: "#sidebar-cart-placeholder",
            replace: false
        });
    } catch (error) {
        console.error(error.code, error.message);
    }
}
</script>
```

Config properties:

| Property | Purpose |
|---|---|
| `module` | Module type to load, for example `"Shop/Cart"`. Always use `module`: a config with only `type` passes validation but sends no module type. |
| `id` | Module instance ID. Keep it stable and unique per purpose, exactly like the `<module>` tag `id`, because settings are stored against it. |
| `template` | Optional skin filename, for example `"mini.dwig"`. Same resolution and fallback as the tag attribute. |
| `target` | CSS selector string or `Element` receiving the HTML. Omit it to get the HTML back with no DOM change. |
| `replace` | Defaults to `true`: the target element is replaced via `replaceWith()`. Set `false` to keep the target and replace only its `innerHTML`. |
| `callback` | Optional function called with the result. If it throws, the returned Promise rejects after the DOM update. |
| `output` | Set to `"json"` only for modules that support raw JSON output, for example `shop/products`. Do not pass `target` in JSON mode. |
| Other properties | Passed to the module endpoint as module inputs, for example `limit`, `current_page`, `paginate`. |

Result and errors:

- With no `target`, the call only returns the HTML: `{ success: true, data: { html }, html, element: null }`.
- With a `target`, `element` is the replacement element (looked up by `id` when possible) or the target itself for `replace: false`.
- JSON mode (`output: "json"`) resolves to the module's JSON envelope instead, for example `response.data.posts` and `response.data.pagination` for `shop/products`.
- Every call uses `try/catch` because failures reject: `VALIDATION_ERROR` for a missing module type, bad `target`, or bad `callback`; `MODULE_REQUEST_FAILED` for a failing HTTP status; `MODULE_INVALID_JSON` when JSON mode returns non-JSON. Never render `error.details.body` to users; it can contain the raw server response.
- Treat returned HTML as trusted server-rendered markup from your own site and insert it only there.

```js
const response = await window.dbEvent.module.load({
    module: "shop/products",
    id: "products-api",
    output: "json",
    paginate: true,
    limit: 12,
    current_page: 1
});
console.log(response.data.posts);
console.log(response.data.pagination);
```

## 8. What each module doc covers

Every per-module file after this introduction documents one module family: the friendly `type` name, the template directory, the exact `data.*` contract with a field table, the skin variants beside `default.dwig`, the `<module>` attributes it accepts, the JavaScript hooks its skins must keep, and one minimal working example.

Per-module docs live beside this introduction, grouped by family directory:

```text
Modules/
+-- introduction.md          # this file: render pipeline, embedding, settings.json
+-- layouts-dwig.md          # Modules/Layouts section skins
+-- Media/
    +-- picture-gallery-dwig.md  # Media/PictureGallery gallery skins
    +-- logo-dwig.md             # Media/Logo brand skins
    +-- slider-dwig.md           # Media/Slider carousel skins
    +-- video-dwig.md            # Media/Video player skins
    +-- audio-dwig.md            # Media/Audio player skins
+-- Social/
    +-- social-sharer-dwig.md  # Social/SocialSharer share buttons
    +-- social-links-dwig.md   # Social/SocialLinks profile icons
+-- Store/
    +-- product-dwig.md          # Store/Product single-product skins
    +-- products-dwig.md         # Store/Products list skins
```

## 9. Adding per-skin settings with settings.json

A `settings.json` file gives editors form fields for a skin — a heading they can retype, an image they can swap, a color they can pick — without touching markup. The template designer declares the fields in JSON; the backend shows them as editable settings for each module instance and passes the saved values to the skin as `data.custom_settings.<key>`.

### 9.1 Where the file lives

One `settings.json` per module skin directory, sitting beside the skins it configures:

```text
Templates/Modules/Layouts/
+-- default.dwig
+-- hero-style-1.dwig
+-- features-style-1.dwig
+-- settings.json            # fields for every skin in this directory
```

Template keys inside the file are skin filenames, so one file covers all skins in the directory. A skin with no entry simply receives an empty `data.custom_settings` object.

### 9.2 File shape

```json
{
    "version": 1,
    "templates": {
        "hero-style-1.dwig": {
            "fields": [
                {
                    "key": "heading",
                    "type": "text",
                    "label": "Heading",
                    "default": "Our story"
                },
                {
                    "key": "hero_image",
                    "type": "image",
                    "label": "Hero image",
                    "default": ""
                },
                {
                    "key": "accent_color",
                    "type": "color",
                    "label": "Accent color",
                    "default": "#1d4ed8"
                }
            ]
        },
        "features-style-1.dwig": {
            "fields": []
        }
    }
}
```

Structural rules, enforced strictly — any violation means the settings are ignored for that file:

1. `version` must be exactly `1`.
2. Template keys are filenames only (`hero-style-1.dwig`), never paths. Subdirectory or `..` segments are rejected.
3. No extra properties anywhere. Only `version` and `templates` at the top level, only `fields` per template, and only `key`, `type`, `label`, `default` per field. An unknown property fails validation for the whole file.
4. At most 50 fields per skin.

### 9.3 Field reference

| Property | Rules |
|---|---|
| `key` | Starts with a letter; letters, numbers, underscores, or hyphens only. Unique within the skin. This is the name used as `data.custom_settings.<key>` in the skin. |
| `type` | One of `text`, `textarea`, `image`, `color`, `video`. Nothing else is accepted. |
| `label` | Plain text, 1–120 characters, shown to editors. No angle brackets or control characters. |
| `default` | String up to 10,000 characters. Used until an editor saves a value for that instance. May be `""` when there is no sensible default. |

### 9.4 The five field types and their value rules

| Type | Editor gets | Saved value rules | Render it as |
|---|---|---|---|
| `text` | Single-line input, max 500 chars | Plain string | `{{ settings.heading\|e }}` |
| `textarea` | Multi-line input, max 10,000 chars | Plain string, may contain line breaks | Escaped text; preserve line breaks with CSS (`white-space: pre-line`) |
| `image` | Image picker, max 2,048 chars | `http(s)` URL or a `userfiles/`, `media/`, or `storage/` path | `<img src="...\|e('html_attr')">`, guarded with `{% if %}` |
| `color` | Color picker, max 500 chars | `#hex` or a named color, or empty | `style` attribute value, guarded with `{% if %}` |
| `video` | Video picker, max 2,048 chars | Same path rules as `image` | `<video src="...\|e('html_attr')">`, guarded with `{% if %}` |

### 9.5 Reading values in the skin

Assign once at the top, then read each key with a fallback:

```twig
{% set settings = data.custom_settings|default({}) %}

<section class="hero py-5 edit" field="layout-hero-{{ data.params.id }}" rel="module"
    {% if settings.accent_color|default('') %}style="--accent: {{ settings.accent_color|e('html_attr') }};"{% endif %}>
    <div class="container">
        <h2>{{ settings.heading|default('Our story')|e }}</h2>
        {% if settings.hero_image|default('') %}
            <img src="{{ settings.hero_image|e('html_attr') }}" alt="{{ settings.heading|default('')|e }}" loading="lazy">
        {% endif %}
    </div>
</section>
```

Rules for reading:

1. Keep the `|default()` fallback on every read, even when the JSON declares a `default`. The fallback covers skins with no `settings.json` entry, where `data.custom_settings` is empty.
2. Guard `image`, `video`, and `color` with `{% if %}`. Any of them can be an empty string, and an empty `src` or `style` is worse than a missing one.
3. Escape by destination: `|e` for text, `|e('html_attr')` for attributes. Values come from editor input.
4. Values are stored per module instance ID, so each placement is configured independently. Like all instance settings, changing the tag's `id` orphans the saved values.

### 9.6 Full example — testimonial band skin

`settings.json` entry:

```json
{
    "version": 1,
    "templates": {
        "testimonials-style-1.dwig": {
            "fields": [
                {
                    "key": "eyebrow",
                    "type": "text",
                    "label": "Eyebrow",
                    "default": "What customers say"
                },
                {
                    "key": "heading",
                    "type": "text",
                    "label": "Heading",
                    "default": "Loved by teams everywhere"
                },
                {
                    "key": "background_color",
                    "type": "color",
                    "label": "Background color",
                    "default": ""
                }
            ]
        }
    }
}
```

`testimonials-style-1.dwig`:

```twig
{#  type: layout
    name: Testimonials style 1
    position: 3
    categories: Content
    screenshot: Assets/Preview/Layouts/testimonials-style-1.jpeg
#}
{% set settings = data.custom_settings|default({}) %}
<section class="testimonials py-5 edit"
         field="layout-testimonials-{{ data.params.id }}" rel="module"
         {% if settings.background_color|default('') %}style="background-color: {{ settings.background_color|e('html_attr') }};"{% endif %}>
    <div class="container text-center">
        <span class="eyebrow">{{ settings.eyebrow|default('What customers say')|e }}</span>
        <h2>{{ settings.heading|default('Loved by teams everywhere')|e }}</h2>
        <module type="Content/Testimonials" id="testimonials-{{ data.params.id }}" template="default.dwig" />
    </div>
</section>
```

### 9.7 Common mistakes

- Adding extra properties (a `help`, `options`, or `required` key) — validation rejects the whole file, not just the field.
- Setting `version` to anything but `1`, or omitting it.
- Using a path (`Layouts/hero-style-1.dwig`) instead of a bare filename as a template key.
- Duplicate `key` values within one skin, or keys starting with a number.
- HTML or angle brackets in `label`, which fails validation.
- Reading `data.custom_settings.heading` without `|default()`, which breaks skins that have no `settings.json` entry.
- Rendering `image`, `video`, or `color` unguarded, which emits empty `src` or `style` attributes before editors configure anything.
- Saving an image or video outside the permitted locations. Values must be full `http(s)` URLs or paths under `userfiles/`, `media/`, or `storage/`.
- Confusing `settings.json` fields with the editable content region (`edit` class with `field` and `rel="module"`). Fields configure the skin's declared options; the editable region holds free content the editor types in place. A section commonly uses both.
