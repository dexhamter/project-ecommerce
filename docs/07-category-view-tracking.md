# 2.7 — Category View Tracking (`view_item_list`)

**Project:** Kova Studio — Ecommerce  
**GTM Container:** Kova Studio (`GTM-NMPNZ4TV`)  
**Google Ads Account:** `AW-18244478477` (Kova Studio Ecommerce)  
**GA4 Measurement ID:** `G-JPWF3JGT5P`  
**GTM version change:** None — no new GTM tags, triggers, or variables required  
**Date completed:** 2026-06-26

---

## What This Does & Why

This subproject fires a GA4 `view_item_list` event every time a customer is shown a list of products — on collection pages, the homepage Featured Products section, and the search results page. For each list, the event captures which products were shown and in which list context (collection handle, fixed search identifier, or homepage section ID).

For Google Ads, `view_item_list` is the top-of-funnel impression signal that feeds the dynamic remarketing pipeline. Google can use list impression data to understand browse behaviour before a user enters the product detail page — enabling broader audience segments (users who saw product in a category) rather than only users who explicitly viewed a product. The event also fires the `GAds - Remarketing` tag, which passes the `shopify_GB_{product_id}_{variant_id}` composite IDs for every visible product to the Google Ads remarketing pool.

For GA4, `view_item_list` is the entry point of the Shopping Behaviour analysis funnel: Items Viewed in List → Items Clicked in List (from `select_item`) → Items Viewed (from `view_item`) → Items Added to Cart → Items Purchased. List CTR (view_item_list → select_item) is the only reliable native metric for evaluating list-level performance. GA4 does not natively persist list attribution through to purchase, so list-to-revenue attribution requires custom Exploration reports — but impression data must exist to build them.

---

## Prerequisites

- [ ] GTM container published at `v2.5.0` or later (Google Ads foundation live)
- [ ] `GA4 - Ecommerce Events` tag exists with an **All Custom Events** trigger
- [ ] `GAds - Remarketing` tag exists with `Trigger - CE - Ecommerce Events (Remarketing)` (regex must include `view_item_list`)
- [ ] `CJS - GAds Items Array` Custom JavaScript variable exists
- [ ] `DLV - ecommerce.items` data layer variable exists
- [ ] Shopify theme is Dawn (`sections/featured-collection.liquid` structure assumed)
- [ ] Subproject 2.6 (Product View Tracking) complete — this subproject mirrors the same Liquid patterns

---

## Business Requirement

Record which products were displayed on every product list surface (collection pages, search results, homepage sections), fire the GA4 `view_item_list` event with complete item parameters and list identity, and populate the Google Ads dynamic remarketing pool with all visible product variant IDs — on every page load that shows a product list.

---

## Data Layer Specification

### Event Name

`view_item_list`

### Event Parameters

| Parameter | Type | Example Value | Notes |
|-----------|------|---------------|-------|
| `item_list_id` | string | `"all"` | Collection handle (collection pages), `"search_results"` (search), `"homepage_featured_products_{section.id}"` (homepage) |
| `item_list_name` | string | `"Products"` | Collection title (collection pages), `"Search Results"` (search), `"Homepage - Featured Products"` (homepage) |
| `items` | array | — | Full items array; see schema below |
| `item_id` | string | `"49500550103263"` | Shopify **variant** ID of the first available variant — cast to string via `String()` |
| `item_group_id` | number | `9389166199007` | Shopify parent product ID — integer from Liquid |
| `item_name` | string | `"Classic Oxford Shirt"` | Product title |
| `item_brand` | string | `"Kova Studio"` | Hardcoded |
| `item_category` | string | `"Shirts"` | Shopify Product Type field |
| `item_variant` | string | `"S / White"` | First available variant title |
| `price` | number | `49.99` | Price of the first available variant |
| `quantity` | integer | `1` | Always `1` on list events |

**Important:** `item_id` on list pages uses `first_available_variant` (not `selected_or_first_available_variant` — no URL variant param context on list pages). This decision is binding for the entire funnel: if list pages push variant ID X and the PDP fires `view_item` with a different variant because the customer selected a different size, the funnel IDs diverge. See ADR-0007.

**`item_list_id` / `item_list_name` by page type:**

