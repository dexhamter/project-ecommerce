# Data Flow Diagram Spec — Kova Studio Ecommerce Measurement

**Tool:** draw.io
**Output:** `data-flow-diagram.png` (export at 2x resolution)
**Orientation:** Landscape
**Layout:** Three vertical swim lanes + one shared destination layer at the bottom

---

## Overall Structure

```
┌─────────────────────────────────────────────────────────────────────┐
│                         DATA SOURCES                                │
│  ┌─────────────────────────┐   ┌─────────────────────────────────┐  │
│  │   LANE 1: STOREFRONT    │   │    LANE 2: CHECKOUT SANDBOX     │  │
│  │  (fashion-sandbox        │   │    (checkout.shopify.com)       │  │
│  │   .myshopify.com)        │   │                                 │  │
│  └─────────────────────────┘   └─────────────────────────────────┘  │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │              LANE 3: SERVER LAYER (Stape.io sGTM)           │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │              LAYER 4: DESTINATIONS                          │   │
│  └─────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Colour Palette

| Element | Colour | Hex |
|---------|--------|-----|
| Lane 1 background (storefront) | Light blue | `#E8F4FD` |
| Lane 2 background (checkout sandbox) | Light orange | `#FEF3E8` |
| Lane 3 background (sGTM server) | Light green | `#EAF6EA` |
| Layer 4 background (destinations) | Light grey | `#F5F5F5` |
| GTM node | Google blue | `#4285F4` |
| Shopify node | Shopify green | `#96BF48` |
| Custom Pixel node | Orange | `#F4A428` |
| sGTM node | Dark green | `#34A853` |
| GA4 node | Orange-red | `#EA4335` |
| Google Ads node | Google blue | `#4285F4` |
| dataLayer node | Dark grey | `#5F6368` |
| Arrow (standard data flow) | Dark grey | `#333333` |
| Arrow (fetch POST) | Orange | `#F4A428` |
| Arrow (blocked / not available) | Red dashed | `#EA4335` |
| Sandbox boundary | Orange dashed border | `#F4A428` |

---

## Lane 1 — Storefront (`fashion-sandbox.myshopify.com`)

### Components

**1. User Browser**
- Shape: Rectangle with rounded corners
- Label: `User Browser\nfashion-sandbox.myshopify.com`
- Icon: Browser/person icon (optional)
- Colour: White with blue border

**2. Shopify Theme**
- Shape: Rectangle
- Label: `Shopify Theme\n(theme.liquid)`
- Colour: Shopify green `#96BF48`, white text

**3. dataLayer**
- Shape: Cylinder or rectangle
- Label: `window.dataLayer`
- Sublabel: `view_item_list\nselect_item\nview_item\nadd_to_cart\nview_cart\nremove_from_cart`
- Colour: Dark grey `#5F6368`, white text

**4. GTM Web Container**
- Shape: Rectangle
- Label: `GTM Web Container\nKova Studio`
- Sublabel: `theme.liquid injection`
- Colour: Google blue `#4285F4`, white text

**5. GA4 Client (in GTM)**
- Shape: Small rectangle inside GTM node or connected to it
- Label: `GA4 Client\n(in GTM)`
- Colour: Orange-red `#EA4335`, white text

### Connections (Lane 1)

```
User Browser → Shopify Theme
  Label: "Page load / interaction"

Shopify Theme → dataLayer
  Label: "dataLayer.push()\necommerce events"
  Note: "ecommerce: null before each push"

dataLayer → GTM Web Container
  Label: "Custom Event triggers\nread DLV variables"

GTM Web Container → GA4 Client
  Label: "Master regex trigger\nGA4 Event tag"
```

---

## Lane 2 — Checkout Sandbox (`checkout.shopify.com`)

### Sandbox Boundary

Draw a **dashed orange border** around this entire lane with a label:

```
⚠️ Shopify Checkout Sandbox
GTM and gtag.js are BLOCKED here
(Checkout Extensibility restriction)
```

### Components

**6. Shopify Checkout Pages**
- Shape: Rectangle
- Label: `Shopify Checkout\ncheckout.shopify.com`
- Sublabel: `Checkout Extensibility\n(locked sandbox)`
- Colour: Shopify green `#96BF48`, white text

**7. Custom Pixel**
- Shape: Rectangle with dashed orange border
- Label: `Shopify Custom Pixel\n(Web Pixels API)`
- Sublabel: `analytics.subscribe()\ncheckout_started\ncheckout_shipping_info_submitted\npayment_info_submitted\ncheckout_completed`
- Colour: Orange `#F4A428`, white text

**8. Blocked GTM (crossed out)**
- Shape: Rectangle with a red ❌ overlay or red dashed border
- Label: `GTM / gtag.js`
- Sublabel: `❌ Unsupported here\n(data loss risk)`
- Colour: Light red / greyed out

### Connections (Lane 2)

```
Shopify Checkout Pages → Custom Pixel
  Label: "analytics.subscribe()\nnative Shopify events"
  Arrow: standard grey

Shopify Checkout Pages → Blocked GTM node
  Label: "❌ Cannot reach\ncheckout.shopify.com"
  Arrow: red dashed
```

