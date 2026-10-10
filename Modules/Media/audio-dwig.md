# Media/Audio — Audio Players and Clips

The `Media/Audio` module renders one audio player: an uploaded audio file or a remote audio URL behind native browser controls. Its skins live in `Templates/Modules/Media/Audio/`. The module's source is a single audio file configured per instance; the skin decides the frame around the player — title, width, and surrounding markup.

## 1. Embedding audio

```twig
<module
    type="Media/Audio"
    id="podcast-episode-12"
    template="default.dwig"
/>
```

- `type` is `Media/Audio`: the path-style name that mirrors the skin path `Templates/Modules/Media/Audio/`. The bare legacy name still resolves, but every example in these docs uses the path-style form.
- `id` identifies this instance and connects it to its audio source. Keep it stable and unique per player. Changing it orphans the file or URL configured under the old ID.
- `template` selects the skin filename from the audio directory. Pass only the filename, never a path. A template chosen in Live Edit is saved against the instance and takes precedence over the inline attribute.
- An unavailable file falls back to `default.dwig`, so the directory keeps a working `default.dwig`.

## 2. Where skins live and how they resolve

```text
Templates/Modules/Media/Audio/
+-- default.dwig     # fallback, keep it working
+-- compact.dwig
+-- ...
```

Resolution order for every skin:

1. The same relative path in the active theme.
2. The bundled default template.
3. `default.dwig` in the same directory.

Keep theme-specific audio skins in the active theme. The bundled skins ship as empty stubs, so writing the audio skins is part of building the theme — follow the player pattern in section 4 so the first skin you write already behaves like the module's native output.

## 3. Configuring the source

One instance plays one audio file, resolved in this order:

1. Saved upload (`data-audio-upload`): the file editors upload to the instance in Live Edit. Wins whenever present.
2. Saved URL (`data-audio-url`): a remote audio file configured on the instance.
3. `data-audio-url` tag attribute: an inline fallback used when neither saved value exists. Handy for hardcoded demo players:

```twig
<module type="Media/Audio"
        id="demo-clip"
        data-audio-url="https://example.com/sample.mp3"
        template="default.dwig" />
```

Rules:

1. Prefer the saved upload or URL over the tag attribute. Attribute sources are frozen in the template; editors cannot change them without editing markup.
2. Serve audio over `https` and from the same truck as the page where possible. Mixed-content and cross-host audio triggers browser warnings and extra latency.
3. With no source configured anywhere, the module shows an editor notice instead of a player. The skin still guards its output (section 4) so the live page never emits an empty `<audio>` tag.

## 4. The player pattern

The player is a native `<audio>` element with controls, inside the module's frame classes. Every audio skin follows this shape:

```twig
{% set source = data.params['data-audio-url']|default('') %}

<div class="mwembed mw-audio" id="mwaudio-{{ data.params.id|default('player')|e('html_attr') }}">
    {% if source %}
        <audio controls src="{{ source|e('html_attr') }}"></audio>
    {% else %}
        <p class="text-body-secondary">Upload an audio file or paste a URL to enable this player.</p>
    {% endif %}
</div>
```

Rules:

1. Always render native `controls`. A player without controls strands keyboard, screen-reader, and mobile users — and most browsers will not autoplay audible audio anyway.
2. Guard the `<audio>` tag with `{% if source %}`. An empty `src` is worse than a placeholder message.
3. Keep the `mwembed mw-audio` classes on the frame. Theme CSS and editor tooling address the player through them.
4. Scope the frame ID with the instance ID (`mwaudio-{{ data.params.id }}`), never a hardcoded value, so two players on one page stay distinct.
5. Escape the source with `|e('html_attr')`. It comes from editor input or tag attributes.
6. Never use `|raw` in an audio skin. There is no trusted HTML fragment here — only a URL string going into one attribute.

## 5. Styling the player

The native element fills its container; the skin controls the container:

```twig
<div class="mwembed mw-audio audio-player" style="max-width: 100%;">
    <audio controls src="{{ source|e('html_attr') }}" style="width: 100%;"></audio>
</div>
```

1. Set the `<audio>` element to `width: 100%` so it follows its column on small screens.
2. Tint native controls with `accent-color` on the element where the design needs brand color. Full visual reskins require a scripted custom player; keep the native controls unless the theme ships one, and keep its hooks intact if it does.
3. Put titles, durations, and transcripts outside the `<audio>` tag as normal markup. Transcripts are escaped text (`|e`); they are also the accessible alternative visitors actually search.

## 6. Use cases

**Podcast episode.** Player plus episode title and a transcript below in the layout's editable region. One instance ID per episode so each keeps its own file.

**Music or voice sample.** Compact skin: player only, constrained width, on product or portfolio pages. Same pattern, tighter frame.

**Course or lesson audio.** Player above the lesson text with a transcript beneath. Native controls give students scrub, speed, and volume for free.

**Testimonial clip.** Short voice note beside a written quote. Keep the frame minimal; the surrounding layout carries the attribution.

**Museum or tour guide clip.** One player per stop on a guide page, each with its own stable ID. Never reuse one ID across stops; shared IDs share the configured source.

**Demo placeholder.** `data-audio-url` attribute with a sample file on a staging page. Replace with the saved upload before launch; attribute sources cannot be edited in Live Edit.

## 7. Common mistakes

- Omitting `controls`, producing a silent unplayable box.
- Rendering `<audio>` with an empty `src` instead of guarding with `{% if %}`.
- Hardcoding the frame ID, so two players on one page collide.
- Dropping the `mwembed mw-audio` frame classes while skinning.
- Expecting autoplay with sound. Browsers block it; audio starts on visitor action.
- Hardcoding the audio URL in the skin instead of using the instance source with a `data-audio-url` fallback, freezing out editor configuration.
- Using `|raw` anywhere in an audio skin. URLs are escaped attribute values, not HTML.
- Reaching for audio when the content is video with sound. Talking-head clips belong in `Media/Video`, not in a player with no picture.
- Forgetting `default.dwig`, so a missing skin selection breaks audio site-wide.
