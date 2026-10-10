# Store/ProductReviews — Ratings, Review Lists, and the Review Form

The `Store/ProductReviews` module renders a product's social proof: the average rating, the approved review list, and — for logged-in visitors only — the form to write a review. Its skins live in `Templates/Modules/Store/ProductReviews/`. The backend loads approved reviews newest-first, computes the count and average, and decides whether the current visitor may write; the skin renders the summary, the list, and the gated form.

## 1. Embedding reviews

```twig
<module
    type="Store/ProductReviews"
    id="product-reviews"
    product_id="{{ data.content.id }}"
    template="default.dwig"
/>
```

- `type` is `Store/ProductReviews`: the path-style name that mirrors the skin path `Templates/Modules/Store/ProductReviews/`. The bare legacy name still resolves, but every example in these docs uses the path-style form.
- `id` identifies this instance and connects it to its settings (such as the minimum-word rule, read via `module_id`). Keep it stable and unique per placement.
- `product_id` selects the product. On a product page it may be omitted: the module resolves the current content ID itself. Inside repeating cards, pass it explicitly and suffix the instance ID per product (`reviews-{{ product.id }}`).
- `template` selects the skin filename from the reviews directory. Pass only the filename, never a path. A template chosen in Live Edit is saved against the instance and takes precedence over the inline attribute.
- An unavailable file falls back to `default.dwig`, so the directory keeps a working `default.dwig`.

## 2. Where skins live and how they resolve

```text
Templates/Modules/Store/ProductReviews/
+-- default.dwig       # fallback, keep it working
+-- review-style-1.dwig
+-- ...
```

Resolution order for every skin:

1. The same relative path in the active theme.
2. The bundled default template.
3. `default.dwig` in the same directory.

Keep theme-specific review skins in the active theme. Changing the bundled fallback changes the default for every generated theme.

## 3. What data a reviews skin receives

| Value | Contents |
|---|---|
| `data.product_id` | Current product ID. Reuse it for hidden form fields and refresh calls. |
| `data.reviews` | Approved reviews, newest first. Each carries `id`, `product_id`, `user_id`, `user_name`, `rating`, `text`, `images`, `image_urls`, `created_at`, `status`. |
| `data.reviews_count` | Number of approved reviews. |
| `data.reviews_average_rating` | Approved-review average. |
| `data.is_logged` | Whether the visitor is logged in. |
| `data.can_review` | Whether this visitor may write now: logged in, and either an admin or without a review for this product today. |
| `data.customer_review` | The visitor's latest review for this product today (pending included), or empty. |
| `data.csrf_token` | Current CSRF token. `common.js` sends it automatically; the skin never handles it by hand. |
| `data.params` | Tag attributes plus the instance ID (`data.params.id`). |

Review image rule: `images` holds storage keys — never render them. Render `image_urls` in `<img>` tags.

Auto-escaping is off in `.dwig` templates: escape names and text with `|e`, attributes with `|e('html_attr')`, and never use `|raw` in a reviews skin.

## 4. Example one — rating summary and review list

Everyone sees this, logged in or guest. Average with star row, count, then one card per review:

```twig
{% set average = data.reviews_average_rating|default(0) %}
{% set count = data.reviews_count|default(0) %}

<div class="rating mb-3">
    <span aria-hidden="true">
    {% for i in 1..5 %}
        {% if average >= i %}
            <i class="fa-solid fa-star"></i>
        {% elseif average > (i - 1) %}
            <i class="fa-solid fa-star-half-stroke"></i>
        {% else %}
            <i class="fa-regular fa-star"></i>
        {% endif %}
    {% endfor %}
    </span>
    <span>{% if count > 0 %}{{ average|number_format(1) }} ({{ count }}){% else %}No ratings yet{% endif %}</span>
</div>

{% for review in data.reviews|default([]) %}
<article class="review-card mb-3" data-review-id="{{ review.id|e('html_attr') }}">
    <strong>{{ review.user_name|default('Customer')|e }}</strong>
    <span aria-label="Rated {{ review.rating|default(0) }} out of 5">{{ review.rating|default(0) }}/5</span>
    <p>{{ review.text|default('')|e }}</p>
    {% for imageUrl in review.image_urls|default([]) %}
        <img src="{{ imageUrl|e('html_attr') }}" alt="Customer review image" loading="lazy">
    {% endfor %}
</article>
{% else %}
    <p>No approved reviews yet. Be the first to write one.</p>
{% endfor %}
```

Rules for the list:

1. Loop `data.reviews|default([])` with an `{% else %}` branch. New products have no reviews; the branch is the normal state, not an error.
2. Render `review.text` escaped and `review.image_urls` — never storage-key `images` — in `<img>` tags.
3. Show the summary from `reviews_average_rating` and `reviews_count`, with a zero state. Never compute averages in the skin.
4. Keep a stable hook per card (`data-review-id`) for styling and moderation tooling.

## 5. Example two — the write form, logged-in visitors only

