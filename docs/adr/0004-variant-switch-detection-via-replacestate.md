# Variant switch detection via history.replaceState interception

Variant switches are detected by intercepting `history.replaceState` and parsing the new `?variant=` URL parameter, not by listening for a custom DOM event.

During implementation (2.6), it was confirmed that Dawn dispatches no `variant:changed` event on `document`, `product-info`, or `variant-selects` — all three were tested with explicit listeners and none fired on swatch click. Dawn's only observable signal for a variant switch is the URL update via `history.replaceState` (visible as `gtm.historyChange-v2` in the GTM dataLayer). Intercepting `replaceState` is therefore the only mechanism that doesn't depend on undocumented theme internals or polling.

Full variant data (price, title) is resolved from `window._kovaVariantsMap`, a lookup table pre-rendered by Liquid at page load time. This avoids an asynchronous AJAX call to `/products/[handle].js`, which would introduce a race condition against the `add_to_cart` intercept (2.8).

## Consequences

`history.replaceState` is patched at snippet load time. GTM also patches `replaceState` for its own history-change detection. Both patches chain correctly regardless of execution order (each stores a reference to the version it found and calls it through). `_origReplaceState` always fires immediately; only the tracking push is debounced.
