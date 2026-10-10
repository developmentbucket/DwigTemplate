# Navigation/Menu — Site Navigation Rows and Trees

The `Navigation/Menu` module renders a menu tree: the header nav, footer links, mobile drawer, sidebar lists. Its skins live in `Templates/Modules/Navigation/Menu/`. Editors build the menu (items, links, nesting) in Live Edit; the backend resolves each item's title and URL and passes the ready tree; the skin renders one level or the full depth.

## 1. Embedding a menu

```twig
<module
    type="Navigation/Menu"
    id="main-navigation"
    template="default.dwig"
/>
```

- `type` is `Navigation/Menu`: the path-style name that mirrors the skin path `Templates/Modules/Navigation/Menu/`. The bare legacy name still resolves, but every example in these docs uses the path-style form.
- `id` identifies this instance and connects it to its menu selection and skin choice. Header, footer, and sidebar menus use different IDs (`main-navigation`, `footer-navigation`) so each is configured independently. Changing an ID orphans the settings saved under the old one.
- `template` selects the skin filename from the menu directory. Pass only the filename, never a path. A template chosen in Live Edit is saved against the instance and takes precedence over the inline attribute.
- An unavailable file falls back to `default.dwig`, so the directory keeps a working `default.dwig`.
- With no menu selected, the module shows an editor notice instead of links. Keep an `{% else %}` or `{% if %}` guard so the live page never renders a broken nav.

Menu selection attributes:

| Attribute | Purpose |
|---|---|
| `menu-name` / `name` | Which menu renders (`header_menu` by default). A saved `menu_name` setting wins over the attribute. |
| `ul_class` | List class hint for the skin. Default `'nav'`. Skins may honor it for the top-level `<ul>`. |

## 2. Where skins live and how they resolve

```text
Templates/Modules/Navigation/Menu/
+-- default.dwig     # fallback, keep it working
+-- footer.dwig      # plain stacked links
+-- sidebar.dwig
+-- ...
```

Resolution order for every skin:

1. The same relative path in the active theme.
2. The bundled default template.
3. `default.dwig` in the same directory.

Keep theme-specific menu skins in the active theme. The bundled skins ship as empty stubs, so writing the header, footer, and mobile skins is part of building the theme.

## 3. What data a menu skin receives

| Value | Contents |
|---|---|
| `data.menu_items` | Menu tree in position order, already nested. Each item carries the fields below; parents carry `children` only when they have visible children. Always loop with `\|default([])`. |
| `data.menuFilter` | Resolved menu filter (`menu_id`, list classes). Informational for most skins. |
| `data.params` | Tag attributes plus the instance ID (`data.params.id`). |

Menu item fields:

| Field | Contents |
|---|---|
| `id` | Item ID. Feeds dropdown toggle IDs and hooks. |
| `title` | Link text, resolved from the item or its linked content/category. Escape with `\|e`. |
| `url` | Resolved link (content, category, or custom URL). Never construct links. |
| `parent_id` | Parent item ID. Informational; nesting already arrives via `children`. |
| `children` | Nested items, same shape. Present only when non-empty — guard with `\|default([])`. |
| `item_type` | Item kind. `mega_menu` marks a mega-menu trigger with `template`, `template_path`, and `mega_menu` payload attached. |

## 4. The tree loop

Render from the data, not from hardcoded links. One recursive macro serves flat rows and nested dropdowns alike:

