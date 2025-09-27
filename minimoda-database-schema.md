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
    email_verified_at TIMESTAMP,
    phone_verified_at TIMESTAMP,
    last_login_at TIMESTAMP,
    last_login_ip INET,
    status VARCHAR(20) DEFAULT 'active' CHECK (status IN ('active', 'suspended', 'deleted')),
    two_factor_enabled BOOLEAN DEFAULT false,
    two_factor_secret VARCHAR(255),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**Purpose:** Store all users (customers and admins)  
**Key Fields:**
- `status`: Account status management
- `email_verified_at`: Email verification tracking
- `two_factor_enabled`: 2FA support

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

### ROLE & PERMISSION TABLES (RBAC)

#### 3.3 roles
```sql
CREATE TABLE roles (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(100) UNIQUE NOT NULL, -- 'super_admin', 'admin', 'staff', 'customer'
    display_name VARCHAR(100) NOT NULL,
    description TEXT,
    is_active BOOLEAN DEFAULT true,
    is_system BOOLEAN DEFAULT false, -- System roles can't be deleted
    priority INTEGER DEFAULT 0, -- Higher priority = more important role
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**Purpose:** Define system roles  
**Key Fields:**
- `is_system`: Protect default roles from deletion
- `priority`: Role hierarchy for conflict resolution

#### 3.4 permissions
```sql
CREATE TABLE permissions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(100) UNIQUE NOT NULL, -- 'product.create', 'order.view'
    display_name VARCHAR(100) NOT NULL,
    description TEXT,
    module VARCHAR(50) NOT NULL, -- 'product', 'order', 'user'
    action VARCHAR(50) NOT NULL, -- 'create', 'read', 'update', 'delete'
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**Purpose:** Define granular permissions  
**Key Fields:**
- `name`: Dot notation for permission (module.action)
- `module`: Group permissions by module
- `action`: CRUD actions

