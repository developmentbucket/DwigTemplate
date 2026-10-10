# Content/FAQ — Frequently Asked Questions

The `Content/FAQ` module renders a question-and-answer list: one collapsible row per question with its answer inside. Its skins live in `Templates/Modules/Content/FAQ/`. Editors define the questions and answers in the module settings; the backend passes the ready list; the skin renders collapsible Q&A rows.

## 1. Embedding FAQs

```twig
<module
    type="Content/FAQ"
    id="support-faq"
    template="default.dwig"
/>
```

- `type` is `Content/FAQ`: the path-style name that mirrors the skin path `Templates/Modules/Content/FAQ/`. The bare legacy name still resolves, but every example in these docs uses the path-style form.
- `id` identifies this instance and connects it to its questions and answers. Support page, product page, and checkout-helper placements use different IDs so each keeps its own list. Changing an ID orphans the Q&A saved under the old ID.
- `template` selects the skin filename from the FAQ directory. Pass only the filename, never a path. A template chosen in Live Edit is saved against the instance and takes precedence over the inline attribute.
- An unavailable file falls back to `default.dwig`, so the directory keeps a working `default.dwig`.
- With no questions defined, the module shows an editor notice instead of the list and renders nothing. Keep an `{% if %}` guard so the live page never shows an empty frame.

## 2. Where skins live and how they resolve

```text
Templates/Modules/Content/FAQ/
+-- default.dwig     # fallback, keep it working
+-- compact.dwig
+-- ...
```

Resolution order for every skin:

1. The same relative path in the active theme.
2. The bundled default template.
3. `default.dwig` in the same directory.

Keep theme-specific FAQ skins in the active theme. The bundled skins ship as empty stubs, so writing the FAQ skins is part of building the theme — follow the Q&A pattern in section 4 so the first skin you write already behaves like the module's native output.

## 3. What data an FAQ skin receives

| Value | Contents |
|---|---|
| `data.faq_items` | Questions in order. Each carries `question` and `answer`. Either can be empty — guard both. |
| `data.settings` | Raw Q&A-settings JSON for the instance. Informational for most skins; the parsed list above is what skins render. |
| `data.params` | Tag attributes plus the instance ID (`data.params.id`). |

## 4. The Q&A pattern

One collapsible row per question: question as the toggle button, answer in the pane, first row open:

```twig
{% set items = data.faq_items|default([]) %}
{% set instance = data.params.id|default('faq') %}

{% if items is not empty %}
<div class="faq-list">
    <h2>Frequently asked questions</h2>
    <div class="accordion" id="faq-accordion-{{ instance|e('html_attr') }}">
    {% for item in items %}
        {% set row_id = 'faq-item-' ~ instance ~ '-' ~ loop.index %}
        <div class="accordion-item">
            <h3 class="accordion-header" id="{{ row_id|e('html_attr') }}-header">
                <button class="accordion-button{% if not loop.first %} collapsed{% endif %}" type="button"
                        data-bs-toggle="collapse" data-bs-target="#{{ row_id|e('html_attr') }}-pane"
                        aria-expanded="{{ loop.first ? 'true' : 'false' }}"
                        aria-controls="{{ row_id|e('html_attr') }}-pane">
                    {{ item.question|default('Question')|e }}
                </button>
            </h3>
            <div id="{{ row_id|e('html_attr') }}-pane"
                 class="accordion-collapse collapse{% if loop.first %} show{% endif %}"
                 aria-labelledby="{{ row_id|e('html_attr') }}-header"
                 data-bs-parent="#faq-accordion-{{ instance|e('html_attr') }}">
                <div class="accordion-body">
                    {% if item.answer|default('') %}
                        <p>{{ item.answer|e }}</p>
                    {% else %}
                        <p class="text-muted">Answer coming soon.</p>
                    {% endif %}
                </div>
            </div>
        </div>
    {% endfor %}
    </div>
</div>
{% endif %}
```

Rules:

1. Derive row, header, and pane IDs from the instance ID plus the loop index. Two FAQ blocks on one page must never share a collapse identity.
2. Open exactly one row (`loop.first`): button expanded without `collapsed`, pane with `show`. Zero or two open rows both break the widget.
3. Pair every control explicitly: `data-bs-target` and `aria-controls` equal the pane `id`; pane `aria-labelledby` equals the header `id`; panes share one `data-bs-parent` scoped to the instance accordion.
4. Guard `answer` with a fallback line. Unanswered questions ship as "Answer coming soon," never as empty panes.
5. Escape questions and answers. FAQ copy is editor input; `|raw` has no place in an FAQ skin.
6. Guard the whole list with `{% if items is not empty %}`. An empty FAQ frame with one dead row is worse than nothing.

## 5. Use cases

**Support FAQ page.** Full question list with the first answer open and a contact CTA beneath. The canonical surface.

**Product Q&A.** Shipping, sizing, and returns questions under the buy box. Per-product-page instance IDs.

**Checkout helper.** Compact rows beside the order form answering payment and delivery questions. Same markup, tighter CSS.

**Policy Q&A.** Returns and warranty questions on policy pages. Short list, no CTA, plain styling.

**Onboarding answers.** New-customer questions on welcome pages. First answer open to invite scrolling.

## 6. Common mistakes

- Shared collapse IDs across two FAQ blocks, expanding the wrong answer.
- Opening zero or multiple rows, or forgetting `data-bs-parent` so rows stop behaving as one list.
- Empty answer panes instead of the coming-soon fallback.
- Unescaped questions or answers.
- Rendering the list frame with no questions instead of guarding with `{% if %}`.
- Using FAQ rows for tabbed parallel content (`Content/Tabs`) or free collapsibles with rich bodies (`Content/Accordion`).
- Forgetting `default.dwig`, so a missing skin selection breaks FAQ lists site-wide.
