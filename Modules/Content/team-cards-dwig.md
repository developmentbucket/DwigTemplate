# Content/TeamCard — Team Members and People Cards

The `Content/TeamCard` module renders people cards: photo, name, role, bio, and link. Its skins live in `Templates/Modules/Content/TeamCard/`. Editors define the people in the module settings; the backend backfills missing keys and passes the ready list; the skin renders one card per person.

## 1. Embedding team cards

```twig
<module
    type="Content/TeamCard"
    id="about-team"
    template="default.dwig"
/>
```

- `type` is `Content/TeamCard`: the path-style name that mirrors the skin path `Templates/Modules/Content/TeamCard/`. The bare legacy name still resolves, but every example in these docs uses the path-style form.
- `id` identifies this instance and connects it to its people list. About page, careers page, and author boxes use different IDs so each keeps its own roster. Changing an ID orphans the people saved under the old ID.
- `template` selects the skin filename from the team-card directory. Pass only the filename, never a path. A template chosen in Live Edit is saved against the instance and takes precedence over the inline attribute.
- An unavailable file falls back to `default.dwig`, so the directory keeps a working `default.dwig`.
- With no people defined, the module shows an editor notice instead of cards. Keep an `{% if %}` guard so the live page never renders an empty grid.

## 2. Where skins live and how they resolve

```text
Templates/Modules/Content/TeamCard/
+-- default.dwig     # fallback, keep it working
+-- slider.dwig
+-- compact.dwig
+-- ...
```

Resolution order for every skin:

1. The same relative path in the active theme.
2. The bundled default template.
3. `default.dwig` in the same directory.

Keep theme-specific people skins in the active theme. The bundled skins ship as empty stubs, so writing the team skins is part of building the theme — follow the card pattern in section 4 so the first skin you write already behaves like the module's native output.

## 3. What data a team-card skin receives

| Value | Contents |
|---|---|
| `data.teamcard_items` | People list in order. Each item carries `name`, `role`, `bio`, `website`, `file` (photo source), and `itemId`. Missing keys arrive backfilled with safe defaults, but photo and website can still be empty — guard both. |
| `data.settings` | Raw people-settings JSON for the instance. Informational for most skins; the parsed list above is what skins render. |
| `data.params` | Tag attributes plus the instance ID (`data.params.id`). |

## 4. The person card loop

One card per person: guarded photo through `thumbnail()`, escaped name and role, guarded website link, escaped bio:

```twig
<div class="row row-cols-1 row-cols-sm-2 row-cols-lg-4 g-4">
{% for person in data.teamcard_items|default([]) %}
    <article class="col">
        <div class="team-card h-100 text-center">
            {% if person.file|default('') %}
                <img class="rounded-circle mx-auto"
                     src="{{ thumbnail(person.file, 400, 400, true)|e('html_attr') }}"
                     alt="{{ person.name|default('Team member')|e('html_attr') }}"
                     width="160" height="160" loading="lazy">
            {% else %}
                <span class="team-card-placeholder rounded-circle mx-auto" aria-hidden="true">
                    {{ person.name|default('?')|slice(0, 1)|e }}
                </span>
            {% endif %}
            <h3>{{ person.name|default('Name')|e }}</h3>
            <p class="text-body-secondary">{{ person.role|default('')|e }}</p>
            {% if person.website|default('') %}
                <a class="d-block mb-2" href="{{ person.website|e('html_attr') }}"
                   target="_blank" rel="noopener">{{ person.website|e }}</a>
            {% endif %}
            {% if person.bio|default('') %}
                <p>{{ person.bio|e }}</p>
            {% endif %}
        </div>
    </article>
{% else %}
    <div class="col-12"><p class="text-muted">Team members coming soon.</p></div>
{% endfor %}
</div>
```

Rules:

1. Guard `file` with `{% if %}` and draw an initial-based placeholder in `{% else %}`. Members without photos are normal; the grid must hold alignment regardless.
2. Pass photos through `thumbnail()` at the displayed size with crop for uniform avatars. Never output the original upload.
3. Guard `website` with `{% if %}` and open it with `target="_blank" rel="noopener"`. Personal sites are off-site by definition.
4. Guard `bio` with `{% if %}`. Bios are optional per person; empty paragraphs collapse card rhythm unevenly.
5. Escape names, roles, bios, and attributes. People data is editor input; `|raw` has no place in a team-card skin.
6. Loop with `|default([])` and an `{% else %}` branch. New instances have no people; the branch is the normal state, not an error.

## 5. Use cases

**About-page team grid.** Photo, name, role, bio, website per member in a responsive grid. The canonical team surface.

**Leadership spotlight.** Single featured card (first item, larger photo and bio) above the grid. Same data, spotlight markup for item one.

**Author boxes.** Compact photo-plus-name skins under articles, one instance per author roster. Per-instance IDs keep rosters independent.

**Careers culture row.** Photos and first names only, no bios, linking to open roles. Same list, minimal card.

**Advisors or partners strip.** Logo-style row of names and roles without photos. Text-first skin for external people.

## 6. Common mistakes

- Rendering `file` unguarded, breaking the grid on photo-less members.
- Outputting original uploads instead of `thumbnail()` sizes.
- Rendering empty `website` links or opening them without `rel="noopener"`.
- Empty bio paragraphs collapsing card rhythm instead of guarded blocks.
- Unescaped names, roles, or bios.
- Sharing one instance ID across rosters that need different people, merging two teams into one settings scope.
- Using this module for testimonials (`Content/Testimonials`) or short quotes — people cards carry roles and bios, not praise.
- Forgetting `default.dwig`, so a missing skin selection breaks people surfaces site-wide.
