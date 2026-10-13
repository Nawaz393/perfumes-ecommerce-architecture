# Supabase Security & Row Level Security (RLS) Policies

[← Main README](../README.md) · [← Documentation Index](README.md)

This document details the multi-role security architecture, Row Level Security (RLS) enforcement, and GoTrue token verification implemented across all 12 PostgreSQL tables in **LUXINPERFUMES.COM**.

---

## 1. Security Philosophy: Fail-Closed Defense

Because Supabase exposes database tables directly over HTTPS via PostgREST, client-facing access is strictly controlled by **PostgreSQL Row Level Security (RLS)**. By default, every table is configured with:

```sql
ALTER TABLE products ENABLE ROW LEVEL SECURITY;
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;
ALTER TABLE order_items ENABLE ROW LEVEL SECURITY;
ALTER TABLE profiles ENABLE ROW LEVEL SECURITY;
ALTER TABLE page_content ENABLE ROW LEVEL SECURITY;
```

If no explicit policy matches a request, PostgreSQL defaults to an empty set (`0 rows returned`), ensuring a zero-trust, fail-closed architecture.

---

## 2. Role Model & Claim Verification

Supabase GoTrue injects user claims into `auth.jwt()`. Roles are assigned in `raw_app_meta_data`:
- **Anonymous / Public User**: No authenticated session; possesses `anon` role. Can only execute `SELECT` queries on approved public storefront resources.
- **Authenticated Customer**: Has `authenticated` role and unique `auth.uid()`. Can read/write their own cart, orders, and profile.
- **Admin Operator**: Possesses `app_metadata.role = 'admin'`. Possesses full read/write access across CMS blocks, orders, stock, and settings.

### Helper Function for Role Extraction

```sql
CREATE OR REPLACE FUNCTION auth.is_admin()
RETURNS boolean AS $$
BEGIN
  RETURN (
    coalesce(
      auth.jwt() -> 'app_metadata' ->> 'role',
      ''
    ) = 'admin'
  );
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;
```

---

## 3. Core Policy Implementations

### 3.1 Fragrance Catalog (`products`)

- **Public Storefront Policy**: Anyone can read active products.
  ```sql
  CREATE POLICY "Public products view"
  ON products FOR SELECT
  USING (is_active = true);
  ```
- **Admin Mutation Policy**: Only admins can insert, update, or delete inventory.
  ```sql
  CREATE POLICY "Admin manage products"
  ON products FOR ALL
  USING (auth.is_admin())
  WITH CHECK (auth.is_admin());
  ```

### 3.2 Customer Orders (`orders`)

- **Customer Ownership Policy**: Customers can only view and insert their own orders.
  ```sql
  CREATE POLICY "Customer view own orders"
  ON orders FOR SELECT
  USING (auth.uid() = user_id OR auth.is_admin());

  CREATE POLICY "Customer insert own orders"
  ON orders FOR INSERT
  WITH CHECK (auth.uid() = user_id);
  ```
- **Update Lockout**: Only admins can update order statuses (e.g. from `pending` to `shipped`).
  ```sql
  CREATE POLICY "Admin update orders"
  ON orders FOR UPDATE
  USING (auth.is_admin())
  WITH CHECK (auth.is_admin());
  ```

### 3.3 Headless CMS Blocks (`page_content`)

- **Public Storefront Read**:
  ```sql
  CREATE POLICY "Public read published CMS"
  ON page_content FOR SELECT
  USING (is_published = true);
  ```
- **Admin Full Control**:
  ```sql
  CREATE POLICY "Admin manage CMS"
  ON page_content FOR ALL
  USING (auth.is_admin())
  WITH CHECK (auth.is_admin());
  ```

Related: [Architecture](architecture.md) · [Checkout & Inventory](checkout-and-inventory.md) · [Code Examples](code-examples.md)