#### 3.5 modules
```sql
CREATE TABLE modules (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(100) UNIQUE NOT NULL, -- 'product_management', 'order_management'
    display_name VARCHAR(100) NOT NULL,
    description TEXT,
    icon VARCHAR(100), -- Icon class for UI
    parent_id UUID REFERENCES modules(id) ON DELETE CASCADE, -- For sub-modules
    route VARCHAR(255), -- Route/URL for the module
    component VARCHAR(255), -- Frontend component name
    sort_order INTEGER DEFAULT 0,
    is_active BOOLEAN DEFAULT true,
    is_visible BOOLEAN DEFAULT true, -- Show in menu
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**Purpose:** Define system modules/menus  
**Key Fields:**
- `parent_id`: Hierarchical menu structure
- `route`: URL/route for navigation
- `is_visible`: Control menu visibility

#### 3.6 user_roles (Junction Table)
```sql
CREATE TABLE user_roles (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role_id UUID NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    assigned_by UUID REFERENCES users(id),
    assigned_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    expires_at TIMESTAMP, -- Optional: temporary roles
    is_active BOOLEAN DEFAULT true,
    UNIQUE(user_id, role_id)
);
```

**Purpose:** Assign roles to users (many-to-many)  
**Key Fields:**
- `assigned_by`: Audit trail
- `expires_at`: Temporary role assignments

#### 3.7 role_permissions (Junction Table)
```sql
CREATE TABLE role_permissions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    role_id UUID NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    permission_id UUID NOT NULL REFERENCES permissions(id) ON DELETE CASCADE,
    granted_by UUID REFERENCES users(id),
    granted_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(role_id, permission_id)
);
```

**Purpose:** Assign permissions to roles

#### 3.8 role_modules (Junction Table)
```sql
CREATE TABLE role_modules (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    role_id UUID NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    module_id UUID NOT NULL REFERENCES modules(id) ON DELETE CASCADE,
    can_view BOOLEAN DEFAULT true,
    can_create BOOLEAN DEFAULT false,
    can_update BOOLEAN DEFAULT false,
    can_delete BOOLEAN DEFAULT false,
    can_export BOOLEAN DEFAULT false,
    custom_permissions JSONB, -- Additional module-specific permissions
    granted_by UUID REFERENCES users(id),
    granted_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(role_id, module_id)
);
```

**Purpose:** Control module access per role  
**Key Fields:**
- CRUD permissions per module
- `custom_permissions`: Flexible additional permissions

#### 3.9 user_permissions (Direct Permissions)
```sql
CREATE TABLE user_permissions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    permission_id UUID NOT NULL REFERENCES permissions(id) ON DELETE CASCADE,
    granted_by UUID REFERENCES users(id),
    granted_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    expires_at TIMESTAMP, -- Temporary permission
    is_granted BOOLEAN DEFAULT true, -- true = grant, false = revoke
    reason TEXT, -- Why this permission was granted/revoked
    UNIQUE(user_id, permission_id)
);
```

**Purpose:** Override permissions for specific users  
**Key Fields:**
- `is_granted`: Can grant or explicitly revoke permissions
- `expires_at`: Temporary permission overrides

#### 3.10 permission_groups
```sql
CREATE TABLE permission_groups (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(100) UNIQUE NOT NULL, -- 'product_full', 'order_read_only'
    display_name VARCHAR(100) NOT NULL,
    description TEXT,
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**Purpose:** Group related permissions

#### 3.11 permission_group_items (Junction Table)
```sql
CREATE TABLE permission_group_items (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    group_id UUID NOT NULL REFERENCES permission_groups(id) ON DELETE CASCADE,
    permission_id UUID NOT NULL REFERENCES permissions(id) ON DELETE CASCADE,
    UNIQUE(group_id, permission_id)
);
```

**Purpose:** Define which permissions belong to each group

#### 3.12 sessions
```sql
CREATE TABLE sessions (
    id VARCHAR(255) PRIMARY KEY,
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    ip_address INET,
    user_agent TEXT,
    payload TEXT,
    last_activity TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**Purpose:** Track user sessions for security

### PRODUCT CATALOG TABLES

#### 3.13 categories
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

#### 3.14 brands
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

#### 3.15 products
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

#### 3.16 product_categories (Junction Table)
```sql
CREATE TABLE product_categories (
    product_id UUID REFERENCES products(id) ON DELETE CASCADE,
    category_id UUID REFERENCES categories(id) ON DELETE CASCADE,
    PRIMARY KEY (product_id, category_id)
);
```

**Purpose:** Many-to-many relationship between products and categories

#### 3.17 product_variants
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

#### 3.18 product_images
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

#### 3.19 carts
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

#### 3.20 cart_items
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

#### 3.21 orders
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

#### 3.22 order_items
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

#### 3.23 payments
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

#### 3.24 payment_logs
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

#### 3.25 shipments
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

#### 3.26 coupons
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

#### 3.27 coupon_usages
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

#### 3.28 activity_logs
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

#### 3.29 settings
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

#### 3.30 notifications
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
-- User & RBAC Relations
customer_addresses.user_id → users.id (CASCADE DELETE)
user_roles.user_id → users.id (CASCADE DELETE)
user_roles.role_id → roles.id (CASCADE DELETE)
role_permissions.role_id → roles.id (CASCADE DELETE)
role_permissions.permission_id → permissions.id (CASCADE DELETE)
role_modules.role_id → roles.id (CASCADE DELETE)
role_modules.module_id → modules.id (CASCADE DELETE)
user_permissions.user_id → users.id (CASCADE DELETE)
user_permissions.permission_id → permissions.id (CASCADE DELETE)
permission_group_items.group_id → permission_groups.id (CASCADE DELETE)
permission_group_items.permission_id → permissions.id (CASCADE DELETE)
modules.parent_id → modules.id (CASCADE DELETE)
sessions.user_id → users.id (CASCADE DELETE)
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
CREATE INDEX idx_users_status ON users(status) WHERE status = 'active';
CREATE INDEX idx_addresses_user_default ON customer_addresses(user_id) WHERE is_default = true;

-- RBAC Indexes
CREATE INDEX idx_roles_active ON roles(is_active) WHERE is_active = true;
CREATE INDEX idx_permissions_module ON permissions(module);
CREATE INDEX idx_permissions_active ON permissions(is_active) WHERE is_active = true;
CREATE INDEX idx_modules_parent ON modules(parent_id);
CREATE INDEX idx_modules_active ON modules(is_active) WHERE is_active = true;
CREATE INDEX idx_user_roles_user ON user_roles(user_id) WHERE is_active = true;
CREATE INDEX idx_user_roles_role ON user_roles(role_id) WHERE is_active = true;
CREATE INDEX idx_role_permissions_role ON role_permissions(role_id);
CREATE INDEX idx_role_modules_role ON role_modules(role_id);
CREATE INDEX idx_user_permissions_user ON user_permissions(user_id);
CREATE INDEX idx_sessions_user ON sessions(user_id);
CREATE INDEX idx_sessions_last_activity ON sessions(last_activity);

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
CREATE INDEX idx_role_modules_custom ON role_modules USING gin(custom_permissions);
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

#### 6.7 User Permissions Check
```sql
-- Get all permissions for a user (including role-based and direct)
WITH user_role_permissions AS (
    -- Permissions from roles
    SELECT DISTINCT p.* 
    FROM permissions p
    JOIN role_permissions rp ON p.id = rp.permission_id
    JOIN user_roles ur ON rp.role_id = ur.role_id
    WHERE ur.user_id = $1 AND ur.is_active = true
        AND (ur.expires_at IS NULL OR ur.expires_at > NOW())
),
user_direct_permissions AS (
    -- Direct permissions
    SELECT p.*
    FROM permissions p
    JOIN user_permissions up ON p.id = up.permission_id
    WHERE up.user_id = $1 
        AND up.is_granted = true
        AND (up.expires_at IS NULL OR up.expires_at > NOW())
),
user_revoked_permissions AS (
    -- Explicitly revoked permissions
    SELECT permission_id
    FROM user_permissions
    WHERE user_id = $1 AND is_granted = false
)
SELECT DISTINCT * FROM (
    SELECT * FROM user_role_permissions
    UNION
    SELECT * FROM user_direct_permissions
) AS all_permissions
WHERE id NOT IN (SELECT permission_id FROM user_revoked_permissions);
```

#### 6.8 User Module Access
```sql
-- Get accessible modules for a user based on roles
SELECT DISTINCT 
    m.*,
    rm.can_view,
    rm.can_create,
    rm.can_update,
    rm.can_delete,
    rm.can_export,
    rm.custom_permissions
FROM modules m
JOIN role_modules rm ON m.id = rm.module_id
JOIN user_roles ur ON rm.role_id = ur.role_id
WHERE 
    ur.user_id = $1 
    AND ur.is_active = true
    AND m.is_active = true
    AND m.is_visible = true
ORDER BY m.sort_order, m.name;
```

#### 6.9 Role Hierarchy
```sql
-- Get role with all its permissions
SELECT 
    r.*,
    array_agg(
        json_build_object(
            'id', p.id,
            'name', p.name,
            'module', p.module,
            'action', p.action
        )
    ) as permissions
FROM roles r
LEFT JOIN role_permissions rp ON r.id = rp.role_id
LEFT JOIN permissions p ON rp.permission_id = p.id
WHERE r.id = $1
GROUP BY r.id;
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
6. CREATE TABLE roles;
7. CREATE TABLE permissions;
8. CREATE TABLE permission_groups;
9. CREATE TABLE modules;

-- Phase 2: First Level Dependencies
10. CREATE TABLE customer_addresses;    -- needs users
11. CREATE TABLE user_roles;            -- needs users, roles
12. CREATE TABLE role_permissions;      -- needs roles, permissions
13. CREATE TABLE role_modules;          -- needs roles, modules
14. CREATE TABLE user_permissions;      -- needs users, permissions
15. CREATE TABLE permission_group_items; -- needs permission_groups, permissions
16. CREATE TABLE sessions;               -- needs users
17. CREATE TABLE products;               -- needs brands
18. CREATE TABLE coupons;

-- Phase 3: Second Level Dependencies
19. CREATE TABLE product_categories;     -- needs products, categories
20. CREATE TABLE product_variants;      -- needs products
21. CREATE TABLE product_images;        -- needs products
22. CREATE TABLE carts;                 -- needs users

-- Phase 4: Third Level Dependencies
23. CREATE TABLE cart_items;            -- needs carts, product_variants
24. CREATE TABLE orders;                -- needs users
25. CREATE TABLE notifications;         -- needs users

-- Phase 5: Order Dependencies
26. CREATE TABLE order_items;           -- needs orders, product_variants
27. CREATE TABLE payments;              -- needs orders
28. CREATE TABLE payment_logs;          -- needs payments
29. CREATE TABLE shipments;             -- needs orders
30. CREATE TABLE shipment_trackings;    -- needs shipments
31. CREATE TABLE coupon_usages;         -- needs coupons, users, orders

-- Phase 6: Logging (can be anytime)
32. CREATE TABLE activity_logs;

-- Phase 7: Indexes (after all tables)
33. CREATE all indexes;
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

## APPENDIX B: DEFAULT RBAC DATA

### Default Roles
```sql
-- System default roles
INSERT INTO roles (name, display_name, description, is_system, priority) VALUES
('super_admin', 'Super Administrator', 'Full system access', true, 100),
('admin', 'Administrator', 'Admin panel access', true, 90),
('staff', 'Staff', 'Limited admin access', true, 50),
('customer', 'Customer', 'Customer account', true, 10);
```

### Default Modules
```sql
-- Admin modules
INSERT INTO modules (name, display_name, icon, route, sort_order) VALUES
('dashboard', 'Dashboard', 'fa-dashboard', '/admin/dashboard', 1),
('product_management', 'Products', 'fa-box', '/admin/products', 2),
('order_management', 'Orders', 'fa-shopping-cart', '/admin/orders', 3),
('customer_management', 'Customers', 'fa-users', '/admin/customers', 4),
('marketing', 'Marketing', 'fa-bullhorn', '/admin/marketing', 5),
('reports', 'Reports', 'fa-chart-bar', '/admin/reports', 6),
('settings', 'Settings', 'fa-cog', '/admin/settings', 7);
```

### Default Permissions
```sql
-- Product permissions
INSERT INTO permissions (name, display_name, module, action) VALUES
('product.view', 'View Products', 'product', 'read'),
('product.create', 'Create Products', 'product', 'create'),
('product.update', 'Update Products', 'product', 'update'),
('product.delete', 'Delete Products', 'product', 'delete'),
('product.export', 'Export Products', 'product', 'export');

-- Order permissions
INSERT INTO permissions (name, display_name, module, action) VALUES
('order.view', 'View Orders', 'order', 'read'),
('order.update', 'Update Orders', 'order', 'update'),
('order.delete', 'Delete Orders', 'order', 'delete'),
('order.export', 'Export Orders', 'order', 'export');

-- Customer permissions
INSERT INTO permissions (name, display_name, module, action) VALUES
('customer.view', 'View Customers', 'customer', 'read'),
('customer.create', 'Create Customers', 'customer', 'create'),
('customer.update', 'Update Customers', 'customer', 'update'),
('customer.delete', 'Delete Customers', 'customer', 'delete');
```

### Role-Permission Assignments
```sql
-- Super Admin gets all permissions
INSERT INTO role_permissions (role_id, permission_id)
SELECT 
    (SELECT id FROM roles WHERE name = 'super_admin'),
    id
FROM permissions;

-- Admin gets most permissions (except delete)
INSERT INTO role_permissions (role_id, permission_id)
SELECT 
    (SELECT id FROM roles WHERE name = 'admin'),
    id
FROM permissions
WHERE action != 'delete';

-- Staff gets view and update only
INSERT INTO role_permissions (role_id, permission_id)
SELECT 
    (SELECT id FROM roles WHERE name = 'staff'),
    id
FROM permissions
WHERE action IN ('read', 'update');
```

### Role-Module Access
```sql
-- Super Admin has full access to all modules
INSERT INTO role_modules (role_id, module_id, can_view, can_create, can_update, can_delete, can_export)
SELECT 
    (SELECT id FROM roles WHERE name = 'super_admin'),
    id,
    true, true, true, true, true
FROM modules;

-- Admin has full access except delete
INSERT INTO role_modules (role_id, module_id, can_view, can_create, can_update, can_delete, can_export)
SELECT 
    (SELECT id FROM roles WHERE name = 'admin'),
    id,
    true, true, true, false, true
FROM modules;

-- Staff has limited access
INSERT INTO role_modules (role_id, module_id, can_view, can_create, can_update, can_delete, can_export)
SELECT 
    (SELECT id FROM roles WHERE name = 'staff'),
    id,
    true, false, true, false, false
FROM modules
WHERE name IN ('dashboard', 'product_management', 'order_management');
```

---

## APPENDIX C: BACKUP & MAINTENANCE

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