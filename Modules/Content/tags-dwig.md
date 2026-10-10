# Content/Tags — Tag Pills and Filters

The `Content/Tags` module renders a record's tags as clickable pills. Its skins live in `Templates/Modules/Content/Tags/`. The backend collects the tags of one content record (or a chosen root page) and passes the plain tag names; the skin renders one pill per tag linking to that tag's filtered view.

## 1. Embedding tags

```twig
<module
    type="Content/Tags"
    id="post-tags"
    content_id="{{ data.content.id }}"
    template="default.dwig"
/>
```

- `type` is `Content/Tags`: the path-style name that mirrors the skin path `Templates/Modules/Content/Tags/`. The bare legacy name still resolves, but every example in these docs uses the path-style form.
- `id` identifies this instance and connects it to its source settings. Keep it stable and unique per placement.
- `content_id` (or `content-id`) selects whose tags render. Without it, the module falls back to the saved root page, then the main page.
- `template` selects the skin filename from the tags directory. Pass only the filename, never a path. A template chosen in Live Edit is saved against the instance and takes precedence over the inline attribute.
- An unavailable file falls back to `default.dwig`, so the directory keeps a working `default.dwig`.
- With no tags found, the module shows an editor notice instead of pills. Keep an `{% if %}` guard so the live page never renders an empty row.

## 2. Where skins live and how they resolve

```text
Templates/Modules/Content/Tags/
+-- default.dwig     # fallback, keep it working
+-- lite.dwig
+-- ...
```

Resolution order for every skin:

1. The same relative path in the active theme.
2. The bundled default template.
3. `default.dwig` in the same directory.

Keep theme-specific tag skins in the active theme. The bundled skins ship as empty stubs, so writing the tag skins is part of building the theme — follow the pill pattern in section 4 so the first skin you write already behaves like the module's native output.

## 3. What data a tags skin receives

| Value | Contents |
|---|---|
| `data.tags` | Tag names as plain strings, in order. Always loop with `\|default([])`. |
| `data.params` | Tag attributes plus the instance ID (`data.params.id`), including `content_id` and any `link-base` override (section 4). |

Tag filtering follows the `tags:<slug>` URL pattern on the source page: opening a pill shows that page filtered to the tag. Keep tag names URL-simple (letters, numbers, hyphens, single spaces) so pill links resolve exactly; exotic characters risk mismatching the stored slug.

## 4. The pill loop

One pill per tag: escaped name, link to the tag view, active style for the currently selected tag:

```twig
{% set tags = data.tags|default([]) %}
{% set link_base = data.params['link-base']|default(url_current()) %}
{% if link_base and not link_base ends with '/' %}{% set link_base = link_base ~ '/' %}{% endif %}
{% set current_url = url_current() %}

{% if tags is not empty %}
<div class="tag-pills d-flex flex-wrap gap-2">
{% for tag in tags %}
    {% set tag_href = link_base ~ 'tags:' ~ tag|url_encode %}
    {% set is_active = tag_href != link_base and tag_href in current_url %}
    <a href="{{ tag_href|e('html_attr') }}"
       class="btn btn-sm rounded-pill {{ is_active ? 'btn-primary' : 'btn-outline-primary' }}"
       {% if is_active %}aria-current="true"{% endif %}>{{ tag|e }}</a>
{% endfor %}
</div>
{% endif %}
```

Rules:

1. Loop `data.tags` with an `{% if %}` guard. Tagless records are normal; an empty pill row is visual noise.
2. URL-encode the tag into the `tags:` link (`|url_encode` before `|e('html_attr')`). Raw names with spaces or ampersands break the href.
3. Resolve the link base from an explicit `link-base` attribute first, falling back to the current page URL with a single trailing slash. Never hardcode the base; tag views live on the source page, wherever it is.
4. Mark the active pill (URL match) with the filled style plus `aria-current`. The match is a convenience highlight, so keep the comparison strict enough to avoid false positives from short tag names.
5. Escape every tag name. Tags are editor input; `|raw` has no place in a tags skin.

## 5. Use cases

**Post footer tags.** Pill row under articles linking each tag's filtered view. The canonical placement, bound to the current post.

**Product tags.** Same pills under product descriptions for material, collection, and feature tags. Bound to the current product.

**Tag cloud.** Larger list with size-weighted pills on blog index sidebars. Same data, frequency styling, capped count.

**Filter chips.** Pills that toggle the listing filter instead of navigating (paired with `Store/Filter` or list reloads). Same loop, button behavior instead of links.

## 6. Common mistakes

- Rendering the pill row with no tags instead of guarding with `{% if %}`.
- Linking raw tag names without `|url_encode`, breaking hrefs on spaces and special characters.
- Hardcoding the link base instead of resolving it per placement.
- Exotic tag characters that never match stored slugs. Keep names URL-simple at the content level.
- Unescaped tag names.
- Active-pill matching so loose that short tags highlight everywhere.
- Using tags for navigation menus (`Navigation/Menu`) or category grids (`Store/StoreCategories`) — pills filter one record set by label.
- Forgetting `default.dwig`, so a missing skin selection breaks tag rows site-wide.
