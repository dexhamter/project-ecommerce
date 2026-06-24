# window._kovaProductData is defined in the view_item snippet, not the add_to_cart snippet

`window._kovaProductData` — the page-level product data store — is defined by the `view_item` Tracking Snippet, even though the `add_to_cart` intercept (Subproject 2.8) is its primary consumer.

The `variant:changed` re-fire listener (added in 2.6) needs access to static parent product fields (`item_name`, `item_group_id`, `item_brand`, `item_category`) that cannot come from the DOM event detail. These fields must be pre-rendered by Liquid at page load time. Defining `window._kovaProductData` in the `view_item` block is the earliest point at which Liquid data is available on the page. The object then acts as a maintained single source of truth: the `variant:changed` listener updates its mutable fields (`item_id`, `price`, optionally `item_variant`) on each genuine switch, so that when the `add_to_cart` intercept runs in 2.8 it always reads the currently-selected variant — not the page-load default.

## Consequences

The `view_item` snippet ships with a variable that looks like it belongs to `add_to_cart`. This is intentional. The 2.8 snippet should consume `window._kovaProductData` with a fallback guard (`window._kovaProductData || {}`) rather than redefining it.