| Page Type | `item_list_id` | `item_list_name` |
|-----------|----------------|------------------|
| Collection (`/collections/all`) | `"all"` (collection.handle) | `"Products"` (collection.title) |
| Collection (`/collections/tops`) | `"tops"` | `"Tops"` |
| Search results | `"search_results"` (fixed) | `"Search Results"` (fixed) |
| Homepage featured | `"homepage_featured_products_{section.id}"` | `"Homepage - Featured Products"` |

Search uses fixed identifiers to prevent GA4 cardinality explosion from per-query list IDs. See ADR-0008.

---

## Data Layer Code — Three Snippets

This subproject requires three distinct Liquid snippets and one section file edit. Collection and search fire synchronously on render. Homepage uses a staging array (`window._kovaLists`) read at `DOMContentLoaded`.

### `snippets/tracking-collection.liquid`

[Screenshot: Shopify Edit Code showing tracking-collection.liquid in the snippets folder]

```liquid
{% comment %} snippets/tracking-collection.liquid {% endcomment %}
{% comment %} 2.7 — view_item_list for collection pages {% endcomment %}
{% comment %}
  Fires one view_item_list push synchronously on page load.
  item_list_id  = collection.handle  (stable, URL-safe — rarely changes)
  item_list_name = collection.title  (human-readable; handle is the join key if title shifts)
  item_id       = first_available_variant.id (keeps item_id consistent across the GA4 funnel)
  price         = that same variant's price   (internally consistent with item_id)
  See ADR-0005, ADR-0007.
{% endcomment %}

<script>
  window.dataLayer = window.dataLayer || [];

  dataLayer.push({ ecommerce: null });
  dataLayer.push({
    event: 'view_item_list',
    ecommerce: {
      item_list_id:   {{ collection.handle | json }},
      item_list_name: {{ collection.title | json }},
      items: [
        {%- for product in collection.products -%}
        {%- assign variant = product.selected_or_first_available_variant -%}
        {
          item_id:       String({{ variant.id | json }}),
          item_group_id: {{ product.id | json }},
          item_name:     {{ product.title | json }},
          item_brand:    "Kova Studio",
          item_category: {{ product.type | json }},
          item_variant:  {{ variant.title | json }},
          price:         {{ variant.price | divided_by: 100.0 }},
          quantity:      1
        }{% unless forloop.last %},{% endunless %}
        {%- endfor -%}
      ]
    }
  });
</script>
```

**Note on `selected_or_first_available_variant` vs `first_available_variant`:** On a collection page, there is never a `?variant=` URL parameter, so both resolve identically — the first in-stock variant. `selected_or_first_available_variant` is used here for consistent pattern with 2.6 (it degrades gracefully to first-available when no selection exists).

---

### `snippets/tracking-search.liquid`

[Screenshot: Shopify Edit Code showing tracking-search.liquid in the snippets folder]

```liquid
{% comment %} snippets/tracking-search.liquid {% endcomment %}
{% comment %} 2.7 — view_item_list for the search results page {% endcomment %}
{% comment %}
  Guards on search.performed and at least one product result before pushing.
  item_list_id  = "search_results" (fixed — prevents per-query cardinality explosion)
  item_list_name = "Search Results" (fixed — aggregates all search impressions into one list)
  search.results contains mixed object types (product, article, page).
  Only product-type results are included in the Items Array.
  The search query itself is captured natively by GA4 via the search_term parameter.
  See ADR-0005, ADR-0008.
{% endcomment %}

{%- if search.performed -%}
  {%- assign product_count = 0 -%}
  {%- for item in search.results -%}
    {%- if item.object_type == 'product' -%}
      {%- assign product_count = product_count | plus: 1 -%}
    {%- endif -%}
  {%- endfor -%}

  {%- if product_count > 0 -%}
<script>
  window.dataLayer = window.dataLayer || [];

  dataLayer.push({ ecommerce: null });
  dataLayer.push({
    event: 'view_item_list',
    ecommerce: {
      item_list_id:   "search_results",
      item_list_name: "Search Results",
      items: [
        {%- assign first_product = true -%}
        {%- for item in search.results -%}
          {%- if item.object_type == 'product' -%}
            {%- assign variant = item.selected_or_first_available_variant -%}
            {%- unless first_product -%},{%- endunless -%}
        {
          item_id:       String({{ variant.id | json }}),
          item_group_id: {{ item.id | json }},
          item_name:     {{ item.title | json }},
          item_brand:    "Kova Studio",
          item_category: {{ item.type | json }},
          item_variant:  {{ variant.title | json }},
          price:         {{ variant.price | divided_by: 100.0 }},
          quantity:      1
        }
            {%- assign first_product = false -%}
          {%- endif -%}
        {%- endfor -%}
      ]
    }
  });
</script>
  {%- endif -%}
{%- endif -%}
```

