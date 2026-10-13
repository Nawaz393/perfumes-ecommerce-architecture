# In-House Headless JSONB Content Management System

[← Main README](../README.md) · [← Documentation index](README.md)

This deep dive details the architecture, design choices, and data modeling of the **custom headless CMS engine** designed for **Luxin Perfumes (LUXINPERFUMES.COM)**.

---

## 1. Architectural Motivation: SaaS Disintermediation

Traditional e-commerce platforms often integrate third-party Headless CMS providers (such as Sanity, Contentful, or Strapi) to empower non-technical administrators to manage marketing banners, brand narratives, and legal policies.

In the context of Luxin Perfumes, this introduced distinct liabilities:
- **Recurring Cost Overheads:** Monthly SaaS tier costs ranging from $50 to $250+/month for standard e-commerce copy.
- **Deployment Coupling:** Hardcoding marketing copy into React JSX components required a Vite rebuild and Vercel/Cloudflare redeployment for minor text or promotional edits.
- **Data Fragmentation:** Fragmented product imagery and CMS media across external CDNs vs. the primary Supabase storage bucket.

### The Architectural Solution
By leveraging **PostgreSQL JSONB** columns inside a dedicated `page_content` table coupled with Supabase Storage, the platform provides a zero-dependency in-house CMS. Store operators modify home hero banners, curated fragrance worlds, infinite marquee tickers, and legal policies in real time with **instant zero-downtime updates**.

```mermaid
flowchart LR
    subgraph AdminConsole["Admin Experience (AdminContent.jsx)"]
        UI_EDITOR["Visual Editor Form\n(Home, About, Brands, Policies)"]
        IMAGE_UPLOADER["CmsImageUpload.jsx\n(Drag & Drop Asset Uploader)"]
    end

    subgraph StoragePipeline["Media Pipeline"]
        S3_STORAGE["Supabase Storage ('products/cms/')"]
        CDN["Public CDN Asset URL"]
    end

    subgraph Persistence["Database Engine"]
        PG_TABLE[("page_content Table\n(Keyed by page_name)")]
        JSONB_DOC["JSONB Document Tree\n(Arbitrary Nested Objects & Arrays)"]
    end

    subgraph StorefrontDelivery["Storefront Hydration (Public Pages)"]
        REACT_QUERY["TanStack Query Cache\n(staleTime: 1 hour)"]
        STOREFRONT_PAGES["HomePage, AboutPage, TermsPage"]
    end

    UI_EDITOR -->|Upload Media| IMAGE_UPLOADER
    IMAGE_UPLOADER -->|uploadContentImage()| S3_STORAGE
    S3_STORAGE -->|getPublicUrl()| CDN
    CDN -->|Embed URL| UI_EDITOR

    UI_EDITOR -->|updatePageContent()| PG_TABLE
    PG_TABLE -->|ON CONFLICT (page_name) DO UPDATE| JSONB_DOC

    STOREFRONT_PAGES -->|usePageContent(pageName)| REACT_QUERY
    REACT_QUERY -->|Cached Read / PostgREST| PG_TABLE
    PG_TABLE -->|JSONB Response| REACT_QUERY
    REACT_QUERY -->|Hydrate Virtual DOM| STOREFRONT_PAGES
```

---

## 2. PostgreSQL Schema & Document Topology

The CMS architecture centers around a single normalized table in PostgreSQL:

```sql
CREATE TABLE page_content (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  page_name VARCHAR(100) UNIQUE NOT NULL,
  content JSONB NOT NULL DEFAULT '{}'::jsonb,
  updated_at TIMESTAMPTZ DEFAULT timezone('utc'::text, now()) NOT NULL
);

-- Indexing for fast key-based lookups
CREATE UNIQUE INDEX idx_page_content_name ON page_content(page_name);
```

### Document Schemas

#### 1. Home Page Schema (`page_name = 'home'`)
Models marketing hero banners, the infinite marquee, and the curated fragrance worlds:

