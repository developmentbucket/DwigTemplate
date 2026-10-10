# Social/SocialSharer — Share-This-Page Buttons

The `Social/SocialSharer` module renders share buttons for the current page: Facebook, X, Pinterest, Viber, WhatsApp, LinkedIn. Its skins live in `Templates/Modules/Social/SocialSharer/`. The backend decides which networks are enabled for the instance and passes their icons; the skin builds the share links and decides the button markup.

Do not confuse it with `Social/SocialLinks`. The sharer shares the current page outward. Social links point visitors at the site's own profiles. Product and post pages want the sharer; headers and footers want social links.

## 1. Embedding a sharer

```twig
<module
    type="Social/SocialSharer"
    id="product-share"
    template="default.dwig"
/>
```

- `type` is `Social/SocialSharer`: the path-style name that mirrors the skin path `Templates/Modules/Social/SocialSharer/`. The bare legacy name still resolves, but every example in these docs uses the path-style form.
- `id` identifies this instance and connects it to its enabled-network settings. Keep it stable and unique per placement. Changing it orphans the network selection saved under the old ID.
- `template` selects the skin filename from the sharer directory. Pass only the filename, never a path. A template chosen in Live Edit is saved against the instance and takes precedence over the inline attribute.
- An unavailable file falls back to `default.dwig`, so the directory keeps a working `default.dwig`.

Network selection via the `enable` attribute:

```twig
{# All networks on #}
<module type="Social/SocialSharer" id="post-share" enable="all" template="default.dwig" />

{# Named subset #}
<module type="Social/SocialSharer" id="product-share" enable="facebook,linkedin,whatsapp" template="default.dwig" />
```

`enable="all"` turns every network on. Otherwise the value is matched by network name against each key, so a comma list enables exactly those networks. Saved per-network settings apply when no `enable` attribute is given.

## 2. Where skins live and how they resolve

```text
Templates/Modules/Social/SocialSharer/
+-- default.dwig     # fallback, keep it working
+-- compact.dwig
+-- ...
```

Resolution order for every skin:

1. The same relative path in the active theme.
2. The bundled default template.
3. `default.dwig` in the same directory.

Keep theme-specific sharer skins in the active theme. Changing the bundled fallback changes the default for every generated theme.

## 3. What data a sharer skin receives

| Value | Contents |
|---|---|
| `data.links` | Networks keyed by name: `facebook`, `x`, `pinterest`, `viber`, `whatsapp`, `linkedin`. Each entry carries `icon`, `svg`, and `enable`. Only entries with `enable` truthy render. |
| `data.params` | Tag attributes plus the instance ID (`data.params.id`), including `enable`. |

The safe pattern for any sharer:

```twig
{% set links = data.links|default({}) %}

<div class="share-buttons">
{% for name, link in links %}
    {% if link.enable|default(false) %}
        ...
    {% endif %}
{% endfor %}
</div>
```

Rules:

1. Iterate `data.links`; never hardcode the network list. The enabled set differs per instance, and hardcoding renders networks the editor switched off.
2. Test `link.enable` on every entry. The backend passes all networks with flags; the flags are the selection.
3. With no network enabled, the module shows an editor notice instead of buttons. Keep the skin harmless when nothing renders — an empty container, not a broken row.

## 4. Building share links for the current page

A sharer always shares the page it sits on. Read the page URL and title from the rendering context and URL-encode them into each network's share endpoint:

```twig
{% set page_url = url_current()|url_encode %}
{% set page_title = (data.seo.title|default(''))|url_encode %}

{% for name, link in data.links|default({}) %}
{% if link.enable|default(false) %}
    {% if name == 'facebook' %}
        <a target="_blank" rel="noopener" aria-label="Share on Facebook"
           href="https://www.facebook.com/sharer/sharer.php?u={{ page_url|e('html_attr') }}">{{ link.svg|raw }}</a>
    {% elseif name == 'x' %}
        <a target="_blank" rel="noopener" aria-label="Share on X"
           href="https://twitter.com/intent/tweet?text={{ page_title|e('html_attr') }}&amp;url={{ page_url|e('html_attr') }}">{{ link.svg|raw }}</a>
    {% elseif name == 'linkedin' %}
        <a target="_blank" rel="noopener" aria-label="Share on LinkedIn"
           href="https://www.linkedin.com/shareArticle?mini=true&amp;url={{ page_url|e('html_attr') }}&amp;title={{ page_title|e('html_attr') }}">{{ link.svg|raw }}</a>
    {% elseif name == 'whatsapp' %}
        <a target="_blank" rel="noopener" aria-label="Share on WhatsApp"
           href="https://wa.me/?text={{ page_title|e('html_attr') }}%20{{ page_url|e('html_attr') }}">{{ link.svg|raw }}</a>
    {% endif %}
{% endif %}
{% endfor %}
```

Rules:

1. Encode with `|url_encode` before escaping with `|e('html_attr')`. Titles contain spaces, quotes, and ampersands; unencoded values truncate or reroute the share.
2. Write `&amp;` between query parameters in markup. A bare `&` is invalid HTML and breaks strict parsers.
3. Open share endpoints with `target="_blank" rel="noopener"`. Never navigate the page itself away to a share dialog.
4. Label every button with `aria-label`. An icon alone says nothing to assistive technology.
5. Render icons from `link.svg` with `|raw`. The artwork is module-supplied icon markup, not editor input — the same trusted-fragment rule as player code, applied to one inline SVG per button.
6. Pinterest, Viber, and app-first networks need their own handlers (pin script, deeplinks). Cover them per network in the same loop; never drop an enabled network silently because its endpoint differs.

## 5. Placement

Sharers belong where sharing intent is highest — beside the content being shared:

```twig
{# Product page: beside the buy box #}
<module type="Social/SocialSharer" id="product-share-{{ product_id }}"
        enable="facebook,x,whatsapp" template="default.dwig" />

{# Blog post: below the article #}
<module type="Social/SocialSharer" id="post-share" enable="all" template="default.dwig" />
```

1. Suffix the ID per record (`product-share-{{ product_id }}`) when placements need independent network selections; share one stable ID when the same row repeats site-wide.
2. Since links always target the current page, placing the same sharer twice on one page duplicates identical buttons. One placement per page is the norm; a second needs a design reason.
3. Keep sharers out of the global header and footer. Those surfaces carry `Social/SocialLinks` profile icons, not per-page share buttons.

## 6. Use cases

**Product share row.** Facebook, X, and WhatsApp beside the buy box, compact icon buttons. Per-product instance ID.

**Blog post share.** Full network row below the article with `enable="all"`. One stable ID for the whole blog.

**Sticky share rail.** Fixed side rail on long articles reusing the post sharer skin. Same data, positioned CSS; no second instance.

**Minimal icon row.** Tight spacing, monochrome icons via CSS on the supplied SVGs, for footers of cards and email-style layouts.

**Launch announcement.** Temporary sharer on a campaign page with three networks. Remove the instance when the campaign ends; the skin stays for reuse.

## 7. Common mistakes

- Hardcoding the network list instead of looping `data.links`, rendering networks the editor disabled.
- Ignoring `link.enable`, so every network shows on every instance.
- Sharing a hardcoded URL instead of `url_current()`, so every page shares the homepage.
- Skipping `|url_encode`, truncating titles and URLs at the first space or ampersand.
- Writing bare `&` between share-endpoint parameters instead of `&amp;`.
- Omitting `target="_blank" rel="noopener"`, navigating the shopper away mid-checkout.
- Escaping `link.svg` with `|e`, printing icon markup as visible text.
- Dropping an enabled network (Pinterest, Viber) because its endpoint differs instead of handling it in the loop.
- Placing a sharer in the header or footer where `Social/SocialLinks` belongs, sharing the wrong thing in the wrong place.
- Forgetting `default.dwig`, so a missing skin selection breaks sharing site-wide.

