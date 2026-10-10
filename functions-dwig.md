# Dwig Template Functions — Complete Reference

Every function callable inside a `.dwig` template, loaded by the theme view renderer. Only the names in this file exist: calling any other function name fails, because only registered names resolve. Auto-escaping is off in `.dwig` templates, so every example escapes with `|e` for text and `|e('html_attr')` for attributes.

Two implementation kinds sit behind one calling convention. Custom functions run dedicated code in the renderer; global functions forward to a same-named backend helper. Four registered names currently return `null` (no backend behind them): `get_variation`, `content_field`, `db_connect_action_url`, `db_request_url`. They are listed at the end so skins stop calling them.

## 1. Media

### `thumbnail(source, width = 200, height = null, crop = null)`

Resolves a thumbnail URL for an image. Omit `height` for proportional scaling; pass a truthy `crop` for cropped squares. A missing source may yield a placeholder URL.

```twig
<img src="{{ thumbnail(product.picture, 600, 600, true)|e('html_attr') }}"
     alt="{{ product.title|e('html_attr') }}">
{{ thumbnail(data.image, 1000)|e('html_attr') }}
```

### `get_media(parameters)`

Media records matching a query hash. Each item carries `filename` among other fields.

```twig
{% set media = get_media({ 'rel_type': 'content', 'rel_id': data.content.id }) %}
{% for item in media|default([]) %}
    <img src="{{ item.filename|e('html_attr') }}" alt="">
{% endfor %}
```

### `get_pictures(parameters)`

Picture records matching the query. Currently the same media-manager query as `get_media()`.

```twig
{% set pictures = get_pictures({ 'rel_id': data.content.id }) %}
{% for picture in pictures|default([]) %}
    <img src="{{ picture.filename|e('html_attr') }}" alt="">
{% endfor %}
```

### `get_picture(contentId, for = 'post', full = false)`

Primary picture for a content item, commonly the URL. Guard it — items without images return empty.

```twig
{% set picture = get_picture(data.content.id) %}
{% if picture %}
    <img src="{{ picture|e('html_attr') }}" alt="">
{% endif %}
```

## 2. Cart and products

### `get_cart(parameters = false)`

Cart lines matching an optional query string or hash.

```twig
{% set items = get_cart() %}
{% for item in items|default([]) %}
    <span>{{ item.title|e }} × {{ item.qty|default(1) }}</span>
{% endfor %}
```

### `cart_total()`

Current cart total from the cart manager. Format it for display.

```twig
<strong>{{ currency_format(cart_total()) }}</strong>
```

### `cart_sum(returnAmount = true)`

Cart sum: monetary sum with `true`, item/quantity sum with `false`.

```twig
<span>{{ cart_sum(false) }} items</span>
```

### `get_products(parameters = false)`

Content records restricted to products unless the query says otherwise. Accepts a query string or hash.

```twig
{% set products = get_products({ 'is_active': 1, 'limit': 8, 'order_by': 'created_at desc' }) %}
{% for product in products|default([]) %}
    <h2>{{ product.title|e }}</h2>
{% endfor %}
```

### `get_product_price(contentId = false)`

First configured price for a product, or `false` when none exists. Without an ID the helper uses the current content.

```twig
{{ currency_format(get_product_price(data.content.id)) }}
```

### `get_product_attributes(productId = null)`

Globally configured attributes with values selected for one product, keyed by attribute key. Each item holds `attribute_key`, the display label in `attribute`, and a `values` hash. Without an ID it tries the product, content, then page context.

```twig
{% for key, item in get_product_attributes(data.content.id) %}
    <dt>{{ item.attribute|e }}</dt>
    <dd>{{ item.values|join(', ')|e }}</dd>
{% endfor %}
```

### `get_product_attribute(attributeKey, productId = null)`

One selected product attribute, or `null` when not selected. Keys are generated configuration keys, not display names.

```twig
{% set color = get_product_attribute('attr_color', data.content.id) %}
{% if color %}
    {{ color.values|join(', ')|e }}
{% endif %}
```

### `get_attributes()`

Every globally configured product attribute with available values. Describes configuration; use `get_product_attributes()` for one product's selections.

```twig
{% for attributeKey, item in get_attributes() %}
    <h3>{{ item.attribute|e }}</h3>
{% endfor %}
```

### `get_related_products(contentId)`

Related records for a content ID. Currently the same related-content call as `get_related_contents()` without enforcing product type — filter `content_type` in the skin when the relationship may mix types.

