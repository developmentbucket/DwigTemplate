# Content/Accordion — Collapsible Content Sections

The `Content/Accordion` module renders collapsible sections: one header button per item, one expandable body each. Its skins live in `Templates/Modules/Content/Accordion/`. Editors define the items (title, icon) in the module settings and edit each body in place; the backend passes the item list and the skin renders headers plus collapsible panes.

## 1. Embedding an accordion

```twig
<module
    type="Content/Accordion"
    id="faq-accordion"
    template="default.dwig"
/>
```

- `type` is `Content/Accordion`: the path-style name that mirrors the skin path `Templates/Modules/Content/Accordion/`. The bare legacy name still resolves, but every example in these docs uses the path-style form.
- `id` identifies this instance and connects it to its items and every item's edited body. Keep it stable and unique per accordion. Changing it orphans the items and bodies saved under the old ID.
- `template` selects the skin filename from the accordion directory. Pass only the filename, never a path. A template chosen in Live Edit is saved against the instance and takes precedence over the inline attribute.
- An unavailable file falls back to `default.dwig`, so the directory keeps a working `default.dwig`.
- With no items defined, the module shows an editor notice instead of the widget. Keep an `{% if %}` guard so the live page never renders an empty frame.

## 2. Where skins live and how they resolve

```text
Templates/Modules/Content/Accordion/
+-- default.dwig     # fallback, keep it working
+-- flush.dwig
+-- ...
```

Resolution order for every skin:

1. The same relative path in the active theme.
2. The bundled default template.
3. `default.dwig` in the same directory.

Keep theme-specific accordion skins in the active theme. The bundled skins ship as empty stubs, so writing the accordion skins is part of building the theme — follow the headers-plus-panes pattern in section 4 so the first skin you write already behaves like the module's native output.

## 3. What data an accordion skin receives

| Value | Contents |
|---|---|
| `data.accordion_items` | Item list in order. Each carries `title`, `icon` (icon HTML), `id`, and `content` (body text). Missing IDs arrive prefilled as `accordion-<instance>-<n>`. |
| `data.settings` | Raw item-settings JSON for the instance. Informational for most skins; the parsed list above is what skins render. |
| `data.use_content_from_live_edit` | Whether bodies are in-place editable. Gate the editable region on this flag (section 4). |
| `data.params` | Tag attributes plus the instance ID (`data.params.id`). |

## 4. Headers plus collapsible panes

One loop renders header buttons and collapsible bodies. First item open, the rest collapsed. Each body holds an editable drop region keyed by its item ID when live-edit bodies are enabled:

```twig
{% set items = data.accordion_items|default([]) %}
{% set instance = data.params.id|default('accordion') %}
{% set editable = data.use_content_from_live_edit|default(false) %}

{% if items is not empty %}
<div class="accordion" id="accordion-{{ instance|e('html_attr') }}">
{% for item in items %}
    {% set key = item.id|default('n' ~ loop.index) %}
    {% set header_id = 'accordion-header-' ~ key %}
    {% set pane_id = 'accordion-pane-' ~ key %}
    <div class="accordion-item">
        <h3 class="accordion-header" id="{{ header_id|e('html_attr') }}">
            <button class="accordion-button{% if not loop.first %} collapsed{% endif %}" type="button"
                    data-bs-toggle="collapse" data-bs-target="#{{ pane_id|e('html_attr') }}"
                    aria-expanded="{{ loop.first ? 'true' : 'false' }}"
                    aria-controls="{{ pane_id|e('html_attr') }}">
                {{ item.icon|default('')|raw }}{{ item.title|default('Title')|e }}
            </button>
        </h3>
        <div id="{{ pane_id|e('html_attr') }}"
             class="accordion-collapse collapse{% if loop.first %} show{% endif %}"
             aria-labelledby="{{ header_id|e('html_attr') }}"
             data-bs-parent="#accordion-{{ instance|e('html_attr') }}">
            <div class="accordion-body">
                {% if editable %}
                    <div class="allow-drop edit"
                         field="accordion-item-{{ key|e('html_attr') }}"
                         rel="module-{{ instance|e('html_attr') }}">
                        <div class="element">{{ item.content|default('Type your text here')|e }}</div>
                    </div>
                {% else %}
                    <p>{{ item.content|default('Type your text here')|e }}</p>
                {% endif %}
            </div>
        </div>
    </div>
{% endfor %}
</div>
{% endif %}
```

Rules:

1. Derive header, pane, and field names from the item's own `id`, never the loop index alone. Reordered items keep their bodies only when the field key follows the item, and the backend already guarantees unique item IDs per instance.
2. Scope the editable region with `field="accordion-item-<item id>"` and `rel="module-<instance id>"`, with the `allow-drop` class — and only when `use_content_from_live_edit` is true. Otherwise render the body as static text.
3. Open exactly one item (`loop.first`): button expanded without `collapsed`, pane with `show`. Zero or two open panes both break the widget.
4. Pair every control explicitly: `data-bs-target` and `aria-controls` equal the pane `id`; pane `aria-labelledby` equals the header `id`; panes share one `data-bs-parent` scoped to the instance wrapper so opening one closes the others.
5. Render `icon` with `|raw` (module-supplied icon markup) and `title` plus `content` with `|e` (editor text). Never swap the two.
6. Guard the whole widget with `{% if items is not empty %}`. An empty accordion frame with one dead header is worse than nothing.

## 5. Use cases

**FAQ list.** Question headers with answer bodies, first answer open. The canonical accordion surface.

**Product details.** Shipping, returns, and care panes below the buy box. One stable ID per product-page placement.

**Policy sections.** Terms, privacy, and cookie panes on legal pages. Long bodies live in the editable regions, not in settings.

**Onboarding steps.** Numbered headers walking new users through setup. Titles carry the step numbers.

**Sidebar help.** Compact accordion in support sidebars linking related answers. Narrow CSS, same markup.

## 6. Common mistakes

- Keying editable fields by loop index instead of item ID, so reordering items scrambles every body.
- Dropping `rel="module-<instance>"` or the `allow-drop` class, breaking content scoping or module drops inside bodies.
- Ignoring `use_content_from_live_edit` and always (or never) rendering the editable region.
- Opening zero or multiple panes, or forgetting `data-bs-parent` so panes stop behaving as one accordion.
- Crossed header/pane IDs, expanding the wrong body.
- Escaping `icon` with `|e` (printing markup as text) or rendering `title` and `content` with `|raw` (injecting editor HTML).
- Rendering the widget frame with no items instead of guarding with `{% if %}`.
- Using an accordion for tabbed workflows with parallel panes (`Content/Tabs`) or single toggles that need no group behavior.
- Forgetting `default.dwig`, so a missing skin selection breaks collapsible sections site-wide.
