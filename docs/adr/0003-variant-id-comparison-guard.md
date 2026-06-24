# Variant ID comparison guard over flag-based guard for view_item deduplication

The `variant:changed` listener suppresses duplicate `view_item` pushes by comparing the incoming variant ID against `window._kovaProductData.item_id`, rather than by checking a boolean "init done" flag.

The flag-based approach (`window._kovaViewItemInitDone = true`) is vulnerable to execution-order race conditions: if Dawn dispatches `variant:changed` synchronously during page initialisation before the Liquid script block has finished executing, the flag is unset and the listener fires a duplicate push. The ID comparison approach checks actual data state rather than execution timing — it is immune to this race condition. It also handles no-op re-selections (user clicks the already-selected swatch) for free, since the IDs will match and the push is skipped. As a side effect, it keeps `window._kovaProductData` in sync after each genuine switch, which the 2.8 `add_to_cart` intercept depends on.

## Implementation note

Both IDs must be cast to strings before comparison (`String(incomingId) === String(window._kovaProductData.item_id)`). Liquid outputs IDs as integers; Dawn's DOM event may output them as strings. Comparing without casting will cause false mismatches and fire duplicate events.
