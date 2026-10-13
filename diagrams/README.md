# Architecture Diagrams

[← Main README](../README.md) · [Documentation Index](../docs/README.md)

This directory contains standalone, reusable [Mermaid](https://mermaid.js.org/) diagram definitions modeling the architecture, component topology, data schemas, and lifecycle workflows of the Luxin Perfumes digital commerce platform.

## Available Diagrams

| Diagram | Source File | Description |
|---|---|---|
| **System Context** | [system-context.mmd](system-context.mmd) | High-level topology across React SPA clients, Vite CDN, Supabase PostgreSQL/Auth/Storage, Node/Express API bridge, and Stripe. |
| **Component Architecture** | [component-architecture.mmd](component-architecture.mmd) | React 18 component tree, route guards, public vs. admin layouts, and dynamic theme switching. |
| **Supabase Database Schema (ERD)** | [supabase-erd.mmd](supabase-erd.mmd) | Entity-relationship model of the 12 core tables including JSONB columns for olfactory notes and variants. |
| **Headless CMS Lifecycle** | [headless-cms-lifecycle.mmd](headless-cms-lifecycle.mmd) | In-house dynamic content authoring flow from admin editors through Supabase Storage/JSONB to TanStack Query hydration. |

## Rendering Notes

GitHub automatically renders Mermaid definitions embedded in Markdown documents. To view or edit these `.mmd` files locally or in documentation portals:
- Use the Mermaid CLI (`mmdc`): `npx @mermaid-js/mermaid-cli -i system-context.mmd -o system-context.svg`
- Or paste the contents into the [Mermaid Live Editor](https://mermaid.live).
