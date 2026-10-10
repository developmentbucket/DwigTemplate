# Content/Testimonials — Customer Quotes and Ratings

The `Content/Testimonials` module renders customer testimonials: quote cards with author names, roles, photos, and star ratings. Its skins live in `Templates/Modules/Content/Testimonials/`. Editors manage testimonials (grouped by project) in the module admin; the backend loads them in position order with an optional limit and character cap; the skin renders one card per testimonial.

## 1. Embedding testimonials

```twig
<module
    type="Content/Testimonials"
    id="home-testimonials"
    template="default.dwig"
/>
```

- `type` is `Content/Testimonials`: the path-style name that mirrors the skin path `Templates/Modules/Content/Testimonials/`. The bare legacy name still resolves, but every example in these docs uses the path-style form.
- `id` identifies this instance and connects it to its group, limit, and character settings. Homepage, product page, and layout-band placements use different IDs when their selections differ. Changing an ID orphans the settings saved under the old one.
- `template` selects the skin filename from the testimonials directory. Pass only the filename, never a path. A template chosen in Live Edit is saved against the instance and takes precedence over the inline attribute.
- An unavailable file falls back to `default.dwig`, so the directory keeps a working `default.dwig`.
- With no testimonials stored, the module shows an editor notice instead of cards. Keep an `{% else %}` branch so the live page never renders an empty band.

## 2. Where skins live and how they resolve

```text
Templates/Modules/Content/Testimonials/
+-- default.dwig     # fallback, keep it working
+-- compact.dwig
+-- ...
```

Resolution order for every skin:

1. The same relative path in the active theme.
2. The bundled default template.
3. `default.dwig` in the same directory.

Keep theme-specific testimonial skins in the active theme. Changing the bundled fallback changes the default for every generated theme.

## 3. What data a testimonials skin receives

| Value | Contents |
|---|---|
| `data.testimonials_items` | Testimonials in position order. Each carries the fields below. Always loop with `\|default([])` and keep an `{% else %}` branch. |
| `data.settings` | Instance settings: `group` (project filter), `testimonials_limit`, `character_limit`. |
| `data.params` | Tag attributes plus the instance ID (`data.params.id`). A `limit` attribute caps the list when no saved limit exists. |

Testimonial fields:

| Field | Contents |
|---|---|
| `name` | Author name. Falls back to `'Anonymous'`; escape with `\|e`. |
| `content` | Quote text, may contain markup. Strip tags before rendering in cards. |
| `rating` | Star rating out of 5. Defaults to `5` when unset. |
| `client_picture` | Author photo, or empty. Guard it; photo-less authors are normal. |
| `client_role`, `client_company` | Author role and company for the attribution line. Either can be empty. |
| `client_website` | Author link. Guard it; open with `target="_blank" rel="noopener"`. |
| `project_name` | Group the testimonial belongs to. Informational; grouping already applied. |

## 4. The quote card loop

Stars from `rating`, quote trimmed to the character limit, author row with guarded photo and attribution:

```twig
{% set testimonials = data.testimonials_items|default([]) %}
{% set character_limit = data.settings.character_limit|default(150)|int %}

<div class="review-track">
{% for testimonial in testimonials %}
    {% set rating = testimonial.rating|default(5)|int %}
    {% if rating < 0 %}{% set rating = 0 %}{% endif %}
    {% if rating > 5 %}{% set rating = 5 %}{% endif %}
    <article class="review-card card">
        <div class="stars" aria-label="Rated {{ rating }} out of 5">
            {% for i in 1..5 %}
                <i class="{{ i <= rating ? 'fas' : 'far' }} fa-star" aria-hidden="true"></i>
            {% endfor %}
        </div>
        <p>"{{ testimonial.content|default('')|striptags|slice(0, character_limit) }}{% if testimonial.content|default('')|length > character_limit %}...{% endif %}"</p>
        <footer>
            {% if testimonial.client_picture|default('') %}
                <img class="avatar" src="{{ testimonial.client_picture|e('html_attr') }}"
                     alt="{{ testimonial.name|default('Reviewer')|e('html_attr') }}"
                     width="48" height="48" loading="lazy">
            {% else %}
                <span class="avatar-placeholder rounded-circle" aria-hidden="true"></span>
            {% endif %}
            <b>{{ testimonial.name|default('Anonymous')|e }}</b>
            {% set credit = [testimonial.client_role|default(''), testimonial.client_company|default('')]|filter(v => v is not empty)|join(', ') %}
            {% if credit %}<small>{{ credit|e }}</small>{% endif %}
        </footer>
    </article>
{% else %}
    <p class="text-muted">No testimonials available at this time.</p>
{% endfor %}
</div>
```

Rules:

1. Clamp `rating` to 0–5 before looping stars. Out-of-range values print wrong star counts.
2. Strip tags from `content` before trimming (`|striptags|slice`). Trimming raw HTML splits tags and leaks markup into the card.
3. Guard `client_picture` with a placeholder fallback. Photo-less authors must hold card alignment like everyone else.
4. Build attribution from non-empty parts only. Stray separators ("· , Verified") read as broken data.
5. Escape names, roles, companies, and attributes. Testimonial copy is third-party input; `|raw` has no place in these skins.
6. Honor the instance `group` and `limit` by rendering whatever arrives in order. Filtering or re-sorting in the skin fights the editor's selection.

## 5. Use cases

**Homepage band.** Quote cards with ratings and author rows under a section heading. The standard social-proof band, often inside a `Layouts` composite section.

**Product reviews strip.** Group-scoped testimonials for one product line. Per-product groups keep praise attached to what it praises.

**Sidebar quote.** Single compact card (first item, trimmed quote, name only) in narrow columns. Same data, minimal chrome.

**Case-study teaser.** Quote plus company and website link driving to the full story. Attribution carries the click, not the quote.

**Empty-state invite.** The `{% else %}` branch inviting the first review instead of a blank band on new sites.

## 6. Common mistakes

- Trimming HTML content without `|striptags`, splitting tags and leaking markup.
- Unclamped ratings printing six stars or negative loops.
- Rendering photo-less authors without a placeholder, collapsing card rhythm.
- Attribution separators printed around empty role or company values.
- Unescaped testimonial copy. Third-party quotes are untrusted input throughout.
- Re-sorting or re-filtering the editor's group and limit in the skin.
- Sharing one instance ID across bands that need different groups, merging every band's selection.
- Using this module for people bios (`Content/TeamCard`) or product star summaries (`Store/ProductReviews`).
- Forgetting `default.dwig`, so a missing skin selection breaks testimonial bands site-wide.
