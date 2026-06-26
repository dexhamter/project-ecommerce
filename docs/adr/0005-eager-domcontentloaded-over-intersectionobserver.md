# Eager DOMContentLoaded firing for view_item_list instead of IntersectionObserver

`view_item_list` events fire synchronously at `DOMContentLoaded` — one per product list section — rather than lazily via IntersectionObserver as sections scroll into the viewport.

IntersectionObserver offers more accurate impression counts (only fires for lists the user actually sees) but introduces an async race condition: a user can click an above-the-fold product before the observer callback fires, causing `select_item` to arrive in the dataLayer before `view_item_list`. GA4 processes events in strict push order and does not retroactively stitch out-of-order events, which permanently breaks List CTR for the affected session. Because List CTR (impressions → clicks) is the only reliable native metric for evaluating list performance in GA4 — downstream list-to-purchase attribution is not natively supported — a broken CTR is more damaging than overcounted impressions for below-fold sections that users never scroll to.

## Considered Options

- **IntersectionObserver (lazy):** Fires per-section as it enters the viewport. Accurate impression counts, but async — cannot guarantee event sequence on above-fold lists.
- **DOMContentLoaded (eager):** Fires all lists synchronously at parse completion. Overcounts impressions for below-fold lists but guarantees `view_item_list` always precedes any user click.

## Consequences

Below-fold list impression counts are inflated, which deflates raw List CTR metrics. This is accepted as the lesser trade-off versus broken attribution. The existing architectural pattern (Liquid-rendered data, synchronous push) is preserved.
