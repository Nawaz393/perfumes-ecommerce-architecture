# Documentation Index

[← Main README](../README.md)

This documentation suite details the architectural design, data modeling, client-side caching, and implementation patterns of **Luxin Perfumes (LUXINPERFUMES.COM)**.

## Architecture Guides

- [Code Examples](code-examples.md) — Production code snippets covering Supabase client isolation, Axios JWT interceptors, CMS React Query hydration, and JSONB queries.
- [In-House Headless JSONB CMS](headless-jsonb-cms.md) — Architecture of the custom zero-dependency CMS engine, schema fallbacks, and real-time content publishing.
- [Architecture Decision Records (ADRs)](decisions/README.md) — Context, decisions, trade-offs, and alternatives for core architectural choices.
- [Diagram Source Files](../diagrams/README.md) — Reusable Mermaid source diagrams for system topology, component hierarchies, ERD, and CMS lifecycle.

## Architectural Decision Records

- [ADR 001 — Supabase Backend-as-a-Service Architecture](decisions/001-supabase-baas-architecture.md)
- [ADR 002 — PostgreSQL JSONB Storage for In-House Headless CMS](decisions/002-jsonb-headless-cms.md)
- [ADR 003 — Client-Side State Architecture and Caching Strategy](decisions/003-client-side-state-and-caching.md)
