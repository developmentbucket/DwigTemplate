# Store/WishList — Saved Items Drawer and List

The `Store/WishList` module renders the visitor's saved items: a wishlist drawer, a full saved-items page, and the heart toggles inside product cards. Its skins live in `Templates/Modules/Store/WishList/`. Wishlist state belongs to the visitor (account or guest cookie), never to the instance; the skin renders the items and wires add, remove, and toggle through `window.dbEvent.shop.wishlist`.

## 1. Embedding a wishlist

```twig
<module
    type="Store/WishList"
    id="wishlist-page"
    template="default.dwig"
/>
```

- `type` is `Store/WishList`: the path-style name that mirrors the skin path `Templates/Modules/Store/WishList/`. The bare legacy name still resolves, but every example in these docs uses the path-style form.
- `id` identifies this instance for its skin selection and settings. Wishlist items themselves are per-visitor, not per-instance: two placements with different IDs show the same visitor's items. Keep the ID stable anyway so skin choice and settings persist.
- `template` selects the skin filename from the wishlist directory. Pass only the filename, never a path. A template chosen in Live Edit is saved against the instance and takes precedence over the inline attribute.
- An unavailable file falls back to `default.dwig`, so the directory keeps a working `default.dwig`.

## 2. Where skins live and how they resolve

```text
Templates/Modules/Store/WishList/
+-- default.dwig     # fallback, keep it working
+-- drawer.dwig
+-- ...
```

Resolution order for every skin:

1. The same relative path in the active theme.
2. The bundled default template.
3. `default.dwig` in the same directory.

Keep theme-specific wishlist skins in the active theme. The bundled skins ship as empty stubs, so writing the wishlist skins is part of building the theme — follow the patterns in sections 4–6 so the first skins you write already behave like the module's native output.

## 3. How wishlist state works

One visitor, one list, everywhere on the site:

- Logged-in customers keep the list on their account. Guests keep it in the wishlist cookie. Both paths accept the same calls, so skins never branch on login state for rendering items.
- Items carry `product_id`, `image` (main product image URL, `null` when the product has no image or no longer exists), `url` (public product URL, `null` when the product no longer exists), and `created_at`.
- State changes go through `window.dbEvent` from `userfiles/modules/developmentbucket/db_lib/events/common.js`, which loads automatically on normal frontend pages. Never reference the `mw` JS library:

```js
// Read
const response = await window.dbEvent.shop.wishlist.get();
const items = response.data.wishlist;

// Save (guests supported via cookie)
await window.dbEvent.shop.wishlist.add({ id: 42 });

// Unsave
await window.dbEvent.shop.wishlist.remove({ id: 42 });
```

Every call uses `try/catch` because failures reject. After any mutation, refresh the rendered list (re-render from `get()` or reload the module) so drawer, page, and card hearts agree.

## 4. The heart toggle inside product cards

Cards carry a toggle button addressed by product ID. Keep this exact hook shape so theme scripts find every heart:

```twig
<button class="wish-btn btn btn-light rounded-circle" type="button"
        aria-label="Add {{ product.title|default('product')|e('html_attr') }} to wishlist"
        aria-pressed="false"
        data-wish="{{ product.id|e('html_attr') }}">
    <i class="fa-regular fa-heart" aria-hidden="true"></i>
</button>
```

Rules:

1. Keep `data-wish="{{ product.id }}"` on every toggle. Scripts collect hearts by this attribute; renaming orphans the button.
2. Flip `aria-pressed` with the saved state (`true` when the product is already saved). A heart that never reflects state lies on every render.
3. Update the `aria-label` with the state change ("Add … to wishlist" versus "Remove … from wishlist"). Icon-only buttons have no other label.
4. Toggle with `add` when absent and `remove` when present, then set the pressed state from the result — never assume the call succeeded before updating the button.

Wire every heart with one delegated handler. It toggles the saved state, flips the button, and refreshes the rendered list so drawer, page, and cards agree:

```html
<script>
function setHeartState(button, saved, title) {
    button.setAttribute('aria-pressed', saved ? 'true' : 'false');
    button.setAttribute('aria-label', (saved ? 'Remove ' : 'Add ') + title + ' ' + (saved ? 'from' : 'to') + ' wishlist');
}

async function refreshWishlistItems() {
    try {
        const response = await window.dbEvent.shop.wishlist.get();
        const items = (response && response.data && response.data.wishlist) || [];
        document.querySelectorAll('[data-wish]').forEach(function (button) {
            const id = Number(button.getAttribute('data-wish'));
            const saved = items.some(function (item) { return Number(item.product_id) === id; });
            setHeartState(button, saved, button.getAttribute('data-wish-title') || 'product');
        });
        return items;
    } catch (error) {
        console.error(error.code, error.message);
        return [];
    }
}

document.addEventListener('click', async function (event) {
    const button = event.target.closest('[data-wish]');
    if (!button) {
        return;
    }
    const id = Number(button.getAttribute('data-wish'));
    const pressed = button.getAttribute('aria-pressed') === 'true';
    button.disabled = true;
    try {
        if (pressed) {
            await window.dbEvent.shop.wishlist.remove({ id: id });
        } else {
            await window.dbEvent.shop.wishlist.add({ id: id });
        }
        await refreshWishlistItems();
    } catch (error) {
        console.error(error.code, error.message);
    } finally {
        button.disabled = false;
    }
});

refreshWishlistItems();
</script>
```

