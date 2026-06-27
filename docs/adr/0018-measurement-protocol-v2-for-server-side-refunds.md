# Measurement Protocol v2 as the production path for refund events

Shopify refund webhooks fire server-side when a merchant processes a return in the Shopify Admin. No browser session is present at that moment, so GTM cannot receive the event. The only viable production path is a server endpoint that receives the Shopify webhook payload and forwards a GA4 `refund` hit via Measurement Protocol v2.

The alternative — firing a client-side `dataLayer.push()` from the storefront — is architecturally impossible in production: the customer is not on the site when a refund is processed.

A third option (routing the webhook through a proxy that pushes to the storefront dataLayer) was rejected: it requires a live browser session, introduces a race condition on session state, and adds unnecessary infrastructure complexity.

The `client_id` required by MP v2 must be captured at purchase time and stored as a Shopify order metafield, so the webhook handler can retrieve it when the refund fires.

**For this project (2.12):** the production path is documented and the data shape is demonstrated, but no server endpoint is deployed. Implementation label: 🟨 Simulated.
