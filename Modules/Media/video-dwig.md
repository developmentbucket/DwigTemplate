# Media/Video — Embedded Players and Uploaded Video

The `Media/Video` module renders one video player: a platform embed (YouTube, Vimeo, and similar), an uploaded video file, or a click-to-play thumbnail. Its skins live in `Templates/Modules/Media/Video/`. Unlike most modules, the backend builds the complete player HTML and hands it to the skin as one trusted fragment; the skin decides the frame around it — aspect-ratio box, width, and lazy-load behavior.

## 1. Embedding a video

```twig
<module
    type="Media/Video"
    id="product-video"
    template="default.dwig"
/>
```

- `type` is `Media/Video`: the path-style name that mirrors the skin path `Templates/Modules/Media/Video/`. The bare legacy name still resolves, but every example in these docs uses the path-style form.
- `id` identifies this instance and connects it to its video source and player settings (embed URL or upload, autoplay, loop, controls, thumbnail, dimensions). Keep it stable and unique per player. Changing it orphans the video configured under the old ID.
- `template` selects the skin filename from the video directory. Pass only the filename, never a path. A template chosen in Live Edit is saved against the instance and takes precedence over the inline attribute.
- An unavailable file falls back to `default.dwig`, so the directory keeps a working `default.dwig`.

Source and playback attributes (each fills a gap only when no saved setting exists):

| Attribute | Purpose |
|---|---|
| `url` | Embed URL or code used when neither a saved embed URL nor a saved upload exists. Handy for hardcoded demo players. |
| `width` / `height` | Player dimensions. Defaults are `100%` and `350px`; the skin's ratio box normally governs display instead. |
| `autoplay` | Starts playback automatically when no saved autoplay setting exists. Browsers block audible autoplay, so pair it with the muted player setting. |

## 2. Where skins live and how they resolve

```text
Templates/Modules/Media/Video/
+-- default.dwig     # fallback, keep it working
+-- wide.dwig
+-- compact.dwig
+-- ...
```

Resolution order for every skin:

1. The same relative path in the active theme.
2. The bundled default template.
3. `default.dwig` in the same directory.

Keep theme-specific video skins in the active theme. Changing the bundled fallback changes the default for every generated theme.

## 3. What data a video skin receives

| Value | Contents |
|---|---|
| `data.code` | Complete ready-to-render player HTML built by the backend (embed iframe or uploaded `<video>` with its thumbnail and settings applied). Trusted server-rendered markup: output it with `\|raw`, never escaped. |
| `data.provider` | Embed provider name for the current source. Useful for per-provider frame tweaks. |
| `data.lazyload` | `true` when click-to-play is enabled for this instance. Gate the lazy-load script on it (section 5). |
| `data.params` | Tag attributes plus the instance ID (`data.params.id`). |

Which source renders follows a fixed precedence: saved embed URL first, saved upload second, `url` tag attribute last. An upload with no embed URL forces the uploaded player. With neither source configured, the module shows an editor notice instead of a player, so the skin needs no empty-state branch for that case — but keep the wrapper harmless when `data.code` is empty:

```twig
<div class="ratio ratio-16x9 mwembed-video">
    {{ data.code|raw }}
</div>
```

Rules:

1. `{{ data.code|raw }}` is the one place where `|raw` is correct. The fragment is player markup your own backend built, not editor free text. Never apply `|e` to it; escaping prints the player as visible text.
2. Never rebuild the player by hand (no hardcoded `<iframe>` or `<video>` tags). Source handling, provider quirks, thumbnails, loop, controls, and muted flags all live inside `data.code`; hand-rolled markup silently drops them.
3. Never render `data.code` on any site but your own or insert player HTML from anywhere else with `|raw`. Treat only this backend-supplied fragment as trusted.

## 4. Framing the player — ratio box and width

The skin's main job is the frame. A Bootstrap ratio box keeps the player responsive at any width:

```twig
{# Widescreen embed #}
<div class="ratio ratio-16x9 mwembed-video">
    {{ data.code|raw }}
</div>
```

```twig
{# Square or portrait social clip #}
<div class="ratio ratio-1x1 mwembed-video">
    {{ data.code|raw }}
</div>
```

```twig
{# Constrained width, centered #}
<div class="mwembed-video" style="max-width: 100%; margin: 0 auto;">
    {{ data.code|raw }}
</div>
```

Rules:

1. Prefer a ratio box (`ratio-16x9` for embeds, `ratio-4x3` or `ratio-1x1` for square and portrait clips) over fixed pixel dimensions. Fixed heights overflow on small screens.
2. Match the ratio to the source: a 16x9 box letterboxes a square clip and crops nothing, but wastes space. Offer one skin per ratio instead of one skin with conditionals.
3. Keep the `mwembed-video` class on the frame. Theme CSS and editor tooling address the player through it.

## 5. Use cases

**Product video.** `ratio-16x9` frame with lazy loading beside the product gallery. Independent `product-video` instance ID so each product page configures its own clip.

**Hero explainer.** Full-width 16x9 embed above the fold with click-to-play, so the stream never delays the page's first paint.

**Testimonial or interview.** 16x9 or 4x3 embed in a two-column section: video one side, quote text in the layout's editable region on the other.

**Tutorial or course lesson.** Constrained-width centered frame (`max-width` plus auto margins) for lesson content. Same player, calmer measure.

**Portrait social clip.** `ratio-1x1` or `ratio-9x16` skin for vertical clips. Separate skin per ratio rather than conditionals in one file.

**Modal teaser.** Thumbnail-style frame that editors pair with a lightbox: the skin keeps the ratio box and lazy-load script; the surrounding layout supplies the modal shell.

**Full-bleed background video.** Does not belong here. Ambient autoplaying backgrounds are the `video_background` module's job; this module always renders a framed, controllable player.

## 7. Common mistakes

- Escaping `data.code` with `|e`, which prints the player as visible text instead of rendering it.
- Hand-writing `<iframe>` or `<video>` tags instead of outputting `data.code`, silently dropping provider handling, thumbnails, loop, controls, and muted flags.
- Using a bare class selector in a custom lazy-load script, so one click starts every player on the page.
- Fixed pixel dimensions instead of a ratio box, overflowing small screens.
- Expecting autoplay with sound. Browsers block it; audible autoplay needs the muted player setting, configured on the instance, not in the skin.
- Adding an empty-state branch for a missing source. The module shows its own editor notice; keep the frame harmless when `data.code` is empty instead.
- Forgetting `default.dwig`, so a missing skin selection breaks video site-wide.

