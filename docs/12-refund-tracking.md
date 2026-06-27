# 2.12 — Refund Tracking (`refund`)

**Implementation label:** 🟨 Simulated — the client-side GTM path is validated. The production server-side path (Shopify webhook → Measurement Protocol v2) is documented but not deployed.

---

## What This Does & Why

When a customer returns an item, Shopify processes the refund server-side in the Shopify Admin. Without tracking this, GA4's revenue figures silently inflate — every order is counted, but no deductions are recorded. Over time, GA4 revenue diverges from Shopify's actual revenue, making attribution and ROAS analysis unreliable.

GA4's `refund` event corrects this. It deducts the refunded amount from the original order, identified by `transaction_id`, keeping reported revenue accurate. This matters for Google Ads specifically because Smart Bidding uses GA4 conversion value signals — inflated revenue causes the algorithm to over-optimise toward orders that were ultimately returned.

**Google Ads does not accept `refund` events directly.** The `refund` dataLayer push is GA4-only. If you need Google Ads to retract a conversion, that requires a separate offline process (Conversion Adjustment Upload via the Google Ads API or CSV upload — see Related Guides). This guide does not cover that flow.

---

## Prerequisites

- [x] GTM Storefront Container `GTM-NMPNZ4TV` at `v2.11.0` or later
- [x] GA4 property `G-JPWF3JGT5P` (Kova Studio) — stream active
- [x] `CE - Ecommerce Events` regex trigger updated to include `refund$` (done in this subproject — see GTM Setup)
- [x] 2.11 Purchase Tracking complete (required for production path context — `transaction_id` must match the original purchase event)
- [ ] *(Production path only)* A server endpoint capable of receiving Shopify webhooks and making outbound HTTPS requests
- [ ] *(Production path only)* GA4 Measurement Protocol API secret generated (see Production Architecture section)

---

## Business Requirement

Record every order refund in GA4 so that reported revenue accurately reflects actual net revenue, enabling reliable ROAS measurement and correct Smart Bidding signals in Google Ads.

---

## Data Layer Specification

### Event Name
`refund`

### Event Parameters

| Parameter | Type | Example Value | Notes |
|-----------|------|---------------|-------|
| `transaction_id` | string | `"KVS-1001"` | Must exactly match the `transaction_id` from the original `purchase` event. GA4 uses this to tie the refund to the order. |
| `value` | number | `65.00` | Amount being refunded, in GBP. Full refund = full order value. Partial refund = partial amount only. |
| `currency` | string | `"GBP"` | Always `"GBP"` for this store. |
| `items` | array | see below | **Full refund:** include all purchased items at original quantities. **Partial refund:** include only the line items being refunded, at the refunded quantity. |
| `item_id` | string | `"44123456789"` | Shopify variant ID. Plain string — no GMC Composite ID prefix here. |
| `item_name` | string | `"Kova Oversized Jacket"` | Human-readable product name. |
| `price` | number | `65.00` | Unit price at time of purchase. |
| `quantity` | number | `1` | Units being refunded (not the original purchase quantity). |

**Minimum `items` array fields:** `item_id`, `item_name`, `price`, `quantity`. GA4 does not require the full items schema for refund events — use the minimum.

### Full dataLayer Code — Full Refund (all items, full value)