```json
{
  "heroTitle": "LUXINPERFUMES.COM",
  "heroSubtitle": "Haute Parfumerie & Pure Extraits",
  "heroDescription": "Exclusive Dubai Oud, Taif Rose, and French Prestige Fragrances.",
  "heroImage": "https://.../cms/hero_banner.webp",
  "marqueeItems": [
    "SAME-DAY DUBAI & UAE DELIVERY",
    "100% GUARANTEED AUTHENTIC",
    "ROYAL COMPLIMENTARY GIFT WRAPPING",
    "EXCLUSIVE DUBAI OUD SIGNATURES"
  ],
  "signatureWorlds": [
    {
      "id": "for-her",
      "title": "For Her",
      "subtitle": "SEDUCTIVE & FLORAL SIGNATURES",
      "description": "Enchanting blends of Taif Rose, Jasmin de Grasse, and warm vanilla orchid.",
      "link": "/products?gender=Women",
      "badge": "Feminine Icons",
      "image": "https://.../cms/for_her.webp",
      "notes": ["Taif Rose", "Jasmin de Grasse", "Vanilla Orchid"]
    },
    {
      "id": "oud-oriental",
      "title": "Oud & Oriental",
      "subtitle": "THE HERITAGE NICHE COLLECTION",
      "description": "Aged Cambodian Oud, golden saffron, and mystical frankincense.",
      "link": "/products?category=Oud",
      "badge": "Royal Heritage",
      "image": "https://.../cms/oud_oriental.webp",
      "notes": ["Cambodian Oud", "Golden Saffron", "Frankincense"]
    }
  ]
}
```

#### 2. Legal Policy Schema (`page_name = 'terms'`, `'refund-policy'`, `'shipping'`)
Models structured compliance documentation:

```json
{
  "title": "Terms & Conditions",
  "lastUpdated": "01/11/2025",
  "introText": "These Terms and Conditions govern your use of the LUXINPERFUMES.COM website...",
  "sections": [
    {
      "title": "1. Intellectual Property",
      "body": "The Service and its original content are the exclusive property of Prime Choose Perfumes...",
      "listType": "none",
      "items": []
    },
    {
      "title": "2. Product Information & Pricing",
      "body": "",
      "listType": "bullet",
      "items": [
        "Descriptions: We strive for accuracy in olfactory descriptions.",
        "Pricing: All prices in United Arab Emirates Dirham (AED)."
      ]
    }
  ]
}
```

---

## 3. Schema Fallback & Zero-Downtime Resilience

A common pitfall with headless CMS architectures is application crash during missing database rows (e.g., initial system bootstrap or catastrophic accidental deletion).

Luxin Perfumes implements a **Two-Tier Resilient Fallback Pattern**:
1. **PGRST116 Trap:** In `src/services/content.js`, if Supabase PostgREST returns error code `PGRST116` (no rows found), the service gracefully returns an empty object rather than throwing.
2. **Default Bundles:** Storefront pages merge server JSONB documents with bundled JavaScript defaults (`src/lib/policy-content-defaults.js`).

```javascript
// Example resilient hydration pattern
const { data: serverContent, isLoading } = usePageContent('terms');
const content = {
  ...termsContentDefaults,
  ...(serverContent || {}),
};
```

Even if the database is offline or unseeded, the application renders complete, legally compliant terms without breaking the UI.

---

## 4. Media Asset Pipeline

CMS image uploads follow an isolated storage workflow:
1. File uploaded via `CmsImageUpload.jsx`.
2. Renamed using timestamp and crypto-random string: `${Date.now()}_${Math.random().toString(36).substring(2, 15)}.${fileExt}`.
3. Uploaded to Supabase Object Storage bucket `products` with path `cms/<fileName>`.
4. Cache-Control header configured to `3600` (1 hour browser cache).
5. Public CDN URL retrieved via `supabase.storage.from('products').getPublicUrl(filePath)`.
6. URL embedded directly into the page's JSONB document.

---

Related: [Architecture](architecture.md) · [Code Examples](code-examples.md) · [ADR 002: JSONB Headless CMS](decisions/002-jsonb-headless-cms.md)