**Two-pass approach for search results:**

- **Pass 1** (lines 1–7): Counts product-type results to guard the entire push. If `product_count == 0` (empty query, articles only, or no results), no `<script>` tag is rendered at all.
- **Pass 2** (inside the `<script>`): Uses `first_product` boolean to handle comma placement — prepends a comma before each item after the first, rather than appending a trailing comma. Avoids the need for a `.filter(Boolean)` hack in JS.
- `item.object_type == 'product'` check inside the items loop ensures articles and pages are excluded even if they appear in `search.results`.

---

### `snippets/tracking-homepage.liquid`

[Screenshot: Shopify Edit Code showing tracking-homepage.liquid in the snippets folder]

```liquid
{% comment %} snippets/tracking-homepage.liquid {% endcomment %}
{% comment %} 2.7 — view_item_list reader for homepage product sections {% endcomment %}
{% comment %}
  Reads window._kovaLists (populated by inline <script> blocks in each homepage
  product section, e.g. featured-collection.liquid) and fires one view_item_list
  event per list entry at DOMContentLoaded.

  DOMContentLoaded is used (not synchronous push) because this snippet may render
  at any position in the template, before or after the section scripts that populate
  window._kovaLists. DOMContentLoaded guarantees all synchronous inline scripts
  have already run. See ADR-0005, ADR-0006.
{% endcomment %}

<script>
  window.dataLayer = window.dataLayer || [];
  window._kovaLists = window._kovaLists || [];

  document.addEventListener('DOMContentLoaded', function() {
    var lists = window._kovaLists;
    if (!lists || lists.length === 0) return;

    for (var i = 0; i < lists.length; i++) {
      var list = lists[i];
      if (!list.items || list.items.length === 0) continue;

      dataLayer.push({ ecommerce: null });
      dataLayer.push({
        event: 'view_item_list',
        ecommerce: {
          item_list_id:   list.item_list_id,
          item_list_name: list.item_list_name,
          items:          list.items
        }
      });
    }
  });
</script>
```

**How the staging array works:** Each homepage section that contains products (e.g. `featured-collection.liquid`) has a synchronous `<script>` block that pushes its product data into `window._kovaLists`. All of these section scripts run during HTML parsing. `tracking-homepage.liquid` waits until `DOMContentLoaded` — which fires after all synchronous scripts have run — then iterates `_kovaLists` and fires one `view_item_list` push per entry. This decouples the reader from section render order. See ADR-0006.

---

### `sections/featured-collection.liquid` — Added push block

[Screenshot: Shopify Edit Code showing the `window._kovaLists` push block at the bottom of featured-collection.liquid]

The following block was added to `sections/featured-collection.liquid`, immediately before `{% schema %}`:

```liquid
{% comment %}
  2.7 — Pushes this section's product data into window._kovaLists for
  tracking-homepage.liquid to consume at DOMContentLoaded.
  item_list_id uses section.id (immutable — survives Theme Editor renames).
  item_list_name is hardcoded (decoupled from section.settings.title).
  Only pushes if the section has a collection assigned with products.
  See ADR-0006.
{% endcomment %}
{%- if section.settings.collection.products.size > 0 -%}
<script>
  window._kovaLists = window._kovaLists || [];
  window._kovaLists.push({
    item_list_id:   "homepage_featured_products_{{ section.id }}",
    item_list_name: "Homepage - Featured Products",
    items: [
      {%- for product in section.settings.collection.products limit: section.settings.products_to_show -%}
      {%- assign variant = product.selected_or_first_available_variant -%}
      {
        item_id:       String({{ variant.id | json }}),
        item_group_id: {{ product.id | json }},
        item_name:     {{ product.title | json }},
        item_brand:    "Kova Studio",
        item_category: {{ product.type | json }},
        item_variant:  {{ variant.title | json }},
        price:         {{ variant.price | divided_by: 100.0 }},
        quantity:      1
      }{% unless forloop.last %},{% endunless %}
      {%- endfor -%}
    ]
  });
</script>
{%- endif -%}
```

