# Use first_available_variant.id as item_id in view_item_list Items Arrays

On Collection Pages, the Search Results page, and homepage sections, `item_id` in the `view_item_list` Items Array is populated with `product.selected_or_first_available_variant.id` (the variant ID), not the parent product ID.

On product detail pages, `item_id` is always a variant ID (set at page load and updated on variant selection). If list pages used the parent product ID instead, the `item_id` would differ across the funnel: `view_item_list` would carry a product ID while downstream `view_item`, `add_to_cart`, and `purchase` events carry a variant ID. GA4 uses `item_id` as the joining key for funnel analysis and cannot match mismatched IDs. Google Merchant Center product feeds are also synced at the variant level using GMC Composite IDs (`shopify_GB_{product_id}_{variant_id}`), so a parent product ID in the list event would fail to match the GMC catalog and break dynamic remarketing. `price` is sourced from the same variant (`first_available_variant.price`) to keep the event payload internally consistent — a variant ID paired with a `price_min` derived from a different variant would be contradictory.

## Considered Options

- **Parent product ID:** Simpler (no variant resolution needed), but breaks funnel matching and GMC remarketing.
- **`first_available_variant.id`:** Requires variant resolution in Liquid, but maintains consistent `item_id` across the entire GA4 funnel.
