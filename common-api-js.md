# `common.js` Usage Guide — All Functions With Examples

Companion to [README.md](./README.md) (the full reference). This guide is for template designers and developers: what each `window.dbEvent` function does and exactly how to call it from a dwig template or theme script.

Terminology: `common.js` is a Promise-based request API (it does not dispatch DOM events). Every call either resolves with a success envelope or rejects with an error envelope — always use `try`/`catch`.

```js
// Success envelope (resolved value)
{ success: true, message: "Optional message", data: {...}, error: null }

// Error envelope (rejected value — catch it, it never resolves as data)
{ success: false, message: "...", data: null,
  error: { code: "VALIDATION_ERROR", message: "...", fields: {...}, details: null } }
```

## 1. Loading — automatic, do not add it yourself

`common.js` is injected automatically by the DB platform itself — you never add it in a dwig template or theme.

The platform (`FrontendController`) inserts this into `</head>` on every normally rendered page:

```html
<script>window.DB_API_URL="https://example.com/api/";
        window.DB_SITE_URL="https://example.com/";
</script>
<script src="https://example.com/userfiles/modules/developmentbucket/db_lib/events/common.js"></script>
```

This happens in **both** places:

- **Website frontend** — every normal storefront/content page.
- **Live edit** — the same injection runs inside the live-edit frame, so `window.dbEvent` is available while editing too (the platform additionally injects `live_edits.js` before `</body>` in edit mode).

Rules for template/theme code:

- Do NOT add a `<script src=".../common.js">` tag yourself — including it twice causes double loading.
- Do NOT redeclare `window.DB_API_URL` / `window.DB_SITE_URL` — the platform sets them.
- Do NOT copy `common.js` into your theme. The platform updates that file automatically, so a vendored copy goes stale. Always use the platform-provided copy.
- Just call `window.dbEvent...` directly. If your script can run very early, guard it: `if (!window.dbEvent) return;`.

Manual loading is only for a **standalone page rendered outside the platform** (external landing page, isolated prototype). Define the globals before loading, and include a CSRF token for POST requests:

```html
<script>
    window.DB_API_URL = "https://example.com/api/";
    window.DB_SITE_URL = "https://example.com/";
</script>
<script src="https://example.com/userfiles/modules/developmentbucket/db_lib/events/common.js"></script>
<meta name="csrf-token" content="...">
```

Needs a modern browser (`fetch`, `async`/`await`, `FormData`, `URLSearchParams`, optional chaining). Both base URLs are captured at load time; changing the globals later has no effect.

Standard error handler pattern used through all examples below:

```js
function showErrors(err, form) {
    // err.error.fields = { fieldName: ["message", ...] }
    if (err?.error?.fields && form) {
        for (const [name, messages] of Object.entries(err.error.fields)) {
            form.querySelector(`[data-error-for="${name}"]`)!.textContent = messages.join(', ');
        }
    } else {
        alert(err?.message || 'Request failed');
    }
}
```

## 2. Cart — `dbEvent.shop.cart`

After `add`, `update`, and `remove`, every element with class `.js-shopping-cart-quantity` updates automatically with the new item count. Put that class on the header cart badge and it stays in sync with zero extra code.

```js
// Read the cart
const cart = await dbEvent.shop.cart.get();
console.log(cart.data); // items, totals, count

// Totals only (mini-cart, header badge)
const totals = await dbEvent.shop.cart.totals();

// Add — content_id required (for_id / rel_id accepted as alternatives)
await dbEvent.shop.cart.add({ content_id: 42, qty: 1 });

// Add with selected options/custom fields (variant product)
await dbEvent.shop.cart.add({
    content_id: 42,
    qty: 2,
    custom_fields: { size: 'XL', color: 'Red' }
});
// Or submit the option form directly:
await dbEvent.shop.cart.add(new FormData(document.querySelector('#product-options-form')));

// Change quantity — id (cart item id) and qty required
await dbEvent.shop.cart.update({ id: 'abc123', qty: 3 });

// Remove one line — id required
await dbEvent.shop.cart.remove({ id: 'abc123' });
```

