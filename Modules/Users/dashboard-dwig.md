# Users/Dashboard — Customer Account Area

The `Users/Dashboard` module renders the customer account area: overview stats, profile editing, order history with detail views, subscriptions, and wishlist. Its skins live in `Templates/Modules/Users/Dashboard/`. The backend passes endpoint URLs and session flags; panels load their data client-side as JSON and the skin renders shell, panels, and forms.

## 1. Embedding the dashboard

```twig
<module
    type="Users/Dashboard"
    id="customer-dashboard"
    template="default.dwig"
/>
```

- `type` is `Users/Dashboard`: the path-style name that mirrors the skin path `Templates/Modules/Users/Dashboard/`. The bare legacy name still resolves, but every example in these docs uses the path-style form.
- `id` identifies this instance. One dashboard per account page is the norm; keep the ID stable so its skin selection persists.
- `template` selects the skin filename from the dashboard directory. Pass only the filename, never a path. A template chosen in Live Edit is saved against the instance and takes precedence over the inline attribute.
- An unavailable file falls back to `default.dwig` (then the bundled default), so the directory keeps a working `default.dwig`.

## 2. Where skins live and how they resolve

```text
Templates/Modules/Users/Dashboard/
+-- default.dwig     # fallback: full panel shell
+-- orders.dwig      # orders-only list
+-- ...
```

Resolution order for every skin:

1. The same relative path in the active theme.
2. The bundled default template.
3. `default.dwig` in the same directory.

Keep theme-specific dashboard skins in the active theme. The full shell and focused lists (orders-only) are different skins over the same endpoints — never one skin with conditionals per surface.

## 3. What data a dashboard skin receives

| Value | Contents |
|---|---|
| `data.module_dom_id` | Unique, instance-derived DOM ID. Scope all CSS and script queries under it. |
| `data.is_logged` | Whether the visitor has a session. Guests see the sign-in prompt, never the panels. |
| `data.csrf_token` | CSRF token for mutating requests (profile save). |
| `data.login_url` / `data.logout_url` | Sign-in link for guests and logout link in the sidebar. Never hardcode either. |
| `data.profile_url` | JSON endpoint for the current profile. |
| `data.profile_update_url` | Endpoint for profile saves. |
| `data.orders_url` | JSON endpoint for the order list. |
| `data.order_url_template` | Order-detail URL with an `__ORDER_ID__` placeholder the skin replaces per row. |
| `data.subscriptions_url` / `data.wishlist_url` | JSON endpoints for subscriptions and wishlist panels. |
| `data.params` | Tag attributes plus the instance ID (`data.params.id`). |

Panel data (profile, orders, subscriptions, wishlist) arrives client-side as JSON, not as template variables. The skin owns the shell, the panel mounts, loading states, and error messaging.

## 4. The shell pattern

Login gate, sidebar navigation with panels, loading and alert regions, all scoped under the DOM ID:

```twig
<div id="{{ data.module_dom_id|e('html_attr') }}">
{% if not data.is_logged %}
    <div class="card mx-auto" style="max-width:520px">
        <div class="card-body p-5 text-center">
            <h2 class="h4 mb-3">Sign in to view your account</h2>
            <p class="text-muted mb-4">Your profile, orders, subscriptions and wishlist are available after login.</p>
            <a href="{{ data.login_url|e('html_attr') }}" class="btn btn-primary px-4">Sign in</a>
        </div>
    </div>
{% else %}
    <div class="row g-0">
        <aside class="col-lg-3">
            <nav aria-label="Account sections">
                <button type="button" data-panel="overview">Overview</button>
                <button type="button" data-panel="profile">Profile</button>
                <button type="button" data-panel="orders">Orders</button>
                <button type="button" data-panel="subscriptions">Subscriptions</button>
                <button type="button" data-panel="wishlist">Wishlist</button>
                <a href="{{ data.logout_url|e('html_attr') }}">Logout</a>
            </nav>
        </aside>
        <main class="col-lg-9">
            <div class="alert d-none" role="alert" data-dashboard-alert></div>
            <div data-dashboard-loading><div class="spinner-border" role="status"></div></div>
            <section data-panel-content="overview">...</section>
            <section data-panel-content="profile">...</section>
            <section data-panel-content="orders">
                <div data-order-list-view><div data-orders></div></div>
                <div data-order-detail-view class="d-none">
                    <button type="button" data-order-back>Back to orders</button>
                    <div data-order-detail></div>
                </div>
            </section>
            <section data-panel-content="subscriptions">...</section>
            <section data-panel-content="wishlist">...</section>
        </main>
    </div>
{% endif %}
</div>
```

