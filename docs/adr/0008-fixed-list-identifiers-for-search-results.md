# Fixed list identifiers for the Search Results page

The Search Results page fires `view_item_list` with fixed identifiers (`item_list_id: "search_results"`, `item_list_name: "Search Results"`) regardless of the search query, rather than embedding the query in the list identity.

Baking the query into the list identity (e.g. `item_list_name: "Search: red dress"`) causes every unique query — including typos, pluralisations, and long-tail variants — to create a separate row in GA4's Item List reports. This makes it impossible to answer the primary business question: "How does search as a channel perform compared to collection pages?" The aggregate signal is destroyed by cardinality. GA4's data model separates the location of the interaction (the list identity) from the user's intent (the query). The query is captured independently via GA4's automatically collected `search_term` parameter, which is the correct mechanism for per-query analysis. Fixed identifiers keep list-level reporting coherent and allow per-query analysis to happen in the `search_term` dimension without polluting the list reports.

## Considered Options

- **Query-scoped identifiers:** Granular per-query, but destroys aggregate list performance metrics.
- **Fixed identifiers:** Aggregates all search impressions and clicks into one list, enabling meaningful List CTR benchmarking against collection pages.
