# Utilities — Site Services (Forms, Map, Consent)

The `Utilities` family holds site services: contact forms, the map embed, and consent notices. Its skins live in `Templates/Modules/Utilities/`. These modules serve the site itself rather than content, products, or accounts: they collect messages, show places, and record consent.

```text
Templates/Modules/Utilities/
+-- Forms/            # contact and enquiry forms (contact_form backend)
+-- GoogleMap/        # map embed slot
+-- CookieNotice/     # cookie banner slot
+-- ConsentPrompt/    # consent dialog slot
```

Only `Forms` carries a full skin contract today; the other three directories ship as empty reserved slots (sections 5–6).

## 1. Embedding utilities

```twig
<module type="Utilities/Forms" id="contact-form" template="contact.dwig" />
```

- `type` is the path-style name mirroring the skin path (`Utilities/Forms`, and later `Utilities/GoogleMap` and siblings). The bare legacy names still resolve, but every example in these docs uses the path-style form.
- `id` identifies the instance and connects it to its fields and settings. Contact page, footer mini-form, and quote form use different IDs so each keeps its own fields. Changing an ID orphans the configuration saved under the old one.
- `template` selects the skin filename from the module's directory. Pass only the filename, never a path. A template chosen in Live Edit is saved against the instance and takes precedence over the inline attribute.
- An unavailable file falls back to `default.dwig`, so every utilities directory keeps a working `default.dwig`.

## 2. Forms — what the skin receives

| Value | Contents |
|---|---|
| `data.form_id` | Unique form name shared by the `<form>` (`name`, `data-form-id`) and the success box (`msg<id>`). Scope all script queries under the wrapper ID derived from the instance. |
| `data.title` | Form heading. Editable in place through the skin's `edit` region (`field="contact_form_title"`). |
| `data.default_fields` | Field set rendered by the nested `custom_fields` module. |
| `data.show_newsletter_subscription` / `data.newsletter_subscribed` / `data.newsletter_label` | Newsletter opt-in row, shown only when enabled and not already subscribed. |
| `data.require_terms` / `data.require_terms_when` | Terms module embed, gated on both values. |
| `data.captcha_enabled` / `data.captcha_label` | CAPTCHA block with the `captcha` module. Render only when enabled. |
| `data.button_text` | Submit label, defaulting to `'Send message'`. |
| `data.thank_you_message` | Success text for the confirmation box. |
| `data.params` | Tag attributes plus the instance ID (`data.params.id`). |

## 3. Forms — the submit pattern

Custom fields render through the nested `custom_fields` module; submission goes through `window.dbEvent.form.submit(form)` from `userfiles/modules/developmentbucket/db_lib/events/common.js`, which loads automatically on normal frontend pages. Never reference the `mw` JS library:

```twig
<form class="mw_form needs-validation" data-form-id="{{ data.form_id }}" name="{{ data.form_id }}"
      method="post" enctype="multipart/form-data" data-contact-form>
    <module type="custom_fields" for-id="{{ data.params.id }}" data-for="module"
            default-fields="{{ data.default_fields }}" />
    {# ...newsletter, terms, captcha, submit button... #}
</form>
<div class="alert alert-success" role="status" aria-live="polite" hidden data-contact-success>
    <p data-contact-success-message>{{ data.thank_you_message }}</p>
</div>
<div class="alert alert-danger" role="alert" aria-live="assertive" hidden data-contact-error></div>
```

```js
form.addEventListener('submit', async function (event) {
    event.preventDefault();
    if (!form.reportValidity()) {
        return;
    }
    try {
        const response = await window.dbEvent.form.submit(form);
        successMessage.textContent = response.message || {{ data.thank_you_message|default('Thank you.')|json_encode|raw }};
        successBox.hidden = false;
        form.reset();
    } catch (response) {
        showFieldErrors(response && response.error ? response.error.fields : {});
        errorBox.textContent = (response && response.message) || 'Unable to send your message.';
        errorBox.hidden = false;
    }
});
```

Rules:

1. Pass the form element directly to preserve multipart encoding for file fields.
2. Run native validation (`reportValidity()`) before submitting; map server field errors back onto inputs with `is-invalid` plus per-field messages and focus the first invalid field.
3. Keep success (`role="status"`) and error (`role="alert"`) regions in markup, announced through `aria-live`. Silent submits strand everyone.
4. Disable the submit button while the request flies and restore it after. Scope the whole script under the instance-derived wrapper ID so two forms never share handlers.
5. Render newsletter, terms, and CAPTCHA blocks only under their flags. Unflagged blocks must be absent, not hidden.

## 4. Forms — use cases

**Contact page.** Full form with custom fields, newsletter opt-in, terms, and CAPTCHA. The canonical `contact.dwig` pattern.

**Quote or bulk enquiry.** Same contract with product-context fields (`bulk-enquiry.dwig`). The enquiry subject travels with the field set, not a forked backend.

**Footer mini form.** Name, email, and message only, compact skin, same submit flow. Different instance ID, same inbox.

**Modal form.** Card without page chrome inside a dialog. Success stays in the modal message box; failures never navigate away.

## 5. Map and consent slots

`GoogleMap`, `CookieNotice`, and `ConsentPrompt` ship as empty reserved slots with no skin contract yet. When skinning them, follow the family rules below and the module-specific shape editors configure (location for the map; copy and buttons for the notices). Do not invent settings keys: render what the instance passes, guarded with `|default()` fallbacks like every other skin.

1. Consent notices must be dismissible, remember the choice, and never block content before a decision where the law requires prior consent. Notices are legal surfaces, not marketing banners.
2. Map embeds load lazily (click-to-load or intersection) and carry a text address fallback for no-JS and blocked-embed cases.
3. Keep third-party embeds behind consent where the installation requires it: no tracking load before the visitor agrees.

## 6. Common mistakes

- Renaming the form hooks (`data-contact-form`, success/error boxes), orphaning the submit script while the form looks fine.
- Submitting without `reportValidity()`, sending invalid data the server rejects field by field.
- Silent submits with no status regions, or one shared message box for two forms.
- Rendering newsletter, terms, or CAPTCHA blocks their flags disable.
- Hand-rolling form posts or calling the `mw` JS library instead of `dbEvent.form.submit`.
- Marketing copy inside consent notices, or undismissable banners.
- Maps without address fallbacks, or tracking embeds loading before consent.
- Forgetting `default.dwig` in a utilities directory, so a missing skin breaks the service site-wide.
