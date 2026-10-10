# Content/Tabs — Tabbed Content Sections

The `Content/Tabs` module renders tabbed content: a button row plus one panel per tab, each panel freely editable. Its skins live in `Templates/Modules/Content/Tabs/`. Editors define the tabs (title, icon) in the module settings; each tab's body is edited in place on the page; the backend passes the tab list and the skin renders nav plus panes.

## 1. Embedding tabs

```twig
<module
    type="Content/Tabs"
    id="product-details-tabs"
    template="default.dwig"
/>
```

- `type` is `Content/Tabs`: the path-style name that mirrors the skin path `Templates/Modules/Content/Tabs/`. The bare legacy name still resolves, but every example in these docs uses the path-style form.
- `id` identifies this instance and connects it to its tab definitions and every tab's edited body. Keep it stable and unique per tab set. Changing it orphans the tabs and their content saved under the old ID.
- `template` selects the skin filename from the tabs directory. Pass only the filename, never a path. A template chosen in Live Edit is saved against the instance and takes precedence over the inline attribute.
- An unavailable file falls back to `default.dwig`, so the directory keeps a working `default.dwig`.
- With no tabs defined, the module shows an editor notice instead of the widget. Keep an `{% if %}` guard so the live page never renders an empty tab frame.

## 2. Where skins live and how they resolve

```text
Templates/Modules/Content/Tabs/
+-- default.dwig     # fallback, keep it working
+-- pills.dwig
+-- ...
```

Resolution order for every skin:

1. The same relative path in the active theme.
2. The bundled default template.
3. `default.dwig` in the same directory.

Keep theme-specific tab skins in the active theme. The bundled skins ship as empty stubs, so writing the tabs skins is part of building the theme — follow the nav-plus-panes pattern in section 4 so the first skin you write already behaves like the module's native output.

## 3. What data a tabs skin receives

| Value | Contents |
|---|---|
| `data.tabs_items` | Tab definitions in order. Each carries `title`, `icon` (icon HTML), and `id`. Missing IDs arrive prefilled as `tab-<instance>-<n>`. Tab bodies are edited in place, not passed here. |
| `data.settings` | Raw tab-settings JSON for the instance. Informational for most skins; the parsed list above is what skins render. |
| `data.params` | Tag attributes plus the instance ID (`data.params.id`). |

## 4. Nav plus editable panes

One loop renders the buttons; a second loop renders the panels. First tab active, the rest hidden. Each panel holds an editable drop region keyed by its tab ID:

```twig
{% set tabs = data.tabs_items|default([]) %}
{% set instance = data.params.id|default('tabs') %}

{% if tabs is not empty %}
<div class="content-tabs" id="content-tabs-{{ instance|e('html_attr') }}">
    <div class="nav nav-tabs" role="tablist">
    {% for tab in tabs %}
        <button class="nav-link{% if loop.first %} active{% endif %}" type="button"
                role="tab" aria-selected="{{ loop.first ? 'true' : 'false' }}"
                aria-controls="tab-pane-{{ tab.id|default('n' ~ loop.index)|e('html_attr') }}"
                id="tab-btn-{{ tab.id|default('n' ~ loop.index)|e('html_attr') }}"
                data-bs-toggle="tab"
                data-bs-target="#tab-pane-{{ tab.id|default('n' ~ loop.index)|e('html_attr') }}">
            {{ tab.icon|default('')|raw }}{{ tab.title|default('Tab')|e }}
        </button>
    {% endfor %}
    </div>

    <div class="tab-content">
    {% for tab in tabs %}
        {% set pane_id = 'tab-pane-' ~ tab.id|default('n' ~ loop.index) %}
        <div class="tab-pane fade{% if loop.first %} show active{% endif %}" role="tabpanel"
             id="{{ pane_id|e('html_attr') }}"
             aria-labelledby="tab-btn-{{ tab.id|default('n' ~ loop.index)|e('html_attr') }}">
            <div class="edit allow-drop"
                 field="tab-item-{{ tab.id|default('n' ~ loop.index)|e('html_attr') }}"
                 rel="module-{{ instance|e('html_attr') }}">
                <div class="element">{{ tab.content|default('Tab content ' ~ loop.index)|e }}</div>
            </div>
        </div>
    {% endfor %}
    </div>
</div>
{% endif %}
```

Rules:

1. Derive button, pane, and field names from the tab's own `id`, never the loop index alone. Reordered tabs keep their bodies only when the field key follows the tab, and the backend already guarantees unique tab IDs per instance.
2. Scope the editable region with `field="tab-item-<tab id>"` and `rel="module-<instance id>"`, with the `allow-drop` class so editors can drop modules inside tab bodies. This exact shape is what makes pane content editable and scoped.
3. Mark exactly one tab active (`loop.first`): button `active` plus `aria-selected="true"`, pane `show active`. Zero or two active panes both break the widget.
4. Pair every control explicitly: `aria-controls`/`data-bs-target` on the button equal the pane `id`; pane `aria-labelledby` equals the button `id`. Duplicate or crossed IDs toggle the wrong panel.
5. Render `icon` with `|raw` (module-supplied icon markup) and `title` with `|e` (editor text). Never swap the two.
6. Guard the whole widget with `{% if tabs is not empty %}`. An empty tab frame with one dead button is worse than nothing.

## 5. Use cases

**Product information tabs.** Description, details, and reviews panes on product pages. One stable ID; bodies edited per product page.

**FAQ tabs.** Question groups split across panes for long help pages. Short titles, generous pane bodies.

**Service detail tabs.** Pricing, process, and coverage panes for service pages. Same widget, service copy.

**Specification tabs.** Tech specs, documents, and compatibility panes for complex products. Tables and file links live in the editable bodies.

**Onboarding steps.** Numbered tabs walking new users through setup. Titles carry the step numbers; panes carry the instructions.

## 6. Common mistakes

- Keying editable fields by loop index instead of tab ID, so reordering tabs scrambles every pane body.
- Dropping `rel="module-<instance>"` or the `allow-drop` class, breaking content scoping or module drops inside panes.
- Marking zero or multiple tabs active.
- Crossed `aria-controls` / pane IDs, toggling the wrong panel.
- Escaping `icon` with `|e` (printing markup as text) or rendering `title` with `|raw` (injecting editor HTML).
- Rendering the widget frame with no tabs instead of guarding with `{% if %}`.
- Rebuilding tab switching by hand instead of the theme's tab behavior, then fighting its active classes.
- Forgetting `default.dwig`, so a missing skin selection breaks tabbed sections site-wide.
