# 2.3 — GTM Foundation (Shopify)

**Project:** Kova Studio — Ecommerce  
**GTM Container:** Kova Studio (`GTM-NMPNZ4TV`)  
**GA4 Measurement ID:** `G-JPWF3JGT5P`  
**Published version:** `v2.3.0 - GTM Foundation`  
**Container export:** `gtm/GTM-NMPNZ4TV_v1.json`  
**Date completed:** 2026-06-18

---

## What This Does & Why

This subproject establishes the baseline GTM container for the Kova Studio Shopify store — the minimum viable state that every subsequent tracking subproject builds on. It covers three things: installing GTM into the Shopify theme, enabling all Built-in Variables, and deploying the GA4 Config tag.

Shopify's GTM installation is meaningfully different from WordPress. There is no plugin to handle snippet injection — the GTM head and body snippets must be manually pasted into `layout/theme.liquid`. The placement relative to Shopify's `{{ content_for_header }}` Liquid tag matters: GTM must load before Shopify's own injected scripts, not after them.

One important decision made here: the Phase 1 GCLID capture pattern (Custom HTML tag writing to `localStorage`) is **not carried over** to this project. On Shopify, `localStorage` is bound to the storefront domain and is inaccessible from the Custom Pixel sandbox (a strict sandboxed iframe) and from `checkout.shopify.com` (a separate domain). GCLID attribution for web conversions is handled by the Conversion Linker tag deployed in 2.5. No manual capture is needed or appropriate.

---

## Prerequisites

- [ ] GTM container created: `Kova Studio` (`GTM-NMPNZ4TV`)
- [ ] GA4 property created: Kova Studio (`G-JPWF3JGT5P`)
- [ ] Shopify Development Store active and accessible
- [ ] Shopify CLI installed and authenticated
- [ ] Theme files downloaded locally via `shopify theme pull`
- [ ] Active theme confirmed as Dawn

---

## Business Requirement

Deploy a clean GTM container baseline on the Shopify storefront: GA4 Config tag firing on all storefront pages, all Built-in Variables available, and the container published and validated before any ecommerce event tags are added.

---

## Data Layer Specification

No data layer events in this subproject. The GA4 Config tag fires on `Initialization - All Pages` and sends a `page_view` via GA4's automatic page view measurement. No custom `dataLayer.push()` is required.

---

## GTM Setup

### Step 1 — Install GTM in theme.liquid

The GTM snippet is injected directly into `layout/theme.liquid` — the single master layout file for the Dawn theme. Do not edit section files or snippet files; all changes go here.

**File path:** `layout/theme.liquid`

**Head snippet placement:** Immediately before `{{ content_for_header }}` (line 66 in the default Dawn installation). Do **not** place it after `{{ content_for_header }}` — Shopify uses that tag to inject its own scripts, and GTM must load before them.

```html
<!-- Google Tag Manager -->
<script>(function(w,d,s,l,i){w[l]=w[l]||[];w[l].push({'gtm.start':
new Date().getTime(),event:'gtm.js'});var f=d.getElementsByTagName(s)[0],
j=d.createElement(s),dl=l!='dataLayer'?'&l='+l:'';j.async=true;j.src=
'https://www.googletagmanager.com/gtm.js?id='+i+dl;f.parentNode.insertBefore(j,f);
})(window,document,'script','dataLayer','GTM-NMPNZ4TV');</script>
<!-- End Google Tag Manager -->

{{ content_for_header }}
```

**Body snippet placement:** Immediately after `<body` opening tag (line 324 in the default Dawn installation):

```html
<body class="gradient{% if settings.animations_hover_elements != 'none' %} animate--hover-{{ settings.animations_hover_elements }}{% endif %}">
<!-- Google Tag Manager (noscript) -->
<noscript><iframe src="https://www.googletagmanager.com/ns.html?id=GTM-NMPNZ4TV"
height="0" width="0" style="display:none;visibility:hidden"></iframe></noscript>
<!-- End Google Tag Manager (noscript) -->
```

> The `<body>` tag in Dawn includes dynamic Liquid classes — preserve them exactly. Only add the noscript block on the line immediately after the opening tag.

### Step 2 — Push theme changes to Shopify for preview

GTM Preview connects to a live URL via Tag Assistant — it cannot inspect local files. Use the Shopify CLI development server to push changes as a development theme (does not affect the published live theme):

```bash
shopify theme dev
```

This command:
1. Uploads your local theme as a development theme
2. Syncs file changes automatically as you save
3. Returns a preview URL in the form: `https://[store].myshopify.com/?preview_theme_id=[dev-theme-id]`

Use this preview URL in GTM Preview.

### Step 3 — Enable Built-in Variables

1. In GTM, go to **Variables** → **Configure** (under Built-In Variables)
2. Enable all groups: **Clicks**, **Forms**, **History**, **Scrolling**, **Videos**, **Visibility**, **Errors**, **Utilities**
3. Save

All 16+ built-in variables are now available to every future tag and trigger without additional setup. Zero performance cost when unused.

### Tag Configuration — GA4 Config

**Tag name:** `GA4 - Config`  
**Tag type:** Google Tag  
**Tag ID:** `{{Const - GA4 Measurement ID}}`  
**Trigger:** Initialization - All Pages