Rules:

1. Gate everything on `data.is_logged`. Guests get the sign-in card; account markup must not render for anonymous visitors.
2. Scope every selector under `#{{ data.module_dom_id }}` — styles and scripts alike. Dashboard CSS lives in the skin; unscoped rules leak onto the whole account page.
3. Keep one panel visible at a time with matching nav states. Exactly one active panel and one active nav button; zero or two breaks orientation.
4. Keep loading and `role="alert"` regions in the shell. Every panel fetch shows loading, then content or a retry message — never a blank panel.
5. Build order-detail URLs by replacing `__ORDER_ID__` in `data.order_url_template` with the row's ID. Never construct API paths by hand.

## 5. Loading panels and saving the profile

Fetch each panel from its endpoint URL, post profile edits with the CSRF token, and guard the binder so re-renders never stack handlers:

```js
const root = document.getElementById('{{ data.module_dom_id }}');
if (!root || root.dataset.dashboardReady === 'true') { /* already bound */ }
root.dataset.dashboardReady = 'true';

async function loadJson(url) {
    const response = await fetch(url, { headers: { Accept: 'application/json' } });
    if (!response.ok) {
        throw new Error('Request failed: ' + response.status);
    }
    return response.json();
}

// Orders list, then detail per row via the URL template.
const orders = await loadJson('{{ data.orders_url|e('js') }}');
const detailUrl = '{{ data.order_url_template|e('js') }}'.replace('__ORDER_ID__', orderId);

// Profile save posts FormData including the CSRF token.
const formData = new FormData(profileForm);
formData.set('_token', '{{ data.csrf_token|e('js') }}');
await fetch('{{ data.profile_update_url|e('js') }}', { method: 'POST', body: formData });
```

Rules:

1. Read endpoint URLs only from `data.*` values injected via `|e('js')` or `|json_encode|raw`. Never hardcode API paths; routed URLs change across installs.
2. Send `_token` with every mutating request. Reads may stay plain GETs; writes never skip the token.
3. Every fetch uses `try/catch`. Write failures to the shell alert region and keep entered values; clearing a profile form on a failed save is the costliest bug on this page.
4. Render money only with `currency_format()` for server-rendered rows; format client-rendered amounts with the locale currency formatter and the order's own currency.
5. Escape every value injected into markup, whether server- or client-rendered. Profile names, addresses, and order notes are user input throughout.

## 6. The orders-only variant

Focused skins like `orders.dwig` render one panel server-side from a provided order list: invoice ID with `id` fallback, guarded date, status badge, `currency_format(amount, currency)` total, and an empty-state alert. Same guards as the shell (escaped values, honest empty state), none of the tab chrome. Use it for emails-adjacent pages, widgets, and print views where the full shell would be noise.

## 7. Use cases

**Full account area.** Overview stats, profile form, orders with detail views, subscriptions, wishlist. The default skin pattern.

**Orders-only page.** Focused list skin for a dedicated order-history route. Same endpoints, no tabs.

**Profile-only embed.** Profile form panel on a settings page. Same save flow, single visible panel.

**Mobile account.** Sidebar collapses to a horizontal scroll nav; panels unchanged. Responsive CSS only, same markup and hooks.

## 8. Common mistakes

- Rendering account panels for guests instead of the sign-in gate.
- Unscoped dashboard CSS or script queries leaking across the page.
- Hardcoded API paths instead of the passed endpoint URLs.
- Mutating requests without `_token`.
- Clearing forms on failed saves, or silent failures with no alert region.
- Zero or two active panels, or nav states that drift from visible content.
- Hand-formatted money instead of `currency_format()` and the order currency.
- Unescaped profile, address, and order-note values in client-rendered rows.
- Forgetting `default.dwig`, so a missing skin selection breaks the account area site-wide.
