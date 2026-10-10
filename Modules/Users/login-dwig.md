# Users/Login — Sign-In Forms

The `Users/Login` module renders sign-in: password login plus email OTP and phone OTP tabs, social-provider buttons, and links to registration and password recovery. Its skins live in `Templates/Modules/Users/Login/`. The backend resolves which methods are enabled, the endpoint and redirect URLs, and the visitor state; the skin renders the tabbed panel and signs visitors in through `window.dbEvent.user.login()`.

## 1. Embedding login

```twig
<module
    type="Users/Login"
    id="site-login"
    template="default.dwig"
/>
```

- `type` is `Users/Login`: the path-style name that mirrors the skin path `Templates/Modules/Users/Login/`. The bare legacy name still resolves, but every example in these docs uses the path-style form.
- `id` identifies this instance. Login forms rarely repeat on one site; keep the ID stable so its skin selection persists.
- `template` selects the skin filename from the login directory. Pass only the filename, never a path. A template chosen in Live Edit is saved against the instance and takes precedence over the inline attribute.
- An unavailable file falls back to `default.dwig`, so the directory keeps a working `default.dwig`.
- A `return` attribute sets where successful sign-ins land. Only same-host destinations are honored; anything else falls back to the site root.

## 2. Where skins live and how they resolve

```text
Templates/Modules/Users/Login/
+-- default.dwig     # fallback, keep it working
+-- compact.dwig
+-- ...
```

Resolution order for every skin:

1. The same relative path in the active theme.
2. The bundled default template.
3. `default.dwig` in the same directory.

Keep theme-specific login skins in the active theme. Changing the bundled fallback changes the default for every generated theme.

## 3. What data a login skin receives

| Value | Contents |
|---|---|
| `data.module_dom_id` | Unique, instance-derived DOM ID for the panel. Scope all CSS and script queries under it. |
| `data.csrf_token` | CSRF token for hidden `_token` fields. |
| `data.login_url` | Login API route. Informational; `dbEvent` posts through its own routing. |
| `data.otp_enabled` / `data.phone_otp_enabled` | Whether the email OTP and phone OTP tabs render. Render each tab only under its flag. |
| `data.registration_enabled` | Whether to show the create-account link. |
| `data.forgot_password_url` / `data.register_url` | Recovery and sign-up links. Never hardcode either. |
| `data.return_url` | Same-host-validated post-login destination. Redirect here on success. |
| `data.social_providers` | Enabled social providers as `key`, `label`, `url` each. Empty when social login is off. |
| `data.is_logged` | Whether the visitor already has a session. Logged-in visitors see the account shortcut, never the forms. |
| `data.params` | Tag attributes plus the instance ID (`data.params.id`). |

## 4. The password form

Email plus password, remember-me, forgot link, per-field errors, and a status region. Submit through `window.dbEvent` from `userfiles/modules/developmentbucket/db_lib/events/common.js`, which loads automatically on normal frontend pages. Never reference the `mw` JS library:

```twig
<form data-login-form novalidate>
    <label>Email address
        <input type="email" name="email" autocomplete="email" required>
    </label>
    <label>Password
        <input type="password" name="password" autocomplete="current-password" required>
    </label>
    <label><input type="checkbox" name="remember"> Keep me signed in on this device</label>
    <button class="btn btn-primary w-100" type="submit">Sign in</button>
    <p class="form-status" role="status" aria-live="polite"></p>
</form>
<p><a href="{{ data.forgot_password_url|e('html_attr') }}">Forgot password?</a></p>
```

```js
form.addEventListener('submit', async function (event) {
    event.preventDefault();
    const button = form.querySelector('[type="submit"]');
    button.disabled = true;
    try {
        await window.dbEvent.user.login({
            login_type: 'email',
            email: form.email.value.trim(),
            password: form.password.value,
            remember: form.remember.checked
        });
        window.location.assign('{{ data.return_url|e('js') }}');
    } catch (error) {
        status.textContent = (error && error.message) || 'Unable to sign you in.';
    } finally {
        button.disabled = false;
    }
});
```

Rules:

1. Keep field names exactly `email`, `password`, `remember`. The API reads these names; renaming breaks sign-in while the form looks fine.
2. Send `login_type: 'email'` explicitly. The default happens to match, but explicit beats ambient when OTP tabs share the panel.
3. Redirect to `data.return_url` on success. It is already same-host validated; never substitute a hardcoded destination or an unvalidated `return` value.
4. Report failures in the `role="status"` region and re-enable the button. Never reveal whether the email or the password was wrong — one generic message for both.
5. Set `autocomplete="email"` and `autocomplete="current-password"`. Password managers depend on them.

## 5. The OTP tabs

Email OTP and phone OTP share one request-then-verify shape with the password panel. Gate each tab on its flag and keep the verify step shared:

```twig
<div role="tablist" aria-label="Sign in method">
    <button type="button" role="tab" aria-selected="true" data-auth-tab="password">Password</button>
    {% if data.otp_enabled %}
        <button type="button" role="tab" aria-selected="false" data-auth-tab="otp">Email OTP</button>
    {% endif %}
    {% if data.phone_otp_enabled %}
        <button type="button" role="tab" aria-selected="false" data-auth-tab="phone_otp">Phone OTP</button>
    {% endif %}
</div>
```

```js
// Request: no code
await window.dbEvent.user.login({
    login_type: 'email_otp',
    email: email
});

// Verify: same address plus code
await window.dbEvent.user.login({
    login_type: 'email_otp',
    email: email,
    code: code
});

// Phone: split fields, repeated unchanged on verify
await window.dbEvent.user.login({
    login_type: 'phone_otp',
    country_code: '+91',
    phone: '9876543210'
});
await window.dbEvent.user.login({
    login_type: 'phone_otp',
    country_code: '+91',
    phone: '9876543210',
    code: '123456'
});
```

Rules:

1. Render each OTP tab only under its flag. Advertising a disabled method invites requests the backend refuses.
2. Call `login` twice per OTP flow with identical identifiers: request without `code`, verify with `code` (the `otp` alias converts automatically). Changed addresses or digits between steps fail verification.
3. Keep `country_code` and `phone` separate; repeat both unchanged on verify. Never merge them into one input.
4. Offer resend by repeating the request call, rate-limited client-side. Show the sent-to address and a way back to correct it before verifying.
5. Every call uses `try/catch` because failures reject. Expired codes surface beside the code input with the request step one step back — never a dead end.

## 6. Social providers and account links

```twig
{% if data.social_providers|default([]) is not empty %}
    {% for provider in data.social_providers %}
        <a class="btn btn-outline-secondary" href="{{ provider.url|e('html_attr') }}" rel="nofollow">{{ provider.label|e }}</a>
    {% endfor %}
{% endif %}
{% if data.registration_enabled %}
    <p>New here? <a href="{{ data.register_url|e('html_attr') }}">Create an account</a></p>
{% endif %}
```

1. Render provider buttons only when the list is non-empty, linking the supplied URLs with `rel="nofollow"`. Never construct provider URLs by hand.
2. Show the registration link only when `registration_enabled` is true.
3. Logged-in visitors get the account shortcut (`site_url('user/dashboard')`), never the forms.

## 7. Use cases

**Standalone login page.** Full tabbed panel with providers and both account links. The session front door.

**Checkout sign-in.** Compact password-first panel with return set to the checkout page, so buyers land back in their order.

**Modal login.** Card without page chrome inside a dialog for header "Sign in" buttons. Success follows `return_url`; failures stay in the modal status region.

**OTP-first markets.** Phone OTP as the default tab where password use is rare. Tab order follows the audience; password stays available.

## 8. Common mistakes

- Renaming `email`, `password`, or `remember` while styling the form.
- Omitting `login_type`, letting OTP-tab state leak into password submits.
- Redirecting to unvalidated destinations instead of `data.return_url`.
- Revealing whether the email or password failed. One generic message for both.
- Rendering OTP tabs their flags disable, inviting refused requests.
- Merging `country_code` and `phone`, or changing identifiers between request and verify.
- Missing `autocomplete` values, breaking password managers and mobile keyboards.
- Showing forms to logged-in visitors instead of the account shortcut.
- Forgetting `default.dwig`, so a missing skin selection breaks sign-in site-wide.
