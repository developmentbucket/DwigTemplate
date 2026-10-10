# Navigation/MegaMenu — Full-Width Dropdown Panels

The `Navigation/MegaMenu` module renders the full-width dropdown panel behind a header menu trigger: category columns, promo cards, featured links. Its skins live in `Templates/Modules/Navigation/MegaMenu/`. The backend passes only the tag attributes; the skin composes the panel from nested modules — categories, products, links — each with its own data and settings.

## 1. Embedding a mega panel

```twig
<module
    type="Navigation/MegaMenu"
    id="shop-mega-menu"
    template="default.dwig"
/>
```

- `type` is `Navigation/MegaMenu`: the path-style name that mirrors the skin path `Templates/Modules/Navigation/MegaMenu/`. The bare legacy name still resolves, but every example in these docs uses the path-style form.
- `id` identifies this instance. One panel per trigger is the norm; keep the ID stable so its skin selection persists.
- `template` selects the skin filename from the mega-menu directory. Pass only the filename, never a path. A template chosen in Live Edit is saved against the instance and takes precedence over the inline attribute.
- An unavailable file falls back to `default.dwig`, so the directory keeps a working `default.dwig`.

The panel is opened from its menu trigger, not rendered inline in the nav flow. Place the module once near the header (or wherever the design anchors the panel) and wire the trigger to it (section 5).

## 2. Where skins live and how they resolve

```text
Templates/Modules/Navigation/MegaMenu/
+-- default.dwig     # fallback, keep it working
+-- shop.dwig
+-- services.dwig
+-- ...
```

Resolution order for every skin:

1. The same relative path in the active theme.
2. The bundled default template.
3. `default.dwig` in the same directory.

Keep theme-specific panels in the active theme. Changing the bundled fallback changes the default for every generated theme.

## 3. What data a mega skin receives

| Value | Contents |
|---|---|
| `data.params` | Tag attributes plus the instance ID (`data.params.id`). That is the entire contract — no item list arrives. |

Every visible row inside the panel comes from a nested `<module>` tag the skin composes. The skin's job is the shell (panel frame, intro column, grid) plus one nested module per content column:

```twig
<div class="mega-menu bg-body border-top shadow-lg" id="megaMenu" aria-hidden="true">
    <div class="container mega-grid row g-4 py-4">
        <div class="mega-intro col-lg-3">
            <span class="eyebrow">Explore our pantry</span>
            <h3>Made the way home remembers.</h3>
            <a class="text-link" href="{{ site_url('shop')|e('html_attr') }}">Shop all products</a>
        </div>
        <div class="col-lg-9">
            <module type="Store/StoreCategories" id="top-mega-menu-categories" template="megaMenu.dwig" />
        </div>
    </div>
</div>
```

Rules:

1. Compose; do not query. The panel has no data of its own — each nested module brings its own contract. Write each column against that module's doc.
2. Give every nested module a stable global ID (`top-mega-menu-categories`). The panel renders site-wide, so one ID means one settings scope, configured once.
3. Keep the panel shell static except for the nested modules. Hardcoded intro copy belongs to the design; editor-editable intro copy belongs in the surrounding layout's editable region, not inside this skin.
4. Never nest a `Navigation/MegaMenu` inside its own skin. Panels compose content modules, never themselves.

## 4. Panel shell — identity and visibility

The shell is one landmark element addressed by the trigger. Keep its identity stable and its hidden state honest:

```twig
<div class="mega-menu" id="megaMenu" aria-hidden="true" aria-label="Shop departments">
```

1. Use one fixed panel ID (`megaMenu`) per panel, matched exactly by the trigger's `aria-controls` and the theme script's selector. A second panel needs a second ID on both sides.
2. Start hidden (`aria-hidden="true"`, CSS-hidden) and toggle both together. A panel that looks closed but stays exposed to assistive technology traps keyboard users.
3. Return focus to the trigger on close and move focus into the panel on open. Escape closes the panel from anywhere inside it.
4. One shared panel serves its trigger across the site. Do not duplicate the panel per page or per menu item; the trigger in the menu skin points at the one panel.

## 5. Wiring the menu trigger

The `Navigation/Menu` skin renders the trigger for `mega_menu` items. The trigger and panel connect through IDs and expanded state:

```twig
<button class="nav-link btn btn-link" type="button"
        aria-expanded="false" aria-controls="megaMenu"
        data-menu-id="{{ item.id|e('html_attr') }}">
    {{ item.title|default('')|e }}
</button>
```

1. Render `mega_menu` items as `<button>`, never links. They open a panel; they navigate nowhere.
2. Keep `aria-controls` equal to the panel ID and flip `aria-expanded` with visibility. The theme script toggles the panel on these attributes.
3. Carry `data-menu-id="{{ item.id }}"` so the script can distinguish triggers when several panels exist.
4. Triggers live in the menu skin; panels live in their own module placement. Splitting them keeps menu editing and panel content independent.

## 6. Use cases

**Shop-all panel.** Intro column plus category cards (`Store/StoreCategories` in a menu-card skin). The catalogue front door behind one nav item.

**Services panel.** Link columns plus a featured card with image and CTA. Same shell, service content, no prices.

**Promo panel.** Campaign links plus one banner card for sales and launches. Swap the nested content per campaign; the shell stays.

**Brand panel.** About links, certifications, and a story blurb for About-style triggers. Static intro, link columns from a menu instance.

## 7. Common mistakes

- Duplicating the panel per page instead of one shared panel per trigger.
- Mismatched panel ID and trigger `aria-controls`, so the button opens nothing (or the wrong panel).
- Hiding the panel visually while leaving `aria-hidden` false, trapping assistive technology.
- Rendering `mega_menu` items as links in the menu skin, navigating away instead of opening the panel.
- Nesting content with throwaway per-page IDs, forking the panel's settings once per page.
- Putting editor-managed copy inside the panel skin instead of the layout's editable region, freezing it in markup.
- Building the panel's columns by hand instead of composing the content modules, duplicating their data logic in static HTML.
- Forgetting `default.dwig`, so a missing skin selection breaks the header panel site-wide.
