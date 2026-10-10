# Users/Register — Account Creation Forms

The `Users/Register` module renders the registration form: name fields, email, password, terms, newsletter opt-in, and social-provider buttons. Its skins live in `Templates/Modules/Users/Register/`. The backend resolves which fields the form shows, the terms and newsletter settings, and the endpoint URLs; the skin renders the form and submits it through `window.dbEvent.user.register()`.

## 1. Embedding registration

```twig
<module
    type="Users/Register"
    id="site-register"
    template="default.dwig"
/>
```

- `type` is `Users/Register`: the path-style name that mirrors the skin path `Templates/Modules/Users/Register/`. The bare legacy name still resolves, but every example in these docs uses the path-style form.
- `id` identifies this instance. Registration forms rarely repeat on one site; keep the ID stable so its skin selection persists.
- `template` selects the skin filename from the register directory. Pass only the filename, never a path. A template chosen in Live Edit is saved against the instance and takes precedence over the inline attribute.
- An unavailable file falls back to `default.dwig`, so the directory keeps a working `default.dwig`.

## 2. Where skins live and how they resolve

```text
Templates/Modules/Users/Register/
+-- default.dwig     # fallback, keep it working
+-- compact.dwig
+-- ...
```

Resolution order for every skin:

1. The same relative path in the active theme.
2. The bundled default template.
3. `default.dwig` in the same directory.

Keep theme-specific registration skins in the active theme. Changing the bundled fallback changes the default for every generated theme.

## 3. What data a register skin receives

| Value | Contents |
|---|---|
| `data.module_dom_id` | Unique, instance-derived DOM ID for the card. Scope all CSS and script queries under it. |
| `data.csrf_token` | CSRF token for the hidden `_token` field. |
| `data.register_api_url` | Registration endpoint. Informational; `dbEvent` posts through its own routing. |
| `data.login_url` / `data.success_url` | Sign-in link and post-registration redirect target. Never hardcode either. |
| `data.social_providers` | Enabled social providers as `key`, `label`, `url` each. Empty when social login is off. |
| `data.show_first_name` / `data.show_last_name` | Whether each name field renders. |
| `data.confirm_password` | Whether the confirmation field renders. |
| `data.terms_required` / `data.terms_label` | Terms checkbox flag and label (label may contain a link). |
| `data.newsletter_enabled` | Whether the newsletter opt-in renders. |
| `data.registration_enabled` | Whether sign-ups are open at all. |
| `data.is_logged` | Whether the visitor already has an account. |
| `data.params` | Tag attributes plus the instance ID (`data.params.id`). |

Three states gate the whole card: logged-in visitors see confirmation, closed registration shows the disabled notice, everyone else gets the form.

## 4. The form pattern

Card shell with state branches, provider buttons, conditional fields, and the `dbEvent` submit handler:

```twig
<div id="{{ data.module_dom_id|e('html_attr') }}" class="mw-auth-card card mx-auto" style="max-width: 520px;">
    <div class="card-body p-4">
    {% if data.is_logged %}
        <div class="alert alert-success mb-0">You are already logged in.</div>
    {% elseif not data.registration_enabled %}
        <div class="alert alert-warning mb-0">New account registration is currently disabled.</div>
    {% else %}
        <h2 class="h3 text-center">Create your account</h2>
        <div class="mw-auth-message alert d-none" role="alert"></div>
        {% if data.social_providers|default([]) is not empty %}
            <div class="row g-2 mb-3">
            {% for provider in data.social_providers %}
                <div class="col-6">
                    <a class="btn btn-outline-secondary w-100"
                       href="{{ provider.url|e('html_attr') }}" rel="nofollow">{{ provider.label|e }}</a>
                </div>
            {% endfor %}
            </div>
        {% endif %}
        <form class="mw-register-form" novalidate>
            <input type="hidden" name="_token" value="{{ data.csrf_token|e('html_attr') }}">
            {# ...conditional name, email, password, confirm, terms, newsletter fields... #}
            <button class="btn btn-primary btn-lg w-100" type="submit">Create account</button>
        </form>
        <p class="text-center">Already have an account? <a href="{{ data.login_url|e('html_attr') }}">Sign in</a></p>
    {% endif %}
    </div>
</div>
```

