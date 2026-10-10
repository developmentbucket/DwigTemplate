# Navigation/PageMenu — Automatic Page-Tree Menus

The `Navigation/PageMenu` module renders the automatic page-tree menu: links generated from the published page hierarchy instead of a hand-built menu. Its skins live in `Templates/Modules/Navigation/PageMenu/`. Where `Navigation/Menu` shows exactly the items editors arranged, the page menu follows the page tree itself — new pages appear without anyone editing a menu — with the current page highlighted.

## 1. Embedding a page menu

```twig
<module
    type="Navigation/PageMenu"
    id="sidebar-page-menu"
    template="default.dwig"
/>
```

- `type` is `Navigation/PageMenu`: the path-style name that mirrors the skin path `Templates/Modules/Navigation/PageMenu/`. The bare legacy name still resolves, but every example in these docs uses the path-style form.
- `id` identifies this instance and connects it to its scope settings (root page, depth) and skin choice. Sidebar, footer, and sitemap placements use different IDs so each keeps its own scope. Changing an ID orphans the settings saved under the old one.
- `template` selects the skin filename from the page-menu directory. Pass only the filename, never a path. A template chosen in Live Edit is saved against the instance and takes precedence over the inline attribute.
- An unavailable file falls back to `default.dwig`, so the directory keeps a working `default.dwig`.
- With no visible pages in scope, the skin renders nothing. Keep an `{% else %}` or `{% if %}` guard so the live page never shows a broken list.

## 2. Menu versus page menu

| Question | Use `Navigation/Menu` | Use `Navigation/PageMenu` |
|---|---|---|
| Who chooses the links? | Editors arrange items by hand. | The page tree decides automatically. |
| New page appears? | Only after someone adds it to the menu. | Immediately, in hierarchy position. |
| Custom order and labels? | Yes — full manual control. | No — tree order and page titles rule. |
| External links and anchors? | Yes. | No — pages only. |
| Typical surface? | Header nav, curated footers. | Sidebars, sitemaps, section indexes. |

If editors must curate, link outward, or reorder freely, that is a menu, not a page menu. If the list must always mirror the site structure with zero maintenance, that is a page menu.

## 3. Where skins live and how they resolve

```text
Templates/Modules/Navigation/PageMenu/
+-- default.dwig     # fallback, keep it working
+-- sidebar.dwig
+-- sitemap.dwig
+-- ...
```

Resolution order for every skin:

1. The same relative path in the active theme.
2. The bundled default template.
3. `default.dwig` in the same directory.

Keep theme-specific page-menu skins in the active theme. The bundled skins ship as empty stubs, so writing the sidebar and sitemap skins is part of building the theme — follow the tree pattern in section 4 so the first skins you write already behave like a page index.

## 4. The page-tree loop

Page-tree items carry the page's resolved link and title plus nested children. Loop defensively, highlight the current page, and recurse with the same guards as menu trees:

```twig
{% macro page_tree(items, level) %}
    {% import _self as nav %}
    <ul class="{{ level == 0 ? 'page-menu' : 'page-menu-children' }}">
    {% for item in items|default([]) %}
        {% set has_children = item.children|default([]) is not empty %}
        {% set item_url = item.url|default('') %}
        {% set is_active = item_url is not empty and (url_current()|trim('/') == item_url|trim('/')) %}
        <li class="page-menu-item{{ is_active ? ' active' : '' }}">
            <a href="{{ item_url|default('#')|e('html_attr') }}"
               {% if is_active %}aria-current="page"{% endif %}>{{ item.title|default('Untitled')|e }}</a>
            {% if has_children %}
                {{ nav.page_tree(item.children, level + 1) }}
            {% endif %}
        </li>
    {% else %}
        {% if level == 0 %}<li class="page-menu-item"><span>No pages available.</span></li>{% endif %}
    {% endfor %}
    </ul>
{% endmacro %}

{% import _self as nav %}
<nav aria-label="Page index">
    {{ nav.page_tree(data.pages|default(data.menu_items|default([])), 0) }}
</nav>
```

Rules:

1. Loop the page list with `|default([])` and an `{% else %}` branch. Draft-only sections and empty subtrees are normal; the branch is the design, not an error.
2. Test `item.children|default([])` before recursing. Childless pages carry no children key.
3. Highlight the current page with `active` plus `aria-current="page"`, comparing normalized URLs. The current-page marker is the page menu's whole advantage over a plain list — never ship the skin without it.
4. Link resolved `url` values only, never constructed paths. Page URLs change with slugs and locales; constructed links rot.
5. Escape titles and attributes. Page titles are editor input; `|raw` has no place in a page-menu skin.
6. Cap recursion depth for sitemap skins (a `level < 3` guard) so pathological nesting cannot blow out the page.

## 5. Use cases

**Docs or account sidebar.** Page-tree index beside long content, current page highlighted, children collapsed under their parents. The canonical page-menu surface.

**Sitemap page.** Full-depth tree with capped levels and counts. One stable instance, sitemap skin, zero maintenance as pages come and go.

**Footer page list.** Flat top-level pages only (depth one) for sitemap-style footers. Same data, minimal markup, no recursion visible.

**Section index.** Subtree under one root page for department, campus, or catalogue sections. Scope comes from the instance; the skin renders whatever arrives.

**Mobile page drawer.** Same tree as the sidebar in an off-canvas panel with collapse toggles per parent. Same scope, drawer-specific skin.

## 6. Common mistakes

- Hand-maintaining in a page menu what belongs in `Navigation/Menu`: custom order, external links, and marketing labels fight the automatic tree.
- Rendering without the current-page highlight, reducing the page menu to a dumber menu.
- Recursing into `children` without `|default([])`, or recursing without a depth cap on sitemap skins.
- Constructing page URLs instead of using resolved links.
- Unescaped titles. Page titles are editor input throughout.
- Sharing one instance ID between differently scoped placements (sidebar subtree versus full sitemap), merging two scopes into one settings scope.
- Using a page menu where products or posts list (`Store/Products`, `Website/Posts`) — it indexes pages, not catalogue records.
- Forgetting `default.dwig`, so a missing skin selection breaks the index site-wide.
