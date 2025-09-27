# TECHNICAL IMPLEMENTATION GUIDE
# MINIMODA E-COMMERCE PLATFORM
## PRACTICAL ARCHITECTURE APPROACH

**Version:** 2.0  
**Date:** December 2024  
**Document Type:** Practical Technical Specification  
**Approach:** Simplified Architecture for 2-Person Team

---

## TABLE OF CONTENTS

1. [Development Environment Setup](#1-development-environment-setup)
2. [Database Implementation with RBAC](#2-database-implementation-with-rbac)
3. [Laravel Project Structure (Practical)](#3-laravel-project-structure-practical)
4. [API Specification](#4-api-specification)
5. [Security & RBAC Implementation](#5-security--rbac-implementation)
6. [Testing Strategy (Simplified)](#6-testing-strategy-simplified)
7. [Deployment Guide](#7-deployment-guide)
8. [Code Standards (Practical)](#8-code-standards-practical)

---

## 1. DEVELOPMENT ENVIRONMENT SETUP

### 1.1 Required Software

| Software | Version | Purpose | Priority |
|----------|---------|---------|----------|
| PHP | 8.3+ | Runtime environment | Required |
| Composer | 2.6+ | PHP dependency manager | Required |
| PostgreSQL | 16 | Database server | Required |
| Redis | 7+ | Cache and queue | Required |
| Node.js | 20 LTS | Frontend tooling | Required |
| Docker | 24+ | Containerization | Optional |
| Git | 2.40+ | Version control | Required |

### 1.2 Quick Setup Guide

```bash
# 1. Create Laravel project
composer create-project laravel/laravel minimoda-backend

# 2. Essential packages only
composer require laravel/sanctum
composer require intervention/image
composer require maatwebsite/excel

# 3. Development helpers (optional)
composer require barryvdh/laravel-debugbar --dev
composer require barryvdh/laravel-ide-helper --dev
```

### 1.3 Environment Configuration

```env
# .env - Minimal configuration
APP_NAME="Minimoda"
APP_ENV=local
APP_URL=http://localhost:8000

# PostgreSQL configuration
DB_CONNECTION=pgsql
DB_HOST=127.0.0.1
DB_PORT=5432
DB_DATABASE=minimoda_db
DB_USERNAME=postgres
DB_PASSWORD=password

# Simple cache & session
CACHE_DRIVER=file
SESSION_DRIVER=file
QUEUE_CONNECTION=database

# Redis (optional for better performance)
REDIS_HOST=127.0.0.1
REDIS_PORT=6379

# Payment & Shipping
MIDTRANS_SERVER_KEY=
MIDTRANS_IS_PRODUCTION=false
RAJAONGKIR_API_KEY=
```

---

## 2. DATABASE IMPLEMENTATION WITH RBAC

### 2.1 Database Setup

```sql
-- Create database with UUID support
CREATE DATABASE minimoda_db;
\c minimoda_db;
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
```

### 2.2 Migration Order (With RBAC)

```
Phase 1: Core Tables
├── 001_create_users_table
├── 002_create_roles_table
├── 003_create_permissions_table
├── 004_create_modules_table
├── 005_create_categories_table
├── 006_create_brands_table

Phase 2: RBAC Relations
├── 007_create_user_roles_table
├── 008_create_role_permissions_table
├── 009_create_role_modules_table
├── 010_create_user_permissions_table

Phase 3: Business Tables
├── 011_create_products_table
├── 012_create_product_variants_table
├── 013_create_carts_table
├── 014_create_orders_table
├── 015_create_payments_table
```

### 2.3 RBAC Seeder Data

```php
// database/seeders/RbacSeeder.php
// Create default roles
$roles = [
    ['name' => 'super_admin', 'display_name' => 'Super Administrator'],
    ['name' => 'admin', 'display_name' => 'Administrator'],
    ['name' => 'staff', 'display_name' => 'Staff'],
    ['name' => 'customer', 'display_name' => 'Customer']
];

// Create default modules
$modules = [
    ['name' => 'dashboard', 'route' => '/admin/dashboard'],
    ['name' => 'products', 'route' => '/admin/products'],
    ['name' => 'orders', 'route' => '/admin/orders'],
    ['name' => 'customers', 'route' => '/admin/customers']
];

// Assign permissions
// super_admin → all permissions
// admin → all except delete
// staff → view and update only
// customer → no admin access
```

---

## 3. LARAVEL PROJECT STRUCTURE (PRACTICAL)

### 3.1 Simplified Folder Structure

```
minimoda-backend/
├── app/
│   ├── Constants/          # Simple constants
│   │   ├── OrderStatus.php
│   │   └── PaymentStatus.php
│   │
│   ├── Helpers/           # Helper functions
│   │   ├── ApiResponse.php
│   │   ├── Format.php
│   │   └── Upload.php
│   │
│   ├── Http/
│   │   ├── Controllers/
│   │   │   ├── Api/       # Customer API
│   │   │   │   ├── AuthController.php
│   │   │   │   ├── ProductController.php
│   │   │   │   ├── CartController.php
│   │   │   │   └── OrderController.php
│   │   │   ├── Admin/     # Admin panel
│   │   │   │   ├── DashboardController.php
│   │   │   │   ├── ProductController.php
│   │   │   │   └── OrderController.php
│   │   │   └── Webhook/
│   │   │       └── MidtransController.php
│   │   │
│   │   ├── Middleware/
│   │   │   ├── CheckPermission.php
│   │   │   └── CheckRole.php
│   │   │
│   │   └── Requests/      # Validation
│   │       └── StoreProductRequest.php
│   │
│   ├── Models/
│   │   ├── Traits/
│   │   │   └── HasUuid.php
│   │   ├── User.php
│   │   ├── Role.php
│   │   ├── Permission.php
│   │   ├── Product.php
│   │   └── Order.php
│   │
│   ├── Repositories/      # Simple repositories
│   │   ├── ProductRepository.php
│   │   ├── OrderRepository.php
│   │   └── UserRepository.php
│   │
│   └── Services/          # Complex logic only
│       ├── CartService.php
│       ├── CheckoutService.php
│       ├── PaymentService.php
│       └── RbacService.php
│
├── database/
│   ├── migrations/
│   └── seeders/
│
├── routes/
│   ├── api.php
│   ├── admin.php
│   └── web.php
│
└── config/
    ├── rbac.php          # RBAC configuration
    └── minimoda.php      # App configuration
```

### 3.2 Model Implementation with RBAC

```php
// app/Models/User.php
class User extends Authenticatable
{
    use HasUuid;
    
    // RBAC relationships
    public function roles()
    {
        return $this->belongsToMany(Role::class, 'user_roles')
            ->wherePivot('is_active', true)
            ->where(function($q) {
                $q->whereNull('expires_at')
                  ->orWhere('expires_at', '>', now());
            });
    }
    
    public function permissions()
    {
        // Get permissions through roles
        return $this->hasManyThrough(Permission::class, Role::class);
    }
    
    public function hasPermission($permission)
    {
        return $this->permissions->contains('name', $permission);
    }
    
    public function hasRole($role)
    {
        return $this->roles->contains('name', $role);
    }
}
```

### 3.3 Repository Pattern (Simplified)

```php
// app/Repositories/ProductRepository.php
class ProductRepository
{
    protected $model;
    
    public function __construct(Product $model)
    {
        $this->model = $model;
    }
    
    public function getActive($perPage = 20)
    {
        return $this->model->where('status', 'active')
            ->with(['brand', 'categories', 'primaryImage'])
            ->paginate($perPage);
    }
    
    public function findBySlug($slug)
    {
        return $this->model->where('slug', $slug)
            ->with(['variants', 'images'])
            ->firstOrFail();
    }
    
    // Other simple database queries
}
```

### 3.4 Service Layer (Only Complex Logic)

```php
// app/Services/CheckoutService.php
class CheckoutService
{
    protected $orderRepo;
    protected $paymentService;
    
    public function process($cart, $data)
    {
        DB::beginTransaction();
        try {
            // 1. Create order
            $order = $this->createOrder($cart, $data);
            
            // 2. Process payment
            $payment = $this->paymentService->create($order);
            
            // 3. Clear cart
            $cart->items()->delete();
            
            DB::commit();
            
            // 4. Send notifications
            event(new OrderCreated($order));
            
            return $order;
        } catch (\Exception $e) {
            DB::rollback();
            throw $e;
        }
    }
}
```

---

## 4. API SPECIFICATION

### 4.1 Customer API Endpoints

```
BASE URL: https://api.minimoda.com/api

Authentication: Bearer Token (Sanctum)

Auth:
POST   /auth/register
POST   /auth/login
POST   /auth/logout

Products:
GET    /products
GET    /products/{slug}
GET    /categories

Cart:
GET    /cart
POST   /cart/add
PUT    /cart/items/{id}
DELETE /cart/items/{id}

Checkout:
POST   /checkout
POST   /shipping/calculate

Orders:
GET    /orders
GET    /orders/{code}
```

### 4.2 Admin API Endpoints

```
BASE URL: https://admin.minimoda.com/admin

Auth: Session-based with CSRF

Dashboard:
GET    /dashboard

Products:
GET    /products
POST   /products
PUT    /products/{id}
DELETE /products/{id}

Orders:
GET    /orders
GET    /orders/{id}
PUT    /orders/{id}/status

Users & RBAC:
GET    /users
GET    /roles
POST   /users/{id}/roles
GET    /permissions
```

---

## 5. SECURITY & RBAC IMPLEMENTATION

### 5.1 Middleware for Permission Check

```php
// app/Http/Middleware/CheckPermission.php
class CheckPermission
{
    public function handle($request, Closure $next, $permission)
    {
        if (!auth()->user()->hasPermission($permission)) {
            abort(403, 'Unauthorized');
        }
        return $next($request);
    }
}

// Usage in routes
Route::post('/products', [ProductController::class, 'store'])
    ->middleware('permission:product.create');
```

### 5.2 Role-Based Menu Display

```php
// app/Services/RbacService.php
class RbacService
{
    public function getUserModules($userId)
    {
        return Module::join('role_modules', 'modules.id', '=', 'role_modules.module_id')
            ->join('user_roles', 'role_modules.role_id', '=', 'user_roles.role_id')
            ->where('user_roles.user_id', $userId)
            ->where('user_roles.is_active', true)
            ->where('modules.is_active', true)
            ->where('modules.is_visible', true)
            ->where('role_modules.can_view', true)
            ->orderBy('modules.sort_order')
            ->select('modules.*', 'role_modules.can_create', 
                     'role_modules.can_update', 'role_modules.can_delete')
            ->distinct()
            ->get();
    }
}
```

### 5.3 Security Best Practices

| Security Measure | Implementation | Priority |
|-----------------|----------------|----------|
| Password Hashing | bcrypt (Laravel default) | Required |
| CSRF Protection | Laravel middleware | Required |
| SQL Injection | Eloquent ORM | Required |
| XSS Prevention | Blade escaping | Required |
| Rate Limiting | throttle:60,1 | Required |
| 2FA | Optional for admin | Nice to have |
| API Authentication | Sanctum tokens | Required |
| Session Security | Secure cookies | Required |

---

## 6. TESTING STRATEGY (SIMPLIFIED)

### 6.1 Priority Testing Areas

```
Critical Paths Only:
├── Authentication (login/register)
├── Cart operations
├── Checkout process
├── Payment webhook
└── RBAC permissions
```

### 6.2 Simple Test Examples

```php
// tests/Feature/AuthTest.php
class AuthTest extends TestCase
{
    public function test_user_can_login()
    {
        $user = User::factory()->create();
        
        $response = $this->postJson('/api/auth/login', [
            'email' => $user->email,
            'password' => 'password'
        ]);
        
        $response->assertStatus(200)
                 ->assertJsonStructure(['token']);
    }
}

// tests/Feature/RbacTest.php
class RbacTest extends TestCase
{
    public function test_admin_can_create_product()
    {
        $admin = User::factory()->create();
        $admin->roles()->attach(Role::where('name', 'admin')->first());
        
        $response = $this->actingAs($admin)
            ->post('/admin/products', [...]);
            
        $response->assertStatus(201);
    }
}
```

---

## 7. DEPLOYMENT GUIDE

### 7.1 Simple VPS Deployment

```bash
# Server Requirements (Minimum)
- Ubuntu 22.04 LTS
- 2 CPU, 4GB RAM
- 40GB SSD

# Software Stack
- Nginx
- PHP 8.3 FPM
- PostgreSQL 16
- Redis (optional)
- Supervisor for queues
```

### 7.2 Deployment Steps

```bash
# 1. Clone repository
git clone https://github.com/your-repo/minimoda-backend.git

# 2. Install dependencies
composer install --no-dev --optimize-autoloader
npm install && npm run build

# 3. Environment setup
cp .env.production .env
php artisan key:generate

# 4. Database setup
php artisan migrate --force
php artisan db:seed --class=RbacSeeder

# 5. Permissions
chmod -R 775 storage bootstrap/cache
chown -R www-data:www-data .

# 6. Optimize
php artisan config:cache
php artisan route:cache
php artisan view:cache
```

### 7.3 Nginx Configuration

```nginx
server {
    listen 80;
    server_name api.minimoda.com;
    root /var/www/minimoda-backend/public;
    
    index index.php;
    
    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }
    
    location ~ \.php$ {
        fastcgi_pass unix:/var/run/php/php8.3-fpm.sock;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        include fastcgi_params;
    }
}
```

---

## 8. CODE STANDARDS (PRACTICAL)

### 8.1 Simplified Standards

```php
// Follow these simple rules:

1. Use meaningful names
   - Good: $productRepository->getActiveProducts()
   - Bad: $repo->get()

2. Keep controllers thin
   - Business logic in services
   - Database queries in repositories

3. Use Laravel conventions
   - Resource controllers
   - Form requests for validation
   - Eloquent relationships

4. Comment complex logic
   // Calculate discount based on user role
   if ($user->hasRole('vip')) {
       $discount = $price * 0.2;
   }

5. Use constants for statuses
   OrderStatus::PENDING instead of 'pending'
```

### 8.2 File Naming

```
Models:         Product.php (singular)
Controllers:    ProductController.php
Repositories:   ProductRepository.php
Services:       CheckoutService.php
Migrations:     2024_01_01_create_products_table.php
```

### 8.3 Git Workflow (Simple)

```bash
main (production)
├── develop (testing)
    ├── feature/add-payment
    └── fix/cart-bug

# Commit messages
feat: Add payment gateway
fix: Fix cart calculation
docs: Update README
```

---

## APPENDIX A: COMMON COMMANDS

### Development
```bash
# Start server
php artisan serve

# Create controller
php artisan make:controller Admin/ProductController --resource

# Create model with migration
php artisan make:model Product -m

# Run migrations
php artisan migrate

# Seed database
php artisan db:seed

# Clear cache
php artisan cache:clear
php artisan config:clear

# Run tests
php artisan test
```

### RBAC Management
```bash
# Create role
php artisan tinker
>>> Role::create(['name' => 'manager', 'display_name' => 'Manager']);

# Assign role to user
>>> $user = User::find(1);
>>> $user->roles()->attach(Role::where('name', 'admin')->first());

# Check permissions
>>> $user->hasPermission('product.create'); // true/false
```

---

## APPENDIX B: TROUBLESHOOTING

### Common Issues

| Issue | Solution |
|-------|----------|
| Permission denied | Run: chmod -R 775 storage |
| Database connection | Check .env DB credentials |
| 419 error | CSRF token missing |
| 500 error | Check storage/logs/laravel.log |
| Slow queries | Add indexes, check EXPLAIN |
| RBAC not working | Clear cache, check middleware |

---

**END OF TECHNICAL GUIDE**

*This practical approach balances clean code with development speed, perfect for a 2-person team*