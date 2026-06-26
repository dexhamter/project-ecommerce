# window._kovaLists staging array for homepage product list data

Homepage sections that render product grids push their GA4 item data into `window._kovaLists` via an inline `<script>` block rendered by Liquid, rather than embedding data in `data-*` HTML attributes on the section container.

Product fields like `title` and `vendor` frequently contain apostrophes, quotation marks, and special characters. Liquid's `| json` filter safely escapes these for inline JavaScript output. Attempting to embed the same JSON into an HTML attribute would cause the `| json` filter's output to conflict with the attribute's surrounding quotes, requiring workarounds like base64 encoding. The inline `<script>` approach eliminates this class of encoding bugs entirely. It also mirrors the established `window._kovaProductData` pattern from `tracking-product.liquid` (ADR-0002), keeping page-level data in JavaScript memory rather than the DOM. Because inline scripts execute synchronously during HTML parsing, `window._kovaLists` is guaranteed to be fully populated before `DOMContentLoaded` fires, making it safe for the Tracking Snippet to read.

## Considered Options

- **`data-*` attribute on section container:** Requires base64 encoding or equivalent to safely embed JSON with special characters in HTML attributes.
- **`window._kovaLists` inline `<script>`:** Uses Liquid's `| json` filter natively, consistent with `window._kovaProductData`, no encoding issues.