5. Give each heart a `data-wish-title` (or read the card title) so the flipped `aria-label` names the product. The handler above falls back to `'product'` when it is missing.
6. Disable the button while its own request is in flight and re-enable in `finally`. Double clicks otherwise fire duplicate add calls.
7. Call `refreshWishlistItems()` on page load so hearts rendered before this script ran still reflect the saved state.

## 5. The drawer pattern

The header heart opens a drawer that exists once per page, usually from a layout partial. The shell is static markup with an empty state; items render into it:

```twig
<aside class="drawer" id="wishDrawer" aria-labelledby="wishTitle" aria-hidden="true">
    <div class="drawer-head d-flex align-items-center justify-content-between p-3 border-bottom">
        <h2 class="h4 mb-0" id="wishTitle">Your wishlist</h2>
        <button class="close-btn btn-close" type="button" aria-label="Close wishlist"></button>
    </div>
    <div class="drawer-body p-3 overflow-auto" id="wishItems">
        <div class="empty-state text-center py-5">
            <h3>Save what you love</h3>
            <p>Your favourites will appear here.</p>
        </div>
    </div>
</aside>
```

1. Keep the drawer shell static with its empty state. An empty wishlist is the normal first-run state, not an error.
2. Render each fetched item with the row pattern from section 6, compacted for the narrow drawer.
3. Toggle `aria-hidden` with the drawer's open state. A hidden drawer must stay out of the tab order and screen-reader flow.

## 6. The saved-items row

One row per item: image, title, price, view link, remove control. Guard every nullable value — items outlive products:

```twig
{% for item in items|default([]) %}
<div class="wishlist-product" data-wishlist-row="{{ item.product_id|e('html_attr') }}">
    <div class="wishlist-product-image">
        {% if item.image|default('') %}
            <a href="{{ item.url|default('#')|e('html_attr') }}">
                <img src="{{ thumbnail(item.image, 250, 250)|e('html_attr') }}" alt="" loading="lazy">
            </a>
        {% endif %}
    </div>
    <div class="wishlist-product-title-wrapper">
        <h3>{{ item.title|default('Unavailable product')|e }}</h3>
        {% if item.price|default(0) %}
            <h4>{{ currency_format(item.price) }}</h4>
        {% endif %}
    </div>
    <a href="{{ item.url|default('#')|e('html_attr') }}">View Product</a>
    <button type="button" data-unwish="{{ item.product_id|e('html_attr') }}" aria-label="Remove from wishlist">Remove</button>
</div>
{% else %}
    <p>Your wishlist is empty.</p>
{% endfor %}
```

Rules:

1. Guard `image` and `url`: both are `null` when the product no longer exists. Never render a link or image from a null value; label the row honestly instead.
2. Pass images through `thumbnail()` at the displayed size with `loading="lazy"`.
3. Format money only with `currency_format()`. Never concatenate symbols by hand.
4. Escape titles and attributes. `|raw` has no place in a wishlist skin.
5. Removal uses `data-unwish` (or the shared toggle) with the product ID, followed by a list refresh so counts and hearts stay consistent.

## 7. Use cases

**Header wishlist drawer.** Heart button with a count badge opening the drawer partial. Present on every page via the layout shell.

**Saved-items page.** Full list rows with prices, view links, and remove controls. One stable instance ID.

**Card hearts.** `data-wish` toggles on every product card (`Store/Products`, `Store/Product`, search rows) sharing one behavior script.

**Post-login merge prompt.** Guests who saved items see them intact after signing in. Skins render the same list; no login branching in markup.

**Empty-state upsell.** Drawer and page empty states link onward to the catalogue instead of dead-ending.

## 8. Common mistakes

- Renaming `data-wish` (or the remove hook), orphaning hearts while cards look fine.
- Never flipping `aria-pressed` or the `aria-label`, so saved items look unsaved.
- Assuming mutation calls succeed before updating buttons and counts.
- Rendering `image` or `url` unguarded when the product no longer exists.
- Branching list markup on login state. Guests have wishlists too, via the cookie.
- Scoping items to the instance ID. Items belong to the visitor; IDs scope skins and settings only.
- Full-size product images in drawer rows instead of `thumbnail()` sizes.
- Hand-formatted prices instead of `currency_format()`.
- Forgetting `default.dwig`, so a missing skin selection breaks saved-items surfaces site-wide.