```twig
{% macro menu_tree(items, level) %}
    {% import _self as nav %}
    <ul class="{{ level == 0 ? 'nav gap-3' : 'dropdown-menu' }}">
    {% for item in items|default([]) %}
        {% set has_children = item.children|default([]) is not empty %}
        {% set is_active = (url_current()|trim('/') == item.url|default('')|trim('/')) %}
        <li class="{{ level == 0 ? 'nav-item' : '' }}{{ has_children ? ' dropdown' : '' }}">
            {% if item.item_type|default('') == 'mega_menu' %}
                <button class="nav-link btn btn-link{{ is_active ? ' active' : '' }}" type="button"
                        data-bs-toggle="dropdown" aria-expanded="false"
                        data-menu-id="{{ item.id|e('html_attr') }}">
                    {{ item.title|default('')|e }}
                </button>
            {% else %}
                <a class="{{ level == 0 ? 'nav-link' : 'dropdown-item' }}{{ is_active ? ' active' : '' }}"
                   href="{{ item.url|default('#')|e('html_attr') }}"
                   {% if is_active %}aria-current="page"{% endif %}>{{ item.title|default('')|e }}</a>
            {% endif %}
            {% if has_children %}
                {{ nav.menu_tree(item.children, level + 1) }}
            {% endif %}
        </li>
    {% else %}
        {% if level == 0 %}<li class="nav-item"><span>No menu items.</span></li>{% endif %}
    {% endfor %}
    </ul>
{% endmacro %}

{% import _self as nav %}
<nav aria-label="Main navigation">
    {{ nav.menu_tree(data.menu_items, 0) }}
</nav>
```

Rules:

1. Loop `data.menu_items`. Hardcoded links freeze the menu in markup and silently ignore everything editors configure — several shipped stub skins do this, and it is the top menu bug to avoid.
2. Test `item.children|default([])` before recursing. The key exists only on parents with visible children.
3. Branch `mega_menu` items into triggers, never plain links. Their payload (`template`, `template_path`, `mega_menu`) feeds the mega-menu module, not an `href`.
4. Mark the current page with `active` plus `aria-current="page"`, comparing normalized URLs. Unnormalized comparison (trailing slash, query) misses the match.
5. Give every dropdown toggle a unique ID derived from `item.id` when wiring `aria-controls` or collapse targets. Duplicated toggle IDs collapse the wrong panel.
6. Escape titles and URLs. Menu text is editor input; `|raw` has no place in a menu skin.

## 5. Which menu renders where

The rendered menu comes from the saved `menu_name` first and the `menu-name` attribute second (`header_menu` when neither is set):

```twig
{# Header: primary menu, dropdown skin #}
<module type="Navigation/Menu" id="main-navigation" menu-name="header_menu" template="default.dwig" />

{# Footer: different menu, plain stacked skin #}
<module type="Navigation/Menu" id="footer-navigation" menu-name="footer_menu" template="footer.dwig" />
```

1. Give each placement its own ID and, where the link sets differ, its own menu. Sharing one ID between header and footer forces both rows into one menu and one skin selection.
2. Match the skin to the placement: dropdown chrome for headers, plain stacked links for footers, collapse panels for mobile drawers. One skin stretched across all three does none well.
3. Mobile and desktop share one menu tree through one ID when their links match (add a second skin, not a second menu); split IDs only when mobile needs different links.

## 6. Use cases

**Desktop header nav.** Top-level row with dropdowns for parents and a mega trigger where `item_type` says so. `aria-current` on the active page.

**Footer link columns.** Flat stacked links, no dropdowns, no active state. One column per menu instance when footers split Explore/Company/Help.

**Mobile drawer.** Same tree as desktop in an off-canvas panel with collapse toggles per parent. Same menu ID, drawer-specific skin.

**Sidebar section nav.** Vertical list for docs, account, or category-adjacent pages. Active trail highlighted through the same comparison as headers.

**Utility row.** Compact login, wishlist, and contact links above the header. Separate micro-menu, minimal skin, no dropdowns.

## 7. Common mistakes

- Hardcoding links instead of looping `data.menu_items`, freezing editors out of their own menu.
- Sharing one instance ID between header and footer, merging two menus into one configuration.
- Recursing into `children` without `|default([])`.
- Rendering `mega_menu` items as plain links, dropping their payload and breaking the mega panel.
- Duplicated dropdown/collapse IDs across items, toggling the wrong panel.
- Missing `aria-current` on the active link, or active comparison that never matches from unnormalized URLs.
- Unescaped titles or URLs. Menus are editor input throughout.
- Using this module for breadcrumbs (per-page trail, not a configured menu) or category grids (`Store/StoreCategories`).
- Forgetting `default.dwig`, so a missing skin selection breaks navigation site-wide.