[Screenshot: GTM Preview → Data Layer tab showing event #12 `refund` with both items in the `items` array and `value: 95`]

```javascript
// 1. Clear previous ecommerce object — mandatory before every ecommerce push
window.dataLayer.push({ ecommerce: null });

// 2. Full refund — all items, full order value
// transaction_id must match the original purchase event exactly
window.dataLayer.push({
  event: 'refund',
  ecommerce: {
    transaction_id: 'KVS-1001',   // Replace with actual Shopify order ID
    value: 95.00,                  // Full order value
    currency: 'GBP',
    items: [
      {
        item_id: '44123456789',    // Shopify variant ID (String)
        item_name: 'Kova Oversized Jacket',
        price: 65.00,
        quantity: 1
      },
      {
        item_id: '44987654321',
        item_name: 'Kova Essential Tee',
        price: 30.00,
        quantity: 1
      }
    ]
  }
});
```

### Full dataLayer Code — Partial Refund (one item only)

[Screenshot: GTM Preview → Data Layer tab showing event #14 `refund` with single item and `value: 65`]

```javascript
// 1. Clear previous ecommerce object
window.dataLayer.push({ ecommerce: null });

// 2. Partial refund — jacket only from the same order
window.dataLayer.push({
  event: 'refund',
  ecommerce: {
    transaction_id: 'KVS-1001',   // Same order ID as full refund example
    value: 65.00,                  // Only the refunded item's value
    currency: 'GBP',
    items: [
      {
        item_id: '44123456789',
        item_name: 'Kova Oversized Jacket',
        price: 65.00,
        quantity: 1                // Quantity being refunded — not original purchase qty
      }
    ]
  }
});
```

**Key difference between full and partial:** the partial refund `items` array contains **only the refunded line item**. The t-shirt is omitted because it is not being returned.

---

## Production Architecture

🟨 **This section is documented but not deployed.** In a real Shopify store, refunds never originate in the browser — they are processed server-side in Shopify Admin. The correct production data flow is:

```
Shopify Admin (merchant processes refund)
  → Shopify fires refund webhook (POST to your server endpoint)
  → Server endpoint parses webhook payload
  → Server reads GA4 client_id from Shopify order metafield
  → Server sends GA4 Measurement Protocol v2 POST
  → GA4 records refund event
```

### Why Measurement Protocol v2 — not GTM

GTM runs in the customer's browser. When a merchant processes a refund in Shopify Admin, the customer is not on the website — there is no browser session to inject a dataLayer push into. The only viable path is server-to-server via MP v2. See ADR-0018.

### The `client_id` problem

GA4 Measurement Protocol v2 requires a `client_id` to attribute the refund event to a user and session. At refund time, the server has no access to the customer's browser cookies. The solution: **store the `client_id` as a Shopify order metafield at purchase time**.

In the Custom Pixel (`checkout_completed` handler — see 2.11), after parsing `client_id` from `browser.cookie.get('_ga')`, POST it to a server endpoint which saves it as an order metafield (namespace: `ga4`, key: `client_id`). When the refund webhook fires, the handler reads the metafield and includes the `client_id` in the MP v2 request.

If `client_id` is omitted, GA4 still records the revenue deduction but cannot attribute it to a session, acquisition channel, or user — defeating the purpose of tracking it.

### Measurement Protocol v2 Request

**Generate an API secret:** GA4 Admin → Data Streams → select stream → Measurement Protocol API secrets → Create. Keep this secret server-side only — never expose it in browser code.

**Endpoint:**
```
POST https://www.google-analytics.com/mp/collect?measurement_id=G-JPWF3JGT5P&api_secret=YOUR_API_SECRET
Content-Type: application/json
```

**Request body — partial refund example (jacket only):**
```json
{
  "client_id": "1234567890.1623456789",
  "events": [
    {
      "name": "refund",
      "params": {
        "transaction_id": "KVS-1001",
        "value": 65.00,
        "currency": "GBP",
        "items": [
          {
            "item_id": "44123456789",
            "item_name": "Kova Oversized Jacket",
            "price": 65.00,
            "quantity": 1
          }
        ]
      }
    }
  ]
}
```

**Validate before sending to production:** use the debug endpoint (same URL, replace `mp/collect` with `debug/mp/collect`) — it returns a JSON response with `validationMessages` without sending data to GA4.

---

## GTM Setup

[Screenshot: GTM workspace — Triggers list before editing]

### What Changed in This Subproject

No new tags, triggers, or variables were created. The only GTM change was adding `refund$` to the existing `CE - Ecommerce Events` regex trigger so `GA4 - Ecommerce Events` fires on refund events.

**Why no dedicated `GA4 - Event - refund` tag:** `GA4 - Ecommerce Events` fires on all events matched by the `CE - Ecommerce Events` regex with Send Ecommerce data enabled. Adding a dedicated `refund` tag would fire two GA4 hits per event. The correct approach is to extend the existing regex.

### Step-by-Step

1. Open **GTM-NMPNZ4TV** → **Triggers**
2. Click **CE - Ecommerce Events**
3. In the **Event name** regex field, append `|refund$` to the end of the existing pattern
4. Save
5. Enter Preview mode and validate (see Validation Steps)
6. Submit → publish as `v2.12.0 - Refund Tracking`

### Trigger Configuration

[Screenshot: CE - Ecommerce Events trigger settings panel showing the full updated regex]

**Trigger type:** Custom Event  
**Trigger name:** `CE - Ecommerce Events`  
**Event name (regex — updated in 2.12):**
```
view_item_list$|select_item$|view_item$|add_to_cart$|view_cart$|remove_from_cart$|begin_checkout$|purchase$|refund$
```
**Use regex matching:** ✅ enabled

### Tag Configuration

No new tag. The existing tag that fires on `refund`:

[Screenshot: GA4 - Ecommerce Events tag details panel — Firing Status: Succeeded, Event Name: "refund"]

**Tag name:** `GA4 - Ecommerce Events`  
**Tag type:** Google Analytics: GA4 Event  
**Measurement ID:** `{{Const - GA4 Measurement ID}}` → `G-JPWF3JGT5P`  
**Event Name:** `{{Event}}` (resolves to `"refund"` at runtime)  
**Send Ecommerce data:** ✅ true  
**Data source:** `dataLayer`  
**Firing trigger:** `CE - Ecommerce Events` (now includes `refund$`)

### Variable Configuration

N/A — no new variables created in this subproject.

### GTM Version

**Version name:** `v2.12.0 - Refund Tracking`  
**Export:** `project-ecommerce/gtm/GTM-NMPNZ4TV_v2.12.0.json`

---

## GA4 Configuration

[Screenshot: GA4 Realtime overview showing `refund` event in "Event count by Event name" with `transaction_id` parameter visible]

- **Event name:** `refund`
- **Marked as conversion:** No — refunds are revenue corrections, not conversion signals
- **Custom dimensions registered:** None — `transaction_id` is already registered as an event-scoped custom dimension from 2.11

---

## Google Ads Configuration

**No Google Ads configuration in this subproject.**

The `refund` dataLayer push is GA4-only. No Google Ads tag fires on `refund` — the `CE - Ecommerce Events (Remarketing)` regex trigger used by `GAds - Remarketing` does not include `refund$`, by design.

> **Google Ads conversion retractions** require a separate offline process: Conversion Adjustment Upload (type: `RETRACTING`) via the Google Ads API or CSV, matched against the original GCLID. This is not connected to the `refund` dataLayer event and is not covered in this subproject.

---

## Validation Steps

### Client-side simulation (QA path — 🟨 Simulated)

Refunds are server-side events in production. The console simulation validates that GTM routes the event correctly and GA4 receives it — it does not exercise the production webhook path.

[Screenshot: GTM Preview event list — left panel showing event #12 `refund` selected]

1. Open **GTM-NMPNZ4TV** → **Preview** → connect to `https://fashion-sandbox.myshopify.com`
2. Navigate to any storefront page (GTM fires on all pages)
3. Open **DevTools → Console** (ensure context is the main page frame, not an iframe)
4. Paste and run the **full refund** snippet
5. In GTM Preview left panel, click the new **`refund`** event

[Screenshot: GTM Preview → Tags tab showing `GA4 - Ecommerce Events` Fired ✅ and `GAds - Remarketing` absent from Tags Fired]

6. **Tags tab:** confirm `GA4 - Ecommerce Events` → **Fired ✅**
7. Confirm `GAds - Remarketing` is **not** in Tags Fired

[Screenshot: GTM Preview → Data Layer tab showing full refund payload — `transaction_id: "KVS-1001"`, `value: 95`, two items]

8. **Data Layer tab:** confirm `transaction_id: "KVS-1001"`, `value: 95`, `currency: "GBP"`, two items
9. Open **GA4 → Reports → Realtime overview**

[Screenshot: GA4 Realtime showing `refund` in "Event count by Event name" with `transaction_id` visible on drill-down]

10. Confirm `refund` appears in "Event count by Event name"
11. Paste and run the **partial refund** snippet
12. Repeat GTM Preview check: `GA4 - Ecommerce Events` **Fired ✅**, `value: 65`, single item in Data Layer

> **GA4 Realtime note:** two `refund` hits fired within seconds in the same session may show as count 1 in the Realtime overview. This is a Realtime sampling behaviour — not a tracking failure. GTM Preview is the authoritative confirmation.

---

## QA Checklist

- [x] `refund$` added to `CE - Ecommerce Events` regex trigger
- [x] `GA4 - Ecommerce Events` fires on `refund` event (GTM Preview ✅)
- [x] `GAds - Remarketing` does NOT fire on `refund` event (GTM Preview ✅)
- [x] Full refund: `value: 95.00`, both items in `items` array (Data Layer tab ✅)
- [x] Partial refund: `value: 65.00`, jacket only in `items` array (Data Layer tab ✅)
- [x] `ecommerce: null` push precedes each refund push
- [x] `transaction_id` visible in GA4 Realtime parameters
- [x] No duplicate GA4 hits (single tag fires — no dedicated refund tag added)
- [x] GTM container published as `v2.12.0 - Refund Tracking`
- [x] Container JSON exported to `project-ecommerce/gtm/GTM-NMPNZ4TV_v2.12.0.json`
- [ ] *(Production path — not built)* MP v2 debug endpoint validates payload
- [ ] *(Production path — not built)* `client_id` order metafield stored at purchase time

---

## Common Errors & Fixes

| Error / Symptom | Root Cause | Fix |
|----------------|------------|-----|
| `refund` appears in GTM Preview event list but no tag in "Tags Fired" | `refund$` missing from `CE - Ecommerce Events` regex — state before this subproject | Edit trigger and append `\|refund$` to the regex pattern |
| Two `refund` events fired but GA4 Realtime shows count of 1 | GA4 Realtime sampling for rapid consecutive hits in the same session | Not a tracking error — verify both events via GTM Preview Data Layer tab |
| Duplicate GA4 hits (two per `refund` push) | A dedicated `GA4 - Event - refund` tag was added alongside the existing catch-all | Remove the dedicated tag; `GA4 - Ecommerce Events` already handles `refund` |
| `value` in GA4 doesn't match expected refund amount | Partial refund `value` set to full order value | `value` must equal the sum of `price × quantity` for the refunded items only |
| MP v2 POST returns 200 OK but event never appears in GA4 | `api_secret` doesn't match the measurement ID's stream, or `client_id` format is wrong | Verify `api_secret` was generated under `G-JPWF3JGT5P` stream specifically; `client_id` must be in `"xxxxxxxxxx.xxxxxxxxxx"` format (parsed from `_ga` cookie) |
| MP v2 debug endpoint returns validation errors | Malformed payload — missing required field or wrong type | Read `validationMessages` array; common causes: missing `client_id`, `measurement_id` in query param vs body |

---

## Reusable Assets

- **GTM Container Export:** `project-ecommerce/gtm/GTM-NMPNZ4TV_v2.12.0.json`
- **Full refund snippet:** see Data Layer Specification above
- **Partial refund snippet:** see Data Layer Specification above
- **MP v2 debug endpoint:** `https://www.google-analytics.com/debug/mp/collect?measurement_id=G-JPWF3JGT5P&api_secret=YOUR_SECRET`

---

## Related Guides

- `guides/02-data-collection/gtm/ecommerce-events.md` — refund tracking section extracted from this subproject
- `project-ecommerce/docs/11-purchase-tracking.md` — purchase event schema; `transaction_id` from the original `purchase` must match `refund` exactly
- `project-ecommerce/docs/adr/0018-measurement-protocol-v2-for-server-side-refunds.md` — decision record for MP v2 as production path
- *(Future)* Google Ads Conversion Adjustment Upload — for retracting Google Ads conversions when a refund occurs
