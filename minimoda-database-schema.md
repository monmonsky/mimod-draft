# DATABASE SCHEMA & ERD DOCUMENTATION
# MINIMODA E-COMMERCE PLATFORM

**Database:** PostgreSQL 16  
**Document Version:** 1.0  
**Last Updated:** December 2024

---

## TABLE OF CONTENTS

1. [Database Overview](#1-database-overview)
2. [ERD (Entity Relationship Diagram)](#2-erd-entity-relationship-diagram)
3. [Table Definitions](#3-table-definitions)
4. [Relationships & Foreign Keys](#4-relationships--foreign-keys)
5. [Index Strategy](#5-index-strategy)
6. [Sample Queries](#6-sample-queries)
7. [Migration Order](#7-migration-order)

---

## 1. DATABASE OVERVIEW

### Database Structure
```
minimoda_db
├── Schema: public (main tables)
├── Schema: audit (logging tables)
└── Extensions: uuid-ossp, pg_trgm, btree_gin
```

### Naming Conventions
- **Tables:** Plural, snake_case (e.g., `products`, `order_items`)
- **Columns:** Singular, snake_case (e.g., `user_id`, `created_at`)
- **Primary Keys:** `id` (UUID)
- **Foreign Keys:** `<table>_id` (e.g., `product_id`)
- **Indexes:** `idx_<table>_<column>` (e.g., `idx_products_slug`)

### PostgreSQL Data Types Used
```sql
UUID        -- Primary keys
VARCHAR     -- Short text (names, codes)
TEXT        -- Long text (descriptions)
NUMERIC     -- Money/prices (12,2)
INTEGER     -- Quantities, counts
BOOLEAN     -- Flags (is_active, is_primary)
TIMESTAMP   -- Dates with timezone
JSONB       -- Flexible data (metadata, settings)
TEXT[]      -- Arrays (tags, categories)
POINT       -- Geolocation coordinates
INET        -- IP addresses
```

---

## 2. ERD (ENTITY RELATIONSHIP DIAGRAM)

### Simplified ERD Text Representation

```
┌─────────────────────────────────────────────────────────────────────┐
│                         MINIMODA ERD                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  users (1)──────(M) customer_addresses                            │
│    │                                                              │
│    ├────(M) carts (1)──────(M) cart_items                        │
│    │                              │                               │
│    ├────(M) orders                │                               │
│    │          │                   │                               │
│    │          ├──(M) order_items  │                               │
│    │          │         │         │                               │
│    │          ├──(1) payments     │                               │
│    │          │         │         │                               │
│    │          └──(1) shipments    │                               │
│    │                    │         │                               │
│    └────(M) notifications         │                               │
│                                   │                               │
│  products (1)────(M) product_variants                             │
│    │                    │         │                               │
│    ├────(M) product_images       │                               │
│    │                              │                               │
│    ├────(M) product_categories    │                               │
│    │              │               │                               │
│    │         categories           │                               │
│    │                              │                               │
│    └──── brands                   │                               │
│                                   │                               │
│  coupons (1)──────(M) coupon_usages                              │
│                                                                     │
│  Legend: (1) = One, (M) = Many                                    │
└─────────────────────────────────────────────────────────────────────┘
```

### Relationship Types

| Relationship | Type | Description |
|--------------|------|-------------|
| **user → addresses** | 1:M | One user has many addresses |
| **user → orders** | 1:M | One user has many orders |
| **product → variants** | 1:M | One product has many variants |
| **product → images** | 1:M | One product has many images |
| **product ↔ categories** | M:M | Many-to-many via product_categories |
| **order → order_items** | 1:M | One order has many items |
| **order → payment** | 1:1 | One order has one payment |
| **cart → cart_items** | 1:M | One cart has many items |
| **variant → cart_items** | 1:M | One variant in many carts |
| **variant → order_items** | 1:M | One variant in many orders |

---

## 3. TABLE DEFINITIONS

### USER & AUTHENTICATION TABLES

#### 3.1 users
```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    phone VARCHAR(20),
    password VARCHAR(255) NOT NULL,
    role VARCHAR(20) DEFAULT 'customer' CHECK (role IN ('customer', 'admin', 'super_admin')),
    email_verified_at TIMESTAMP,
    phone_verified_at TIMESTAMP,
    last_login_at TIMESTAMP,
    status VARCHAR(20) DEFAULT 'active' CHECK (status IN ('active', 'suspended', 'deleted')),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**Purpose:** Store all users (customers and admins)  
**Key Fields:**
- `role`: Determines access level
- `status`: Account status management
- `email_verified_at`: Email verification tracking

#### 3.2 customer_addresses
```sql
CREATE TABLE customer_addresses (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    label VARCHAR(100), -- 'Home', 'Office', etc
    recipient_name VARCHAR(255) NOT NULL,
    phone VARCHAR(20) NOT NULL,
    province_id INTEGER NOT NULL,
    province_name VARCHAR(100),
    city_id INTEGER NOT NULL,
    city_name VARCHAR(100),
    subdistrict_id INTEGER,
    subdistrict_name VARCHAR(100),
    address TEXT NOT NULL,
    postal_code VARCHAR(10),
    coordinates POINT, -- PostgreSQL geographic point
    is_default BOOLEAN DEFAULT false,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**Purpose:** Multiple shipping addresses per user  
**Key Fields:**
- `province_id`, `city_id`: Links to RajaOngkir
- `coordinates`: For map integration
- `is_default`: Default shipping address

### PRODUCT CATALOG TABLES

#### 3.3 categories
```sql
CREATE TABLE categories (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(255) UNIQUE NOT NULL,
    parent_id UUID REFERENCES categories(id) ON DELETE CASCADE,
    image VARCHAR(500),
    description TEXT,
    sort_order INTEGER DEFAULT 0,
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**Purpose:** Hierarchical product categories  
**Key Fields:**
- `parent_id`: For subcategories (self-referencing)
- `slug`: SEO-friendly URL
- `sort_order`: Display ordering

#### 3.4 brands
```sql
CREATE TABLE brands (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(255) UNIQUE NOT NULL,
    logo VARCHAR(500),
    description TEXT,
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**Purpose:** Product brand management

#### 3.5 products
```sql
CREATE TABLE products (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(255) UNIQUE NOT NULL,
    description TEXT,
    brand_id UUID REFERENCES brands(id) ON DELETE SET NULL,
    age_min INTEGER, -- Minimum age (months)
    age_max INTEGER, -- Maximum age (months)
    tags TEXT[], -- PostgreSQL array
    status VARCHAR(20) DEFAULT 'active' CHECK (status IN ('active', 'inactive', 'draft')),
    seo_meta JSONB DEFAULT '{}', -- SEO metadata
    view_count INTEGER DEFAULT 0,
    is_featured BOOLEAN DEFAULT false,
    created_by UUID REFERENCES users(id),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**Purpose:** Main product information  
**Key Fields:**
- `tags`: Array of searchable tags
- `seo_meta`: JSONB for flexible SEO data
- `age_min/max`: Age range for kids clothing

#### 3.6 product_categories (Junction Table)
```sql
CREATE TABLE product_categories (
    product_id UUID REFERENCES products(id) ON DELETE CASCADE,
    category_id UUID REFERENCES categories(id) ON DELETE CASCADE,
    PRIMARY KEY (product_id, category_id)
);
```

**Purpose:** Many-to-many relationship between products and categories

#### 3.7 product_variants
```sql
CREATE TABLE product_variants (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_id UUID NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    sku VARCHAR(100) UNIQUE NOT NULL,
    size VARCHAR(50) NOT NULL,
    color VARCHAR(50),
    weight_gram INTEGER NOT NULL, -- For shipping calculation
    price NUMERIC(12,2) NOT NULL,
    compare_at_price NUMERIC(12,2), -- Original price for discount
    stock_quantity INTEGER DEFAULT 0,
    reserved_quantity INTEGER DEFAULT 0, -- Reserved in carts
    barcode VARCHAR(100),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**Purpose:** Product variations (size, color)  
**Key Fields:**
- `sku`: Unique stock keeping unit
- `weight_gram`: For shipping cost calculation
- `reserved_quantity`: Stock in active carts

#### 3.8 product_images
```sql
CREATE TABLE product_images (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_id UUID NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    url VARCHAR(500) NOT NULL,
    alt_text VARCHAR(255),
    is_primary BOOLEAN DEFAULT false,
    sort_order INTEGER DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**Purpose:** Multiple images per product

### SHOPPING CART TABLES

#### 3.9 carts
```sql
CREATE TABLE carts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    guest_token VARCHAR(100), -- For guest checkout
    expires_at TIMESTAMP, -- Cart expiration
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT cart_user_or_guest CHECK (
        (user_id IS NOT NULL AND guest_token IS NULL) OR
        (user_id IS NULL AND guest_token IS NOT NULL)
    )
);
```

**Purpose:** Shopping cart for users and guests  
**Key Fields:**
- `guest_token`: Supports guest checkout
- `expires_at`: Auto-cleanup old carts

#### 3.10 cart_items
```sql
CREATE TABLE cart_items (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    cart_id UUID NOT NULL REFERENCES carts(id) ON DELETE CASCADE,
    product_variant_id UUID NOT NULL REFERENCES product_variants(id),
    quantity INTEGER NOT NULL CHECK (quantity > 0),
    price_snapshot NUMERIC(12,2) NOT NULL, -- Price at time of adding
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**Purpose:** Items in shopping cart  
**Key Fields:**
- `price_snapshot`: Preserves price when added to cart

### ORDER TABLES

#### 3.11 orders
```sql
CREATE TABLE orders (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_code VARCHAR(50) UNIQUE NOT NULL, -- ORD-20241201-001
    user_id UUID REFERENCES users(id),
    
    -- Customer Info (for guest checkout)
    customer_name VARCHAR(255) NOT NULL,
    customer_email VARCHAR(255) NOT NULL,
    customer_phone VARCHAR(20) NOT NULL,
    
    -- Amounts
    subtotal NUMERIC(12,2) NOT NULL,
    shipping_cost NUMERIC(12,2) DEFAULT 0,
    discount_total NUMERIC(12,2) DEFAULT 0,
    tax_amount NUMERIC(12,2) DEFAULT 0,
    grand_total NUMERIC(12,2) NOT NULL,
    
    -- Status
    payment_status VARCHAR(20) DEFAULT 'pending' 
        CHECK (payment_status IN ('pending', 'paid', 'failed', 'refunded')),
    fulfillment_status VARCHAR(20) DEFAULT 'unfulfilled'
        CHECK (fulfillment_status IN ('unfulfilled', 'processing', 'packed', 'shipped', 'delivered', 'returned')),
    
    -- Shipping Info
    courier VARCHAR(50),
    service VARCHAR(100),
    airwaybill VARCHAR(100),
    shipping_address_json JSONB NOT NULL,
    
    -- Additional
    notes TEXT,
    cancelled_at TIMESTAMP,
    cancelled_reason TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**Purpose:** Main order information  
**Key Fields:**
- `order_code`: Human-readable order number
- `shipping_address_json`: Complete address snapshot
- Dual status tracking (payment & fulfillment)

#### 3.12 order_items
```sql
CREATE TABLE order_items (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id UUID NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
    product_variant_id UUID NOT NULL REFERENCES product_variants(id),
    
    -- Snapshot data (preserve at order time)
    product_name_snapshot VARCHAR(255) NOT NULL,
    size VARCHAR(50) NOT NULL,
    color VARCHAR(50),
    
    -- Pricing
    price NUMERIC(12,2) NOT NULL,
    quantity INTEGER NOT NULL CHECK (quantity > 0),
    weight_gram INTEGER NOT NULL,
    subtotal NUMERIC(12,2) NOT NULL,
    
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**Purpose:** Items within an order  
**Key Fields:**
- Snapshot fields preserve product info at order time

### PAYMENT & SHIPPING TABLES

#### 3.13 payments
```sql
CREATE TABLE payments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id UUID NOT NULL REFERENCES orders(id),
    provider VARCHAR(50) NOT NULL, -- 'midtrans', 'manual'
    transaction_id VARCHAR(100),
    method VARCHAR(50), -- 'bank_transfer', 'credit_card', 'e-wallet'
    amount NUMERIC(12,2) NOT NULL,
    status VARCHAR(20) DEFAULT 'pending'
        CHECK (status IN ('pending', 'processing', 'success', 'failed', 'expired', 'refunded')),
    raw_payload JSONB, -- Store complete payment gateway response
    paid_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**Purpose:** Payment transaction records  
**Key Fields:**
- `raw_payload`: Complete gateway response for debugging
- `transaction_id`: External payment reference

#### 3.14 payment_logs
```sql
CREATE TABLE payment_logs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    payment_id UUID NOT NULL REFERENCES payments(id),
    event VARCHAR(100) NOT NULL, -- 'webhook_received', 'status_change'
    data JSONB NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**Purpose:** Payment webhook and event logging

#### 3.15 shipments
```sql
CREATE TABLE shipments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id UUID NOT NULL REFERENCES orders(id),
    courier VARCHAR(50) NOT NULL, -- 'jne', 'jnt', 'sicepat'
    service VARCHAR(100) NOT NULL, -- 'REG', 'YES', 'OKE'
    cost NUMERIC(12,2) NOT NULL,
    etd VARCHAR(50), -- Estimated time delivery
    airwaybill VARCHAR(100),
    raw_tracking JSONB, -- Complete tracking data
    shipped_at TIMESTAMP,
    delivered_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**Purpose:** Shipping information and tracking

### PROMOTION TABLES

#### 3.16 coupons
```sql
CREATE TABLE coupons (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code VARCHAR(50) UNIQUE NOT NULL,
    description TEXT,
    type VARCHAR(20) CHECK (type IN ('percentage', 'amount')),
    value NUMERIC(12,2) NOT NULL,
    min_order_amount NUMERIC(12,2) DEFAULT 0,
    max_discount_amount NUMERIC(12,2), -- For percentage type
    max_uses INTEGER,
    per_user_limit INTEGER DEFAULT 1,
    used_count INTEGER DEFAULT 0,
    applicable_categories UUID[], -- Array of category IDs
    excluded_products UUID[], -- Array of product IDs
    start_at TIMESTAMP,
    end_at TIMESTAMP,
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**Purpose:** Discount coupon management  
**Key Fields:**
- Arrays for category/product restrictions
- Usage limits and tracking

#### 3.17 coupon_usages
```sql
CREATE TABLE coupon_usages (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    coupon_id UUID NOT NULL REFERENCES coupons(id),
    user_id UUID NOT NULL REFERENCES users(id),
    order_id UUID NOT NULL REFERENCES orders(id),
    discount_amount NUMERIC(12,2) NOT NULL,
    used_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**Purpose:** Track coupon usage per user/order

### SYSTEM TABLES

#### 3.18 activity_logs
```sql
CREATE TABLE activity_logs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    actor_type VARCHAR(50) NOT NULL, -- 'user', 'admin', 'system'
    actor_id UUID,
    action VARCHAR(100) NOT NULL, -- 'order.created', 'product.updated'
    object_type VARCHAR(50), -- 'order', 'product'
    object_id UUID,
    meta JSONB DEFAULT '{}', -- Additional data
    ip_address INET,
    user_agent TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**Purpose:** Audit trail for all system activities

#### 3.19 settings
```sql
CREATE TABLE settings (
    key VARCHAR(100) PRIMARY KEY,
    value JSONB NOT NULL,
    description TEXT,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**Purpose:** System configuration storage  
**Example Keys:**
- `shipping.origin`: Origin address for shipping
- `payment.midtrans.server_key`: Payment gateway config
- `email.smtp`: Email configuration

#### 3.20 notifications
```sql
CREATE TABLE notifications (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    type VARCHAR(50) NOT NULL, -- 'order', 'payment', 'shipping'
    title VARCHAR(255) NOT NULL,
    message TEXT NOT NULL,
    data JSONB DEFAULT '{}',
    read_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**Purpose:** User notification system

---

## 4. RELATIONSHIPS & FOREIGN KEYS

### Foreign Key Constraints Summary

```sql
-- User Relations
customer_addresses.user_id → users.id (CASCADE DELETE)
carts.user_id → users.id (CASCADE DELETE)
orders.user_id → users.id (SET NULL)
notifications.user_id → users.id (CASCADE DELETE)

-- Product Relations
products.brand_id → brands.id (SET NULL)
product_variants.product_id → products.id (CASCADE DELETE)
product_images.product_id → products.id (CASCADE DELETE)
product_categories.product_id → products.id (CASCADE DELETE)
product_categories.category_id → categories.id (CASCADE DELETE)

-- Cart Relations
cart_items.cart_id → carts.id (CASCADE DELETE)
cart_items.product_variant_id → product_variants.id (RESTRICT)

-- Order Relations
order_items.order_id → orders.id (CASCADE DELETE)
order_items.product_variant_id → product_variants.id (RESTRICT)
payments.order_id → orders.id (RESTRICT)
shipments.order_id → orders.id (RESTRICT)

-- Coupon Relations
coupon_usages.coupon_id → coupons.id (RESTRICT)
coupon_usages.user_id → users.id (RESTRICT)
coupon_usages.order_id → orders.id (RESTRICT)
```

### Cascade Rules

| Action | Rule | Description |
|--------|------|-------------|
| **CASCADE DELETE** | Parent delete → Children delete | User deleted → Addresses deleted |
| **SET NULL** | Parent delete → FK becomes NULL | Brand deleted → Product brand_id NULL |
| **RESTRICT** | Prevent parent deletion | Can't delete variant with orders |

---

## 5. INDEX STRATEGY

### Primary Indexes (Automatic)
```sql
-- All PRIMARY KEY columns are automatically indexed
```

### Performance Indexes
```sql
-- User & Auth
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_role ON users(role) WHERE role != 'customer';
CREATE INDEX idx_addresses_user_default ON customer_addresses(user_id) WHERE is_default = true;

-- Products
CREATE INDEX idx_products_slug ON products(slug);
CREATE INDEX idx_products_status ON products(status) WHERE status = 'active';
CREATE INDEX idx_products_featured ON products(is_featured) WHERE is_featured = true;
CREATE INDEX idx_variants_sku ON product_variants(sku);
CREATE INDEX idx_variants_product ON product_variants(product_id);
CREATE INDEX idx_variants_stock ON product_variants(product_id) WHERE stock_quantity > 0;

-- Categories
CREATE INDEX idx_categories_slug ON categories(slug);
CREATE INDEX idx_categories_parent ON categories(parent_id);

-- Orders
CREATE INDEX idx_orders_code ON orders(order_code);
CREATE INDEX idx_orders_user ON orders(user_id);
CREATE INDEX idx_orders_status ON orders(payment_status, fulfillment_status);
CREATE INDEX idx_orders_date ON orders(created_at DESC);

-- Cart
CREATE INDEX idx_carts_user ON carts(user_id);
CREATE INDEX idx_carts_guest ON carts(guest_token);
CREATE INDEX idx_carts_expires ON carts(expires_at);

-- Full Text Search
CREATE INDEX idx_products_search ON products 
    USING gin(to_tsvector('english', name || ' ' || COALESCE(description, '')));

-- JSONB Indexes
CREATE INDEX idx_products_seo ON products USING gin(seo_meta);
CREATE INDEX idx_orders_address ON orders USING gin(shipping_address_json);
CREATE INDEX idx_payments_payload ON payments USING gin(raw_payload);
```

### Composite Indexes
```sql
-- Frequently queried together
CREATE INDEX idx_orders_user_status ON orders(user_id, payment_status);
CREATE INDEX idx_products_brand_status ON products(brand_id, status);
CREATE INDEX idx_cart_items_cart_variant ON cart_items(cart_id, product_variant_id);
```

---

## 6. SAMPLE QUERIES

### Common Query Examples

#### 6.1 Get Product with All Details
```sql
-- Get product with variants, images, and categories
SELECT 
    p.*,
    b.name as brand_name,
    json_agg(DISTINCT pv.*) as variants,
    json_agg(DISTINCT pi.*) as images,
    array_agg(DISTINCT c.name) as categories
FROM products p
LEFT JOIN brands b ON p.brand_id = b.id
LEFT JOIN product_variants pv ON p.id = pv.product_id
LEFT JOIN product_images pi ON p.id = pi.product_id
LEFT JOIN product_categories pc ON p.id = pc.product_id
LEFT JOIN categories c ON pc.category_id = c.id
WHERE p.slug = 'kaos-anak-dinosaurus'
GROUP BY p.id, b.name;
```

#### 6.2 Cart with Items
```sql
-- Get cart items with product details
SELECT 
    ci.*,
    pv.size,
    pv.color,
    pv.price as current_price,
    pv.stock_quantity,
    p.name as product_name,
    p.slug as product_slug,
    pi.url as image_url
FROM cart_items ci
JOIN product_variants pv ON ci.product_variant_id = pv.id
JOIN products p ON pv.product_id = p.id
LEFT JOIN product_images pi ON p.id = pi.product_id AND pi.is_primary = true
WHERE ci.cart_id = $1;
```

#### 6.3 Order Summary
```sql
-- Get complete order with items
SELECT 
    o.*,
    json_agg(
        json_build_object(
            'product_name', oi.product_name_snapshot,
            'size', oi.size,
            'color', oi.color,
            'quantity', oi.quantity,
            'price', oi.price,
            'subtotal', oi.subtotal
        )
    ) as items,
    p.transaction_id,
    p.status as payment_status,
    s.courier,
    s.airwaybill,
    s.delivered_at
FROM orders o
LEFT JOIN order_items oi ON o.id = oi.order_id
LEFT JOIN payments p ON o.id = p.order_id
LEFT JOIN shipments s ON o.id = s.order_id
WHERE o.order_code = 'ORD-20241201-001'
GROUP BY o.id, p.transaction_id, p.status, s.courier, s.airwaybill, s.delivered_at;
```

#### 6.4 Product Search
```sql
-- Full text search with filters
SELECT 
    p.*,
    ts_rank(to_tsvector('english', p.name || ' ' || p.description), 
            plainto_tsquery('english', $1)) as rank
FROM products p
WHERE 
    to_tsvector('english', p.name || ' ' || p.description) 
    @@ plainto_tsquery('english', $1)
    AND p.status = 'active'
    AND EXISTS (
        SELECT 1 FROM product_variants pv 
        WHERE pv.product_id = p.id AND pv.stock_quantity > 0
    )
ORDER BY rank DESC, p.created_at DESC
LIMIT 20;
```

#### 6.5 Sales Report
```sql
-- Monthly sales summary
SELECT 
    DATE_TRUNC('month', o.created_at) as month,
    COUNT(DISTINCT o.id) as total_orders,
    COUNT(DISTINCT o.user_id) as unique_customers,
    SUM(o.grand_total) as revenue,
    AVG(o.grand_total) as avg_order_value
FROM orders o
WHERE 
    o.payment_status = 'paid'
    AND o.created_at >= NOW() - INTERVAL '12 months'
GROUP BY DATE_TRUNC('month', o.created_at)
ORDER BY month DESC;
```

#### 6.6 Low Stock Alert
```sql
-- Products with low stock
SELECT 
    p.name,
    pv.sku,
    pv.size,
    pv.color,
    pv.stock_quantity,
    pv.reserved_quantity,
    (pv.stock_quantity - pv.reserved_quantity) as available
FROM product_variants pv
JOIN products p ON pv.product_id = p.id
WHERE 
    (pv.stock_quantity - pv.reserved_quantity) < 5
    AND p.status = 'active'
ORDER BY available ASC, p.name;
```

---

## 7. MIGRATION ORDER

### Correct Migration Sequence

```sql
-- Phase 1: Independent Tables
1. CREATE EXTENSION "uuid-ossp";
2. CREATE TABLE users;
3. CREATE TABLE brands;
4. CREATE TABLE categories;
5. CREATE TABLE settings;

-- Phase 2: First Level Dependencies
6. CREATE TABLE customer_addresses;    -- needs users
7. CREATE TABLE products;               -- needs brands
8. CREATE TABLE coupons;

-- Phase 3: Second Level Dependencies
9. CREATE TABLE product_categories;     -- needs products, categories
10. CREATE TABLE product_variants;      -- needs products
11. CREATE TABLE product_images;        -- needs products
12. CREATE TABLE carts;                 -- needs users

-- Phase 4: Third Level Dependencies
13. CREATE TABLE cart_items;            -- needs carts, product_variants
14. CREATE TABLE orders;                -- needs users
15. CREATE TABLE notifications;         -- needs users

-- Phase 5: Order Dependencies
16. CREATE TABLE order_items;           -- needs orders, product_variants
17. CREATE TABLE payments;              -- needs orders
18. CREATE TABLE payment_logs;          -- needs payments
19. CREATE TABLE shipments;             -- needs orders
20. CREATE TABLE shipment_trackings;    -- needs shipments
21. CREATE TABLE coupon_usages;         -- needs coupons, users, orders

-- Phase 6: Logging (can be anytime)
22. CREATE TABLE activity_logs;

-- Phase 7: Indexes (after all tables)
23. CREATE all indexes;
```

### Rollback Sequence
```sql
-- Drop in reverse order to avoid foreign key conflicts
DROP TABLE IF EXISTS activity_logs CASCADE;
DROP TABLE IF EXISTS coupon_usages CASCADE;
DROP TABLE IF EXISTS shipment_trackings CASCADE;
-- ... continue in reverse
```

---

## APPENDIX A: DATABASE SIZING ESTIMATES

### Storage Estimates (1 Year)

| Table | Avg Row Size | Est. Rows/Year | Storage |
|-------|--------------|----------------|---------|
| users | 500 bytes | 10,000 | 5 MB |
| products | 2 KB | 1,000 | 2 MB |
| product_variants | 200 bytes | 5,000 | 1 MB |
| orders | 1 KB | 30,000 | 30 MB |
| order_items | 200 bytes | 90,000 | 18 MB |
| cart_items | 150 bytes | 100,000 | 15 MB |
| activity_logs | 500 bytes | 500,000 | 250 MB |
| **TOTAL** | | | **~350 MB** |

*Note: Actual size will be larger with indexes (typically 2-3x)*

### Performance Targets

| Operation | Target Time | Query Type |
|-----------|------------|------------|
| Product listing | < 100ms | SELECT with JOIN |
| Cart operations | < 50ms | INSERT/UPDATE |
| Order creation | < 200ms | Transaction |
| Search | < 200ms | Full-text search |
| Reports | < 1s | Aggregation |

---

## APPENDIX B: COMMON PITFALLS & SOLUTIONS

### Pitfall 1: N+1 Query Problem
**Problem:** Loading products with images separately  
**Solution:** Use JSON aggregation or eager loading

### Pitfall 2: Slow Category Tree
**Problem:** Recursive queries for category hierarchy  
**Solution:** Use materialized path or nested set model

### Pitfall 3: Cart Abandonment
**Problem:** Carts never cleaned up  
**Solution:** Scheduled job to delete expired carts

### Pitfall 4: Stock Inconsistency
**Problem:** Race condition on stock updates  
**Solution:** Use row-level locking or optimistic locking

### Pitfall 5: Large JSONB Fields
**Problem:** Slow queries on large JSON  
**Solution:** Extract frequently queried fields to columns

---

## APPENDIX C: BACKUP & MAINTENANCE

### Backup Strategy
```bash
# Daily backup
pg_dump -h localhost -U minimoda -d minimoda_db -F custom -f backup_$(date +%Y%m%d).dump

# Restore
pg_restore -h localhost -U minimoda -d minimoda_db -v backup_20241201.dump
```

### Maintenance Tasks
```sql
-- Weekly
VACUUM ANALYZE;

-- Monthly
REINDEX DATABASE minimoda_db;

-- Quarterly
VACUUM FULL;
```

### Monitoring Queries
```sql
-- Table sizes
SELECT 
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS size
FROM pg_tables
WHERE schemaname = 'public'
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC;

-- Slow queries
SELECT 
    query,
    calls,
    mean_exec_time,
    total_exec_time
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 10;
```

---

**END OF DATABASE SCHEMA DOCUMENTATION**

*This document should be updated whenever schema changes are made.*

© 2024 Minimoda Development Team