Full cart-line AJAX refresh (re-render the module region after a change):

```js
async function refreshCart() {
    await dbEvent.module.load({
        type: 'Store/Cart',
        template: 'default.dwig',
        target: '#cart-region'   // innerHTML replaced with fresh cart HTML
    });
}

document.querySelector('#cart-region').addEventListener('click', async (e) => {
    const btn = e.target.closest('[data-cart-remove]');
    if (!btn) return;
    try {
        await dbEvent.shop.cart.remove({ id: btn.dataset.cartRemove });
        await refreshCart();
    } catch (err) { showErrors(err); }
});
```

Re-order a past order (adds its items to the current cart, badge updates itself):

```js
await dbEvent.shop.reorder(orderId);
```

## 3. Coupons — `dbEvent.shop.cart.coupon`

```js
// Apply — coupon_code required
await dbEvent.shop.cart.coupon.add({ coupon_code: 'SAVE10' });
await refreshCart();

// Remove the active coupon
await dbEvent.shop.cart.coupon.remove();

// Read the coupon stored in session
const session = await dbEvent.shop.cart.coupon.sessionCoupon();
```

## 4. Wishlist — `dbEvent.shop.wishlist`

```js
// All saved items for the current customer (login required)
const list = await dbEvent.shop.wishlist.get();

// Toggle a heart button — id (product id) required for add/remove
async function toggleWishlist(button, productId) {
    try {
        if (button.classList.contains('is-active')) {
            await dbEvent.shop.wishlist.remove({ id: productId });
            button.classList.remove('is-active');
        } else {
            await dbEvent.shop.wishlist.add({ id: productId });
            button.classList.add('is-active');
        }
    } catch (err) { showErrors(err); }
}
document.querySelectorAll('[data-wishlist-toggle]').forEach((btn) => {
    btn.addEventListener('click', () => toggleWishlist(btn, btn.dataset.wishlistToggle));
});

// Render the wishlist region via the module (or loop list.data in script)
await dbEvent.module.load({
    type: 'Store/WishList',
    template: 'default.dwig',
    target: '#wishlist-region'
});

## 5. Live search — `dbEvent.shop.search`

Accepts a keyword string or a config object. Identical in-flight requests share one network call, so typing fast is safe — but debounce anyway. Returns the standard envelope plus a top-level `html` string.

```js
// Simplest form
const res = await dbEvent.shop.search('sneakers');
document.querySelector('#search-results').innerHTML = res.html;

// With options — limit clamped to 1..50, default 10
const res2 = await dbEvent.shop.search({
    keyword: 'sneakers',
    limit: 8,
    template: 'default.dwig'   // skin used for server-rendered html
});

// Debounced search field
let timer;
document.querySelector('#search-input').addEventListener('input', (e) => {
    clearTimeout(timer);
    timer = setTimeout(async () => {
        const keyword = e.target.value.trim();
        if (!keyword) return;
        try {
            const res = await dbEvent.shop.search({ keyword, limit: 8 });
            document.querySelector('#search-results').innerHTML = res.html;
        } catch (err) { showErrors(err); }
    }, 300);
});
```

## 6. Product filter — `dbEvent.shop.filter`

POSTs the filter set and returns server-rendered `html` plus structured data. Pass `target` to inject the HTML automatically, and `callback` to also run custom code. Defaults: `page: 1`, `limit: 20`, `sort: 'date_desc'`, `stock_status: 'all'`, `template: 'default.dwig'`.

```js
await dbEvent.shop.filter({
    target: '#shop-results',          // CSS selector — innerHTML replaced
    categories: [4, 9],               // single value also accepted
    price: { min: 25, max: 150 },
    custom_data: { size: ['XL'] },    // scalars auto-wrapped to arrays
    attributes: { color: ['Red'] },
    sort: 'price_asc',
    page: 1,
    limit: 12,
    template: 'default.dwig',
    callback: (result) => {
        console.log(result.data.total); // structured data alongside html
        window.scrollTo({ top: 0, behavior: 'smooth' });
    }
});

