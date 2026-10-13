# ADR 001: Supabase Backend-as-a-Service (BaaS) Architecture

[← ADR index](README.md) · [← Main README](../../README.md)

- **Status:** Accepted and in production.
- **Context:** The platform requires relational data modeling for products, orders, cart items, coupons, and customer profiles, paired with object storage for high-resolution perfume bottle photography and fine-grained authentication. Building and maintaining a standalone custom Node/Express backend with custom JWT sessions, migration pipelines, and file upload endpoints increases operational overhead, hosting infrastructure costs, and delivery timeline for a luxury fragrance commerce brand.
- **Decision:** Adopt **Supabase** (PostgreSQL 15+, GoTrue Auth, PostgREST, and Supabase Storage) as the primary Backend-as-a-Service. Leverage PostgREST for direct client-to-database queries protected by Row-Level Security (RLS) policies, and encapsulate sensitive business calculations (cart totals, coupon redemption, loyalty point calculations) within 9 PostgreSQL PL/pgSQL stored procedures (`RPCs`). Maintain a minimal Node.js/Express bridge exclusively for PCI-compliant Stripe session creation and webhook processing.
- **Consequences:** 
  - Significant reduction in infrastructure maintenance and backend boilerplate.
  - Sub-millisecond direct PostgreSQL queries over HTTP via PostgREST.
  - Native integration between Supabase Storage CDN and client image uploaders.
  - Zero database credential leakage to the frontend by relying exclusively on public anon keys and JWT-scoped RLS policies.
- **Trade-offs:** 
  - Business logic is partitioned between client React Query hooks and PostgreSQL stored procedures, requiring PL/pgSQL expertise for complex cart and discount procedures.
  - Vendor dependency on the Supabase platform, though mitigated by Supabase being open-source PostgreSQL.
- **Alternatives:**
  - **Full Custom Node/Express Monolith:** Requires dedicated server management, Dockerization, Redis session storage, and manual S3 AWS SDK integration.
  - **Serverless AWS Lambda + DynamoDB:** Schema rigidity unsuitable for multi-variant perfume pyramids, relational coupons, and complex filtering.
