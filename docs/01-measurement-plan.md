# 2.1 — Measurement Plan

## What This Does & Why

The measurement plan is the architectural blueprint for everything that follows. It defines what we track, why we track it, and how the data flows through the stack before a single tag is created. For an ecommerce store, a measurement plan is especially important because the GA4 enhanced ecommerce schema is opinionated — the event names, parameter names, and `items` array structure are all prescribed by Google, and any deviation causes broken funnel reports that are difficult to fix retroactively.

This plan also resolves the Shopify-specific architectural challenge upfront: Shopify's checkout runs on a sandboxed domain (`checkout.shopify.com`) that GTM via `theme.liquid` cannot reach. The plan documents the correct hybrid architecture — GTM for the storefront, Custom Pixel + server-side GTM for checkout — so that implementation proceeds in the right order.

## Prerequisites

- [ ] Shopify development store live and accessible (fashion-sandbox.myshopify.com)
- [ ] Google Tag Manager account access
- [ ] GA4 account access
- [ ] Google Ads MCC access (440-830-5844)
- [ ] Stape.io account created (free plan)
- [ ] draw.io for data flow diagram

## Business Requirement

Track the complete Kova Studio customer journey — from product discovery through purchase — as Google Ads conversion data, with accurate revenue attribution, enhanced conversions, and a server-side data pipeline resilient to ad blockers and ITP.

---

## Business Persona

**Brand:** Kova Studio
**Type:** D2C fashion brand — minimalist clothing
**Demo store:** fashion-sandbox.myshopify.com
**Currency:** GBP

---

## Business Objectives & KPIs

| # | Objective | KPI |
|---|-----------|-----|
| 1 | Maximise purchase revenue | Total revenue / month |
| 2 | Reduce cart abandonment | Funnel conversion rate (`add_to_cart` → `purchase`) |
| 3 | Improve ROAS | Google Ads ROAS (revenue ÷ ad spend) |
| 4 | Build remarketing audiences | Audience size per segment (cart abandoners, category viewers, purchasers) |
| 5 | Increase average order value | AOV (total revenue ÷ purchase count) |

---

## Conversion Event Taxonomy

### Macro Conversions — sent to Google Ads as conversion actions

These are the events Google Ads Smart Bidding optimises toward. Only events that directly indicate business value are marked as macros.

| Event | GA4 Name | Trigger | Rationale |
|-------|----------|---------|-----------|
| Purchase | `purchase` | Order confirmation page | Primary revenue event — the only event with real conversion value |
| Begin Checkout | `begin_checkout` | User enters checkout flow | Secondary signal — provides volume while purchase numbers are low |

`add_to_cart` is intentionally excluded from macros — it is too high in the funnel to be a reliable Smart Bidding signal and would inflate conversion counts.

### Micro Conversions — GA4 only, not sent to Google Ads

These inform funnel drop-off analysis and remarketing audience building.

| Stage | Event | GA4 Name | Shopify Native Event |
|-------|-------|----------|---------------------|
| Discovery | Collection page view | `view_item_list` | `collection_viewed` |
| Discovery | Product click from list | `select_item` | — |
| Discovery | Product detail page view | `view_item` | `product_viewed` |
| Cart | Add to cart | `add_to_cart` | `product_added_to_cart` |
| Cart | Cart page view | `view_cart` | `cart_viewed` |
| Cart | Remove from cart | `remove_from_cart` | `product_removed_from_cart` |
| Checkout | Begin checkout | `begin_checkout` | `checkout_started` |
| Checkout | Shipping info submitted | `add_shipping_info` | `checkout_shipping_info_submitted` |
| Checkout | Payment info submitted | `add_payment_info` | `payment_info_submitted` |

### Data Correction Events — not conversions

| Event | GA4 Name | Purpose |
|-------|----------|---------|
| Refund | `refund` | Revenue correction — keeps GA4 revenue accurate when orders are refunded |

### Full Funnel (9 steps)

```
view_item_list → select_item → view_item → add_to_cart → view_cart
  → begin_checkout → add_shipping_info → add_payment_info → purchase
```

---

## Data Layer — `items` Array Schema

Every ecommerce event carries an `items` array. All parameters follow the GA4 ecommerce schema.

### `item_id` Decision

**Using Shopify variant ID** (not product handle).

Rationale: variant ID is unique per size/colour combination, enabling accurate deduplication and future-proofing for Google Ads Dynamic Remarketing. The human-readable `item_name` and `item_variant` fields provide readability alongside it.

### Items Object Schema

| Parameter | Type | Source | Example |
|-----------|------|--------|---------|
| `item_id` | string | Shopify variant ID | `"43291847123"` |
| `item_name` | string | Product title | `"Oversized Linen Jacket"` |
| `item_brand` | string | Hardcoded | `"Kova Studio"` |
| `item_category` | string | Collection handle | `"jackets"` |
| `item_variant` | string | Variant title | `"Black / M"` |
| `price` | number | Variant price (no currency symbol) | `89.00` |
| `quantity` | integer | Quantity added/purchased | `1` |