// Pagination button — keep current filters, change only the page
document.querySelector('#shop-results').addEventListener('click', async (e) => {
    const btn = e.target.closest('[data-page-number]');
    if (!btn) return;
    await dbEvent.shop.filter({ target: '#shop-results', page: btn.dataset.pageNumber });
});
```

## 7. Product price and reviews — `dbEvent.shop.product` (alias `dbEvent.product`)

```js
// Live price for selected options — product_id required
const price = await dbEvent.shop.product.get_price({
    product_id: 42,
    custom_fields: { size: 'XL' }   // must be an object when present
});
document.querySelector('#live-price').textContent = price.data.price;

// Update price when an option changes
document.querySelector('#product-options-form').addEventListener('change', async (e) => {
    const form = e.currentTarget;
    const payload = Object.fromEntries(new FormData(form).entries());
    try {
        const res = await dbEvent.product.get_price({ product_id: 42, ...payload });
        document.querySelector('#live-price').textContent = res.data.price;
    } catch (err) { showErrors(err); }
});

// Add a review — product_id, rating 1..5, and text required; images optional
await dbEvent.product.productReviews.add({
    product_id: 42,
    rating: 5,
    text: 'Fits perfectly.',
    images: fileInput.files   // FileList or array of File accepted
});
// A <form> or FormData with the same field names works as well:
await dbEvent.product.productReviews.add(new FormData(reviewForm));

// List reviews for a product — product_id required
const reviews = await dbEvent.product.productReviews.list({ product_id: 42 });
```

## 8. Auth — `dbEvent.user.login`, `.register`, `.forgotPassword`

One method covers all three flows; the payload selects the flow. `otp` is accepted as an alias of `code` in both OTP flows.

```js
// Email + password
await dbEvent.user.login({ email: 'person@example.com', password: 'secret' });
// username works instead of email:
await dbEvent.user.login({ username: 'person', password: 'secret', login_type: 'email' });

// Email OTP — step 1 requests the code, step 2 (with code) verifies and logs in
await dbEvent.user.login({ login_type: 'email_otp', email: 'person@example.com' });
await dbEvent.user.login({ login_type: 'email_otp', email: 'person@example.com', code: '123456' });

// Phone OTP — same two steps with phone
await dbEvent.user.login({ login_type: 'phone_otp', phone: '+919999999999' });
await dbEvent.user.login({ login_type: 'phone_otp', phone: '+919999999999', code: '123456' });

// Registration — type auto-detected (password present → password; phone →
// phone_otp; email → email_otp) or set explicitly with registration_type
await dbEvent.user.register({ email: 'new@example.com', password: 'secret', first_name: 'New' });
await dbEvent.user.register({ registration_type: 'email_otp', email: 'new@example.com' });
await dbEvent.user.register({ registration_type: 'email_otp', email: 'new@example.com', code: '123456' });
await dbEvent.user.register({ registration_type: 'phone_otp', phone: '+919999999999' });

// Forgotten password — email or username required
await dbEvent.user.forgotPassword({ email: 'person@example.com' });
```

Login/register form example (dwig template script):

```html
<form id="login-form">
    <input type="email" name="email" required>
    <input type="password" name="password" required>
    <p data-error-for="email"></p>
    <button type="submit">Login</button>
</form>
<script>
document.querySelector('#login-form').addEventListener('submit', async (e) => {
    e.preventDefault();
    const form = e.currentTarget;
    try {
        await dbEvent.user.login(Object.fromEntries(new FormData(form).entries()));
        window.location.href = '{{ site_url('dashboard') }}';
    } catch (err) { showErrors(err, form); }
});
</script>
```

## 9. Customer addresses and orders — `dbEvent.user`

```js
// Addresses (login required)
const addresses = await dbEvent.user.getAddresses();          // via profile payload
await dbEvent.user.addAddress({
    name: 'Home',                 // or title
    type: 'shipping',             // 'shipping' (default) or 'billing'
    address_street_1: '221B Baker Street',
    city: 'Mumbai', country: 'IN', zip: '400001', phone: '+919999999999'
});
await dbEvent.user.updateAddress(7, { name: 'Home', address_street_1: '...' });
// Or pass the id inside the object:
await dbEvent.user.updateAddress({ id: 7, name: 'Home', address_street_1: '...' });
// Namespaced alias: dbEvent.user.address.get() / .add() / .update()

