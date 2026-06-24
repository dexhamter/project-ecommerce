# view_item re-fires on variant switch

The `view_item` event re-fires each time the user selects a different variant on a product detail page, not only on initial page load.

Dynamic remarketing requires the exact variant ID to match the Google Merchant Center feed entry (`shopify_GB_{product_id}_{variant_id}`). If only the page-load variant is pushed, the remarketing tag passes the wrong ID whenever the user views a different variant — Google Ads falls back to generic brand ads instead of serving the specific product. Variant prices also differ, so a page-load-only push sends inaccurate `value` data to Smart Bidding. Re-firing on the `variant:changed` DOM event keeps both the remarketing audience membership and the value signal accurate.

## Considered Options

**Page-load only** — simpler, no listener required. Rejected because it silently corrupts dynamic remarketing for any user who views a variant other than the page default.

**Re-fire on variant:changed** — chosen. Introduces a 500ms debounce to suppress rapid multi-click noise, and a Variant ID Comparison Guard to prevent duplicate pushes on no-op selections and potential init-time fires.