**Note:** `item_list_id` and `item_list_name` are added only on `view_item_list` events — they do not belong on individual product events.

### Required Event-Level Parameters

| Parameter | Type | Events | Example |
|-----------|------|--------|---------|
| `currency` | string | All ecommerce events | `"GBP"` |
| `value` | number | `view_item`, `add_to_cart`, `begin_checkout`, `purchase` | `89.00` |
| `transaction_id` | string | `purchase` only | `"5901234567304"` (Shopify `order.id` — long numeric) |
| `tax` | number | `purchase` only | `0.00` |
| `shipping` | number | `purchase` only | `3.99` |

**Note on `transaction_id`:** Shopify's Web Pixels API (Custom Pixel sandbox) exposes `event.data.checkout.order.id` — a long numeric string like `"5901234567304"`. The human-readable order name (`#1001`, via `order.name`) is **not** exposed in the sandbox. Use `order.id` for `transaction_id` in all purchase event implementations.

### ecommerce: null Rule

**Every ecommerce `dataLayer.push()` must be preceded by `dataLayer.push({ ecommerce: null })`.** This clears the previous ecommerce object and prevents parameter bleed between events. This is a GA4 requirement — failure to do this causes events to inherit parameters from the previous push.

---

## Tracking Architecture

### The Shopify Constraint

Shopify's checkout runs on `checkout.shopify.com` — a separate, locked-down domain. GTM installed via `theme.liquid` **does not fire on this domain**. Shopify deprecated `checkout.liquid` and "Additional Scripts" for new stores as part of Checkout Extensibility. Running GTM or `gtag.js` inside Shopify's Custom Pixel sandbox is officially unsupported by Google and causes significant data loss (Enhanced Conversions, cross-domain measurement, and consent mode URL passthrough all fail in the sandbox).

### Architecture Decision

**Hybrid: Client-side GTM (storefront) + Custom Pixel fetch() (checkout) + Server-side GTM (routing layer)**

```
Storefront (fashion-sandbox.myshopify.com)
  └── GTM via theme.liquid
        └── view_item_list, select_item, view_item, add_to_cart,
            view_cart, remove_from_cart
              └── GA4 client → sGTM server

Checkout (checkout.shopify.com)
  └── Custom Pixel (analytics.subscribe())
        └── begin_checkout, add_shipping_info,
            add_payment_info, purchase
              └── fetch() POST → sGTM server directly
                  (bypasses client-side GTM entirely)

sGTM (Stape.io)
  └── receives all events
        ├── routes to GA4 (Measurement Protocol)
        └── routes to Google Ads (conversion tags)
```

### Why This Architecture

| Component | Role | Why |
|-----------|------|-----|
| GTM via theme.liquid | Storefront tracking | Full GTM functionality — Preview Mode works, all tag types supported |
| Custom Pixel + fetch() | Checkout tracking | Only reliable way to capture checkout events; no script loading in sandbox |
| sGTM (Stape.io) | Routing layer | Bypasses ad blockers and ITP; first-party server; 20–40% more conversion data vs client-only |

### What to Avoid

- ❌ Loading GTM inside the Custom Pixel — unsupported by Google, Preview Mode breaks, data loss
- ❌ Loading `gtag.js` inside the Custom Pixel — same sandbox limitations apply
- ❌ Native Google & YouTube Shopify app — not installed; do not install (would double-count purchases)
- ❌ "Additional Scripts" on order status page — deprecated by Shopify

### sGTM Hosting

**Implementation:** Stape.io free plan
- Up to 10,000 requests/month (sufficient for sandbox demo volume)
- Custom domain included in free plan
- Stape-provided subdomain used for this implementation (e.g., `[container-id].stape.io`)

**Production path (documented in guide, not implemented):** Custom subdomain on the store's own domain (e.g., `metrics.kovastore.com`) — required for true first-party cookies and maximum ITP bypass. Requires DNS configuration pointing the subdomain to the Stape.io or GCP endpoint.

### GTM Efficiency — Master Regex Trigger

Instead of individual triggers per event, a single regex trigger handles all GA4 ecommerce events:

```
Trigger name: Trigger - CE - Ecommerce Events (Regex)
Type: Custom Event
Event name: view_item_list|select_item|view_item|add_to_cart|view_cart|remove_from_cart|begin_checkout|add_shipping_info|add_payment_info|purchase
Use regex matching: ✅
```

Paired with a single GA4 tag (`GA4 - Event - Ecommerce (Master)`) using `{{Event}}` as the event name and "Send Ecommerce data" enabled with Data Layer as the source. GTM parses the `ecommerce` object automatically.

---

## Account Structure

| Account | Name | ID |
|---------|------|----|
| GTM container | Kova Studio | TBD on creation |
| GA4 property | Kova Studio | TBD on creation |
| GA4 stream URL | https://fashion-sandbox.myshopify.com | TBD on creation |
| Google Ads (child) | Kova Studio Ecommerce | 912-474-4048 |
| Google Ads conversion ID | — | AW-18244478477 |
| Google Ads MCC | Hammad Yousuf Analytics | 440-830-5844 |
| sGTM host | Stape.io | TBD on provisioning |

