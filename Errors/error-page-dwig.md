# Templates/Errors — Error and Maintenance Pages

The `Templates/Errors/` directory holds the failure pages: 404 (not found), 403 (denied), 500 (server error), and maintenance (planned downtime). Each file is a complete page template selected by status, not an embedded module: no `<module>` tag places them, and they receive no instance data. Every error page extends the site layout, fills the content block with one centered card, and offers the way back.

## 1. The four templates

```text
Templates/Errors/
+-- 404.dwig           # page not found
+-- 403.dwig           # access denied
+-- 500.dwig           # something went wrong
+-- maintenance.dwig   # planned downtime, no home button
```

Resolution is theme-first, bundled-default-second per filename. Keep theme-specific error pages in the active theme. The bundled files ship as empty stubs, so writing these four pages is part of building the theme — a missing error template leaves failures unbranded or blank.

## 2. Shared anatomy

Every error page follows the same shape. Extend the site layout, fill `content`, center one card with code, heading, message, and (except maintenance) a home button:

```twig
{% extends "Layouts/main.dwig" %}
{% block content %}
<main class="container py-5 text-center">
    <div class="card border-0 shadow-sm mx-auto" style="max-width: 680px;">
        <div class="card-body p-5">
            <div class="display-1 fw-bold text-primary">404</div>
            <h1 class="h2">Page not found</h1>
            <p class="text-body-secondary mb-4">The page may have moved or no longer exists.</p>
            <a class="btn btn-primary" href="{{ site_url() }}">Return home</a>
        </div>
    </div>
</main>
{% endblock %}
```

Rules for all four:

1. Extend `Layouts/main.dwig` and fill only the `content` block. Error pages wear the site chrome (header, footer, styles); only the message changes.
2. Keep one centered card (`max-width: 680px`) with the status code, one plain-language heading, one explanatory line, and one way back. Error pages are read under stress; density is the enemy.
3. Link home with `href="{{ site_url() }}"`. Never hardcode the domain, and never link back to the failing page.
4. Keep error pages dependency-light. No forms, no carts, no scripts beyond the layout's own. A 500 page whose widgets also fail doubles the outage; a maintenance page must assume backends may be down.
5. Static copy only. Error templates receive no instance data — no `data.*` contract, no settings, no editable regions. Write the message in markup, in words a non-technical visitor understands.

## 3. Per-template copy and behavior

| File | When | Code | Heading | Message | Button |
|---|---|---|---|---|---|
| `404.dwig` | Unknown URL | 404 | Page not found | The page may have moved or no longer exists. | Return home |
| `403.dwig` | Permission refused | 403 | Access denied | You do not have permission to view this page. | Return home |
| `500.dwig` | Server failure | 500 | Something went wrong | Please try again in a moment. | Return home |
| `maintenance.dwig` | Planned downtime | None (icon) | We'll be back shortly | Short, brand-voiced reason plus "check again soon." | None |

Rules per template:

1. 404 copy blames the move, never the visitor. No "you typed it wrong."
2. 403 copy states the refusal plainly without explaining the permission model. Never leak which rule, role, or record denied access.
3. 500 copy says nothing technical. No exception text, no trace IDs in markup, no retry loops — one line and the way back.
4. Maintenance carries no home button: home may be down too. A decorative icon plus the return-soon line is the whole page. Keep it brand-voiced; this is the only error page allowed personality.

## 4. Use cases

**Redesigned 404.** On-brand card with search box beside the home button, so lost visitors can recover by keyword as well as by navigation.

**Login-gated 403.** Access-denied card with a sign-in button next to home, for pages that exist behind accounts. Same template, second action.

**Quiet 500.** Minimal card with no imagery beyond type, for failing fast under load. The lighter the page, the likelier it renders during the outage.

**Branded maintenance.** Icon, warm headline, and short reason with no navigation at all. Temporary file, removed when the window ends.

## 5. Common mistakes

- Forgetting one of the four files, leaving that failure unbranded. All four ship as stubs; all four need writing.
- Technical detail on 500 pages: traces, codes, and internals that alarm visitors and aid attackers.
- Blaming the visitor on 404 pages, or explaining the permission model on 403 pages.
- A home button on maintenance pages that may itself be down.
- Heavy widgets (forms, carts, scripts) inside error pages that fail along with everything else.
- Hardcoded home URLs instead of `site_url()`.
- Expecting instance data, settings, or editable regions. Error templates are static documents by design.