Writing is gated by `data.can_review`: logged in, and (for non-admin customers) no review submitted for this product today. Admins bypass the daily limit. The skin renders three states and nothing else:

```twig
{% if data.can_review %}
    <form data-db-product-review-form enctype="multipart/form-data">
        <input type="hidden" name="product_id" value="{{ data.product_id|e('html_attr') }}">
        <input type="hidden" name="module_id" value="{{ data.params.id|default('')|e('html_attr') }}">
        <label>Your rating
            <select name="rating" required>
                <option value="5">5 — Excellent</option>
                <option value="4">4 — Good</option>
                <option value="3">3 — Average</option>
                <option value="2">2 — Poor</option>
                <option value="1">1 — Terrible</option>
            </select>
        </label>
        <label>Your review
            <textarea name="text" required minlength="1"></textarea>
        </label>
        <label>Photos (up to 5)
            <input type="file" name="images[]" accept="image/jpeg,image/png,image/webp,image/gif" multiple>
        </label>
        <button type="submit">Post review</button>
    </form>
{% elseif not data.is_logged %}
    <p><a href="{{ site_url('login')|e('html_attr') }}">Sign in</a> to write a review.</p>
{% elseif data.customer_review %}
    <p>Thanks — your review from today is awaiting approval.</p>
{% endif %}
```

Submit through `window.dbEvent` from `userfiles/modules/developmentbucket/db_lib/events/common.js`, which loads automatically on normal frontend pages. Never reference the `mw` JS library. Add this script after the form:

```twig
<script>
document.querySelectorAll('[data-db-product-review-form]').forEach(function (form) {
    if (form.dataset.dbReviewBound === 'true') {
        return;
    }
    form.dataset.dbReviewBound = 'true';

    form.addEventListener('submit', async function (event) {
        event.preventDefault();
        const submitButton = form.querySelector('[type="submit"]');
        submitButton.disabled = true;
        try {
            const response = await window.dbEvent.product.productReviews.add(form);
            form.reset();
            form.hidden = true;
        } catch (response) {
            console.error(response.error?.message || 'Review submission failed.');
            submitButton.disabled = false;
        }
    });
});
</script>
```

Rules for the form:

1. Gate on `data.can_review`, never on `data.is_logged` alone. Logged-in is necessary but not sufficient: a customer who already reviewed today must see the waiting state, not a second form. A second same-day submit is rejected with `DAILY_REVIEW_LIMIT_REACHED`.
2. Never send `user_id` or `status`. The backend derives the user from the session and always creates a pending review.
3. Keep the field names exactly `product_id`, `module_id`, `rating`, `text`, `images[]`. The API reads these names; renaming breaks submission while the form looks fine.
4. The form needs no `action` or `method`. Passing the form element directly preserves the selected `images[]` files as multipart data. Limits: rating 1–5, text up to 5,000 characters, at most 5 images of 4 MB each (JPEG, PNG, WebP, GIF).
5. Every call uses `try/catch` because failures reject. Re-enable the button on error so the visitor can retry; hide and reset the form on success since the saved review is pending approval and will not appear in the list yet.
6. Scope the binder with the `data-db-product-review-form` hook and the bound guard, so two review forms on one page (card plus modal) do not double-submit.

## 6. Use cases

**Product page reviews.** Summary plus full list plus gated form below the buy box. One stable `product-reviews` ID; `product_id` omitted since the page resolves it.

**Quick-view modal reviews.** Compact summary and list inside the modal, form hidden to keep the modal light. Per-product instance IDs.

**Card rating badge.** Average stars plus count only (section 4's summary block) inside `Store/Products` and `Store/Product` cards. No list, no form.

**Post-purchase prompt.** Gated form on the order-success page for the purchased product with explicit `product_id`. Eligible buyers see it; guests see the sign-in prompt.

**Paginated browsing.** Long lists load more pages through `window.dbEvent.product.productReviews.list({ product_id, page, limit })` (page default 1, limit default 10, max 50). Render returned `response.data.reviews` with the same card markup as the server list.

## 7. Common mistakes

- Gating the form on `is_logged` instead of `can_review`, inviting double submits that fail with `DAILY_REVIEW_LIMIT_REACHED`.
- Sending `user_id` or `status` from the skin. The backend owns both; the form must not.
- Renaming `product_id`, `rating`, `text`, or `images[]` while styling the form.
- Rendering storage-key `images` in `<img>` tags instead of `image_urls`.
- Computing averages or counts in the skin instead of reading `reviews_average_rating` and `reviews_count`.
- Using `|raw` on review text or names. Reviews are visitor input; escape everything.
- Adding an `action`/`method` or a second `common.js` script tag, or calling the `mw` JS library.
- Forgetting the `{% else %}` list branch, breaking new products with zero reviews.
- Sharing one instance ID across products in a repeating card, merging every card's minimum-word and template settings.
- Forgetting `default.dwig`, so a missing skin selection breaks reviews site-wide.