#### Variable — Const - GA4 Measurement ID

| Field | Value |
|-------|-------|
| Variable type | Constant |
| Name | `Const - GA4 Measurement ID` |
| Value | `G-JPWF3JGT5P` |

> Always use a Constant variable for the Measurement ID — never hardcode it directly in the tag. This makes future property changes a single-field edit.

### Step 4 — Publish

Publish the container as version `v2.3.0 - GTM Foundation`.

### Step 5 — Push theme to live

Once GTM Preview validation passes and the container is published:

```bash
shopify theme push
```

This promotes the development theme changes (GTM snippet in `theme.liquid`) to the live published theme.

### Step 6 — Export container JSON

Export the container from GTM → Admin → Export Container → Current version.

Save to: `project-ecommerce/gtm/GTM-NMPNZ4TV_v1.json`

---

## GA4 Configuration

No GA4 property configuration changes in this subproject. The property was created before 2.3 (Measurement ID: `G-JPWF3JGT5P`) — property-level configuration (Enhanced Measurement, custom dimensions, data retention, internal traffic filter) is covered in 2.4.

---

## Google Ads Configuration

No Google Ads work in this subproject. All three Google Ads baseline tags (`GAds - Config`, `GAds - Conversion Linker`, `GAds - Remarketing - All Pages`) are deployed in 2.5.

---

## Validation Steps

1. Run `shopify theme dev` to get the preview URL
2. In GTM, click **Preview** and enter the preview URL
3. Tag Assistant opens the store — navigate to any storefront page
4. In the GTM Preview panel, confirm:
   - `GA4 - Config` appears under **Tags Fired** on the **Initialization** event
   - No tags appear under **Tags Not Fired** (other than those intentionally not yet created)
   - The **Variables** tab shows `Const - GA4 Measurement ID` = `G-JPWF3JGT5P`
5. In GA4 DebugView, confirm a `page_view` event appears for the storefront page

---

## QA Checklist

- [ ] GTM head snippet is placed **before** `{{ content_for_header }}` in `theme.liquid`
- [ ] GTM body snippet is placed immediately after `<body` in `theme.liquid`
- [ ] `GA4 - Config` fires on **Initialization** in GTM Preview
- [ ] `Const - GA4 Measurement ID` resolves to `G-JPWF3JGT5P` in the Variables tab
- [ ] No console errors related to GTM or GA4
- [ ] GA4 DebugView confirms `page_view` on a storefront page
- [ ] GTM does **not** fire on `checkout.shopify.com` pages (expected — document this)
- [ ] Container published as `v2.3.0 - GTM Foundation`
- [ ] Container exported to `gtm/GTM-NMPNZ4TV_v1.json`
- [ ] `shopify theme push` run after validation

---

## Common Errors & Fixes

| Error | Cause | Fix |
|-------|-------|-----|
| GTM fires but `page_view` not in GA4 DebugView | GA4 Config tag not associated with correct GA4 property / wrong Measurement ID | Verify `Const - GA4 Measurement ID` = `G-JPWF3JGT5P`; check GA4 DebugView is open on the correct property |
| GTM Preview shows no tags firing | Preview URL does not include the development theme ID | Ensure you're using the URL returned by `shopify theme dev`, not the plain store URL |
| GTM snippet appears after Shopify-injected scripts in page source | Head snippet placed after `{{ content_for_header }}` | Move the head snippet to before `{{ content_for_header }}` |
| `<body>` tag Liquid classes broken after snippet insertion | Body snippet inserted inside the `<body` tag attributes instead of after it | The noscript block must be on the line **after** the closing `>` of the body tag |
| GTM does not fire on checkout pages | Expected — GTM via `theme.liquid` only fires on storefront pages; `checkout.shopify.com` is sandboxed | No fix needed; document as known limitation; checkout events handled by Custom Pixel in 2.11 |

---

## Design Decision — Why GCLID localStorage Capture Is Not Used

In Phase 1 (Lead Gen), a `Custom HTML - GCLID Capture` tag wrote the GCLID to `localStorage._gclid` for use in offline conversion imports. This pattern is **not carried over** to the Shopify ecommerce project for three reasons:

1. **Custom Pixel sandbox**: Shopify's checkout tracking (2.11) runs via the Web Pixels API in a strict sandboxed iframe. The sandbox cannot access the parent window's `localStorage` — the stored GCLID would be invisible at purchase time.

2. **Domain boundary**: Standard Shopify checkout runs on `checkout.shopify.com`, a separate domain. `localStorage` is strictly domain-bound and does not transfer across origins.

3. **Native alternative exists**: GTM's Conversion Linker tag (deployed in 2.5) writes the GCLID to a `_gcl_aw` cookie and handles GCLID attribution natively for web conversion tags. No custom capture is required.

---

## Reusable Assets

- `gtm/GTM-NMPNZ4TV_v1.json` — baseline container export (GA4 Config + Built-in Variables only)

---

## Related Guides

- `guides/02-data-collection/gtm/shopify-gtm-install.md` — library guide extracted from this subproject
- `guides/02-data-collection/gtm/tags-triggers-variables.md` — GTM foundation guide from Phase 1 (WordPress context)
- `docs/02-data-layer-spec.md` — ecommerce data layer spec (prerequisite context)
