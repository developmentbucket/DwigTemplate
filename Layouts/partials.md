# Layouts/Partials — Shared Fragments for main.dwig

`Templates/Layouts/Partials/` holds small reusable `.dwig` fragments that `main.dwig` pulls in with `{% include %}`. The bundled directory ships empty: header, footer, and every fragment below are theme-authored conventions, not supplied files. Create only the ones your theme needs.

## 1. Why partials exist

`main.dwig` should stay a thin shell: doctype, head, header, content block, footer, scripts. Everything with real markup moves into a partial so the shell stays readable and each piece can evolve alone:

```twig
{# Templates/Layouts/main.dwig #}
<body class="{{ helper_body_classes() }}">
    {% include "Layouts/Partials/header.dwig" %}
    {% block content %}{% endblock %}
    {% include "Layouts/Partials/footer.dwig" %}
</body>
```

Rules for includes:

- Paths are relative to the `Templates` root, and the active theme is searched before the bundled defaults.
- A partial is a fragment. It must not contain `<html>`, `<head>`, or `<body>`, and it must not extend a layout.
- A partial can contain `<module>` tags. That is the normal way to give a fragment live behavior with saved settings.

## 2. header.dwig — site header and navigation

The header combines branding, navigation, search, and shop controls in one place so every page renders them identically.

```twig
{# Templates/Layouts/Partials/header.dwig #}
<header class="site-header sticky-top bg-white shadow-sm">
    <div class="container d-flex align-items-center gap-4 py-3">
        <a class="navbar-brand" href="{{ site_url()|e('html_attr') }}">
            <module type="Media/Logo" id="site-logo" template="default.dwig" />
        </a>
        <nav class="flex-grow-1" aria-label="Main navigation">
            <module type="Navigation/Menu" id="main-navigation" template="default.dwig" />
        </nav>
        <module type="Store/Search" id="header-search" template="default.dwig" />
        <a class="btn btn-outline-secondary position-relative"
           href="{{ get_option('cart_page_url', 'shop')|e('html_attr') }}"
           aria-label="Open cart">
            Cart
            <span class="badge bg-primary js-shopping-cart-quantity">0</span>
        </a>
    </div>
    {% include "Layouts/Partials/announcement-bar.dwig" ignore missing %}
</header>
```

Notes:

- Every module gets a stable global ID (`site-logo`, `main-navigation`). The header renders on all pages, so one ID means one settings scope for the whole site. Per-page IDs here would silently fork the header settings per page.
- The bundled `Navigation/Menu` and `Media/Logo` skins ship as empty stubs, so writing these two skins is part of building the header. Keep the module's JS hooks and `data-*` attributes when you skin them.
- `ignore missing` on the announcement bar lets pages render before that optional partial exists.
- Use case variants: a transparent over-hero header for the home page (add a `header_transparent` block or a second partial), a slim header without search for checkout and landing shells, and a multilingual header that adds a language-switcher module.

## 3. footer.dwig — site footer

The footer mirrors the header: link columns, contact summary, social icons, and legal row.

```twig
{# Templates/Layouts/Partials/footer.dwig #}
<footer class="site-footer bg-dark text-light mt-5">
    <div class="container py-5">
        <div class="row g-4">
            <div class="col-md-4">
                <module type="Media/Logo" id="footer-logo" template="footer.dwig" />
                <p class="mt-3 mb-0">{{ get_option('website_description')|default('')|e }}</p>
            </div>
            <div class="col-md-4">
                <h2 class="h6">Explore</h2>
                <module type="Navigation/Menu" id="footer-navigation" template="footer.dwig" />
            </div>
            <div class="col-md-4">
                <h2 class="h6">Follow us</h2>
                <module type="Social/SocialLinks" id="footer-social" template="default.dwig" />
            </div>
        </div>
        <hr>
        <p class="mb-0 small">&copy; {{ "now"|date("Y") }} {{ get_option('website_title', 'website')|e }}. All rights reserved.</p>
    </div>
</footer>
```

Notes:

- Footer module IDs differ from header IDs (`footer-navigation`, not `main-navigation`) because the footer menu is configured independently in Live Edit.
- Skins are per placement: `footer.dwig` variants of Logo and Menu can be simpler than the header ones.
- Use case variants: a minimal legal-only footer for checkout and landing shells, and a newsletter footer that embeds a contact-form module above the legal row.

## 4. cart-sidebar.dwig — mini cart drawer

The full cart page uses the `Store/Cart` module, but the header button needs a lightweight drawer on every page. Build it as a partial containing the cart module in a compact skin:

```twig
{# Templates/Layouts/Partials/cart-sidebar.dwig #}
<aside id="cart-sidebar" class="offcanvas offcanvas-end" aria-label="Shopping cart">
    <div class="offcanvas-header">
        <h2 class="h5 mb-0">Your cart</h2>
        <button type="button" class="btn-close" data-bs-dismiss="offcanvas" aria-label="Close"></button>
    </div>
    <div class="offcanvas-body">
        <module type="Store/Cart" id="global-mini-cart" template="mini.dwig" />
    </div>
</aside>
```

Notes:

- The bundled `Store/Cart/default.dwig` is a full-page table. The `mini.dwig` skin reuses the same data (`data.items`, `data.total`, `data.checkout_page_link`) with compact markup, and it must keep equivalent cart calls from section 6 or quantity editing and removal silently stop working.
- The drawer partial is included once in `main.dwig`, after the footer, and opened from any `data-bs-target="#cart-sidebar"` button.
- Put the `js-shopping-cart-quantity` class on the count badge. `common.js` updates every such element automatically after each cart mutation, so the theme never syncs it by hand.

## 5. More partials worth adding

- `announcement-bar.dwig` — a dismissible promo or notice strip above the header, fed by a site option so marketing can edit it without touching markup.
- `mobile-menu.dwig` — the off-canvas navigation for small screens, reusing the same `main-navigation` menu module ID so mobile and desktop share one menu tree.
- `search-overlay.dwig` — a full-width search panel embedding the `Store/Search` module, opened from the header search button.
- `wishlist-drawer.dwig` — same drawer pattern as the cart sidebar around the `Store/Wishlist` module. Its bundled skin is an empty stub, so this skin is theme work.
- `breadcrumbs.dwig` — a category and content trail included at the top of `{% block content %}` in page templates, not in the shell, since invoices and landing pages skip it.
- `cookie-notice.dwig` — consent banner wired to the cookie-consent settings. Include it in the shell so it appears on every page, and keep its accept and decline hooks intact.
- `back-to-top.dwig` — a small floating button with theme JS. Trivial markup, but keeping it a partial keeps the shell clean.


## 6. Frontend JavaScript -- always use dbEvent

All frontend JavaScript in templates goes through window.dbEvent, the request API in userfiles/modules/developmentbucket/db_lib/events/common.js. Never reference the mw JS library from .dwig templates or theme scripts.

The script loads automatically on normal frontend pages with the site and API URLs already set. Do not include it a second time. Standalone pages must define those two globals plus a csrf-token meta tag before loading it. The full method reference lives in userfiles/modules/developmentbucket/db_lib/events/README.md.

Cart calls used by cart skins and drawers:

```html
<script>
async function updateCartQty(id, qty) {
    try {
        await window.dbEvent.shop.cart.update({ id, qty });
    } catch (error) {
        console.error(error.code, error.message);
    }
}

async function removeCartItem(id) {
    try {
        await window.dbEvent.shop.cart.remove({ id });
    } catch (error) {
        console.error(error.code, error.message);
    }
}
</script>
```

```twig
<input type="number" min="1" value="{{ item.qty }}"
       onchange="updateCartQty('{{ item.id|e('js') }}', this.value);">
<button type="button" onclick="removeCartItem('{{ item.id|e('js') }}');">Remove</button>
```

Wishlist calls follow the same shape: window.dbEvent.shop.wishlist.get(), .add({ id }), and .remove({ id }). Every call uses try/catch because API errors reject instead of resolving. See the common.js README for the service, user, comment, form, module, and shop search and filter calls.

## 7. Partial versus module versus block

- Use a partial when the markup is static structure shared across pages, such as header, footer, or drawer shells.
- Use a `<module>` tag inside the partial when the fragment needs behavior or saved settings, such as menus, logo, cart, or search.
- Use a `{% block %}` in `main.dwig` when individual pages must replace or extend the region, such as `content`, `head_extra`, or `scripts`. A partial that every page overrides should have been a block.

## 8. Common mistakes

- Putting `<html>` or `<body>` tags in a partial, which nests a document inside the shell.
- Reusing one module ID for two different purposes, such as the header menu and the footer menu sharing `main-navigation`. Shared IDs share settings.
- Skinning the cart drawer and dropping the `dbEvent.shop.cart.update()` and `dbEvent.shop.cart.remove()` calls its buttons rely on.
- Hardcoding shop, cart, or checkout URLs instead of using resolved links and options.
- Including page-specific partials like breadcrumbs in the shell instead of in the page templates that need them.
