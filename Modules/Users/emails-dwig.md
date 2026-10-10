# Users/Emails — Transactional Email Skins

The `Users/Emails` directory holds the user-account email skins: the registration welcome (with verification link) and the forgot-password message (with reset link). Its files live in `Templates/Modules/Users/Emails/`. Unlike every other module doc here, these skins have no `<module>` tag and no settings: they render as standalone email documents when the corresponding account event fires.

Know where email copy actually sends from today. Account emails send through the mail-template system, which substitutes `{placeholders}` (user fields plus `{verify_email_link}`) into the configured subject and message. The `.dwig` files below are the theme-owned skin slot for those messages: write them as complete, client-safe email documents following the rules in this file.

## 1. The email skins

```text
Templates/Modules/Users/Emails/
+-- register.dwig    # welcome + verify your email
+-- forgot.dwig      # reset your password
```

Resolution is theme-first, bundled-default-second, with per-filename fallback. There is no `default.dwig` fallback across different emails: each message is its own file, and a missing file means that message has no themed skin.

Rules for both skins:

1. Standalone documents. Never extend a layout (`Layouts/main.dwig`), include site partials, or assume page CSS. Email clients render this file alone.
2. Table-based layout with inline styles only. External stylesheets, `<style>`-block sophistication, and web fonts do not survive most inboxes.
3. Absolute URLs everywhere through `site_url()` and resolved links. Relative paths break the moment the message leaves the site.
4. No JavaScript, no forms, no interactive widgets. Scripts are stripped; the single link is the entire interaction.
5. Escape every value (`|e`, `|e('html_attr')`). Names and addresses are user input even inside email.

## 2. register.dwig — welcome and verify

Sent after sign-up when address verification is on. One job: confirm the account exists and get the link clicked.

```twig
<table role="presentation" width="100%" cellpadding="0" cellspacing="0">
<tr><td align="center">
<table role="presentation" width="600" cellpadding="24" cellspacing="0">
    <tr><td>
        <h1>Welcome, {{ data.name|default('friend')|e }}!</h1>
        <p>Your account at {{ data.site_name|default(site_url())|e }} is ready. Confirm your email address to finish setup:</p>
        <p><a href="{{ data.verify_link|default('#')|e('html_attr') }}"
              style="display:inline-block;padding:12px 24px;background:#0d6efd;color:#ffffff;text-decoration:none;">Verify email address</a></p>
        <p>If the button does not work, paste this link into your browser:<br>
        <span>{{ data.verify_link|default('')|e }}</span></p>
        <p>If you did not create this account, ignore this message.</p>
    </td></tr>
</table>
</td></tr>
</table>
```

Rules:

1. Lead with the human outcome (account ready), then exactly one CTA: the verification link, rendered both as a button and as pastable text for clients that break buttons.
2. Read the name, site, and link defensively with `|default()` fallbacks. A missing value must degrade to plain prose, never to an empty link.
3. Include the ignore-this-message line. Misdirected mail without it reads as phishing.
4. Never include a password, code, or anything beyond the link. The link is the credential.

## 3. forgot.dwig — reset link

Sent on password-reset request. One job: get the legitimate owner to the reset form, and nobody else.

```twig
<table role="presentation" width="100%" cellpadding="0" cellspacing="0">
<tr><td align="center">
<table role="presentation" width="600" cellpadding="24" cellspacing="0">
    <tr><td>
        <h1>Reset your password</h1>
        <p>Someone requested a password reset for {{ data.email|default('this address')|e }}. If that was you, choose a new password:</p>
        <p><a href="{{ data.reset_link|default('#')|e('html_attr') }}"
              style="display:inline-block;padding:12px 24px;background:#0d6efd;color:#ffffff;text-decoration:none;">Choose a new password</a></p>
        <p>This link expires soon and works once. If you did not ask for this, ignore this message — your password stays unchanged.</p>
    </td></tr>
</table>
</td></tr>
</table>
```

Rules:

1. Never state or imply the account is compromised. "If that was you / if not, ignore" covers both cases without alarming anyone.
2. State the expiry and single-use nature in plain words. Time pressure without explanation reads as phishing.
3. Never include the old password, a temporary password, or a bare code when a link serves. Links carry the authority; secrets in bodies leak through forwards.
4. Render nothing account-identifying beyond the address the request came with. No usernames, IDs, or order history in reset mail.

## 4. Use cases

**Welcome series first touch.** `register.dwig` with brand header, verify CTA, and support contact. Sent once per account.

**Verification reminder.** Same skin, same link, adjusted headline for re-sends. One skin serves both sends; the headline comes from message data, not a forked file.

**Reset request.** `forgot.dwig` with reset CTA and expiry note. Plain, fast, no marketing chrome.

**Reset confirmation.** Success notice after the password changes ("signed in everywhere else stays signed in; contact support if this wasn't you"). Same document family, follow-up message.

## 5. Common mistakes

- Extending the site layout or depending on theme CSS inside email skins.
- Relative image or link paths that die outside the browser.
- JavaScript, forms, or multi-step interactions that clients strip.
- Missing pastable-text fallback under CTA buttons.
- Passwords, codes, or personal data beyond the request address in message bodies.
- No ignore-this-message line on unsolicited-feeling mail.
- Unescaped names and addresses. Email values are user input throughout.
- Forking one skin per campaign instead of message-driven headlines in shared files.