---

## Lane 3 — Server Layer (Stape.io sGTM)

### Components

**9. sGTM Server Container**
- Shape: Large rectangle (server/cloud icon optional)
- Label: `Server-Side GTM\n(Stape.io — free plan)`
- Sublabel: `[container-id].stape.io\nReceives all events`
- Colour: Dark green `#34A853`, white text

**10. sGTM — GA4 Tag**
- Shape: Small rectangle inside or adjacent to sGTM node
- Label: `GA4 Tag\n(sGTM)`
- Colour: Orange-red `#EA4335`, white text

**11. sGTM — Google Ads Conversion Tag**
- Shape: Small rectangle inside or adjacent to sGTM node
- Label: `Google Ads\nConversion Tag (sGTM)`
- Colour: Google blue `#4285F4`, white text

### Connections INTO Lane 3

```
GA4 Client (Lane 1) → sGTM Server Container
  Label: "HTTPS\n(GA4 event payload)"
  Arrow: standard grey, solid

Custom Pixel (Lane 2) → sGTM Server Container
  Label: "fetch() POST\n(checkout events)"
  Arrow: ORANGE, solid, thicker weight
  Note: "Direct server call\nbypasses GTM entirely"
```

### Connections within Lane 3

```
sGTM Server Container → sGTM GA4 Tag
  Label: "event routing"

sGTM Server Container → sGTM Google Ads Tag
  Label: "purchase / begin_checkout\nconversion routing"
```

---

## Layer 4 — Destinations

### Components

**12. Google Analytics 4**
- Shape: Rectangle
- Label: `Google Analytics 4\nKova Studio property`
- Icon: GA4 logo (optional)
- Colour: Orange-red `#EA4335`, white text

**13. Google Ads**
- Shape: Rectangle
- Label: `Google Ads\n912-474-4048\n(AW-18244478477)`
- Sublabel: `Conversions:\n• Purchase (30-day window)\n• Begin Checkout (7-day window)`
- Icon: Google Ads logo (optional)
- Colour: Google blue `#4285F4`, white text

**14. Looker Studio** *(downstream — optional, lighter treatment)*
- Shape: Rectangle, lighter style
- Label: `Looker Studio\nReporting Dashboard`
- Colour: Grey outline, white fill

### Connections OUT of Lane 3 to Layer 4

```
sGTM GA4 Tag → Google Analytics 4
  Label: "Measurement Protocol\n(all ecommerce events)"
  Arrow: standard grey

sGTM Google Ads Tag → Google Ads
  Label: "Conversion hits\n(purchase + begin_checkout)"
  Arrow: standard grey

Google Analytics 4 → Google Ads
  Label: "GA4 ↔ Google Ads link\n(audiences + reporting)"
  Arrow: bidirectional, grey dashed

Google Analytics 4 → Looker Studio
  Label: "GA4 connector"
  Arrow: grey, thinner weight

Google Ads → Looker Studio
  Label: "Google Ads connector"
  Arrow: grey, thinner weight
```

---

## Annotations / Call-out Boxes

Add these as call-out shapes (speech bubble or rectangle with coloured border) near the relevant nodes:

**Call-out 1** (near Custom Pixel → sGTM arrow):
```
⚡ fetch() POST — not a script load
Bypasses sandbox restrictions.
No GTM or gtag.js inside the pixel.
```

**Call-out 2** (near sGTM node):
```
🛡️ First-party server
• Bypasses ad blockers
• Extended cookie lifetimes
• 20–40% more conversion data
  vs client-only tracking
```

**Call-out 3** (near Blocked GTM node in Lane 2):
```
⚠️ Common mistake
Loading GTM or gtag() here
causes silent data loss.
Google Support won't help.
```

**Call-out 4** (near GA4 Client → sGTM arrow):
```
ecommerce: null pushed
before every event to
prevent parameter bleed
```

---

## Legend (bottom-right corner)

```
Legend:
─────  Standard data flow
━━━━━  fetch() POST (direct to server)
- - -  Bidirectional link / reference
- - -  Blocked / not available (red)

[  ]   Client-side component
[  ]   Server-side component
[  ]   Destination platform
⚠️     Restriction / warning
```

---

## Export Settings

- **Format:** PNG
- **Resolution:** 2x (for retina screenshots)
- **Filename:** `data-flow-diagram.png`
- **Save to:** `project-ecommerce/screenshots/01-measurement-plan/`
- **Also export as:** `data-flow-diagram.drawio` (source file, same folder)

---

## Notes on Layout

- Place Lane 1 (Storefront) on the **left**
- Place Lane 2 (Checkout Sandbox) in the **centre**
- Place Lane 3 (sGTM Server) on the **right**, slightly below Lanes 1 and 2 — it receives from both
- Place Layer 4 (Destinations) at the **bottom**, spanning full width
- The orange `fetch()` arrow from Custom Pixel to sGTM should be the most visually prominent arrow in the diagram — it is the key architectural decision
- The red dashed "blocked" arrow and the crossed-out GTM node in Lane 2 should be immediately readable as "wrong path"