```twig
{% for product in get_related_products(data.content.id)|default([]) %}
    {{ product.title|e }}
{% endfor %}
```

## 3. Menus and categories

### `get_menu(parameters = false)`

One menu record matching optional filters (the manager adds single-result limits).

```twig
{% set menu = get_menu({ 'name': 'header-menu' }) %}
{% if menu %}
    {{ menu.title|e }}
{% endif %}
```

Prefer the Navigation/Menu module for rendering the complete tree with behavior.

### `get_categories(parameters)`

Categories matching a query string or hash.

```twig
{% set categories = get_categories({ 'parent_id': 0 }) %}
{% for category in categories|default([]) %}
    {{ category.title|e }}
{% endfor %}
```

### `get_shop_categories()`

Categories recognized as shop categories. No arguments.

```twig
{% for category in get_shop_categories()|default([]) %}
    {{ category.title|e }}
{% endfor %}
```

### `get_service_categories()`

Active, non-deleted service-subtype categories ordered by position and title. Each item carries a resolved `url` and `picture`.

```twig
{% for category in get_service_categories()|default([]) %}
    <a href="{{ category.url|e('html_attr') }}">{{ category.title|e }}</a>
{% endfor %}
```

### `content_categories(contentId = false, dataType = 'categories')`

Categories related to a content item. Without an ID the helper tries the current content; returns `false` when neither resolves.

```twig
{% for category in content_categories(data.content.id)|default([]) %}
    {{ category.title|e }}
{% endfor %}
```

### `content_tags(contentId = false, returnFull = false)`

Tags for a content item: compact form by default, full records with `true`.

```twig
{% for tag in content_tags(data.content.id, true)|default([]) %}
    {{ tag.name|default(tag.title|default(''))|e }}
{% endfor %}
```

## 4. Content records and fields

### `get_page_id()` / `get_post_id()` / `get_content_id()`

All three currently return the `PAGE_ID` constant (`null` when undefined) through one shared registration. Prefer the prepared record in skins:

```twig
{% set pageId = get_page_id() %}
{% set postId = data.content.id %}
```

### `get_content_by_id(id = false)`

One content record by ID. The stored URL field can be a slug rather than a resolved link — prefer prepared `link` values from renderers.

```twig
{% set item = get_content_by_id(42) %}
{% if item %}
    <a href="{{ item.url|default('#')|e('html_attr') }}">{{ item.title|e }}</a>
{% endif %}
```

### `content_data(contentId, fieldName = false)`

Extra content-data fields: the whole collection without a name, one value with a name.

```twig
{% set sku = content_data(data.content.id, 'sku') %}
{{ sku|default('')|e }}
```

### `get_custom_fields(table, id = 0, returnFull = false, fieldFor = false, debug = false, fieldType = false, forSession = false)`

Global custom-field query. The first argument is the relation table/type the backend helper expects.

```twig
{% set fields = get_custom_fields('content', data.content.id, true) %}
{% for field in fields|default([]) %}
    <span>{{ field.name|e }}: {{ field.value|default('')|e }}</span>
{% endfor %}
```

### `prev_post(contentId = false)` / `next_post(contentId = false)`

Adjacent records of type `post` relative to the given ID. Either can return false or empty at the ends.

```twig
{% set previous = prev_post(data.content.id) %}
{% if previous %}
    <a href="{{ previous.link|default('#')|e('html_attr') }}">{{ previous.title|e }}</a>
{% endif %}
{% set next = next_post(data.content.id) %}
{% if next %}
    <a href="{{ next.link|default('#')|e('html_attr') }}">{{ next.title|e }}</a>
{% endif %}
```

### `get_related_contents(contentId)`

Content records related to the given ID.

```twig
{% for related in get_related_contents(data.content.id)|default([]) %}
    {{ related.title|e }}
{% endfor %}
```

## 5. Users

### `is_login()`

`true` with an authenticated user, `false` for guests.

```twig
{% if is_login() %}
    <module type="Users/Dashboard" />
{% else %}
    <module type="Users/Login" />
{% endif %}
```

### `get_user_by_id(parameters = false)`

User record through the user manager; a numeric ID is the common argument. Render only public-intended fields — never dump whole records into HTML.

```twig
{% set author = get_user_by_id(data.content.created_by) %}
{% if author %}
    {{ author.first_name|default(author.username)|e }}
{% endif %}
```

## 6. URLs and assets

### `site_url(path = false)`

Website base URL with the optional path appended. Pass relative paths without assuming domain-root hosting.

