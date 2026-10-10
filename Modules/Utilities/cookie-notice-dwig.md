# Utilities/CookieNotice — Cookie Consent Banner

The `Utilities/CookieNotice` module renders the cookie consent banner: the notice visitors accept or refuse, the policy link, and per-tracker toggles. Its skins live in `Templates/Modules/Utilities/CookieNotice/`. Editors configure colors, policy URL, banner position, the default state, and which trackers the banner governs; the skin renders the banner and wires consent so unapproved tracking never loads.

## 1. Embedding a cookie notice

```twig
<module
    type="Utilities/CookieNotice"
    id="cookie-notice"
    template="default.dwig"
/>
```

- `type` is `Utilities/CookieNotice`: the path-style name that mirrors the skin path `Templates/Modules/Utilities/CookieNotice/`. The bare legacy name still resolves, but every example in these docs uses the path-style form.
- `id` identifies this instance. One banner per site is the norm; keep the ID stable so its settings persist.
- `template` selects the skin filename from the cookie-notice directory. Pass only the filename, never a path. A template chosen in Live Edit is saved against the instance and takes precedence over the inline attribute.
- An unavailable file falls back to `default.dwig`, so the directory keeps a working `default.dwig`.
- The banner renders only when the cookie policy is enabled in settings. When disabled, the module renders nothing at all — the skin needs no disabled branch.

## 2. Where skins live and how they resolve

```text
Templates/Modules/Utilities/CookieNotice/
+-- default.dwig     # fallback, keep it working
+-- compact.dwig
+-- ...
```

Resolution order for every skin:

1. The same relative path in the active theme.
2. The bundled default template.
3. `default.dwig` in the same directory.

Keep theme-specific notice skins in the active theme. The bundled skins ship as empty stubs, so writing the notice skin is part of building the theme — follow the banner pattern in section 4 so the first skin you write already behaves like the module's native output.

## 3. What the banner is configured with

| Setting | Purpose |
|---|---|
| `cookies_policy` | Master switch (`y` shows the banner; anything else renders nothing). |
| `cookiePolicyURL` | Policy page link from the banner. Default points at the privacy policy. |
| `textColor` / `backgroundColor` | Banner colors. Empty inherits the theme. |
| `panelTogglePosition` | Banner placement. Default `right`. |
| `unsetDefault` | State of undecided trackers. Default `blocked`: nothing unapproved loads. |
| `showLiveChatMessage` | Live-chat notice flag inside the banner. |
| Tracker entries | Per-service `{enabled, label, code}` rows (analytics, pixel, chat, heatmaps). Only enabled services appear as toggles; their codes stay in settings, never in markup. |

Rules:

1. Never render tracker codes, IDs, or keys into markup. Codes configure loading server-side; the skin handles labels and toggles only.
2. Keep `blocked` as the undecided default. Flipping unknowns to allowed trades a consent violation for a metrics bump.
3. Link the policy URL on every banner variant. A consent control without its policy reference fails the purpose test.

## 4. The banner pattern

Notice text, policy link, per-tracker toggles for enabled services, accept/refuse actions, remembered choice:

```twig
<div class="cookie-notice" role="dialog" aria-modal="false" aria-labelledby="cookie-notice-title" hidden data-cookie-notice>
    <h2 id="cookie-notice-title">We value your privacy</h2>
    <p>We use cookies to improve your experience and analyse traffic.
       Read our <a href="{{ policy_url|default(site_url('privacy-policy'))|e('html_attr') }}">cookie policy</a>.</p>
    <ul>
    {% for tracker in trackers|default([]) %}
        {% if tracker.enabled %}
        <li><label><input type="checkbox" name="tracker-{{ tracker.key|e('html_attr') }}"
                          {% if tracker.accepted %}checked{% endif %}>
            {{ tracker.label|e }}</label></li>
        {% endif %}
    {% endfor %}
    </ul>
    <button type="button" data-cookie-accept-all>Accept all</button>
    <button type="button" data-cookie-save>Save choices</button>
    <button type="button" data-cookie-refuse>Refuse all</button>
</div>
```

Rules:

1. Offer accept-all, save-choices, and refuse-all as equal actions. A banner with no genuine refusal is decoration, not consent.
2. Render toggles only for enabled services. Disabled trackers must be absent, not pre-checked.
3. Persist the choice (cookie or storage) and suppress the banner on return visits. Re-asking every page view trains visitors to click anything to dismiss.
4. Gate every tracker load on the stored choice: enabled-and-accepted loads, everything else stays blocked. No tracking fires before the visitor decides.
5. Keep the banner out of the tab order once dismissed, and return focus sensibly on close. Modal-like focus traps belong to blocking dialogs, not dismissible notices.
6. Escape labels and URLs. Tracker labels are merchant input; `|raw` has no place in a notice skin.

## 5. Use cases

**Bottom banner.** Full-width notice with text, policy link, and three actions. The standard placement.

**Corner card.** Compact floating card (`panelTogglePosition` right) with toggles behind a "customize" disclosure. Same choices, smaller chrome.

**Policy page embed.** Full tracker list with descriptions beside the policy text for visitors revisiting their choices.

**Pre-consent placeholder.** Blocked embeds (maps, videos, chat) render a consent-gated placeholder until the matching tracker is accepted. Same choice store, different surface.

## 6. Common mistakes

- No genuine refuse path, or disabled services rendered as pre-checked toggles.
- Tracker codes or IDs rendered into markup instead of staying in settings.
- Tracking loads before the visitor decides, or undecided defaults flipped to allowed.
- Missing policy link on a banner variant.
- Re-asking on every page view instead of persisting the choice.
- Marketing copy or dark patterns in a legal surface.
- Banner trapping focus like a modal, or dismissed banners lingering in the tab order.
- Forgetting `default.dwig`, so a missing skin selection breaks consent site-wide.
