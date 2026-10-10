# Social/SocialLinks — Follow-Us Profile Icons

The `Social/SocialLinks` module renders links to the site's own social profiles: Facebook, Instagram, YouTube, X, LinkedIn, and more. Its skins live in `Templates/Modules/Social/SocialLinks/`. Editors configure profile URLs and per-network toggles once; the backend resolves each URL (instance setting first, site-wide website setting second) and passes the ready list; the skin renders the icon row.

Do not confuse it with `Social/SocialSharer`. Social links point visitors at the site's profiles. The sharer shares the current page outward. Headers, footers, and contact pages want social links; product and post pages want the sharer.

## 1. Embedding social links

```twig
<module
    type="Social/SocialLinks"
    id="footer-social"
    template="default.dwig"
/>
```

- `type` is `Social/SocialLinks`: the path-style name that mirrors the skin path `Templates/Modules/Social/SocialLinks/`. The bare legacy name still resolves, but every example in these docs uses the path-style form.
- `id` identifies this instance and connects it to its profile URLs and network toggles. Footer and header rows use different IDs (`footer-social`, `header-social`) when their networks differ. Changing an ID orphans the URLs saved under the old one.
- `template` selects the skin filename from the social-links directory. Pass only the filename, never a path. A template chosen in Live Edit is saved against the instance and takes precedence over the inline attribute.
- An unavailable file falls back to `default.dwig`, so the directory keeps a working `default.dwig`.

Tag attributes that adjust the row:

| Attribute | Purpose |
|---|---|
| `show-icons` | Comma-separated network names forced on for this placement (for example `show-icons="facebook,instagram"`), even when their saved toggles are off. Use sparingly; saved toggles are the normal control. |
| `option-group` | Reads configuration saved under a different group, so two placements share one profile set. Prefer distinct IDs unless the rows must stay identical. |

## 2. Where skins live and how they resolve

```text
Templates/Modules/Social/SocialLinks/
+-- default.dwig     # fallback, keep it working
+-- footer.dwig
+-- ...
```

Resolution order for every skin:

1. The same relative path in the active theme.
2. The bundled default template.
3. `default.dwig` in the same directory.

Keep theme-specific link rows in the active theme. Changing the bundled fallback changes the default for every generated theme.

## 3. What data a social-links skin receives

| Value | Contents |
|---|---|
| `data.links` | The profile list. Each item carries `title` (display name), `platform` (network key), `status` (enabled flag), `url` (resolved profile URL), and `icon` (icon class). |
| `data.settings` | Per-network `<platform>_enabled` flags plus `social_links_has_enabled`, true when at least one core network is on. |
| `data.params` | Tag attributes plus the instance ID (`data.params.id`). |

Profile URLs resolve per network as: instance URL first, site-wide website URL second. Either can be empty, so the skin must test both the flag and the URL:

```twig
<div class="socials d-flex flex-wrap gap-2">
{% for link in data.links|default([]) %}
    {% if link.status and link.url|default('') is not empty %}
        <a href="{{ link.url|e('html_attr') }}" aria-label="{{ link.title|e('html_attr') }}">
            ...
        </a>
    {% endif %}
{% endfor %}
</div>
```

Auto-escaping is off in `.dwig` templates, so escape URLs with `|e('html_attr')` and titles with `|e`. Never render a link value with `|raw`.

## 4. Mapping platforms to icons

The backend passes an icon class per link, but skins normally map `platform` to the theme's own icon set with a fallback for networks the map does not name:

```twig
{% set iconClass = {
    'facebook': 'fa-brands fa-facebook-f',
    'twitter': 'fa-brands fa-twitter',
    'youtube': 'fa-brands fa-youtube',
    'instagram': 'fa-brands fa-instagram',
    'whatsapp': 'fa-brands fa-whatsapp',
    'linkedin': 'fa-brands fa-linkedin-in'
}[link.platform]|default('fa-brands fa-link') %}

<a class="btn btn-outline-light rounded-circle"
   href="{{ link.url|e('html_attr') }}"
   aria-label="{{ link.title|e('html_attr') }}">
    <i class="{{ iconClass|e('html_attr') }}" aria-hidden="true"></i>
</a>
```

Rules:

1. Iterate `data.links`; never hardcode the network list. The enabled set differs per instance, and hardcoding renders profiles the editor switched off or never configured.
2. Test both `link.status` and a non-empty `link.url`. An enabled toggle with no URL still has nowhere to go; a URL with the toggle off must stay hidden.
3. Keep the `|default()` icon fallback. New or uncommon networks arrive with a platform key the map never saw; the fallback keeps them visible instead of icon-less.
4. Label every link with `aria-label` from `link.title` and mark the icon `aria-hidden="true"`. An icon alone says nothing to assistive technology.
5. Decide link target deliberately and keep it uniform across the row. Off-site profiles conventionally open a new tab (`target="_blank" rel="noopener"`); same-tab navigation keeps visitors but drops them off-site on return.

## 5. Placement

Social links belong in persistent surfaces — the rows visitors expect on every page:

```twig
{# Footer follow row #}
<module type="Social/SocialLinks" id="footer-social" template="default.dwig" />

{# Header icon set, different networks #}
<module type="Social/SocialLinks" id="header-social" template="compact.dwig" />
```

1. Give each placement its own ID when its networks differ. Sharing one ID across header and footer forces both rows into one configuration.
2. Use `option-group` only when two placements must mirror one profile set exactly. Otherwise the mirror breaks the day one row needs its own network.
3. Keep share buttons out of these rows. A follow icon that opens a share dialog (or vice versa) is the classic mix-up between this module and `Social/SocialSharer`.

## 6. Use cases

**Footer follow row.** Circular outline buttons for the configured profiles under a "Follow us" heading. The standard `footer-social` instance.

**Header icon set.** Compact subset (three or four networks) in the top bar via `show-icons` or a dedicated `header-social` ID with its own toggles.

**Contact page profiles.** Larger labeled row (icon plus network name) beside address and phone. Same data, text-visible skin for clarity.

**Author or about box.** Small row under a team member or author bio linking personal profiles. Per-instance IDs keep each person's links independent.

**Coming-soon page.** Single social row on a minimal shell so an unlaunched site still collects followers. Reuses the footer skin; no new skin needed.

## 7. Common mistakes

- Hardcoding the network list instead of looping `data.links`, rendering profiles the editor disabled or never set.
- Testing only `status` and emitting `href=""` links, or testing only `url` and showing toggled-off networks.
- Missing the icon-map fallback, leaving uncommon networks icon-less.
- Hardcoding profile URLs in the skin instead of reading resolved `link.url`, freezing out editor configuration.
- Sharing one ID (or `option-group`) across rows that need different networks, so configuring one rewrites the other.
- Putting `Social/SocialSharer` where follow icons belong, or follow icons where share buttons belong.
- Icon-only links with no `aria-label`.
- Forgetting `default.dwig`, so a missing skin selection breaks the row site-wide.
