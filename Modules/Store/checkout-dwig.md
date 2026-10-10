# Store/Checkout — Address Form, Methods, and Order Placement

The `Store/Checkout` module renders the checkout page: delivery and billing address, shipping and payment method selection, order summary, and order placement. Its skins live in `Templates/Modules/Store/Checkout/`. The backend pre-resolves countries, states, saved visitor data, available methods, totals, and endpoint URLs; the skin renders the form, hydrates the method selectors, and posts the order without a page reload.

## 1. Embedding checkout

```twig
<module
    type="Store/Checkout"
    id="checkout-page"
    template="default.dwig"
/>
```

- `type` is `Store/Checkout`: the path-style name that mirrors the skin path `Templates/Modules/Store/Checkout/`. The bare legacy name still resolves, but every example in these docs uses the path-style form.
- `id` identifies this instance and connects it to its settings (shipping and payment visibility). One checkout per page is the norm; keep the ID stable so settings persist.
- `template` selects the skin filename from the checkout directory. Pass only the filename, never a path. A template chosen in Live Edit is saved against the instance and takes precedence over the inline attribute.
- An unavailable file falls back to `default.dwig`, so the directory keeps a working `default.dwig`.

Visibility attributes:

| Attribute | Purpose |
|---|---|
| `data-show-shipping` | Set to `"n"` to hide the delivery-method section (digital goods, pickup-only flows). Shown otherwise. |
| `data-show-payments` | Set to `"n"` to hide the payment-method section. Shown otherwise. |

## 2. Where skins live and how they resolve

```text
Templates/Modules/Store/Checkout/
+-- default.dwig     # fallback: full checkout page
+-- compact.dwig
+-- ...
```

Resolution order for every skin:

1. The same relative path in the active theme.
2. The bundled default template.
3. `default.dwig` in the same directory.

Keep theme-specific checkout skins in the active theme. Changing the bundled fallback changes the default for every generated theme.

## 3. What data a checkout skin receives

Cart and page state:

| Value | Contents |
|---|---|
| `data.items` | Cart lines (same shape as `Store/Cart` items). An empty cart means the checkout has nothing to sell — link back to the shop instead of rendering the form. |
| `data.cart_totals` | Totals breakdown for the summary column. |
| `data.step` | Checkout step number. |
| `data.user` | Logged-in customer record, or empty for guests. Never render its fields raw; prefill form inputs only. |
| `data.requires_registration` | Guests must register before ordering. Show the account prompt instead of (or before) the address form. |
| `data.requires_terms` | Terms acceptance is mandatory. Render the checkbox and block submission until checked. |
| `data.payment_success` / `data.payment_failure` | Return state from gateway redirects. Render confirmation or recovery messaging, never the form. |
| `data.show_shipping` / `data.show_payments` | Section visibility resolved from attributes. Gate the two method sections on these flags. |

Address and method data:

| Value | Contents |
|---|---|
| `data.countries` | Country code-to-name map for every country select. |
| `data.defaultCountry` / `data.paymentDefaultCountry` | Pre-selected delivery and billing countries. |
| `data.defaultStates` / `data.paymentDefaultStates` | State lists for the pre-selected countries. Further states load per country through `data.urls.states`. |
| `data.visitorCheckoutData` | Saved values to prefill inputs (`first_name`, `last_name`, `phone`, `email`, `address`, `city`, `state`, `zip`, `payment_*`, `shippingGateway`). Read every key with `\|default()`. |
| `data.shippingMethods` | Available delivery methods with rates, rendered client-side into the shipping mount. |
| `data.payment_modules` | Available payment methods, rendered client-side into the payment mount. |
| `data.checkoutCustomFields` | Extra per-order fields. When non-empty, embed the custom-fields module (section 4). |
| `data.csrfToken` | CSRF token for the order form's hidden `_token` field. |
| `data.urls` | Endpoint URLs: `processOrder`, `paymentMethodChange`, `data`, `options`, `states`. Post and fetch through these — never hardcode API paths. |
| `data.params` | Tag attributes plus the instance ID (`data.params.id`). |

Auto-escaping is off in `.dwig` templates: escape prefilled values with `|e('html_attr')`, pass server data into scripts only through `|json_encode|raw`, and never use `|raw` on visitor or merchant text.

## 4. Page anatomy

Scope everything under one root ID derived from the instance, and keep the error box, form token, and section mounts exactly hooked:

```twig
{% set checkoutId = 'checkout-' ~ data.params.id|default('page') %}

<div id="{{ checkoutId|e('html_attr') }}" class="container py-4">
    <div class="alert alert-danger" role="alert" aria-live="polite" data-checkout-error hidden></div>

    <form data-checkout-form method="post" enctype="multipart/form-data">
        <input type="hidden" name="_token" value="{{ data.csrfToken|e('html_attr') }}">

        <div class="checkout-card">
            <h3>Delivery address</h3>
            <select name="country" required>
                {% for countryCode, countryName in data.countries|default({}) %}
                    <option value="{{ countryCode|e('html_attr') }}"
                        {% if countryCode == data.defaultCountry|default('') %}selected{% endif %}>
                        {{ countryName|e }}
                    </option>
                {% endfor %}
            </select>
            <input type="text" name="first_name" value="{{ data.visitorCheckoutData.first_name|default('')|e('html_attr') }}" required>
            <input type="tel" name="phone" value="{{ data.visitorCheckoutData.phone|default('')|e('html_attr') }}" required>
            <input type="email" name="email" value="{{ data.visitorCheckoutData.email|default('')|e('html_attr') }}" required>
            {# ...address, city, state (data.defaultStates), zip... #}
            <label><input type="checkbox" name="billDifferentAddress" value="1" data-different-billing> Use a different billing address</label>
        </div>

        <div class="checkout-card d-none" data-billing-address>
            {# ...billing fields, prefilled from payment_* keys... #}
        </div>

        {% if data.checkoutCustomFields|default([]) is not empty %}
            <module type="custom_fields" for="shop_checkout" for-id="1"
                    id="checkout-custom-fields-{{ data.params.id|default('page') }}" />
        {% endif %}

        {% if data.show_shipping|default(true) %}
            <div class="checkout-card" data-shipping-section>
                <h3>Delivery method</h3>
                <div data-shipping-methods></div>
            </div>
        {% endif %}

        {% if data.show_payments|default(true) %}
            <div class="checkout-card" data-payment-section>
                <h3>Payment method</h3>
                <div data-payment-methods></div>
            </div>
        {% endif %}

        <button type="submit" class="btn btn-primary btn-lg w-100" data-submit-order>Place Order</button>
    </form>

    <aside aria-label="Order summary">
        <module type="Store/Cart" id="checkout-cart-{{ data.params.id|default('page') }}"
                template="cart-summary.dwig" data-checkout-link-enabled="n" />
    </aside>
</div>
```

Rules:

1. Derive the root ID from `data.params.id` and scope all CSS and script queries under it. Checkout CSS lives in the skin; unscoped rules leak onto the whole shop.
2. Keep the `data-checkout-error` alert box with `role="alert"`. Every failure path writes its message there; silent order failures strand paying customers.
3. Keep the hidden `_token` field from `data.csrfToken`. Submissions without it are rejected.
4. Prefill every input from `data.visitorCheckoutData` with `|default()` fallbacks and escaped attributes. Never invent field names: the order endpoint reads its own contract, and renamed inputs arrive empty.
5. Reveal the billing block only through the `data-different-billing` toggle. Billing fields submit only when shown.
6. Embed the order summary as a `Store/Cart` module in a summary skin with the checkout link disabled — never a second hand-built totals table that can drift from the cart.
7. Keep the `data-shipping-methods` and `data-payment-methods` mounts empty in markup. Method lists render client-side from `data.shippingMethods` and `data.payment_modules` so changing address, country, or coupon re-renders rates without a reload.

## 5. Client flow — hydrate, validate, place

The skin script hydrates selects from server JSON, refreshes methods on address change, and posts the order to `data.urls.processOrder`:

1. Inject server data into the script only via `|json_encode|raw` (`data.shippingMethods`, `data.payment_modules`, endpoint URLs, saved selections). Never interpolate values into JS strings by hand.
2. Rebuild states when the country changes through `data.urls.states`, then refresh shipping methods for the new destination. Stale rates for the wrong country mischarge delivery.
3. Re-render payment methods on method change through the payment-change URL where the shop needs it. Keep the selected method checked across re-renders.
4. On submit: prevent the default post, validate required fields and terms inline, lock the form with a loading state, post `FormData` to the process-order URL, and route gateway handoffs (redirects, modals) exactly as the response directs.
5. Every request uses `try/catch`. Write failures to the error box, unlock the form, and keep all entered values. Clearing the form on a failed order is the costliest UX bug on this page.
6. Emit checkout update events (`mw:checkout-cart-updated`, `mw:checkout-total-updated` and siblings) after cart, coupon, and total changes so the summary column and any listening module stay in sync.

## 6. Use cases

**Standard checkout.** Address card, billing toggle, shipping and payment mounts, summary column, place-order button. The default skin pattern.

**Guest checkout.** Same form with saved-data prefills empty; account creation handled by the registration prompt when `requires_registration` is true.

**Digital-goods checkout.** Shipping section hidden via `data-show-shipping="n"`. Address stays for billing and receipts; no delivery method is offered.

**Terms-gated checkout.** `requires_terms` renders the acceptance checkbox; submission stays blocked until checked, with the error box naming the missing acceptance.

**Post-payment return.** `payment_success` renders confirmation with order reference; `payment_failure` renders recovery (retry payment, contact support) — form hidden in both.

**Mobile summary-first.** Collapsible order summary above the form on small screens (`aria-expanded` toggle), full column on desktop. Same modules, responsive CSS only.

## 7. Common mistakes

- Hand-building the summary totals instead of embedding the cart summary module, letting checkout and cart disagree.
- Renaming order inputs, so the endpoint receives empty values while the form looks complete.
- Interpolating server data into JS by hand instead of `|json_encode|raw`, breaking on quotes and opening injection holes.
- Hardcoding API URLs instead of reading `data.urls`, breaking gated, localized, or relocated shops.
- Omitting the `_token` field or the error box, producing rejected orders with no explanation.
- Clearing entered values on a failed submission.
- Showing shipping or payment sections when their `show_*` flags are false.
- Unscoped checkout CSS leaking across the shop, or duplicate root IDs from a second checkout instance.
- Forgetting `default.dwig`, so a missing skin selection breaks the single page that takes money.
