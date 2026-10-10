# Utilities/ConsentPrompt — Blocking Agreement Modal

The `Utilities/ConsentPrompt` module renders a blocking agreement modal: a heading, a message, and one "I Agree" action that visitors must accept before the modal leaves. Its skins live in `Templates/Modules/Utilities/ConsentPrompt/`. Editors write the heading and message and switch the prompt on; the choice persists per browser so return visitors are never asked twice.

Do not confuse it with its sibling. `Utilities/CookieNotice` is the granular banner with per-tracker accept/refuse. The consent prompt is the single blocking gate: agree to continue. Banners negotiate; prompts gate.

## 1. Embedding a consent prompt

```twig
<module
    type="Utilities/ConsentPrompt"
    id="site-consent"
    template="default.dwig"
/>
```

- `type` is `Utilities/ConsentPrompt`: the path-style name that mirrors the skin path `Templates/Modules/Utilities/ConsentPrompt/`. The bare legacy name still resolves, but every example in these docs uses the path-style form.
- `id` identifies this instance. One prompt per site is the norm; keep the ID stable so its settings persist.
- `template` selects the skin filename from the consent-prompt directory. Pass only the filename, never a path. A template chosen in Live Edit is saved against the instance and takes precedence over the inline attribute.
- An unavailable file falls back to `default.dwig`, so the directory keeps a working `default.dwig`.
- The prompt renders only when enabled in settings. When disabled, nothing renders at all — the skin needs no disabled branch.

## 2. Where skins live and how they resolve

```text
Templates/Modules/Utilities/ConsentPrompt/
+-- default.dwig     # fallback, keep it working
+-- compact.dwig
+-- ...
```

Resolution order for every skin:

1. The same relative path in the active theme.
2. The bundled default template.
3. `default.dwig` in the same directory.

Keep theme-specific prompt skins in the active theme. The bundled skins ship as empty stubs, so writing the prompt skin is part of building the theme — follow the modal pattern in section 4 so the first skin you write already behaves like the module's native output.

## 3. What the prompt is configured with

| Setting | Purpose |
|---|---|
| Enabled flag | Master switch. Anything but enabled renders nothing. |
| Heading | Modal title (for example age confirmation or terms update). Plain text. |
| Message | The agreement body. Plain text with line breaks; no links, no markup. |

Rules:

1. Keep heading and message short enough to fit a centered dialog on a phone screen. The prompt blocks the page; long copy punishes every visitor.
2. Say exactly what agreeing means and what happens on refusal. A gate without consequences stated is a dark pattern.
3. Never put trackers, codes, or settings values into the modal markup. The prompt records one decision; it configures nothing.

## 4. The modal pattern

Blocking dialog, single agree action, persisted choice, honest dismissal rules:

```twig
<div class="modal fade show d-block" id="consentPromptModal" tabindex="-1" role="dialog" aria-modal="true"
     aria-labelledby="consentPromptTitle" hidden data-consent-prompt>
    <div class="modal-dialog modal-dialog-centered">
        <div class="modal-content shadow">
            <div class="modal-header">
                <h5 class="modal-title fw-bold" id="consentPromptTitle">{{ heading|default('Please confirm')|e }}</h5>
            </div>
            <div class="modal-body">
                <p>{{ message|default('')|e|nl2br }}</p>
            </div>
            <div class="modal-footer">
                <button id="accept-consent-prompt" class="btn btn-primary" type="button">I Agree</button>
            </div>
        </div>
    </div>
</div>
```

```js
const storageKey = 'mw_consent_prompt_agreed';
const modal = document.getElementById('consentPromptModal');
if (!localStorage.getItem(storageKey)) {
    modal.hidden = false;
    document.getElementById('accept-consent-prompt').addEventListener('click', function () {
        localStorage.setItem(storageKey, '1');
        modal.remove();
    });
} else {
    modal.remove();
}
```

Rules:

1. Show only when no stored agreement exists; remove (not merely hide) the modal once agreed or when a choice is already stored. Leftover hidden modals trap focus and confuse audits.
2. Keep exactly one action: agree. A blocking gate with three equal choices is a banner wearing a modal's clothes — that belongs in `Utilities/CookieNotice`.
3. Trap focus inside the dialog while it shows and return focus sensibly on removal. Blocking means modal semantics throughout: `role="dialog"`, `aria-modal`, labelled title.
4. Escape heading and message. Prompt copy is merchant input; `|raw` has no place here. Preserve line breaks for readability.
5. Scope IDs per instance when two prompts could ever coexist. Duplicated modal IDs agree the wrong gate.

## 5. Use cases

**Age gate.** Confirm legal age before restricted content. Short heading, one-line consequence, agree to enter.

**Terms update.** Re-confirm changed terms with a versioned storage key so the new text re-prompts exactly once.

**Announcement gate.** Critical notice (downtime, relocation) that every visitor must acknowledge. Retire the prompt when the event passes.

**Entry disclaimer.** Beta, medical, or financial disclaimers ahead of sensitive content. Message states the limit; agreement unlocks browsing.

## 6. Common mistakes

- Two equal actions (or a preferences link) turning the gate into a banner. Granular choice lives in the cookie notice.
- Showing the prompt to visitors who already agreed, by skipping the stored-choice check.
- Hiding instead of removing the agreed modal, leaving focus traps and audit failures.
- Unstated refusal consequences. If there is no alternative path, say so plainly.
- Long legal prose in a blocking dialog. Terms belong on a page with a link, not in a gate.
- Unescaped heading or message.
- Duplicated modal IDs across two prompts.
- Using a blocking gate where a dismissible banner suffices — gates cost goodwill on every visit.
- Forgetting `default.dwig`, so a missing skin selection breaks the gate site-wide.