Field rules:

1. Render each field only under its flag (`show_first_name`, `show_last_name`, `confirm_password`, `terms_required`, `newsletter_enabled`). Unflagged fields must be absent from markup, not merely hidden — extra posted fields confuse validation.
2. Keep field names exactly: `first_name`, `last_name`, `email`, `password`, `confirm_password`, `terms`, `newsletter_subscribe`, plus hidden `_token`. The API reads these names; renaming breaks submission while the form looks fine.
3. Pair every input with its `<label>` (`for` equals input `id`, IDs prefixed by the DOM ID). Unlabeled inputs fail assistive technology and shrink tap targets.
4. Render `terms_label` with `|raw` — it is admin-configured label HTML that may contain the terms link. This is the one deliberate raw output in auth skins; everything else stays escaped.
5. Set correct `autocomplete` values (`given-name`, `family-name`, `email`, `new-password`). Password managers and mobile keyboards depend on them.

Submit through `window.dbEvent` from `userfiles/modules/developmentbucket/db_lib/events/common.js`, which loads automatically on normal frontend pages. Never reference the `mw` JS library:

```js
const response = await window.dbEvent.user.register(form);
```

Passing the form element preserves multipart encoding. On success show the message, reset the form, and follow `payload.redirect` (falling back to `data.success_url`) after a short beat; on failure show the first validation message and re-enable the button. Every call uses `try/catch` because failures reject. Guard the binder with a ready flag on the DOM ID so re-renders never double-submit.

## 5. Registering with email OTP and phone OTP

Passwordless registration creates the account only after the visitor proves the address: request a code, then verify it. Both OTP methods share one two-step skin pattern — identifier first, code second — and both log the customer in on success.

### 5.1 Email OTP — request, then verify

Step one collects the email and requests the code. Step two collects the code and creates the account:

```twig
<div id="{{ data.module_dom_id|e('html_attr') }}-email-otp">
    <div class="mw-auth-message alert d-none" role="alert"></div>
    <form data-otp-request-form novalidate>
        <label>Email address
            <input class="form-control" type="email" name="email" autocomplete="email" required>
        </label>
        <button class="btn btn-primary w-100" type="submit">Send code</button>
    </form>
    <form data-otp-verify-form class="d-none" novalidate>
        <p>Enter the code sent to <strong data-otp-sent-to></strong>.</p>
        <label>Verification code
            <input class="form-control" type="text" name="code" inputmode="numeric" autocomplete="one-time-code" required>
        </label>
        <button class="btn btn-primary w-100" type="submit">Verify and create account</button>
        <button class="btn btn-link w-100" type="button" data-otp-resend>Resend code</button>
    </form>
</div>
```

```js
const requestForm = root.querySelector('[data-otp-request-form]');
const verifyForm = root.querySelector('[data-otp-verify-form]');
let pendingEmail = '';

requestForm.addEventListener('submit', async function (event) {
    event.preventDefault();
    pendingEmail = requestForm.email.value.trim();
    try {
        await window.dbEvent.user.register({
            registration_type: 'email_otp',
            email: pendingEmail
        });
        root.querySelector('[data-otp-sent-to]').textContent = pendingEmail;
        requestForm.classList.add('d-none');
        verifyForm.classList.remove('d-none');
    } catch (error) {
        showAuthError(error);
    }
});

verifyForm.addEventListener('submit', async function (event) {
    event.preventDefault();
    try {
        const response = await window.dbEvent.user.register({
            registration_type: 'email_otp',
            email: pendingEmail,
            code: verifyForm.code.value.trim()
        });
        window.setTimeout(function () {
            window.location.assign('{{ data.success_url|e('js') }}');
        }, 800);
    } catch (error) {
        showAuthError(error);
    }
});
```

