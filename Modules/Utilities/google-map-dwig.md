# Utilities/GoogleMap — Location Map Embed

The `Utilities/GoogleMap` module renders a location map: an embedded map frame for an address with zoom, size, and style controls. Its skins live in `Templates/Modules/Utilities/GoogleMap/`. Editors set the address (free text or structured parts) and display options; the skin renders the embed frame with a text-address fallback.

## 1. Embedding a map

```twig
<module
    type="Utilities/GoogleMap"
    id="contact-map"
    data-address="221B Baker Street, London"
    template="default.dwig"
/>
```

- `type` is `Utilities/GoogleMap`: the path-style name that mirrors the skin path `Templates/Modules/Utilities/GoogleMap/`. The bare legacy name still resolves, but every example in these docs uses the path-style form.
- `id` identifies this instance and connects it to its address and display settings. Contact page, footer mini-map, and store-locator placements use different IDs. Changing an ID orphans the settings saved under the old one.
- `template` selects the skin filename from the map directory. Pass only the filename, never a path. A template chosen in Live Edit is saved against the instance and takes precedence over the inline attribute.
- An unavailable file falls back to `default.dwig`, so the directory keeps a working `default.dwig`.

Address and display attributes (each fills a gap only when no saved setting exists):

| Attribute | Purpose |
|---|---|
| `data-address` | Free-text address. Structured parts (country, city, street, zip) join into the address when set; otherwise this value (or its saved setting) is used. |
| `data-zoom` | Zoom level. Default `14`. |
| `data-width` / `data-height` | Frame dimensions. Both default to `100%`; the skin's ratio wrapper normally governs display. |
| `data-map-type` | Map style: `roadmap`, `satellite`, `hybrid`, or `terrain`. Default `roadmap`. |

## 2. Where skins live and how they resolve

```text
Templates/Modules/Utilities/GoogleMap/
+-- default.dwig     # fallback, keep it working
+-- compact.dwig
+-- ...
```

Resolution order for every skin:

1. The same relative path in the active theme.
2. The bundled default template.
3. `default.dwig` in the same directory.

Keep theme-specific map skins in the active theme. The bundled skins ship as empty stubs, so writing the map skin is part of building the theme — follow the frame pattern in section 4 so the first skin you write already behaves like the module's native output.

## 3. What data a map skin receives

| Value | Contents |
|---|---|
| `data.params` | Tag attributes plus the instance ID (`data.params.id`), including `data-address`, `data-zoom`, `data-width`, `data-height`, and `data-map-type`. Saved settings fill the same keys when attributes are absent. |

Resolve the working values once at the top, mirroring the backend's own chain:

```twig
{% set address = data.params['data-address']|default('')|trim %}
{% set zoom = data.params['data-zoom']|default(14)|int %}
{% set map_type = data.params['data-map-type']|default('roadmap') %}
{% if zoom < 1 or zoom > 21 %}{% set zoom = 14 %}{% endif %}
```

Rules:

1. Treat a missing address as missing, never as a default city. An empty address renders the address fallback (section 4), not a surprise location.
2. Clamp zoom to the valid range with default `14`. Out-of-range zooms render ocean or errors depending on provider mood.
3. Accept only the four known map types; anything else falls back to `roadmap`.
4. Escape the address for URLs (`|url_encode`) going into the embed `src` and for HTML (`|e`) going into visible text. Addresses are editor input with commas, hashes, and ampersands.

## 4. The frame pattern

Ratio wrapper, lazily loaded embed, text-address fallback for blocked embeds and no-JS:

```twig
{% set embed_src = 'https://maps.google.com/maps?q=' ~ address|url_encode ~ '&z=' ~ zoom ~ '&t=' ~ map_type|first|lower ~ '&output=embed' %}

<div class="map-frame ratio ratio-16x9">
    {% if address %}
        <iframe title="Map: {{ address|e('html_attr') }}"
                src="{{ embed_src|e('html_attr') }}"
                loading="lazy" referrerpolicy="no-referrer-when-downgrade"
                allowfullscreen></iframe>
    {% else %}
        <div class="d-flex align-items-center justify-content-center bg-body-secondary p-4">
            <p class="mb-0">Map address not configured yet.</p>
        </div>
    {% endif %}
</div>
<p class="map-address mt-2">{{ address|default('Address coming soon')|e }}</p>
```

Rules:

1. Prefer a ratio box over fixed pixel dimensions so the frame follows its column on small screens.
2. Load lazily (`loading="lazy"`). Map embeds are among the heaviest third-party frames on contact pages; eager loading taxes every visitor for a map most never scroll to.
3. Always print the human address beneath the frame. Blocked embeds, consent-gated loads, and no-JS clients still show visitors where to go.
4. Title the iframe with the address. An untitled frame is a dead end for assistive technology.
5. Gate third-party loading on consent where the installation requires it: render the address block and a load-on-click cover until the visitor agrees, matching the `Utilities/CookieNotice` choice store.

## 5. Use cases

**Contact page map.** Full-width 16x9 frame with the street address beneath and directions link. The canonical placement.

**Footer mini map.** Compact 4x3 frame in the footer contact column. Same data, smaller chrome, same fallback.

**Store locator entries.** One map per branch with per-instance IDs and addresses. Never share one ID across branches; shared IDs share the address.

**Event venue.** Map plus date and transit notes for event pages. Temporary instance, removed when the event ends.

## 6. Common mistakes

- Fixed pixel dimensions instead of a ratio box, overflowing small screens.
- Eager-loading a heavy third-party frame every visitor pays for.
- Missing address rendering a surprise default location instead of the fallback block.
- Unclamped zoom or unknown map types passed straight into the embed URL.
- Unencoded addresses breaking the embed `src` on commas, hashes, and ampersands.
- Untitled iframes with no address text for blocked-embed and no-JS visitors.
- Third-party map loading before consent where the installation gates it.
- Sharing one instance ID across branches, pointing every map at one address.
- Forgetting `default.dwig`, so a missing skin selection breaks maps site-wide.
