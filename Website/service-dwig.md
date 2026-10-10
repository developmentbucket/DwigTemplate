# Website/Service — Appointment Service Pages

The `Templates/Website/Service/` templates render individual appointment-service pages: service title, description, price, images, assigned staff, custom fields, and the booking module. `default.dwig` is the fallback every service resolves to unless the service selects another layout.

## 1. Service page shape

```twig
{% extends "Layouts/main.dwig" %}

{% block content %}
<main class="container py-5">
    <article>
        <h1>{{ data.content.title|default('Service')|e }}</h1>

        {% if data.content.description|default('') %}
            <div class="mb-4">{{ data.content.description|raw }}</div>
        {% endif %}

        <div class="edit" data-layout-container rel="content" field="content">
            <module type="appointments" service_id="{{ data.content.id }}" />
        </div>
    </article>
</main>
{% endblock %}
```

- Extends `Layouts/main.dwig` and fills only the `content` block. Page chrome comes from the layout; this file owns the service story.
- Carries the editable layout container (`edit` with `data-layout-container`, `rel="content"`, `field="content"`) so editors compose around the booking module.
- Embeds the appointments module with the current service ID. The booking flow belongs to the module; the page supplies context and surrounding copy.

## 2. Where service templates live and how they resolve

```text
Templates/Website/Service/
+-- default.dwig     # fallback, keep it working
+-- featured-service.dwig
+-- ...
```

Resolution: the filename saved as the service's `layout_file` wins; empty, `inherit`, invalid, or unavailable values fall back to `default.dwig`. Keep theme-specific service layouts in the active theme.

## 3. What data the service page receives

| Value | Contents |
|---|---|
| `data.content` | Current service record with `is_service` set to `true`: `id`, `title`, `description` (sanitized HTML, render with `\|raw`). |
| `data.content_data` | Extra service fields by key. Read every key with `\|default()` — custom keys differ per site. |
| `data.service` | Enriched service: `price`, `content_data`, `custom_fields`, `pictures`, `staff_ids`, and resolved `staff`. |

## 4. Rules

1. Pass `service_id="{{ data.content.id }}"` to the appointments module explicitly. The booking flow does not inherit the service from the surrounding page.
2. Render `description` with `|raw` as sanitized HTML, with a coming-soon fallback when empty. Titles, prices, and custom values stay escaped.
3. Show the price from `data.service.price` formatted for the shop locale. Never hand-format money or hardcode currency.
4. Render staff from resolved `data.service.staff` with photo, name, and role guards. Staff IDs alone render nothing — resolution happens before display.
5. Pass service images through `thumbnail()` at display size with guards. Imageless services hold layout like any other page.
6. Read `custom_fields` and `content_data` keys with `|default()` fallbacks. Service extras are per-site configuration, not a stable contract.
7. Escape everything except body HTML. Service copy and staff details are editor input throughout.

## 5. Use cases

**Standard service page.** Title, description, price, staff, and booking module. The default skin pattern.

**Featured service.** Hero image plus highlighted price and priority booking placement for flagship offerings. Same data, premium chrome.

**Staff-led service.** Practitioner profiles above the booking module for people-chosen services. Staff resolution carries the section.

**Multi-location service.** Branch list with per-branch booking context. Same module, location-scoped instances.

## 6. Common mistakes

- Omitting `service_id` on the appointments module, leaving booking unbound.
- Escaped description HTML printing tags as text, or `|raw` on titles, prices, and custom values.
- Hand-formatted prices or hardcoded currency.
- Rendering raw staff IDs instead of resolved staff records.
- Unthumbnailed originals or unguarded images breaking imageless services.
- Reading service extras without `|default()`, breaking services missing site-specific keys.
- Dropping the editable container, freezing editors out of page composition.
- Skipping `default.dwig`, so services with unset layouts break instead of falling back.
