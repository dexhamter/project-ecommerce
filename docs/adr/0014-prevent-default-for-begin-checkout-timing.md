# Prevent Default Form Submit for begin_checkout Timing

Dawn's checkout buttons (`#CartDrawer-Checkout` in the cart drawer, `#checkout` on the cart page) are `<button type="submit" name="checkout">` elements inside native HTML forms. Submitting either form causes the browser to navigate immediately to `checkout.shopify.com`. A naive `dataLayer.push()` inside a `submit` listener is unreliable: GTM processes tags asynchronously after the push returns, and the browser may begin tearing down the JavaScript execution context before GTM has time to build the GA4 payload and hand it to `navigator.sendBeacon()`. In GTM Preview this manifests as tags stuck in "Still Running".

We prevent the default form submission (`e.preventDefault()`), push `begin_checkout` with GTM's `eventCallback` and `eventTimeout: 1000`, then programmatically call `form.submit()` — which bypasses submit event listeners and avoids an infinite loop — once the callback fires or 1000ms elapses. This guarantees GTM has completed tag execution before navigation begins, while capping the user-visible delay at one second in the worst case (ad-blocker or GTM failure). 1000ms was chosen over 2000ms to minimise checkout friction; increase only if event drop-off is observed in GTM Preview.

## Considered Options

**Simple push without preventDefault** — rejected. GTM's async processing creates a race condition with page navigation. Beacon transport protects the hit once queued, but not before GTM queues it.

**Click listener on the checkout button** — rejected in favour of a `submit` listener. A click fires regardless of whether the form will actually submit (future validation, third-party apps), and does not protect against double-clicks.

## Consequences

- A `checkoutFiring` boolean guard is required. Because the default is prevented and the actual navigation is delayed by up to 1000ms, repeated clicks during that window re-fire the `submit` listener. The guard swallows all submissions after the first. It never needs resetting — the page navigates away once `form.submit()` executes.

- A fallback fetch is required. `window._kovaCartState` is seeded asynchronously on page load. High-intent users who click checkout before the seed resolves would be silently dropped by a naive null bail. Instead, a `Promise.race([fetch('/cart.js').then(r => r.json()), timeout(1000)])` is used: if the fetch resolves in time, cart data is available for the event; if the timeout wins, the form submits without tracking rather than blocking the user.

- Covers both checkout surfaces. The same listener pattern is attached to `CartDrawer-Form` (cart drawer, available on all pages) and `cart` (main cart page at `/cart`). Both lead to the same checkout flow and must be instrumented identically.
