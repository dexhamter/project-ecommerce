# GTM Container Initialised Inside Custom Pixel Sandbox for Checkout Tracking

The Shopify thank-you page runs inside a checkout environment where the Storefront Container (GTM-NMPNZ4TV via `theme.liquid`) does not fire. Three approaches were considered for tracking the `purchase` event:

- **Path A — Direct fetch to sGTM**: Custom Pixel code manually constructs the full event payload and POSTs it to the Stape.io sGTM endpoint via `fetch()`.
- **Path B — postMessage bridge**: Custom Pixel sends a message to the parent storefront page, which relays it to `window.dataLayer`. Not viable — checkout is a new page load; no storefront JS is running to receive the message.
- **Path C — GTM container inside the sandbox**: A dedicated GTM container is initialised inside the Custom Pixel's own iframe. Checkout events are pushed to the sandbox's local `window.dataLayer` and processed by GTM.

We chose Path C. The Custom Pixel runs in a lax sandbox (an `<iframe>` with `allow-scripts`) that has its own isolated `window` and `document`. `document.createElement('script')` works; GTM initialises normally inside the iframe. This keeps tag management in the GTM interface — familiar workflow, full version history, and Custom Event triggers that work identically to storefront tag configuration.

Path A requires all event payload construction in raw pixel JavaScript. Any future schema change (new parameter, new event) requires editing the Custom Pixel in Shopify Admin and redeploying — there is no GTM abstraction layer. Path C keeps that abstraction in place.

GA4 client_id continuity is maintained by reading `_ga` via `browser.cookie.get()` (Shopify's sandbox-safe cookie API) and overriding the generated phantom ID via GTM's "Fields to Set" on the GA4 Config tag.
