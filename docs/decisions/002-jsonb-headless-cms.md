# ADR 002: PostgreSQL JSONB Storage for In-House Headless CMS

[← ADR index](README.md) · [← Main README](../../README.md)

- **Status:** Accepted and in production.
- **Context:** Marketing teams and store operators require continuous autonomy to update homepage hero banners, curate "Fragrance World" collections, modify promotional marquee texts, and revise legal compliance policies (Terms, Shipping, Returns) without requesting engineering code updates or triggering Vite application rebuilds and cloud deployments.
- **Decision:** Implement an in-house Headless CMS by storing polymorphic document structures inside a single PostgreSQL table (`page_content`) utilizing native **JSONB** columns. Provide dedicated visual editors in `AdminContent.jsx` (`HomeContentEditor`, `AboutContentEditor`, `PolicyContentEditor`) that execute direct upserts against `page_content` keyed by `page_name`. Rehydrate storefront pages through TanStack React Query with a 1-hour stale time and instant administrative mutation invalidation.
- **Consequences:**
  - Completely disintermediates recurring third-party SaaS CMS subscription costs ($50–$250/month for Sanity or Contentful).
  - Eliminates deployment latency: marketing copy updates are live instantly upon saving in the admin console.
  - Keeps media assets inside the shared Supabase Storage `products/cms/` bucket rather than splitting media across external CDNs.
  - High resilience: storefront pages merge server JSONB documents with bundled JavaScript defaults (`policy-content-defaults.js`), preventing page crashes even on unseeded databases.
- **Trade-offs:**
  - Lacks the advanced editorial collaborative workflows (draft/publish branching, field-level revision history) provided by enterprise commercial CMS suites.
  - Schema changes in nested JSONB structures require defensive client-side parsing and sensible defaults.
- **Alternatives:**
  - **Commercial Headless CMS (Sanity / Contentful):** Introduces recurring monthly SaaS subscriptions, API rate limits, and extra third-party client dependencies.
  - **Relational Entity-Attribute-Value (EAV) Modeling:** Introduces severe SQL query complexity, join proliferation, and slower performance compared to native PostgreSQL JSONB document retrieval.
