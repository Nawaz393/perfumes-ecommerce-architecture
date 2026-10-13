# ADR 003: Client-Side State Architecture & Caching Strategy

[← ADR index](README.md) · [← Main README](../../README.md)

- **Status:** Accepted and in production.
- **Context:** An ultra-luxury perfume e-commerce SPA involves varied state types: asynchronous server cache (product lists, categories, CMS content, user orders), persistent customer state across browser reloads (active cart items, applied coupons, loyalty redemption), and ephemeral UI state (dropdowns, dialogs, visual comparison drawers). Introducing heavy global state management libraries like Redux Toolkit can lead to unnecessary boilerplate, complex synchronization of server data, and potential cache inconsistency.
- **Decision:** Establish a dual state architecture:
  1. **Server State & Cache:** Use **TanStack React Query v5** (`@tanstack/react-query`) for all remote Supabase and API communications, configuring stratified `staleTime` values (1 hour for CMS content, 5 minutes for product catalog, instant invalidation for cart mutations).
  2. **Synchronous & Persistent Client State:** Use **React Context API** (`AuthContext`, `OrderContext`) combined with `localStorage` and `sessionStorage` synchronization within specialized domain controller hooks (`useCartController`).
- **Consequences:**
  - Zero Redux boilerplate (no actions, slices, or dispatch reducers for simple server queries).
  - Built-in deduplication of network requests, automatic background revalidation on window focus, and out-of-the-box optimistic updates.
  - Cart persistence survives browser refreshes and tab closures without requiring premature database mutations for anonymous shoppers.
- **Trade-offs:**
  - Developers must understand the distinction between server state (TanStack Query) and ephemeral client state (React Context/useState) to avoid duplicate state tracking.
  - Complex multi-step checkout state requires explicit storage synchronization rules.
- **Alternatives:**
  - **Redux Toolkit / RTK Query:** Heavy bundle size overhead and excessive boilerplate for an application where Supabase PostgREST already provides structured endpoints.
  - **Zustand:** Excellent client store, but requires manual handling of cache invalidation, background refetching, and request deduplication that TanStack Query solves natively.
