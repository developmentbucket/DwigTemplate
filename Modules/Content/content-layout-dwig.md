# Content/ContentLayout — Titled Card Columns

The `Content/ContentLayout` module renders a titled column section: a heading block plus a row of image cards, each with its own text and button, plus an optional section CTA. Its skins live in `Templates/Modules/Content/ContentLayout/`. Editors define the heading, alignment, column count, and card items in the module settings; the backend passes them ready-shaped; the skin renders the section.

## 1. Embedding a content layout

```twig
<module
    type="Content/ContentLayout"
    id="home-highlights"
    template="default.dwig"
/>
```

- `type` is `Content/ContentLayout`: the path-style name that mirrors the skin path `Templates/Modules/Content/ContentLayout/`. The bare legacy name still resolves, but every example in these docs uses the path-style form.
- `id` identifies this instance and connects it to its heading, columns, and card items. Keep it stable and unique per section. Changing it orphans the content saved under the old ID.
- `template` selects the skin filename from the content-layout directory. Pass only the filename, never a path. A template chosen in Live Edit is saved against the instance and takes precedence over the inline attribute.
- An unavailable file falls back to `default.dwig`, so the directory keeps a working `default.dwig`.
- With no items defined, the module shows an editor notice instead of the section. Keep an `{% if %}` guard so the live page never renders an empty frame.

## 2. Where skins live and how they resolve

```text
Templates/Modules/Content/ContentLayout/
+-- default.dwig     # fallback, keep it working
+-- wide.dwig
+-- ...
```

Resolution order for every skin:

1. The same relative path in the active theme.
2. The bundled default template.
3. `default.dwig` in the same directory.

Keep theme-specific sections in the active theme. Changing the bundled fallback changes the default for every generated theme.

## 3. What data a content-layout skin receives

| Value | Contents |
|---|---|
| `data.settings` | Section settings: `title`, `description`, `align` (default `'center'`), `maxColumns` (default `3`), `buttonLink` (default `'#'`), `buttonText` (default `''`, meaning no section CTA). |
| `data.content_items` | Card list in order. Each item carries `image`, `title`, `description`, `buttonText`, `buttonLink`. Any field can be empty — guard each one. |
| `data.params` | Tag attributes plus the instance ID (`data.params.id`). |

## 4. The section pattern

Heading block gated on `title`, card columns sized from `maxColumns`, per-card image/text/button guards, section CTA gated on `buttonText`:

```twig
{% set settings = data.settings|default({}) %}
{% set align = settings.align|default('center') %}
{% set max_columns = settings.maxColumns|default(3)|int %}
{% if max_columns < 1 %}{% set max_columns = 3 %}{% endif %}

<div class="layout-content-holder">
    {% if settings.title|default('') %}
        <div class="text-{{ align|e('html_attr') }}">
            <h2>{{ settings.title|e }}</h2>
            {% if settings.description|default('') %}
                <p>{{ settings.description|e }}</p>
            {% endif %}
        </div>
    {% endif %}

    {% set items = data.content_items|default([]) %}
    {% if items is not empty %}
    <div class="row mt-4">
        {% for item in items %}
            <div class="col-md-{{ (12 // max_columns)|e('html_attr') }} py-3 text-{{ align|e('html_attr') }}">
                {% if item.image|default('') %}
                    <img loading="lazy"
                         src="{{ thumbnail(item.image, 1000, 1000, true)|e('html_attr') }}"
                         alt="{{ item.title|default('')|e('html_attr') }}">
                {% endif %}
                <h5>{{ item.title|default('')|e }}</h5>
                {% if item.description|default('') %}
                    <p>{{ item.description|e }}</p>
                {% endif %}
                {% if item.buttonText|default('') %}
                    <a href="{{ item.buttonLink|default('#')|e('html_attr') }}"
                       class="btn btn-primary" target="_blank" rel="noopener">{{ item.buttonText|e }}</a>
                {% endif %}
            </div>
        {% endfor %}
    </div>
    {% endif %}

    {% if settings.buttonText|default('') %}
        <div class="text-{{ align|e('html_attr') }} mt-4">
            <a href="{{ settings.buttonLink|default('#')|e('html_attr') }}"
               class="btn btn-primary" target="_blank" rel="noopener">{{ settings.buttonText|e }}</a>
        </div>
    {% endif %}
</div>
```

Rules:

1. Read `align` and `maxColumns` once with validated fallbacks. Clamp `maxColumns` to a sane range (1–4, default 3) before dividing the grid — a zero or absurd value breaks every column.
2. Gate the heading on `title`, the description on `description`, each card image, button, and the section CTA on its own value. This module is mostly optional parts; unguarded empties print blank headings, broken images, and dead buttons.
3. Pass card images through `thumbnail()` at the displayed size with crop for uniform cards. Never output original uploads.
4. Escape titles, descriptions, and attributes. Card copy is editor input; `|raw` has no place in a content-layout skin.
5. Keep `target="_blank" rel="noopener"` paired on off-site buttons. Same-site links stay in the same tab; decide per button, not per skin.

## 5. Use cases

**Feature highlights.** Three columns with icon-ish images, titles, and one-line copy. The homepage standard.

**Service cards.** Image, title, short description, and per-card CTA linking each service page. Same data, service copy.

**Steps or process.** Numbered titles with descriptions in 3–4 columns. No buttons; the section CTA carries the single action.

**Press or logo wall.** Images only, titles as `alt`, no copy or buttons. Minimal skin over the same items.

**Testimonial-ish quotes.** Short quote as description with author as title. For attributed praise prefer `Content/Testimonials`; this skin fits unattributed pull quotes.

## 6. Common mistakes

- Dividing the grid by unvalidated `maxColumns`, breaking columns on zero or text values.
- Rendering heading, images, or buttons unguarded, printing blanks and dead controls before editors configure anything.
- Outputting original uploads instead of `thumbnail()` sizes.
- Unescaped titles or descriptions.
- Same-tab versus new-tab decided never: off-site buttons need the `noopener` pair, same-site links must not open tabs.
- Rebuilding this section per page instead of reusing one instance, forking identical content into many settings scopes.
- Using this module for people (`Content/TeamCard`), tabbed content (`Content/Tabs`), or product grids (`Store/Products`).
- Forgetting `default.dwig`, so a missing skin selection breaks sections site-wide.
