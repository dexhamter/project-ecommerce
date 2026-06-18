# 2.4 — GA4 Foundation (Shopify)

**Project:** Kova Studio — Ecommerce  
**GA4 Property:** Kova Studio (`G-JPWF3JGT5P`)  
**Date completed:** 2026-06-18

---

## What This Does & Why

This subproject configures the GA4 property to correctly receive, process, and report on ecommerce data from the Kova Studio Shopify store. The GA4 Config tag was deployed in 2.3 — this subproject is entirely property-level configuration: Enhanced Measurement settings, custom dimension registration, data retention, and internal traffic filtering.

Two corrections to the original playbook were made during grilling:

1. **Custom dimension scope:** `item_category`, `item_brand`, `item_variant`, and `item_list_name` are native GA4 ecommerce schema parameters. GA4 automatically parses them from the `items` array and populates built-in dimensions (Item Brand, Item Category, Item Variant, Item List Name) in Monetization reports. They cannot and should not be registered as custom dimensions — the GA4 interface blocks duplicate registration of reserved parameter names. Only `transaction_id` requires registration.

2. **No custom metrics:** GA4's native ecommerce schema handles all standard transaction fields (`value`, `tax`, `shipping`, `price`, `quantity`) automatically. Custom metrics would only be needed for non-standard numeric data such as profit margin or loyalty points.

---

## Prerequisites

- [ ] GA4 property created (`G-JPWF3JGT5P`) — done before 2.3
- [ ] GA4 Config tag deployed in GTM and firing on storefront pages — 2.3 complete
- [ ] Data layer spec finalised — 2.2 complete
- [ ] Own IP address known for internal traffic filter

---

## Business Requirement

Configure the GA4 property so that all ecommerce events from 2.6 onwards are correctly parsed, dimensioned, and attributed — with internal traffic excluded and data retained long enough to support year-over-year analysis.

---

## Data Layer Specification

No data layer changes in this subproject. All configuration is inside GA4 Admin.

---

## GA4 Configuration

### Enhanced Measurement

Navigate to: **Admin → Data Streams → [stream] → Enhanced Measurement**

| Setting | Status | Notes |
|---------|--------|-------|
| Page views | ✅ On | Standard |
| Scrolls | ✅ On | Standard |
| Outbound clicks | ✅ On | Works on storefront pages; does not work inside Custom Pixel sandbox (checkout) — see caveat below |
| Video engagement | ✅ On | Standard |
| File downloads | ✅ On | Standard |
| Site search | ✅ On | Shopify Dawn uses query parameter `q` — set this in the Site Search settings |
| Form interactions | ❌ Off | Auto form tracking fires on validation errors and UI controls (quantity steppers, variant selectors, filter toggles) — all noise. Custom ecommerce events handle all meaningful interactions. |

**Site Search configuration:** Click the gear icon next to Site Search → confirm query parameter is set to `q`. This tracks `view_search_results` events automatically whenever a user searches the Shopify store.

**Outbound clicks caveat:** Enhanced Measurement outbound click tracking fires from the GA4 Config tag on storefront pages — it works correctly there. It is not available inside Shopify's Custom Pixel sandbox (checkout.shopify.com), where the GA4 tag does not run. Outbound click tracking in checkout would require manual custom event implementation, which is outside scope for this project.

---

### Custom Dimension

Navigate to: **Admin → Custom definitions → Custom dimensions → Create custom dimension**

Only one custom dimension is registered. All other ecommerce item parameters (`item_category`, `item_brand`, `item_variant`, `item_list_name`, etc.) are native GA4 ecommerce schema fields — GA4 populates them automatically from the `items` array and they appear as built-in dimensions in Monetization reports. Do not attempt to register them.

| Dimension name | Parameter | Scope | Reason for registration |
|----------------|-----------|-------|------------------------|
| Transaction ID | `transaction_id` | Event | Surfaces Order ID as a filterable dimension in Explorations and custom reports. GA4 shows it in standard Ecommerce purchases report automatically, but registration is required to use it as a dimension in Explorations. |

**Why item parameters are not registered:**

`item_category`, `item_brand`, `item_variant`, `item_list_name`, `item_list_id`, `item_id`, `item_name`, `price`, `quantity` are all part of GA4's predefined ecommerce item schema. GA4 parses these automatically from the `items` array on every ecommerce event and populates dedicated built-in dimensions for them in Monetization → Ecommerce reports. The GA4 interface will block registration of these reserved parameter names to prevent duplication.