**Why `section.id` for `item_list_id`:** `section.id` is Shopify's internal immutable identifier for a template section. It survives Theme Editor renames and content changes. `section.settings.title` is editable and would cause the list ID to change in GA4 if a merchant renames the heading.

**Why `section.settings.products_to_show` limit:** Only the products actually rendered in the grid are included. The section setting controls how many cards appear — pushing more products than are visible would inflate impression counts.

---

## GTM Setup

### No new GTM configuration required for this subproject

Both existing tags already handle `view_item_list` automatically:

- **`GA4 - Ecommerce Events`** fires on an **All Custom Events** trigger — picks up any `event` key pushed to the dataLayer, including `view_item_list`.
- **`GAds - Remarketing`** fires on `Trigger - CE - Ecommerce Events (Remarketing)`, which uses the regex `view_item_list$|select_item$|view_item$|add_to_cart$|view_cart$|remove_from_cart$|purchase$`. `view_item_list` is already in the regex.

[Screenshot: GTM Preview showing GA4 - Ecommerce Events and GAds - Remarketing both Fired on the view_item_list event]

**GTM version:** No new publish required. Container remains at the version deployed in 2.6.

---

## Shopify Theme Setup

### Step 1 — Create the three snippet files

In Shopify Admin → Online Store → Themes → **Edit code** → `snippets/` folder → **Add a new snippet** three times:

1. `tracking-collection` → paste `snippets/tracking-collection.liquid`
2. `tracking-search` → paste `snippets/tracking-search.liquid`
3. `tracking-homepage` → paste `snippets/tracking-homepage.liquid`

Save each after pasting.

### Step 2 — Edit `sections/featured-collection.liquid`

In Shopify Admin → Online Store → Themes → **Edit code** → `sections/featured-collection.liquid`.

Scroll to the bottom of the file. Add the `window._kovaLists` push block (see above) immediately before `{% schema %}`. Save.

