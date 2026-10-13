# Representative Code Examples

[← Main README](../README.md) · [← Documentation index](README.md)

This document provides production code examples from the luxury fragrance e-commerce platform, illustrating architectural patterns, Supabase data access, client-side caching, and transactional safety.

---

## 1. Supabase Client Isolation & Environment Configuration

The application encapsulates the Supabase JS client as a singleton, pulling credentials from Vite's ESM environment variables.

```javascript
// src/lib/supabaseClient.js
import { createClient } from '@supabase/supabase-js';

const supabaseUrl = import.meta.env.VITE_SUPABASE_URL;
const supabaseAnonKey = import.meta.env.VITE_SUPABASE_ANON_KEY;

export const supabase = createClient(supabaseUrl, supabaseAnonKey);
```

---

## 2. Session-Bearing Axios Interceptor for Express Bridge

When calling the Node.js/Express API bridge (for Stripe checkout creation and secure server tasks), an Axios request interceptor dynamically injects the active Supabase JWT session token.

```javascript
// src/lib/axios.js
import axios from 'axios';
import { supabase } from './supabaseClient';

const api = axios.create({
  baseURL: import.meta.env.VITE_API_BASE_URL,
  withCredentials: true,
  timeout: 45000,
});

api.interceptors.request.use(async (config) => {
  try {
    const {
      data: { session },
    } = await supabase.auth.getSession();
    if (session?.access_token) {
      config.headers = {
        ...config.headers,
        Authorization: `Bearer ${session.access_token}`,
      };
    }
  } catch (err) {
    console.error('Failed to attach Supabase session to request:', err);
  }
  return config;
});

export default api;
```

---

## 3. Headless CMS Hydration & Cache Invalidation Hook

Custom React Query hooks manage the lifecycle of JSONB page documents. A 1-hour stale time prevents redundant database reads, while mutations instantly invalidate the target cache key.

```javascript
// src/hooks/content.jsx
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { getPageContent, updatePageContent, uploadContentImage } from '@/services/content';

/**
 * Fetch dynamic JSONB content for a specific storefront page
 * Cached for 1 hour to optimize database bandwidth
 */
export const usePageContent = (pageName) => {
  return useQuery({
    queryKey: ['page_content', pageName],
    queryFn: () => getPageContent(pageName),
    staleTime: 1000 * 60 * 60, // 1-hour TTL
  });
};

/**
 * Upsert dynamic CMS content and immediately invalidate the page cache
 */
export const useUpdatePageContent = () => {
  const queryClient = useQueryClient();
  
  return useMutation({
    mutationFn: updatePageContent,
    onSuccess: (data, variables) => {
      // Invalidate the specific page's cache for instant client rehydration
      queryClient.invalidateQueries(['page_content', variables.pageName]);
    },
  });
};
```

---

## 4. PostgreSQL JSONB Querying: Sizing & Stock Availability

Product variants (bottle sizes, prices, and stock counts) are modeled in a JSONB array (`sizes`). PostgREST operators query deep into the document structure for filtering and ordering.

```javascript
// src/services/product.js (Representative Extract)
import { supabase } from '@/lib/supabaseClient';

export const getProducts = async ({
  search = '',
  brand = 'all',
  category = 'all',
  priceRange = [0, 999999],
  inStock = false,
  sortBy = 'name',
}) => {
  let query = supabase.from('products').select('*', { count: 'exact' });

  // Filter across JSONB variant prices (indices 0 to 4)
  const priceConditions = [0, 1, 2, 3, 4]
    .map((i) => `and(sizes->${i}->price.gte.${priceRange[0]},sizes->${i}->price.lte.${priceRange[1]})`)
    .join(',');
  query = query.or(priceConditions);

  // In-stock filter checking variant stock > 0
  if (inStock) {
    query = query.or(
      'sizes->0->stock.gt.0,sizes->1->stock.gt.0,sizes->2->stock.gt.0,sizes->3->stock.gt.0'
    );
  }

  // Dynamic sorting on JSONB base variant price
  switch (sortBy) {
    case 'price-low':
      query = query.order('sizes->0->price', { ascending: true });
      break;
    case 'price-high':
      query = query.order('sizes->0->price', { ascending: false });
      break;
    default:
      query = query.order('name', { ascending: true });
  }

  const { data, error, count } = await query;
  if (error) throw error;
  return { data, count };
};
```

---

## 5. Cart Controller & Variant Stock Validation

The `useCartController` hook orchestrates multi-variant bottle additions, stock boundary enforcement, coupon application, and browser storage persistence.

```javascript
// src/hooks/useCartController.jsx (Representative Extract)
export default function useCartController() {
  const { user } = useAuth();
  const { data: cart = [], isLoading } = useCart(user?.id);
  const updateQuantityMutation = useUpdateCartQuantity();
  const applyCartCouponMutation = useApplyCartCoupon();

  const handleUpdateQuantity = async (productId, size, newQuantity) => {
    if (!user?.id) return;

    const item = cart.find((i) => i.product_id === productId && i.size === size);
    if (!item) return;

    // Enforce variant inventory invariant
    if (newQuantity > item.stock) {
      toast({
        title: 'Stock Limit Reached',
        description: `Only ${item.stock} units available for ${size}.`,
        variant: 'destructive',
      });
      return;
    }

    await updateQuantityMutation.mutateAsync({
      userId: user.id,
      productId,
      size,
      quantity: newQuantity,
    });
  };

  return {
    cart,
    handleUpdateQuantity,
    isLoading,
  };
}
```

---

## 6. Server-Side Cart Pricing & Coupon RPCs

To prevent client-side price tampering, all cart subtotal, discount, and loyalty point calculations are evaluated in PostgreSQL stored procedures.

```javascript
// src/services/cart.js
import { supabase } from '@/lib/supabaseClient';

export async function calculateCartTotal(userId) {
  const { data, error } = await supabase
    .rpc('calculate_cart_total', { p_user_id: userId });
  if (error || !data) throw new Error('Failed to calculate cart total');
  return data;
}

export async function applyCouponToCart(userId, couponCode) {
  const { data, error } = await supabase
    .rpc('apply_coupon', { p_user_id: userId, p_coupon_code: couponCode });
  if (error || !data) throw new Error('Invalid or expired coupon code');
  return data;
}
```

---

Related: [Architecture](architecture.md) · [Supabase Security & RLS](supabase-security-and-rls.md) · [Headless JSONB CMS](headless-jsonb-cms.md)