---

## Google Ads Conversion Actions

| Conversion Action | Event | Category | Value | Count | Click Window | View Window | Engaged-View Window | Attribution |
|------------------|-------|----------|-------|-------|-------------|-------------|--------------------|-|
| Purchase | `purchase` | Purchase | Dynamic (`value` from dataLayer) | One | 30 days | 1 day | 3 days | Linear |
| Begin Checkout | `begin_checkout` | Begin checkout | No value | Many | 7 days | 1 day | 3 days | Linear |

**Attribution model:** Linear (fallback while conversion volume is below Data-driven threshold).
Migrate to **Data-driven** attribution as soon as the account qualifies (requires sufficient conversion volume — Google evaluates this automatically and will prompt when eligible).

**Why not Last Click:** Last click attributes 100% of credit to the final interaction and ignores upper-funnel ad contributions. For a fashion brand with a multi-touch customer journey, this leads to poor optimisation decisions and falsely signals that awareness-driving campaigns are failing.

**Conversion lag note:** Google Ads attributes the conversion to the date of the ad click, not the purchase date. A user who clicks an ad on May 1st and buys on May 25th shows revenue on May 1st in reporting. Recent days will always show a temporary dip — this is attribution lag, not a tracking failure.

---

## GTM Naming Conventions

Follows Phase 1 conventions exactly.

| Type | Format | Examples |
|------|--------|---------|
| Tags | `[Provider] - [Type] - [Description]` | `GA4 - Event - Ecommerce (Master)`, `GAds - Conversion - Purchase` |
| Triggers | `Trigger - [Type] - [Description]` | `Trigger - CE - Ecommerce Events (Regex)`, `Trigger - CE - purchase` |
| Variables | `[VarType] - [Description]` | `DLV - ecommerce.value`, `DLV - ecommerce.transaction_id`, `Const - GA4 Measurement ID` |

Variable type abbreviations: `Const` = Constant, `DLV` = Data Layer Variable, `JS` = Custom JavaScript, `URL` = URL Variable

---

## Data Sources

| Source | Data | Destination |
|--------|------|-------------|
| Shopify theme.liquid (GTM) | Storefront ecommerce events | sGTM → GA4 + Google Ads |
| Shopify Custom Pixel | Checkout + purchase events | sGTM (via fetch()) → GA4 + Google Ads |
| sGTM (Stape.io) | All events (routing layer) | GA4, Google Ads |
| GA4 | Aggregated behavioural data | Linked to Google Ads for audiences + reporting |
| Google Ads | Campaign spend + click data | Linked to GA4 for ROAS reporting |

---

## Validation Steps

1. Confirm Shopify dev store accessible at `fashion-sandbox.myshopify.com`
2. Confirm Google & YouTube app is NOT installed in Shopify admin
3. Confirm new GTM container created and named `Kova Studio`
4. Confirm new GA4 property created and named `Kova Studio`
5. Confirm Google Ads child account `912-474-4048` exists under MCC `440-830-5844`
6. Confirm Stape.io account created and sGTM container provisioned

---

## QA Checklist

- [ ] All 5 business objectives documented with measurable KPIs
- [ ] All 9 micro-conversion events listed with correct GA4 names
- [ ] Both macro conversions listed with correct GA4 names
- [ ] `items` array schema documented with all 7 fields
- [ ] `ecommerce: null` rule documented
- [ ] Tracking architecture diagram created (draw.io) → saved to `screenshots/01-measurement-plan/`
- [ ] Account IDs all confirmed and recorded above
- [ ] Google Ads conversion windows confirmed in account
- [ ] No Google & YouTube app installed on store

---

## Common Errors & Fixes

| Error | Cause | Fix |
|-------|-------|-----|
| Revenue double-counted in GA4 | Native Google & YouTube app running alongside custom pixel | Remove the app or disable its GA4/conversion tracking features |
| Checkout events missing entirely | Tried to use GTM via theme.liquid for checkout events | Use Custom Pixel + fetch() to sGTM for all checkout events |
| GTM fires on `checkout.shopify.com` but tags fail | Loaded GTM inside Custom Pixel sandbox | Remove GTM from Custom Pixel; use fetch() to sGTM directly |
| `ecommerce` parameters bleeding between events | Missing `dataLayer.push({ ecommerce: null })` before each push | Add the null push before every ecommerce event push |
| Attribution looks wrong in Google Ads | Using Last Click attribution | Switch to Linear; migrate to Data-driven when eligible |

---

## Reusable Assets

- Data flow diagram → `screenshots/01-measurement-plan/data-flow-diagram.png`
- This measurement plan → `project-ecommerce/docs/01-measurement-plan.md`

---

## Related Guides

- `google-ads-measurement-library/docs/guides/01-planning-architecture/ecommerce-measurement-plan.md`
- `google-ads-measurement-library/docs/guides/02-data-collection/shopify-data-layer.md` *(written after 2.2)*
- `google-ads-measurement-library/docs/guides/08-privacy-consent/consent-mode-v2.md` *(written after 2.15)*