```twig
<a href="{{ site_url('contact')|e('html_attr') }}">Contact</a>
<a href="{{ site_url()|e('html_attr') }}">Home</a>
```

### `url_current(skipAjax = false, noGet = false)`

Current request URL. `skipAjax = true` prefers the referrer for AJAX requests; `noGet = true` strips the query string.

```twig
<link rel="canonical" href="{{ url_current(false, true)|e('html_attr') }}">
{% if 'sale' in url_current() %}...{% endif %}
```

### `assets(filePath)`

Versioned URL for a file in the active theme's `Assets` directory. Uses CDN configuration when set, otherwise the theme CDN path, plus a `?v=` file-version value. Supply the path relative to `Assets`.

```twig
<link rel="stylesheet" href="{{ assets('css/app.css')|e('html_attr') }}">
<script src="{{ assets('js/app.js')|e('html_attr') }}"></script>
<img src="{{ assets('images/logo.svg')|e('html_attr') }}" alt="">
```

Prefer `assets()` over hardcoded theme paths and over `template_url()` for Theme Studio assets.

### `template_url(path = false)`

URL below the active legacy template directory. Only for legacy template files; Theme Studio assets belong to `assets()`.

```twig
<img src="{{ template_url('assets/logo.svg')|e('html_attr') }}" alt="">
```

### `query_string(source = 'get', key = null)`

One request value: `'get'` reads the query string, `'post'` reads posted data. The key is required — without it the helper returns `''`. Missing values return `''`; multi-values return the array.

```twig
{% set coupon = query_string('get', 'coupon') %}
{% if coupon %}
    <p>Coupon applied: {{ coupon|e }}</p>
{% endif %}
```

### `helper_body_classes()`

Space-separated context classes for `<body>`: page, post, content, category, slug, and active-template classes depending on the request.

```twig
<body class="{{ helper_body_classes()|e('html_attr') }}">
```

## 7. Formatting, debug, and matching

### `currency_format(amount, currency = false)`

Money through the shop manager with the supplied currency; without one, shop configuration decides.

```twig
{{ currency_format(data.product.price|default(0)) }}
{{ currency_format(order_details.amount, order_details.currency) }}
```

### `barcode(value, type = 'png', color = '#000000', height = 30, width = 2)`

Code 128 barcode as a base64 data URI. Types `png`, `jpeg`, `svg` (anything else falls back to `png`).

```twig
<img src="{{ barcode(data.content_data.sku, 'png', '#000000', 50, 2)|e('html_attr') }}"
     alt="Product barcode">
```

### `get_option(key, optionGroup = false, returnFull = false, orderBy = false, module = false)`

Stored backend option value. Use `returnFull = true` only when the skin needs the complete option row.

```twig
<title>{{ get_option('website_title', 'website')|default('Website')|e }}</title>
```

### `print_data(value)`

HTML `<pre>` debug dump of any value. Development only: output is unescaped and can expose sensitive data.

```twig
{{ print_data(data) }}
```

### `json_encode(value, flags = 0, depth = 512)` / `json_decode(json, associative = null, depth = 512, flags = 0)`

Native PHP JSON pair. Encode for embedding data in scripts (with context-appropriate escaping); decode with `true` second for associative arrays.

```twig
<script type="application/json" id="product-data">
    {{ json_encode(data.product)|e }}
</script>
{% set settings = json_decode(optionValue, true) %}
{{ settings.layout|default('default')|e }}
```

### `preg_match_all(pattern, subject)`

Native match counter. Twig cannot receive the by-reference matches argument, so use it for counts only.

```twig
{% set digits = preg_match_all('/\\d+/', data.content.title) %}
```

### `is_mobile_device()`

`1` when the request User-Agent matches a small mobile-device expression, `0`/`false` otherwise. Heuristic: prefer responsive CSS for layout and call this only when server output must genuinely differ.

```twig
{% if is_mobile_device() %}
    <span class="mobile-only">Mobile content</span>
{% endif %}
```

## 8. Registered but unavailable

These four names resolve but always return `null` — no backend helper or custom implementation stands behind them. Do not call them; use the replacement beside each:

| Name | Use instead |
|---|---|
| `get_variation` | Prepared `data.product.variants` collection. |
| `content_field` | Prepared fields (`data.content.title`, `data.content.content_body`) or `content_data()`. |
| `db_connect_action_url` | Endpoint URLs the renderer passes (for example checkout `data.urls`). |
| `db_request_url` | Endpoint URLs the renderer passes. |
