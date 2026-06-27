# Dedicated GTM Container for the Custom Pixel Sandbox

Once the decision to run GTM inside the Custom Pixel sandbox was made (ADR-0015), a second choice arose: reuse the existing Storefront Container (GTM-NMPNZ4TV) or create a dedicated container.

Reusing GTM-NMPNZ4TV would cause every tag with an "Initialization - All Pages" or "All Pages" trigger to fire the moment the sandbox iframe initialises: `GA4 - Config` sends a phantom page_view from the sandbox URL, `GAds - Remarketing` fires with no ecommerce data, `GAds - Conversion Linker` fires in an environment where it has no useful effect. Mitigating this requires blocking triggers on each affected tag — technical debt that grows with every future tag added to the storefront container, because each addition must be assessed for whether it also needs a sandbox exception.

We use a dedicated container (the Checkout Pixel Container) that contains only what checkout tracking requires. The sandbox initialises a clean container; no storefront tags fire; no blocking triggers accumulate; no phantom data reaches GA4 or Google Ads. The Storefront Container export remains a clean record of storefront measurement only.

**Considered alternatives**: Reuse GTM-NMPNZ4TV with blocking triggers and a URL-cleaning Custom JavaScript variable. Viable, but creates ongoing maintenance overhead and structural phantom-data risk. The dedicated container eliminates the problem at the source.

**Consequences**: Variables shared between containers (Measurement ID, Conversion ID) must be defined independently in each. The Checkout Pixel Container version history is independent of the Storefront Container — first published version is `v1.0.0 - Purchase Tracking`.
