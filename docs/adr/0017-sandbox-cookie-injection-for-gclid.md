# Sandbox document.cookie Injection for GCLID Attribution

The `GAds - Conversion - Purchase` tag inside the Checkout Pixel Container must attach the Google Click ID (GCLID) to the conversion hit so Google Ads can attribute the purchase to the originating ad click. The Google Ads tag reads GCLID from `document.cookie` via its own internal scraping behaviour — there is no GTM "Fields to Set" parameter that accepts a GCLID value from a Data Layer Variable.

The Custom Pixel sandbox is an isolated iframe. Its `document.cookie` is empty on initialisation; the parent storefront's `_gcl_aw` cookie is not inherited across the iframe boundary. If the conversion tag fires without `_gcl_aw` in the sandbox's `document.cookie`, the conversion is recorded but unattributed — Google Ads cannot link it to an ad click.

We use the `browser.cookie.get('_gcl_aw')` API (Shopify's sandbox-safe cross-boundary cookie reader) to retrieve the live `_gcl_aw` value from the storefront domain before GTM initialises. The retrieved value is then written to the sandbox's own `document.cookie`:

```js
if (gclAwRaw) {
  document.cookie = `_gcl_aw=${gclAwRaw}; path=/`;
}
```

The Checkout Pixel Container is then loaded. When the `GAds - Conversion - Purchase` tag fires on `checkout_completed`, it queries `document.cookie`, finds the manually planted `_gcl_aw`, and attaches the GCLID to the conversion hit exactly as it would on any standard storefront page.

`browser.cookie.get('_gcl_aw')` is preferred over reading `_gcl_aw` from `checkout.attributes` (the value written by the GCLID Cart Stamp, ADR documented in the Storefront Container). The Cart Stamp pattern was designed for architectures that have no access to the `browser` API — under Path C (ADR-0015) the `browser` API is available, providing the live cookie value directly without relying on the cart attribute write having succeeded or being current.

**Note**: if `browser.cookie.get('_gcl_aw')` returns undefined (non-paid traffic, ITP, cookie blocked), no cookie is written and the conversion is recorded without GCLID attribution — correct behaviour. There is no fallback to `checkout.attributes` for simplicity; Enhanced Conversions (2.14) provides email-based attribution as a supplementary signal for cases where GCLID is absent.
