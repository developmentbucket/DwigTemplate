# DWIG Templates — Introduction Guide

This guide explains how to use `.dwig` template files to build a custom website.

## 1. What you are building

A custom website is a layout shell plus page templates plus reusable module skins plus theme assets. DWIG files control presentation only. The backend loads the data, and the `.dwig` file decides the HTML.

## 2. What is a `.dwig` file

A `.dwig` file is a Theme Studio template. It uses Twig syntax. Three rules apply to every file.

1. It controls presentation only, never data loading or settings saving.
2. Auto-escaping is off, so escape output yourself with `|e` for text and `|e('html_attr')` for attributes. Use `|raw` only for trusted HTML.
3. Use safe defaults for anything optional, for example `data.posts|default([])`.
4. All frontend JavaScript goes through `window.dbEvent` from `userfiles/modules/developmentbucket/db_lib/events/common.js`. Never reference the `mw` JS library.

## 3. The five template families

Every template lives under the theme's `Templates/` root.

- `Templates/Layouts/` holds the shared document shell, such as `main.dwig`.
- `Templates/Website/` holds full pages: `Page/`, `Post/`, `Service/`, and `PostCategory/`.
- `Templates/Store/` holds shop pages, search fragments, invoices, and order emails.
- `Templates/Modules/` holds reusable skins for one module, such as menus, sliders, and carts.
- `Templates/Errors/` holds `404.dwig`, `500.dwig`, and `maintenance.dwig`.

Full pages extend the layout. Fragments and module skins do not.

```twig
{% extends "Layouts/main.dwig" %}
{% block content %}...{% endblock %}
```

## 4. Naming and fallback rules

1. Every renderable file must end in `.dwig`.
2. `default.dwig` is the fallback in each directory, so keep it working.
3. Extra skins sit beside it, for example `landing-page.dwig` or `compact.dwig`.
4. Save only the filename, never a full path. An empty or missing value falls back to `default.dwig`.

## 5. Bundled defaults versus your theme

Theme Studio looks in the active theme first and uses the bundled defaults when the file is not there. Customize by copying the same relative path into your theme, for example `Templates/Modules/Navigation/Menu/default.dwig`. Do not edit the bundled fallback for a single site's design.

## 6. How pages are composed

```text
Layouts/main.dwig -> Website/Page/default.dwig -> <module type="..." template="....dwig" />
```

Pages embed behavior with `<module>` tags.

```twig
<module
  type="Navigation/Menu"
  id="main-navigation"
  template="default.dwig"
/>
```

The `type` selects the module, `template` selects the skin, and `id` must be stable and unique per configurable instance because settings are stored against it.

## 7. Data rules

There is no single data contract for all templates. Each module passes its own `data.*` values, so check that module's doc and its `default.dwig` before writing a skin. Common page values include `data.content`, `data.content_data`, `data.post`, `data.product`, `data.posts`, `data.products`, and `data.category`. Never hardcode URLs or currency. Use resolved `link` and `url` values with `currency_format()`, and load theme files with `assets()`.