**Why edit a section file:** Unlike product templates (which support Custom Liquid blocks via `main-product`'s block schema), `main-collection-product-grid` and `main-search` do not declare a `blocks` setting that accepts `custom_liquid`. There is no block-based injection path for collection or search sections. The homepage Featured Products section also does not support blocks that can access `section.settings.collection` — the section's own Liquid file is the only context where those settings are available.

### Step 3 — Inject tracking sections into template JSON files

The three tracking snippets are injected as standalone `custom-liquid` sections added to each page template's JSON. This is equivalent to adding a hidden "Custom Liquid" section via the Theme Editor, with padding set to 0.

[Screenshot: Shopify Edit Code showing templates/collection.json with tracking-collection section]

**`templates/collection.json`** — Add a `"tracking-collection"` section and append to `order`:

```json
{
  "sections": {
    "banner": { ... },
    "tracking-collection": {
      "type": "custom-liquid",
      "settings": {
        "custom_liquid": "{% render 'tracking-collection' %}",
        "color_scheme": "scheme-1",
        "padding_top": 0,
        "padding_bottom": 0
      }
    },
    "product-grid": { ... }
  },
  "order": ["banner", "product-grid", "tracking-collection"]
}
```

[Screenshot: Shopify Edit Code showing templates/search.json with tracking-search section]

**`templates/search.json`** — Add `"tracking-search"` and prepend to `order` (must fire before the search section renders its results in the DOM):

```json
{
  "sections": {
    "tracking-search": {
      "type": "custom-liquid",
      "settings": {
        "custom_liquid": "{% render 'tracking-search' %}",
        "color_scheme": "scheme-1",
        "padding_top": 0,
        "padding_bottom": 0
      }
    },
    "main": { ... }
  },
  "order": ["tracking-search", "main"]
}
```

[Screenshot: Shopify Edit Code showing templates/index.json with tracking-homepage section]

**`templates/index.json`** — Add `"tracking-homepage"` and append to `order`:

```json
{
  "sections": {
    "image_banner": { ... },
    "featured_collection": { ... },
    "tracking-homepage": {
      "type": "custom-liquid",
      "settings": {
        "custom_liquid": "{% render 'tracking-homepage' %}",
        "color_scheme": "scheme-1",
        "padding_top": 0,
        "padding_bottom": 0
      }
    }
  },
  "order": ["image_banner", "featured_collection", "tracking-homepage"]
}
```

**`custom-liquid` vs `custom_liquid`:** The `"type": "custom-liquid"` value (with a hyphen) refers to the **standalone section** defined in `sections/custom-liquid.liquid`, which exposes a `custom_liquid` textarea setting. This is different from the `"type": "custom_liquid"` (with an underscore) used for **blocks within sections** like `main-product`. Using the wrong one causes a 404 on the entire template.

### Step 4 — Upload the theme

Download the full theme, make the edits locally, then upload via Shopify Admin → Online Store → Themes → **Add theme** → **Upload zip file**. Publish after confirming preview works.

Alternatively use Shopify CLI: `shopify theme push --theme-id=THEME_ID`.

---

## GA4 Configuration

[Screenshot: GA4 DebugView showing view_item_list event with item_list_id, item_list_name, and items array]

- **Event name:** `view_item_list`
- **Marked as conversion:** No
- **Custom dimensions registered:** None — `item_list_id`, `item_list_name` are native GA4 ecommerce parameters for list events; `item_*` fields are part of the predefined item schema. No custom dimension registration required.

**GA4 List reports:** Under **Reports → Monetisation → Ecommerce purchases**, the "Items viewed in list" and "Items added to cart from list" metrics only populate when `view_item_list` and `select_item` are both present. `select_item` is a 2.x subproject not yet implemented — list click-through data will appear blank until then.

---

## Google Ads Configuration

This subproject does not create a conversion action. `view_item_list` feeds the **dynamic remarketing audience** for list-level browse signals.

- **Conversion action:** None (remarketing only)
- **Remarketing tag:** `GAds - Remarketing` (deployed in 2.5) — fires on all `view_item_list` events via the ecommerce regex trigger
- **Dynamic remarketing:** `CJS - GAds Items Array` constructs `shopify_GB_{item_group_id}_{item_id}` composite IDs for all items in the list. This means every product visible in a list view is added to the remarketing pool — broader than the per-product audience from `view_item`.

---

## Validation Steps

### Test 1 — Collection page

[Screenshot: GTM Preview event list showing view_item_list firing on /collections/all or any collection URL]

1. Open GTM Preview → connect to `https://fashion-sandbox.myshopify.com`
2. Navigate to any collection page (e.g., `/collections/all`)
3. Confirm `view_item_list` appears in the GTM Preview event stream
4. Click the event → **Data Layer** tab → confirm:
   - `ecommerce.item_list_id` matches the collection handle (e.g. `"all"`)
   - `ecommerce.item_list_name` matches the collection title (e.g. `"Products"`)
   - `ecommerce.items` array populated — all products on the page
   - Each item has `item_id` (string), `item_group_id` (number), `item_name`, `item_brand: "Kova Studio"`, `item_category`, `item_variant`, decimal `price`, `quantity: 1`
5. Click → **Tags** tab → confirm `GA4 - Ecommerce Events` and `GAds - Remarketing` both show **Fired**

[Screenshot: GTM Preview Data Layer tab showing full ecommerce object for the collection page view_item_list event]

[Screenshot: GTM Preview Tags tab showing GA4 - Ecommerce Events and GAds - Remarketing both Fired]

### Test 2 — Homepage

[Screenshot: GTM Preview event list showing view_item_list firing on the homepage after DOM Ready]

6. Navigate to the homepage (`/`)
7. Confirm `view_item_list` appears **after** the `DOM Ready` event in the GTM Preview stream (DOMContentLoaded fires after DOM Ready in GTM's model)
8. Confirm:
   - `item_list_id` starts with `"homepage_featured_products_"`
   - `item_list_name: "Homepage - Featured Products"`
   - Items array populated with homepage featured products

[Screenshot: GTM Preview Data Layer tab for the homepage view_item_list event]

### Test 3 — Search results (with results)

[Screenshot: GTM Preview event list showing view_item_list firing on /search?q=dress]

9. Navigate to `/search?q=dress` (or any query that returns products)
10. Confirm `view_item_list` fires with:
    - `item_list_id: "search_results"`
    - `item_list_name: "Search Results"`
    - Items array contains only products (no articles or pages)
    - Each product item uses first available variant ID

[Screenshot: GTM Preview Data Layer tab showing search view_item_list event with search_results identifiers]

### Test 4 — Search results (empty)

11. Navigate to `/search?q=xyznonexistent`
12. Confirm **no** `view_item_list` event fires (the guard prevents the push when `product_count == 0`)

---

## QA Checklist

- [ ] `view_item_list` fires on collection page load
- [ ] `view_item_list` fires on homepage with `window._kovaLists` pattern (after DOM Ready)
- [ ] `view_item_list` fires on search results page when products are returned
- [ ] `view_item_list` does **not** fire on search page when zero product results
- [ ] `item_list_id` on collection pages equals the collection handle (URL-safe slug, not title)
- [ ] `item_list_id` on search results is always `"search_results"` regardless of query string
- [ ] `item_list_id` on homepage starts with `"homepage_featured_products_"` followed by the section ID
- [ ] `item_list_name` on collection pages equals the collection title
- [ ] `item_list_name` on search results is always `"Search Results"`
- [ ] `item_list_name` on homepage is `"Homepage - Featured Products"`
- [ ] Items array is present and non-empty on all three page types
- [ ] Search items array contains **only** products (no articles or pages)
- [ ] Each item has `item_id` as a **string** (variant ID, not product ID)
- [ ] Each item has `item_group_id` as an integer (parent product ID)
- [ ] Each item has `item_brand: "Kova Studio"`
- [ ] `price` is a decimal float (e.g. `49.99`), not cents
- [ ] `quantity` is `1` on all items
- [ ] `ecommerce: null` push precedes every `view_item_list` push (no parameter bleed)
- [ ] `GA4 - Ecommerce Events` fires on all three page types (GTM Preview → Tags)
- [ ] `GAds - Remarketing` fires on all three page types (GTM Preview → Tags)
- [ ] `CJS - GAds Items Array` resolves with `shopify_GB_` composite IDs for list items
- [ ] All three snippet files exist in Shopify theme `snippets/` folder
- [ ] `window._kovaLists` push block is present at the bottom of `sections/featured-collection.liquid`
- [ ] `templates/collection.json` includes `tracking-collection` section with `type: "custom-liquid"`
- [ ] `templates/search.json` includes `tracking-search` section with `type: "custom-liquid"`
- [ ] `templates/index.json` includes `tracking-homepage` section with `type: "custom-liquid"`

---

## Common Errors & Fixes

| Error / Symptom | Root Cause | Fix |
|----------------|------------|-----|
| 404 on the entire collection, search, or homepage after theme upload | `"type"` in template JSON set to `"custom_liquid"` (underscore — block type) instead of `"custom-liquid"` (hyphen — standalone section type) | Open the affected template JSON → change `"type": "custom_liquid"` to `"type": "custom-liquid"` for the tracking sections |
| `view_item_list` fires but items array is empty (`items: []`) | `collection.products` resolving to empty, or `search.results` has no products | Collection: confirm the collection has products assigned in Shopify Admin. Search: confirm `search.performed` is true — the snippet only renders inside `search.json`, where the search context is available |
| `view_item_list` does not fire on collection page at all | `tracking-collection` section missing from `templates/collection.json` `order` array, or `tracking-collection.liquid` snippet not uploaded | Check `templates/collection.json` → `order` array; check `snippets/` folder in Shopify Edit Code |
| `view_item_list` does not fire on homepage | `window._kovaLists` never populated (section push block missing from `featured-collection.liquid`) or `tracking-homepage.liquid` not injected into `templates/index.json` | Check `sections/featured-collection.liquid` for the `_kovaLists.push` block above `{% schema %}`; check `templates/index.json` for `tracking-homepage` section |
| `view_item_list` fires on homepage with empty items array | `featured-collection.liquid` push block conditional `{% if section.settings.collection.products.size > 0 %}` never passes — no collection assigned in Theme Editor or collection is empty | Go to Shopify Admin → Themes → Customize → Homepage → Featured Collection section → assign a collection |
| `view_item_list` fires on search page even for empty searches | `product_count` guard not present in `tracking-search.liquid` | Confirm the two-pass structure: first `for` loop counts products, outer `if product_count > 0` wraps the `<script>` block |
| Search items array contains articles or pages | `item.object_type == 'product'` check missing inside the items `for` loop in `tracking-search.liquid` | Add the object_type guard inside the items loop; the `first_product` flag must reset only when `object_type == 'product'` |
| `item_id` is the product ID (7-digit) instead of variant ID (11-digit) | Used `product.id` or `item.id` instead of `variant.id` in the items loop | Change to `product.selected_or_first_available_variant.id` for collection/homepage; `item.selected_or_first_available_variant.id` for search |
| `item_category` is empty string for some products | Product Type not set in Shopify Admin for that product | Go to Shopify Admin → Products → [product] → set Product type |
| `price` is a large integer (e.g. `4999`) | `divided_by: 100.0` filter missing from the variant price Liquid output | Re-add `\| divided_by: 100.0` to all `variant.price` Liquid assignments in all three snippets and the featured-collection block |
| Remarketing `id` is `shopify_GB_undefined_undefined` | `item_group_id` or `item_id` missing from items array | Check `CJS - GAds Items Array` variable — verify it reads from `ecommerce.items[i].item_group_id` and `ecommerce.items[i].item_id` |
| Multiple `view_item_list` events fire for the same page | For homepage: extra section pushed data into `_kovaLists` — expected if multiple product sections exist on the homepage | Each section that pushes to `_kovaLists` produces one event. This is correct. To merge, change tracking-homepage to concatenate all list items into a single push. |
| `view_item_list` fires twice on homepage | `tracking-homepage.liquid` injected twice into `templates/index.json` | Check the `sections` object and `order` array in `templates/index.json` — `tracking-homepage` key should appear exactly once |

---

## Architectural Decisions

Four ADRs were recorded during the design of this subproject. See `docs/adr/` for full context.

| ADR | Decision |
|-----|----------|
| `0005-eager-domcontentloaded-over-intersectionobserver.md` | `view_item_list` fires at `DOMContentLoaded` (all lists on page load) rather than lazily via IntersectionObserver as sections scroll into view. Reason: IntersectionObserver is async — a user can click a product in an above-fold list before the observer callback fires, causing `select_item` to arrive before `view_item_list`. GA4 does not retroactively stitch out-of-order events; broken List CTR is more damaging than overcounted impressions. |
| `0006-window-kova-lists-staging-array.md` | Homepage product sections push item data into `window._kovaLists` (a staging array) rather than directly pushing to the dataLayer. Reason: `tracking-homepage.liquid` is injected as a standalone template section and may render before or after `featured-collection.liquid` in the DOM output. The staging array decouples writer (section scripts) from reader (tracking snippet) — `DOMContentLoaded` guarantees all synchronous section scripts have completed before the reader iterates. |
| `0007-first-available-variant-as-item-id-on-list-pages.md` | `item_id` on list pages uses the first available variant ID (not the product ID). Reason: the entire GA4 funnel uses variant IDs as `item_id`. If list pages pushed product IDs, GA4's Shopping Behaviour funnel would fail to match list impressions to downstream `view_item` and `add_to_cart` events. Google Merchant Centre composite IDs (`shopify_GB_{product_id}_{variant_id}`) also require a variant ID to be valid. |
| `0008-fixed-list-identifiers-for-search-results.md` | `item_list_id` for search results is always `"search_results"` (not `"search_results_{query}"`). Reason: per-query list IDs would create one list per unique search string in GA4 — hundreds of cardinality-exploded list IDs that make reports unusable. The actual query string is captured by GA4's native Enhanced Measurement `search_term` parameter. |

---

## Reusable Assets

- **Snippet files:**
  - `theme_export__fashion-sandbox-myshopify-com/snippets/tracking-collection.liquid`
  - `theme_export__fashion-sandbox-myshopify-com/snippets/tracking-search.liquid`
  - `theme_export__fashion-sandbox-myshopify-com/snippets/tracking-homepage.liquid`
- **Section edit:** `theme_export__fashion-sandbox-myshopify-com/sections/featured-collection.liquid` (bottom of file, before `{% schema %}`)
- **Template edits:** `theme_export__fashion-sandbox-myshopify-com/templates/collection.json`, `search.json`, `index.json`
- **ADRs:** `project-ecommerce/docs/adr/0005` through `0008`
- **Domain glossary update:** `project-ecommerce/CONTEXT.md` — added `window._kovaLists`, `item_list_id`, `item_list_name`, `Collection Page`, `Search Results Page`

---

## Related Guides

- `project-ecommerce/docs/02-data-layer-spec.md` — full items array schema and universal rules
- `project-ecommerce/docs/05-google-ads-foundation.md` — `GAds - Remarketing` tag and `CJS - GAds Items Array` variable that this guide depends on
- `project-ecommerce/docs/06-product-view-tracking.md` — 2.6 Liquid patterns this guide mirrors; `window._kovaProductData` data store pattern this guide extends with `window._kovaLists`
- `project-ecommerce/docs/adr/0005–0008` — architectural decisions for this subproject