// Past orders (login required)
const orders = await dbEvent.user.order.list();   // alias: .getAll()
```

## 10. Affiliate — `dbEvent.user.affiliate`

```js
const dash = await dbEvent.user.affiliate.dashboard();  // alias: .get()
const comms = await dbEvent.user.affiliate.commissions({ page: 1, per_page: 20 });
const payouts = await dbEvent.user.affiliate.payouts({ status: 'pending' });
const ledger = await dbEvent.user.affiliate.ledger({ page: 1 });
const clicks = await dbEvent.user.affiliate.clicks({ page: 1 });
const coupons = await dbEvent.user.affiliate.coupons();

// Request a payout — amount > 0 and payment_method required
await dbEvent.user.affiliate.requestPayout({ amount: 150, payment_method: 'bank_transfer' });

// Track a referral visit — string code or object
await dbEvent.user.affiliate.track('PARTNER42');
await dbEvent.user.affiliate.track({ code: 'PARTNER42' });
```

## 11. Comments, forms, modules

```js
// Blog/product comment — rel_id, rel_type, comment_body required
await dbEvent.comment.add({
    rel_id: 15,               // content id being commented on
    rel_type: 'content',
    comment_body: 'Great article!',
    comment_name: 'Reader',   // optional display name
    comment_email: 'reader@example.com'
});

// Contact/newsletter form — must pass the <form> element or its FormData
await dbEvent.form.submit(document.querySelector('#contact-form'));

// Render any module into a page region (used by cart/search/filter examples)
const result = await dbEvent.module.load({
    type: 'Store/Cart',        // path-style module type (also accepts module / data-type)
    id: 'cart-1',
    template: 'default.dwig',
    target: '#cart-region',    // selector string or Element; omitted = return html only
    replace: true,             // true (default) replaces node; false sets innerHTML
    output: 'html',            // 'json' returns parsed JSON instead of injecting HTML
    callback: (res) => console.log(res.html, res.element)
});
```

## 12. Appointments and services

```js
// One appointment-enabled service with pricing, duration, staff
const service = await dbEvent.service.get(42);          // alias: .getDetails(42)

// Dates with availability — days = calendar days inspected (1..90, default 10)
const dates = await dbEvent.service.getAvailableDates(42, 15);
const dates2 = await dbEvent.service.getAvailableDates(42, {
    days: 15, start_date: '2026-09-10', timezone: 'Asia/Kolkata'
});

// Times for one date (YYYY-MM-DD), aggregated across active staff
const times = await dbEvent.service.getAvailableTimes(42, '2026-09-12', {
    timezone: 'Asia/Kolkata'
});

// Logged-in customer's appointments — status filter optional
const appts = await dbEvent.user.appointment.list({ status: 'confirmed', page: 1, per_page: 10 });
// Aliases: dbEvent.user.appointment.getAll(...), dbEvent.appointments.list(...)
```

## 13. Pickup locations — `dbEvent.shop.locations`

```js
const all = await dbEvent.shop.locations.get();
await dbEvent.shop.locations.select(3);                 // or { location_id: 3 } / { id: 3 }
const current = await dbEvent.shop.locations.getSelected();
await dbEvent.shop.locations.clearSelection();
// Every location carries zip_code (null when unset) for uniform templates.
```

## 14. Security notes

- POST calls send the `csrf-token` meta value automatically — keep that meta tag on every page using `common.js`.
- Never render `error.details.body` to users; an `INVALID_RESPONSE` body can contain a full raw server response.
- Client validation only improves UX — real authorization and pricing stay on the backend; never trust prices or totals computed in the browser.
```