---

### Data Retention

Navigate to: **Admin → Data Settings → Data Retention**

| Setting | Value |
|---------|-------|
| Event data retention | 14 months |
| Reset user data on new activity | On (default) |

14 months provides 12 months of full data plus a two-month buffer — sufficient for year-over-year comparisons in standard GA4 reports.

---

### Internal Traffic Filter

Navigate to: **Admin → Data Streams → [stream] → Configure tag settings → Define internal traffic**

1. Create an IP rule with your current IP address
2. `traffic_type` value: `internal`

Then navigate to: **Admin → Data Filters → Internal Traffic**

3. Set filter state to **Active** (not Testing)

> All traffic during implementation is your own. Testing mode serves no practical purpose here — set to Active immediately.

---

### Google Ads Link

**Deferred to 2.5.** The Google Ads account (`AW-18244478477`) is ready, but the GA4 → Google Ads link is done alongside the three baseline Google Ads GTM tags in 2.5 to keep all Google Ads setup in one subproject. This matches the Phase 1 pattern (link done in 1.5, not 1.4).

---

### Key Events

None marked in this subproject. Key events are marked per event subproject as each conversion tag is deployed and validated (first marked in 2.11 for `purchase`).

---

## GTM Setup

No GTM changes in this subproject. All work is in GA4 Admin.

---

## Google Ads Configuration

No Google Ads work in this subproject. Deferred to 2.5.

---

## Validation Steps

1. Navigate to any storefront page
2. Open **GA4 → Admin → DebugView**
3. Confirm `page_view` event appears (fired by GA4 Config tag from 2.3)
4. Confirm `scroll` event appears after scrolling 90% down the page
5. Perform a search on the Shopify store → confirm `view_search_results` event appears with `search_term` parameter populated
6. Open **GA4 → Realtime** — confirm your own visits do not appear (internal traffic filter active)
7. Navigate to **Admin → Custom definitions** → confirm `Transaction ID` dimension is listed with event scope

---

## QA Checklist

- [ ] Enhanced Measurement: form interactions confirmed **Off**
- [ ] Enhanced Measurement: site search confirmed **On**, query parameter set to `q`
- [ ] `view_search_results` fires in DebugView after a storefront search
- [ ] `Transaction ID` custom dimension registered, event-scoped, parameter `transaction_id`
- [ ] No item parameters (`item_category` etc.) registered as custom dimensions
- [ ] Data retention set to 14 months
- [ ] Internal traffic IP rule created
- [ ] Internal traffic filter set to **Active**
- [ ] Realtime confirms own visits excluded
- [ ] GA4 → Google Ads link noted as deferred to 2.5

---

## Common Errors & Fixes

| Error | Cause | Fix |
|-------|-------|-----|
| `form_submit` events appearing in DebugView on product pages | Form interactions left On | Disable Form interactions in Enhanced Measurement |
| `view_search_results` not firing after storefront search | Site search query parameter set incorrectly | Confirm parameter is `q` in Enhanced Measurement Site Search settings |
| GA4 blocks registration of `item_brand` as custom dimension | Reserved native ecommerce parameter | Do not register — GA4 populates it automatically from the `items` array |
| Own visits still showing in Realtime | Internal traffic filter set to Testing, not Active | Change filter state to Active |
| `transaction_id` not available as dimension in Explorations | Not registered as custom dimension | Register as event-scoped custom dimension with parameter `transaction_id` |

---

## Design Decisions

### No Custom Metrics
GA4's built-in ecommerce schema automatically surfaces `Purchase revenue`, `Item revenue`, `Purchases`, `Average purchase revenue`, and related metrics from the `value`, `price`, `quantity`, `tax`, and `shipping` parameters. Custom metrics are only warranted for non-standard numeric values (e.g., profit margin, loyalty points, COGS). This project uses the standard schema — no custom metrics needed.

### item_category etc. Not Registered
This is worth documenting explicitly because many guides incorrectly recommend registering these as custom dimensions. They are GA4 reserved parameters, automatically populated in Monetization reports. Registering them would be redundant at best and blocked by GA4 at worst. Trust the native schema.

---

## Reusable Assets

No new GTM assets in this subproject.

---

## Related Guides

- `docs/03-gtm-foundation.md` — GTM baseline (prerequisite)
- `docs/02-data-layer-spec.md` — defines all parameters referenced by custom dimensions
- `docs/05-google-ads-foundation.md` — GA4 → Google Ads link (deferred here)