Rules:

1. Call `register` twice with the same `registration_type` and `email`: first without `code` to request, then with `code` to verify and create. The account exists only after verification, and the customer is logged in immediately.
2. Keep the verified email in script state between steps and resend it unchanged on verify. A retyped address that differs from the requested one fails verification.
3. Offer resend by repeating the request call. Rate-limit it client-side (disable briefly after each send) so visitors cannot spam the endpoint.
4. Accept `otp` as equivalent to `code` if the input is named that way; the client converts it.

### 5.2 Phone OTP — country code plus number, then code

Identical two-step shape with split phone fields. Send the dialing prefix in `country_code` and the national number in `phone`, and repeat both unchanged on verify:

```twig
<form data-otp-request-form novalidate>
    <label>Country code
        <input class="form-control" type="text" name="country_code" value="+91" required>
    </label>
    <label>Phone number
        <input class="form-control" type="tel" name="phone" autocomplete="tel-national" required>
    </label>
    <button class="btn btn-primary w-100" type="submit">Send code</button>
</form>
```

```js
await window.dbEvent.user.register({
    registration_type: 'phone_otp',
    country_code: '+91',
    phone: '9876543210'
});

const response = await window.dbEvent.user.register({
    registration_type: 'phone_otp',
    country_code: '+91',
    phone: '9876543210',
    code: '123456'
});
```

Rules:

1. Keep `country_code` and `phone` separate fields. The backend stores them separately; a leading `+` is normalized, and calling codes run one to four digits. Never merge them into one input.
2. Repeat the exact same `country_code` and `phone` on the verify call. Changed digits verify against a code that was never sent to them.
3. Prefer `tel-national` autocomplete on the number field so mobile keyboards offer the right layout.
4. When `registration_type` is omitted, the client infers the OTP method from the fields present (`phone` implies phone OTP, `email` implies email OTP). Set it explicitly anyway — inference is a fallback, not a contract skins should lean on.

### 5.3 Shared OTP rules

1. Gate the OTP forms behind the same three states as passwords: hide them for logged-in visitors and when registration is disabled.
2. Never invent a verification UI beyond request and verify. Code checking happens server-side; the skin collects and forwards, nothing more.
3. Every call uses `try/catch` because failures reject. Expired or wrong codes surface as error messages beside the code input with the request form one step back — never a dead end.
4. Do not mix OTP and password fields in one submit. One registration call carries one method; combined payloads resolve to password registration whenever `password` is present.

## 6. Use cases

**Standalone register page.** Full card with providers, all fields, and sign-in link. The account front door.

**Checkout-adjacent signup.** Compact skin beside express checkout for guests creating accounts mid-purchase. Same contract, tighter chrome.

**Modal registration.** Card without page chrome inside a dialog for header "Join" buttons. Success redirects through `success_url`; failures stay in the modal message box.

**Invite-only notice.** Closed-registration state styled as "request an invite" with contact link. The disabled branch carries the design, not an afterthought.

## 7. Common mistakes

- Rendering unflagged fields instead of omitting them, confusing validation with stray posted values.
- Renaming `email`, `password`, `terms`, or `_token` while styling the form.
- Unlabeled inputs or duplicated input IDs across two forms.
- Escaping `terms_label` with `|e`, printing the terms link as visible text — or applying `|raw` to anything else in the form.
- Hardcoding login or success URLs instead of `data.login_url` / `data.success_url`.
- Showing the form to logged-in visitors, or a dead form when registration is disabled, instead of the two notice branches.
- Missing `autocomplete` values, breaking password managers and mobile keyboards.
- Forgetting `default.dwig`, so a missing skin selection breaks sign-ups site-wide.
