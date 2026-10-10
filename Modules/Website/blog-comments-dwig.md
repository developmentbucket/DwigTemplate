# Website/BlogComments — Article Comments and Reply Form

The `Website/BlogComments` module renders comments under content: the comment list for the current page plus the form to add one. Its skins live in `Templates/Modules/Website/BlogComments/`. The backend loads the current content's comments and passes them as rows; the skin renders the list, optional one-level replies, and the submit form through `window.dbEvent.comment.add()`.

## 1. Embedding comments

```twig
<module
    type="Website/BlogComments"
    id="post-comments"
    template="default.dwig"
/>
```

- `type` is `Website/BlogComments`: the path-style name that mirrors the skin path `Templates/Modules/Website/BlogComments/`. The bare legacy name still resolves, but every example in these docs uses the path-style form.
- `id` identifies this instance. One comments block per page is the norm; keep the ID stable so its skin selection persists.
- `template` selects the skin filename from the comments directory. Pass only the filename, never a path. A template chosen in Live Edit is saved against the instance and takes precedence over the inline attribute.
- An unavailable file falls back to `default.dwig`, so the directory keeps a working `default.dwig`.
- Comments render only when the `allow_comments` setting permits. When disabled, the module renders nothing at all — the skin needs no disabled branch, but keep the list guarded for empty states.

## 2. Where skins live and how they resolve

```text
Templates/Modules/Website/BlogComments/
+-- default.dwig     # fallback, keep it working
+-- compact.dwig
+-- ...
```

Resolution order for every skin:

1. The same relative path in the active theme.
2. The bundled default template.
3. `default.dwig` in the same directory.

Keep theme-specific comment skins in the active theme. The bundled skins ship as empty stubs, so writing the comments skins is part of building the theme — follow the list-plus-form pattern in sections 4–5 so the first skin you write already behaves like the module's native output.

## 3. What data a comments skin receives

| Value | Contents |
|---|---|
| `data.comments` | Comment rows for the current content. Each carries `id`, `rel_id`, `rel_type`, `reply_to_comment_id`, `comment_name`, `comment_body`, `comment_email`, `comment_website`, and timestamps. Always loop with `\|default([])`. |
| `data.params` | Tag attributes plus the instance ID (`data.params.id`). |

The list arrives flat. Threading is one level: rows whose `reply_to_comment_id` matches a parent `id` render nested under it. The skin groups replies; the backend does not nest them.

## 4. The comment list

Group replies under parents, escape everything, and keep an inviting empty state:

```twig
{% set comments = data.comments|default([]) %}
{% set parents = comments|filter(c => not c.reply_to_comment_id|default(0)) %}

<h2>{{ comments|length }} comment{{ comments|length == 1 ? '' : 's' }}</h2>

{% for comment in parents %}
<article class="comment" data-comment-id="{{ comment.id|e('html_attr') }}">
    <strong>{{ comment.comment_name|default('Visitor')|e }}</strong>
    {% if comment.created_at|default('') %}<small>{{ comment.created_at|e }}</small>{% endif %}
    <p>{{ comment.comment_body|default('')|e }}</p>
    <button type="button" data-comment-reply="{{ comment.id|e('html_attr') }}">Reply</button>
    {% set replies = comments|filter(r => (r.reply_to_comment_id|default(0)) == comment.id) %}
    {% if replies is not empty %}
    <div class="comment-replies">
        {% for reply in replies %}
        <article class="comment comment-reply" data-comment-id="{{ reply.id|e('html_attr') }}">
            <strong>{{ reply.comment_name|default('Visitor')|e }}</strong>
            <p>{{ reply.comment_body|default('')|e }}</p>
        </article>
        {% endfor %}
    </div>
    {% endif %}
</article>
{% else %}
    <p class="text-muted">No comments yet. Start the conversation below.</p>
{% endfor %}
```

Rules:

1. Split parents from replies with `reply_to_comment_id` filters. Nested-nesting (replies to replies) is not supported — render one level only.
2. Escape names, bodies, dates, and attributes. Comments are visitor input, often anonymous; `|raw` has no place in a comments skin.
3. Never render `comment_email` publicly. It identifies the author to moderators, not to readers.
4. Link `comment_website` only when present, with `target="_blank" rel="nofollow noopener"`. Author links are untrusted outbound links by definition.
5. Keep the `{% else %}` branch inviting. An empty thread is the normal state on new posts, not an error.

## 5. The comment form

Name, body, and optional website fields posting through `window.dbEvent` from `userfiles/modules/developmentbucket/db_lib/events/common.js`, which loads automatically on normal frontend pages. Never reference the `mw` JS library:

```twig
{% set content_id = data.params.content_id|default(data.params['content-id']|default(0)) %}

<form data-comment-form novalidate>
    <input type="hidden" name="rel_id" value="{{ content_id|e('html_attr') }}">
    <input type="hidden" name="rel_type" value="content">
    <input type="hidden" name="reply_to_comment_id" value="">
    <label>Your name
        <input type="text" name="comment_name" autocomplete="name" required>
    </label>
    <label>Comment
        <textarea name="comment_body" required></textarea>
    </label>
    <button class="btn btn-primary" type="submit">Post comment</button>
    <p class="form-status" role="status" aria-live="polite"></p>
</form>
```

```js
form.addEventListener('submit', async function (event) {
    event.preventDefault();
    const button = form.querySelector('[type="submit"]');
    button.disabled = true;
    try {
        await window.dbEvent.comment.add({
            rel_id: Number(form.rel_id.value),
            rel_type: 'content',
            comment_body: form.comment_body.value.trim(),
            comment_name: form.comment_name.value.trim()
        });
        form.reset();
        status.textContent = 'Thanks! Your comment is awaiting moderation.';
    } catch (error) {
        status.textContent = (error && error.message) || 'Unable to post your comment.';
    } finally {
        button.disabled = false;
    }
});
```

Rules:

1. Pass a plain object with `rel_id`, `rel_type`, and `comment_body`. Unlike the user methods, `comment.add()` does not convert forms or `FormData` — read values out of the form first.
2. Carry `rel_id` (the content ID) and `rel_type: 'content'` on every submit, plus `reply_to_comment_id` when answering (set by the reply buttons, cleared after). A missing target posts nowhere.
3. Expect moderation: successful posts may not appear immediately. Say so in the success message instead of promising instant visibility.
4. Name, email, CAPTCHA, login, and rate-limit requirements vary by installation. Read the current rules from the installation's behavior and reflect them in the form (required markers, login prompt) rather than hardcoding one policy.
5. Every call uses `try/catch` because failures reject. Report in the `role="status"` region and re-enable the button.

## 6. Use cases

**Post footer thread.** Full list with one-level replies plus the form under articles. The canonical placement, bound to the current content.

**Compact sidebar count.** Comment count linking down to the thread. Same data, one line, no list or form.

**Product Q&A thread.** Questions and answers on product pages using the same list-plus-form shape. Bound to the current product.

**Moderated community.** List with visible awaiting-moderation notices for the author's own pending posts. Same markup, status-aware copy.

## 7. Common mistakes

- Rendering `comment_email` publicly, or linking `comment_website` without `nofollow noopener`.
- Nesting replies more than one level deep, which the data model does not support.
- Passing the form element or `FormData` to `comment.add()`, which validates plain objects only.
- Omitting `rel_id` / `rel_type`, posting comments with no target.
- Promising instant visibility when moderation queues the post.
- Hardcoding one name/login/CAPTCHA policy instead of reflecting the installation's rules.
- Unescaped comment bodies. Anonymous visitor input is the least trusted content on the page.
- Using this module for product star ratings (`Store/ProductReviews`) — comments carry discussion, not scores.
- Forgetting `default.dwig`, so a missing skin selection breaks discussion site-wide.
