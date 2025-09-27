# CLEAN ARCHITECTURE & REPOSITORY PATTERN
# MINIMODA BACKEND - PROFESSIONAL STRUCTURE

**Laravel Version:** 11.x  
**PHP Version:** 8.3+  
**Architecture Pattern:** Repository, Service Layer, DTO  
**Document Version:** 1.0

---

## TABLE OF CONTENTS

1. [Architecture Overview](#1-architecture-overview)
2. [Complete Project Structure](#2-complete-project-structure)
3. [Repository Pattern Implementation](#3-repository-pattern-implementation)
4. [Service Layer Pattern](#4-service-layer-pattern)
5. [Data Transfer Objects (DTOs)](#5-data-transfer-objects-dtos)
6. [Helpers & Utilities](#6-helpers--utilities)
7. [Development Workflow](#7-development-workflow)
8. [Code Examples & Templates](#8-code-examples--templates)

---

## 1. ARCHITECTURE OVERVIEW

### Layer Architecture
```
┌─────────────────────────────────────────────────────────────┐
│                     PRESENTATION LAYER                      │
│          Controllers (API/Web) → Resources → Views          │
├─────────────────────────────────────────────────────────────┤
│                      APPLICATION LAYER                      │
│     Services → Actions → DTOs → Form Requests → Jobs       │
├─────────────────────────────────────────────────────────────┤
│                        DOMAIN LAYER                         │
│      Models → Value Objects → Events → Policies → Rules    │
├─────────────────────────────────────────────────────────────┤
│                    INFRASTRUCTURE LAYER                     │
│   Repositories → Cache → External APIs → File Storage      │
└─────────────────────────────────────────────────────────────┘
```

### Request Flow
```
Request → Middleware → Controller → Service → Repository → Database
                           ↓           ↓          ↓
                        Response   Business    Data
                                    Logic     Access
```

### Key Principles
1. **Single Responsibility:** Each class has one reason to change
2. **Dependency Inversion:** Depend on abstractions, not concretions
3. **Separation of Concerns:** Clear boundaries between layers
4. **DRY (Don't Repeat Yourself):** Reusable components
5. **SOLID Principles:** Throughout the architecture

---

## 2. COMPLETE PROJECT STRUCTURE

```
minimoda-backend/
├── app/
│   ├── Actions/                      # Single-purpose action classes
│   │   ├── Cart/
│   │   │   ├── AddItemToCartAction.php
│   │   │   ├── CalculateCartTotalAction.php
│   │   │   ├── ClearExpiredCartsAction.php
│   │   │   ├── RemoveItemFromCartAction.php
│   │   │   └── UpdateCartItemQuantityAction.php
│   │   ├── Order/
│   │   │   ├── CreateOrderAction.php
│   │   │   ├── GenerateOrderNumberAction.php
│   │   │   ├── ProcessOrderPaymentAction.php
│   │   │   ├── UpdateOrderStatusAction.php
│   │   │   ├── CalculateOrderTotalsAction.php
│   │   │   └── GenerateInvoiceAction.php
│   │   ├── Payment/
│   │   │   ├── ProcessPaymentAction.php
│   │   │   ├── RefundPaymentAction.php
│   │   │   ├── VerifyPaymentAction.php
│   │   │   └── HandlePaymentWebhookAction.php
│   │   ├── Product/
│   │   │   ├── UpdateProductStockAction.php
│   │   │   ├── ReserveStockAction.php
│   │   │   ├── ReleaseStockAction.php
│   │   │   ├── UploadProductImagesAction.php
│   │   │   └── GenerateProductSlugAction.php
│   │   └── Shipping/
│   │       ├── CalculateShippingCostAction.php
│   │       ├── CreateShipmentAction.php
│   │       ├── UpdateTrackingAction.php
│   │       └── GenerateShippingLabelAction.php
│   │
│   ├── Cache/                        # Caching layer
│   │   ├── CacheKeys.php
│   │   ├── Contracts/
│   │   │   └── CacheableInterface.php
│   │   ├── Repositories/
│   │   │   ├── ProductCacheRepository.php
│   │   │   ├── CategoryCacheRepository.php
│   │   │   └── SettingsCacheRepository.php
│   │   └── Tags/
│   │       └── CacheTags.php
│   │
│   ├── Console/
│   │   └── Commands/
│   │       ├── ClearExpiredCartsCommand.php
│   │       ├── GenerateRepositoryCommand.php
│   │       ├── GenerateServiceCommand.php
│   │       ├── GenerateDTOCommand.php
│   │       ├── UpdateSearchIndexCommand.php
│   │       └── SendAbandonedCartRemindersCommand.php
│   │
│   ├── DTOs/                        # Data Transfer Objects
│   │   ├── BaseDTO.php
│   │   ├── Auth/
│   │   │   ├── LoginDTO.php
│   │   │   ├── RegisterDTO.php
│   │   │   └── ResetPasswordDTO.php
│   │   ├── Cart/
│   │   │   ├── AddToCartDTO.php
│   │   │   ├── UpdateCartItemDTO.php
│   │   │   └── CartSummaryDTO.php
│   │   ├── Order/
│   │   │   ├── CreateOrderDTO.php
│   │   │   ├── OrderItemDTO.php
│   │   │   ├── OrderFilterDTO.php
│   │   │   └── CheckoutDTO.php
│   │   ├── Payment/
│   │   │   ├── PaymentRequestDTO.php
│   │   │   ├── PaymentResponseDTO.php
│   │   │   └── RefundDTO.php
│   │   ├── Product/
│   │   │   ├── CreateProductDTO.php
│   │   │   ├── UpdateProductDTO.php
│   │   │   ├── ProductVariantDTO.php
│   │   │   ├── ProductFilterDTO.php
│   │   │   └── ProductSearchDTO.php
│   │   └── Shipping/
│   │       ├── ShippingAddressDTO.php
│   │       ├── ShippingCalculateDTO.php
│   │       └── TrackingUpdateDTO.php
│   │
│   ├── Enums/                        # Enumerations
│   │   ├── OrderStatus.php
│   │   ├── PaymentStatus.php
│   │   ├── PaymentMethod.php
│   │   ├── ShippingStatus.php
│   │   ├── ProductStatus.php
│   │   ├── UserRole.php
│   │   ├── UserStatus.php
│   │   ├── CouponType.php
│   │   ├── DiscountType.php
│   │   └── NotificationType.php
│   │
│   ├── Events/
│   │   ├── Cart/
│   │   │   ├── ItemAddedToCart.php
│   │   │   └── CartAbandoned.php
│   │   ├── Order/
│   │   │   ├── OrderCreated.php
│   │   │   ├── OrderPaid.php
│   │   │   ├── OrderShipped.php
│   │   │   ├── OrderDelivered.php
│   │   │   └── OrderCancelled.php
│   │   ├── Payment/
│   │   │   ├── PaymentReceived.php
│   │   │   ├── PaymentFailed.php
│   │   │   └── RefundProcessed.php
│   │   └── Product/
│   │       ├── ProductCreated.php
│   │       ├── LowStockAlert.php
│   │       └── ProductOutOfStock.php
│   │
│   ├── Exceptions/
│   │   ├── Handler.php
│   │   ├── BaseException.php
│   │   ├── Business/
│   │   │   ├── InsufficientStockException.php
│   │   │   ├── InvalidCouponException.php
│   │   │   ├── OrderNotFoundException.php
│   │   │   └── PaymentFailedException.php
│   │   ├── Validation/
│   │   │   ├── ValidationException.php
│   │   │   └── InvalidInputException.php
│   │   └── External/
│   │       ├── PaymentGatewayException.php
│   │       ├── ShippingApiException.php
│   │       └── ExternalServiceException.php
│   │
│   ├── Helpers/
│   │   ├── ApiResponse.php
│   │   ├── ArrayHelper.php
│   │   ├── DateHelper.php
│   │   ├── FileHelper.php
│   │   ├── ImageHelper.php
│   │   ├── MoneyHelper.php
│   │   ├── PhoneHelper.php
│   │   ├── PriceHelper.php
│   │   ├── StringHelper.php
│   │   ├── UploadHelper.php
│   │   └── helpers.php              # Global helper functions
│   │
│   ├── Http/
│   │   ├── Controllers/
│   │   │   ├── BaseController.php
│   │   │   ├── Api/
│   │   │   │   ├── BaseApiController.php
│   │   │   │   ├── V1/
│   │   │   │   │   ├── Auth/
│   │   │   │   │   │   └── AuthController.php
│   │   │   │   │   ├── Cart/
│   │   │   │   │   │   └── CartController.php
│   │   │   │   │   ├── Checkout/
│   │   │   │   │   │   └── CheckoutController.php
│   │   │   │   │   ├── Order/
│   │   │   │   │   │   └── OrderController.php
│   │   │   │   │   ├── Product/
│   │   │   │   │   │   └── ProductController.php
│   │   │   │   │   └── User/
│   │   │   │   │       └── ProfileController.php
│   │   │   │   └── V2/               # Future API version
│   │   │   └── Admin/
│   │   │       ├── BaseAdminController.php
│   │   │       ├── DashboardController.php
│   │   │       ├── ProductManagementController.php
│   │   │       ├── OrderManagementController.php
│   │   │       ├── CustomerManagementController.php
│   │   │       └── SettingsController.php
│   │   │
│   │   ├── Middleware/
│   │   │   ├── AdminAuthenticate.php
│   │   │   ├── ApiAuthenticate.php
│   │   │   ├── ApiVersion.php
│   │   │   ├── CacheResponse.php
│   │   │   ├── CheckUserStatus.php
│   │   │   ├── ForceJsonResponse.php
│   │   │   ├── LogApiRequests.php
│   │   │   ├── RateLimiter.php
│   │   │   ├── SanitizeInput.php
│   │   │   ├── ThrottleRequests.php
│   │   │   ├── TransformInput.php
│   │   │   ├── TrimStrings.php
│   │   │   ├── ValidateSignature.php
│   │   │   └── VerifyApiKey.php
│   │   │
│   │   ├── Requests/
│   │   │   ├── BaseFormRequest.php
│   │   │   ├── Api/
│   │   │   │   ├── Auth/
│   │   │   │   │   ├── LoginRequest.php
│   │   │   │   │   ├── RegisterRequest.php
│   │   │   │   │   └── ForgotPasswordRequest.php
│   │   │   │   ├── Cart/
│   │   │   │   │   ├── AddToCartRequest.php
│   │   │   │   │   └── UpdateCartRequest.php
│   │   │   │   ├── Order/
│   │   │   │   │   └── CreateOrderRequest.php
│   │   │   │   └── Product/
│   │   │   │       └── ProductSearchRequest.php
│   │   │   └── Admin/
│   │   │       ├── Product/
│   │   │       │   ├── StoreProductRequest.php
│   │   │       │   └── UpdateProductRequest.php
│   │   │       └── Order/
│   │   │           └── UpdateOrderStatusRequest.php
│   │   │
│   │   └── Resources/
│   │       ├── BaseResource.php
│   │       ├── Api/
│   │       │   ├── CartResource.php
│   │       │   ├── CategoryResource.php
│   │       │   ├── OrderResource.php
│   │       │   ├── ProductResource.php
│   │       │   ├── UserResource.php
│   │       │   └── Collections/
│   │       │       ├── ProductCollection.php
│   │       │       └── OrderCollection.php
│   │       └── Admin/
│   │           ├── AdminProductResource.php
│   │           └── AdminOrderResource.php
│   │
│   ├── Jobs/
│   │   ├── Cart/
│   │   │   └── ClearExpiredCart.php
│   │   ├── Order/
│   │   │   ├── ProcessOrder.php
│   │   │   └── GenerateInvoice.php
│   │   ├── Payment/
│   │   │   └── ProcessPayment.php
│   │   └── Notification/
│   │       ├── SendOrderConfirmation.php
│   │       └── SendShippingNotification.php
│   │
│   ├── Listeners/
│   │   ├── Cart/
│   │   │   └── UpdateCartActivity.php
│   │   ├── Order/
│   │   │   ├── SendOrderNotification.php
│   │   │   ├── UpdateInventory.php
│   │   │   └── CreateShipment.php
│   │   └── User/
│   │       └── LogUserActivity.php
│   │
│   ├── Models/
│   │   ├── Concerns/                # Model traits
│   │   │   ├── HasUuid.php
│   │   │   ├── HasSlug.php
│   │   │   ├── HasStatus.php
│   │   │   ├── Auditable.php
│   │   │   ├── Cacheable.php
│   │   │   └── Searchable.php
│   │   ├── Scopes/
│   │   │   ├── ActiveScope.php
│   │   │   ├── PublishedScope.php
│   │   │   └── OrderByLatestScope.php
│   │   ├── User.php
│   │   ├── Product.php
│   │   ├── Category.php
│   │   ├── Order.php
│   │   ├── Payment.php
│   │   └── ... (other models)
│   │
│   ├── Observers/
│   │   ├── ProductObserver.php
│   │   ├── OrderObserver.php
│   │   ├── UserObserver.php
│   │   └── CategoryObserver.php
│   │
│   ├── Policies/
│   │   ├── ProductPolicy.php
│   │   ├── OrderPolicy.php
│   │   └── UserPolicy.php
│   │
│   ├── Providers/
│   │   ├── AppServiceProvider.php
│   │   ├── AuthServiceProvider.php
│   │   ├── EventServiceProvider.php
│   │   ├── HelperServiceProvider.php
│   │   ├── ObserverServiceProvider.php
│   │   ├── RepositoryServiceProvider.php
│   │   ├── RouteServiceProvider.php
│   │   └── ServiceLayerProvider.php
│   │
│   ├── QueryBuilders/
│   │   ├── ProductQueryBuilder.php
│   │   ├── OrderQueryBuilder.php
│   │   └── UserQueryBuilder.php
│   │
│   ├── Repositories/
│   │   ├── Contracts/               # Interfaces
│   │   │   ├── BaseRepositoryInterface.php
│   │   │   ├── BrandRepositoryInterface.php
│   │   │   ├── CartRepositoryInterface.php
│   │   │   ├── CategoryRepositoryInterface.php
│   │   │   ├── CouponRepositoryInterface.php
│   │   │   ├── OrderRepositoryInterface.php
│   │   │   ├── PaymentRepositoryInterface.php
│   │   │   ├── ProductRepositoryInterface.php
│   │   │   ├── ShipmentRepositoryInterface.php
│   │   │   └── UserRepositoryInterface.php
│   │   ├── Eloquent/                # Implementations
│   │   │   ├── BaseRepository.php
│   │   │   ├── BrandRepository.php
│   │   │   ├── CartRepository.php
│   │   │   ├── CategoryRepository.php
│   │   │   ├── CouponRepository.php
│   │   │   ├── OrderRepository.php
│   │   │   ├── PaymentRepository.php
│   │   │   ├── ProductRepository.php
│   │   │   ├── ShipmentRepository.php
│   │   │   └── UserRepository.php
│   │   └── Criteria/                # Query criteria
│   │       ├── ActiveProductsCriteria.php
│   │       ├── OrderByDateCriteria.php
│   │       └── WithRelationsCriteria.php
│   │
│   ├── Rules/                       # Custom validation rules
│   │   ├── PhoneNumber.php
│   │   ├── StockAvailable.php
│   │   ├── ValidCoupon.php
│   │   ├── UniqueSlug.php
│   │   └── ValidShippingAddress.php
│   │
│   ├── Services/
│   │   ├── BaseService.php
│   │   ├── Auth/
│   │   │   ├── AuthService.php
│   │   │   ├── PasswordResetService.php
│   │   │   └── VerificationService.php
│   │   ├── Cart/
│   │   │   ├── CartService.php
│   │   │   ├── CartCalculationService.php
│   │   │   └── CartValidationService.php
│   │   ├── Checkout/
│   │   │   ├── CheckoutService.php
│   │   │   └── CheckoutValidationService.php
│   │   ├── Order/
│   │   │   ├── OrderService.php
│   │   │   ├── OrderStatusService.php
│   │   │   ├── InvoiceService.php
│   │   │   └── OrderNumberGeneratorService.php
│   │   ├── Payment/
│   │   │   ├── PaymentService.php
│   │   │   ├── PaymentGateway/
│   │   │   │   ├── PaymentGatewayInterface.php
│   │   │   │   ├── MidtransGateway.php
│   │   │   │   ├── StripeGateway.php
│   │   │   │   └── PaypalGateway.php
│   │   │   └── PaymentWebhookService.php
│   │   ├── Product/
│   │   │   ├── ProductService.php
│   │   │   ├── InventoryService.php
│   │   │   ├── ProductSearchService.php
│   │   │   └── ProductImageService.php
│   │   ├── Shipping/
│   │   │   ├── ShippingService.php
│   │   │   ├── ShippingProvider/
│   │   │   │   ├── ShippingProviderInterface.php
│   │   │   │   ├── RajaOngkirProvider.php
│   │   │   │   ├── JNEProvider.php
│   │   │   │   └── SiCepatProvider.php
│   │   │   ├── ShippingCalculatorService.php
│   │   │   └── TrackingService.php
│   │   └── Notification/
│   │       ├── NotificationService.php
│   │       ├── Channels/
│   │       │   ├── EmailChannel.php
│   │       │   ├── SMSChannel.php
│   │       │   └── WhatsAppChannel.php
│   │       └── Templates/
│   │           ├── OrderConfirmationTemplate.php
│   │           └── ShippingUpdateTemplate.php
│   │
│   ├── Specifications/              # Business rule specifications
│   │   ├── Product/
│   │   │   ├── ProductIsAvailable.php
│   │   │   └── ProductHasStock.php
│   │   └── Order/
│   │       ├── OrderCanBeCancelled.php
│   │       └── OrderCanBeShipped.php
│   │
│   ├── Traits/
│   │   ├── Controllers/
│   │   │   ├── ApiResponser.php
│   │   │   └── HasMediaUpload.php
│   │   ├── Models/
│   │   │   ├── Filterable.php
│   │   │   ├── Sortable.php
│   │   │   └── HasMetadata.php
│   │   └── Services/
│   │       └── Loggable.php
│   │
│   └── ValueObjects/
│       ├── Address.php
│       ├── Email.php
│       ├── Money.php
│       ├── Name.php
│       ├── PhoneNumber.php
│       ├── Price.php
│       ├── Quantity.php
│       ├── SKU.php
│       └── Weight.php
│
├── bootstrap/
│   ├── app.php
│   ├── cache/
│   └── providers.php
│
├── config/
│   ├── app.php
│   ├── auth.php
│   ├── cache.php
│   ├── constants.php               # App constants
│   ├── database.php
│   ├── filesystems.php
│   ├── logging.php
│   ├── mail.php
│   ├── media.php                   # Media configuration
│   ├── minimoda.php                # App specific config
│   ├── payment.php                 # Payment gateways
│   ├── queue.php
│   ├── services.php                # Third-party services
│   ├── shipping.php                # Shipping providers
│   └── settings.php                # Default app settings
│
├── database/
│   ├── factories/
│   │   ├── Concerns/
│   │   │   └── HasStatus.php
│   │   ├── UserFactory.php
│   │   ├── ProductFactory.php
│   │   └── OrderFactory.php
│   ├── migrations/
│   ├── seeders/
│   │   ├── Testing/
│   │   │   ├── TestUserSeeder.php
│   │   │   └── TestProductSeeder.php
│   │   └── Production/
│   │       ├── AdminUserSeeder.php
│   │       └── CategorySeeder.php
│   └── sql/
│       ├── functions/              # PostgreSQL functions
│       ├── triggers/               # Database triggers
│       └── views/                  # Database views
│
├── docs/
│   ├── api/
│   │   ├── openapi.yaml           # OpenAPI specification
│   │   └── postman/
│   │       └── minimoda.postman_collection.json
│   ├── architecture/
│   │   ├── clean-architecture.md
│   │   ├── repository-pattern.md
│   │   └── service-layer.md
│   ├── database/
│   │   ├── erd.png
│   │   └── schema.md
│   └── development/
│       ├── coding-standards.md
│       ├── git-workflow.md
│       └── testing-guide.md
│
├── public/
│   ├── index.php
│   └── .htaccess
│
├── resources/
│   ├── lang/
│   │   └── en/
│   │       ├── auth.php
│   │       ├── messages.php
│   │       ├── pagination.php
│   │       └── validation.php
│   └── views/
│       ├── admin/
│       └── emails/
│
├── routes/
│   ├── api/
│   │   ├── v1.php                 # API version 1 routes
│   │   └── v2.php                 # API version 2 routes
│   ├── admin.php
│   ├── api.php
│   ├── channels.php
│   ├── console.php
│   ├── web.php
│   └── webhook.php
│
├── storage/
│   ├── app/
│   ├── debugbar/
│   ├── framework/
│   └── logs/
│
├── tests/
│   ├── Concerns/
│   │   ├── InteractsWithCart.php
│   │   ├── InteractsWithPayment.php
│   │   └── RefreshDatabase.php
│   ├── Feature/
│   │   ├── Api/
│   │   └── Admin/
│   ├── Integration/
│   │   ├── PaymentGatewayTest.php
│   │   └── ShippingProviderTest.php
│   ├── Unit/
│   │   ├── Actions/
│   │   ├── DTOs/
│   │   ├── Repositories/
│   │   └── Services/
│   ├── Fixtures/
│   │   └── payment-response.json
│   ├── Mocks/
│   │   ├── MockPaymentGateway.php
│   │   └── MockShippingProvider.php
│   └── TestCase.php
│
├── .env.example
├── .env.testing
├── .gitignore
├── .gitlab-ci.yml
├── .php-cs-fixer.php
├── Makefile
├── README.md
├── artisan
├── composer.json
├── docker-compose.yml
├── package.json
├── phpstan.neon
├── phpunit.xml
└── webpack.mix.js
```

---

## 3. REPOSITORY PATTERN IMPLEMENTATION

### Base Repository Interface
```php
// app/Repositories/Contracts/BaseRepositoryInterface.php
interface BaseRepositoryInterface
{
    public function all(array $columns = ['*']);
    public function paginate(int $perPage = 15, array $columns = ['*']);
    public function create(array $data);
    public function update(array $data, $id);
    public function delete($id);
    public function find($id, array $columns = ['*']);
    public function findBy(string $field, $value, array $columns = ['*']);
    public function findWhere(array $where, array $columns = ['*']);
    public function whereIn(string $field, array $values, array $columns = ['*']);
    public function with(array $relations);
    public function withCount(array $relations);
    public function orderBy(string $column, string $direction = 'asc');
    public function updateOrCreate(array $attributes, array $values = []);
}
```

### Product Repository Interface
```php
// app/Repositories/Contracts/ProductRepositoryInterface.php
interface ProductRepositoryInterface extends BaseRepositoryInterface
{
    public function getActiveProducts(int $perPage = 15);
    public function getFeaturedProducts(int $limit = 10);
    public function getProductBySlug(string $slug);
    public function getProductsByCategory(int $categoryId, int $perPage = 15);
    public function getProductsByBrand(int $brandId, int $perPage = 15);
    public function searchProducts(string $query, array $filters = []);
    public function getProductsWithLowStock(int $threshold = 10);
    public function updateStock(int $productId, int $variantId, int $quantity);
    public function getRelatedProducts(int $productId, int $limit = 4);
}
```

### Repository Implementation
```php
// app/Repositories/Eloquent/BaseRepository.php
abstract class BaseRepository implements BaseRepositoryInterface
{
    protected $model;
    
    public function __construct($model)
    {
        $this->model = $model;
    }
    
    // Implementation of all base methods
}

// app/Repositories/Eloquent/ProductRepository.php
class ProductRepository extends BaseRepository implements ProductRepositoryInterface
{
    public function __construct(Product $model)
    {
        parent::__construct($model);
    }
    
    // Implementation of product-specific methods
}
```

### Repository Service Provider
```php
// app/Providers/RepositoryServiceProvider.php
class RepositoryServiceProvider extends ServiceProvider
{
    public function register()
    {
        // Bind interfaces to implementations
        $this->app->bind(ProductRepositoryInterface::class, ProductRepository::class);
        $this->app->bind(OrderRepositoryInterface::class, OrderRepository::class);
        $this->app->bind(UserRepositoryInterface::class, UserRepository::class);
        // ... other bindings
    }
}
```

---

## 4. SERVICE LAYER PATTERN

### Base Service
```php
// app/Services/BaseService.php
abstract class BaseService
{
    protected $repository;
    
    public function __construct($repository)
    {
        $this->repository = $repository;
    }
    
    protected function beginTransaction()
    {
        DB::beginTransaction();
    }
    
    protected function commit()
    {
        DB::commit();
    }
    
    protected function rollback()
    {
        DB::rollback();
    }
}
```

### Product Service
```php
// app/Services/Product/ProductService.php
class ProductService extends BaseService
{
    protected $imageService;
    protected $inventoryService;
    
    public function __construct(
        ProductRepositoryInterface $repository,
        ProductImageService $imageService,
        InventoryService $inventoryService
    ) {
        parent::__construct($repository);
        $this->imageService = $imageService;
        $this->inventoryService = $inventoryService;
    }
    
    public function createProduct(CreateProductDTO $dto): Product
    {
        // Business logic for creating product
    }
    
    public function updateProduct(int $id, UpdateProductDTO $dto): Product
    {
        // Business logic for updating product
    }
}
```

### Order Service
```php
// app/Services/Order/OrderService.php
class OrderService extends BaseService
{
    protected $cartService;
    protected $paymentService;
    protected $shippingService;
    protected $inventoryService;
    
    public function createOrder(CreateOrderDTO $dto): Order
    {
        $this->beginTransaction();
        
        try {
            // 1. Validate cart
            // 2. Calculate totals
            // 3. Create order
            // 4. Create order items
            // 5. Update inventory
            // 6. Process payment
            // 7. Clear cart
            
            $this->commit();
            
            // 8. Send notifications
            // 9. Trigger events
            
            return $order;
        } catch (\Exception $e) {
            $this->rollback();
            throw $e;
        }
    }
}
```

---

## 5. DATA TRANSFER OBJECTS (DTOs)

### Base DTO
```php
// app/DTOs/BaseDTO.php
abstract class BaseDTO
{
    public function __construct(array $data = [])
    {
        foreach ($data as $key => $value) {
            if (property_exists($this, $key)) {
                $this->$key = $value;
            }
        }
    }
    
    public static function fromRequest(Request $request): static
    {
        return new static($request->validated());
    }
    
    public static function fromArray(array $data): static
    {
        return new static($data);
    }
    
    public function toArray(): array
    {
        return get_object_vars($this);
    }
}
```

### Product DTO
```php
// app/DTOs/Product/CreateProductDTO.php
class CreateProductDTO extends BaseDTO
{
    public string $name;
    public string $description;
    public int $brandId;
    public array $categories;
    public array $variants;
    public array $images;
    public ?array $seoMeta;
    public ?array $tags;
    
    public static function fromRequest(StoreProductRequest $request): static
    {
        return new static([
            'name' => $request->input('name'),
            'description' => $request->input('description'),
            'brandId' => $request->input('brand_id'),
            'categories' => $request->input('categories', []),
            'variants' => $request->input('variants', []),
            'images' => $request->file('images', []),
            'seoMeta' => $request->input('seo_meta'),
            'tags' => $request->input('tags', [])
        ]);
    }
}
```

---

## 6. HELPERS & UTILITIES

### API Response Helper
```php
// app/Helpers/ApiResponse.php
class ApiResponse
{
    public static function success($data = null, string $message = 'Success', int $code = 200)
    {
        return response()->json([
            'success' => true,
            'message' => $message,
            'data' => $data
        ], $code);
    }
    
    public static function error(string $message = 'Error', int $code = 400, $errors = null)
    {
        return response()->json([
            'success' => false,
            'message' => $message,
            'errors' => $errors
        ], $code);
    }
    
    public static function paginated($data, string $message = 'Success')
    {
        return response()->json([
            'success' => true,
            'message' => $message,
            'data' => $data->items(),
            'meta' => [
                'total' => $data->total(),
                'per_page' => $data->perPage(),
                'current_page' => $data->currentPage(),
                'last_page' => $data->lastPage()
            ]
        ], 200);
    }
}
```

### Money Helper
```php
// app/Helpers/MoneyHelper.php
class MoneyHelper
{
    public static function format($amount, string $currency = 'IDR'): string
    {
        return 'Rp ' . number_format($amount, 0, ',', '.');
    }
    
    public static function toFloat($amount): float
    {
        return (float) str_replace(['.', ','], ['', '.'], $amount);
    }
    
    public static function calculateDiscount($price, $discount, $type = 'percentage'): float
    {
        if ($type === 'percentage') {
            return $price * ($discount / 100);
        }
        return $discount;
    }
}
```

### Upload Helper
```php
// app/Helpers/UploadHelper.php
class UploadHelper
{
    public static function uploadImage($file, string $path = 'products'): string
    {
        // Handle image upload
        // Resize if needed
        // Generate unique filename
        // Return path
    }
    
    public static function deleteImage(string $path): bool
    {
        // Delete image from storage
    }
    
    public static function uploadMultiple(array $files, string $path = 'products'): array
    {
        // Handle multiple file uploads
    }
}
```

---

## 7. DEVELOPMENT WORKFLOW

### Controller Flow Example
```php
// app/Http/Controllers/Api/V1/Product/ProductController.php
class ProductController extends BaseApiController
{
    protected ProductService $productService;
    
    public function __construct(ProductService $productService)
    {
        $this->productService = $productService;
    }
    
    public function index(ProductSearchRequest $request)
    {
        // 1. Create DTO from request
        $dto = ProductSearchDTO::fromRequest($request);
        
        // 2. Call service
        $products = $this->productService->searchProducts($dto);
        
        // 3. Return resource
        return ApiResponse::paginated($products, 'Products retrieved successfully');
    }
    
    public function store(StoreProductRequest $request)
    {
        // 1. Create DTO
        $dto = CreateProductDTO::fromRequest($request);
        
        // 2. Call service
        $product = $this->productService->createProduct($dto);
        
        // 3. Return resource
        return ApiResponse::success(
            new ProductResource($product),
            'Product created successfully',
            201
        );
    }
}
```

### Service Layer Flow
```php
// app/Services/Product/ProductService.php
class ProductService
{
    public function createProduct(CreateProductDTO $dto): Product
    {
        DB::beginTransaction();
        
        try {
            // 1. Create product using repository
            $productData = $dto->toArray();
            $product = $this->repository->create($productData);
            
            // 2. Handle categories
            if (!empty($dto->categories)) {
                $product->categories()->sync($dto->categories);
            }
            
            // 3. Handle variants using action
            foreach ($dto->variants as $variantData) {
                $this->createProductVariantAction->execute($product, $variantData);
            }
            
            // 4. Handle images
            if (!empty($dto->images)) {
                $this->imageService->uploadProductImages($product, $dto->images);
            }
            
            DB::commit();
            
            // 5. Clear cache
            $this->clearProductCache();
            
            // 6. Dispatch events
            event(new ProductCreated($product));
            
            return $product->fresh(['categories', 'variants', 'images']);
            
        } catch (\Exception $e) {
            DB::rollback();
            throw new ProductCreationException($e->getMessage());
        }
    }
}
```

---

## 8. CODE EXAMPLES & TEMPLATES

### Custom Artisan Commands

#### Generate Repository Command
```bash
php artisan make:repository Product
# Creates:
# - app/Repositories/Contracts/ProductRepositoryInterface.php
# - app/Repositories/Eloquent/ProductRepository.php
# - Adds binding to RepositoryServiceProvider
```

#### Generate Service Command
```bash
php artisan make:service Product
# Creates:
# - app/Services/Product/ProductService.php
# - app/Services/Product/ProductServiceInterface.php
```

#### Generate DTO Command
```bash
php artisan make:dto Product/CreateProduct
# Creates:
# - app/DTOs/Product/CreateProductDTO.php
```

### Makefile for Common Tasks
```makefile
# Makefile
.PHONY: help

help:
	@grep -E '^[a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) | sort | awk 'BEGIN {FS = ":.*?## "}; {printf "\033[36m%-30s\033[0m %s\n", $$1, $$2}'

install: ## Install dependencies
	composer install
	npm install
	cp .env.example .env
	php artisan key:generate

fresh: ## Fresh migration with seed
	php artisan migrate:fresh --seed

test: ## Run tests
	php artisan test

coverage: ## Run tests with coverage
	php artisan test --coverage

format: ## Format code
	./vendor/bin/php-cs-fixer fix

analyze: ## Static analysis
	./vendor/bin/phpstan analyse

cache: ## Clear and cache
	php artisan cache:clear
	php artisan config:cache
	php artisan route:cache
	php artisan view:cache

queue: ## Start queue worker
	php artisan queue:work --tries=3

docs: ## Generate API documentation
	php artisan l5-swagger:generate

deploy: ## Deploy to production
	git pull origin main
	composer install --no-dev --optimize-autoloader
	npm run production
	php artisan migrate --force
	php artisan cache:clear
	php artisan config:cache
	php artisan route:cache
	php artisan view:cache
	php artisan queue:restart
```

### PHPStan Configuration
```neon
# phpstan.neon
parameters:
    level: 5
    paths:
        - app
        - tests
    excludePaths:
        - app/Http/Middleware/RedirectIfAuthenticated.php
    checkMissingIterableValueType: false
```

### PHP CS Fixer Configuration
```php
// .php-cs-fixer.php
<?php
$finder = PhpCsFixer\Finder::create()
    ->in(__DIR__)
    ->exclude(['bootstrap', 'storage', 'vendor'])
    ->name('*.php')
    ->notName('*.blade.php')
    ->ignoreDotFiles(true)
    ->ignoreVCS(true);

return (new PhpCsFixer\Config())
    ->setRules([
        '@PSR12' => true,
        'array_syntax' => ['syntax' => 'short'],
        'ordered_imports' => ['sort_algorithm' => 'alpha'],
        'no_unused_imports' => true,
        'single_quote' => true,
        'trailing_comma_in_multiline' => true,
    ])
    ->setFinder($finder);
```

### Environment Configuration
```env
# .env.example
APP_NAME=Minimoda
APP_ENV=local
APP_KEY=
APP_DEBUG=true
APP_URL=http://localhost:8000
API_URL=http://localhost:8000/api
ADMIN_URL=http://localhost:8000/admin

# Database
DB_CONNECTION=pgsql
DB_HOST=127.0.0.1
DB_PORT=5432
DB_DATABASE=minimoda_db
DB_USERNAME=postgres
DB_PASSWORD=password

# Cache & Session
CACHE_DRIVER=redis
SESSION_DRIVER=redis
QUEUE_CONNECTION=redis

# Redis
REDIS_HOST=127.0.0.1
REDIS_PASSWORD=null
REDIS_PORT=6379
REDIS_CACHE_DB=1
REDIS_SESSION_DB=2
REDIS_QUEUE_DB=3

# Payment Gateway
MIDTRANS_SERVER_KEY=
MIDTRANS_CLIENT_KEY=
MIDTRANS_IS_PRODUCTION=false

# Shipping
RAJAONGKIR_API_KEY=
RAJAONGKIR_TYPE=starter

# Storage
FILESYSTEM_DISK=local
AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
AWS_DEFAULT_REGION=us-east-1
AWS_BUCKET=

# Mail
MAIL_MAILER=smtp
MAIL_HOST=smtp.mailtrap.io
MAIL_PORT=2525
MAIL_USERNAME=
MAIL_PASSWORD=
MAIL_ENCRYPTION=tls
MAIL_FROM_ADDRESS=noreply@minimoda.com
MAIL_FROM_NAME="${APP_NAME}"

# Monitoring
SENTRY_DSN=
DEBUGBAR_ENABLED=false
```

---

## APPENDIX A: PACKAGE RECOMMENDATIONS

### Essential Packages
```json
{
    "require": {
        "php": "^8.3",
        "laravel/framework": "^11.0",
        "laravel/sanctum": "^3.3",
        "laravel/horizon": "^5.21",
        "predis/predis": "^2.2",
        "intervention/image": "^2.7",
        "maatwebsite/excel": "^3.1",
        "spatie/laravel-permission": "^6.0",
        "spatie/laravel-medialibrary": "^11.0",
        "spatie/laravel-query-builder": "^5.6",
        "spatie/laravel-data": "^4.0",
        "spatie/laravel-activitylog": "^4.7"
    },
    "require-dev": {
        "barryvdh/laravel-debugbar": "^3.9",
        "barryvdh/laravel-ide-helper": "^2.13",
        "fakerphp/faker": "^1.23",
        "mockery/mockery": "^1.6",
        "nunomaduro/collision": "^8.0",
        "phpunit/phpunit": "^10.5",
        "phpstan/phpstan": "^1.10",
        "friendsofphp/php-cs-fixer": "^3.40",
        "pestphp/pest": "^2.28",
        "pestphp/pest-plugin-laravel": "^2.2"
    }
}
```

---

## APPENDIX B: GIT WORKFLOW

### Branch Strategy
```
main (production)
├── develop (staging)
│   ├── feature/product-management
│   ├── feature/payment-integration
│   ├── bugfix/cart-calculation
│   └── hotfix/security-patch
```

### Commit Message Convention
```
feat: Add product search functionality
fix: Resolve cart total calculation issue  
docs: Update API documentation
style: Format code according to PSR-12
refactor: Extract payment logic to service
test: Add unit tests for order service
chore: Update dependencies
```

### Git Hooks (.husky)
```json
{
  "husky": {
    "hooks": {
      "pre-commit": "php artisan test && ./vendor/bin/php-cs-fixer fix",
      "pre-push": "./vendor/bin/phpstan analyse"
    }
  }
}
```

---

**END OF CLEAN ARCHITECTURE GUIDE**

*This structure ensures maintainable, testable, and scalable code following industry best practices.*