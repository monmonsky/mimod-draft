# PRACTICAL ARCHITECTURE GUIDE
# MINIMODA - SIMPLIFIED BUT STRUCTURED

**Approach:** Balanced between clean code and development speed  
**Team Size:** 2 Freelancers  
**Timeline:** 4 Months MVP  
**Focus:** Get things done, but maintainable

---

## 1. SIMPLIFIED PROJECT STRUCTURE

```
minimoda-backend/
├── app/
│   ├── Helpers/                    # Helper functions
│   │   ├── ApiResponse.php
│   │   ├── Upload.php
│   │   ├── Format.php
│   │   └── General.php
│   │
│   ├── Http/
│   │   ├── Controllers/
│   │   │   ├── Api/
│   │   │   │   ├── AuthController.php
│   │   │   │   ├── ProductController.php
│   │   │   │   ├── CartController.php
│   │   │   │   ├── CheckoutController.php
│   │   │   │   ├── OrderController.php
│   │   │   │   └── ProfileController.php
│   │   │   ├── Admin/
│   │   │   │   ├── DashboardController.php
│   │   │   │   ├── ProductController.php
│   │   │   │   ├── OrderController.php
│   │   │   │   ├── CustomerController.php
│   │   │   │   └── SettingController.php
│   │   │   └── Webhook/
│   │   │       ├── MidtransController.php
│   │   │       └── RajaOngkirController.php
│   │   │
│   │   ├── Middleware/
│   │   │   ├── ApiAuth.php
│   │   │   ├── AdminAuth.php
│   │   │   └── VerifyWebhook.php
│   │   │
│   │   ├── Requests/             # Form validation
│   │   │   ├── LoginRequest.php
│   │   │   ├── RegisterRequest.php
│   │   │   ├── StoreProductRequest.php
│   │   │   ├── AddToCartRequest.php
│   │   │   └── CheckoutRequest.php
│   │   │
│   │   └── Resources/             # API responses
│   │       ├── ProductResource.php
│   │       ├── OrderResource.php
│   │       ├── CartResource.php
│   │       └── UserResource.php
│   │
│   ├── Models/                    # Eloquent models
│   │   ├── Traits/
│   │   │   ├── HasUuid.php
│   │   │   └── HasSlug.php
│   │   ├── User.php
│   │   ├── Product.php
│   │   ├── Category.php
│   │   ├── Order.php
│   │   ├── Cart.php
│   │   └── Payment.php
│   │
│   ├── Repositories/              # Simple repository pattern
│   │   ├── ProductRepository.php
│   │   ├── OrderRepository.php
│   │   ├── UserRepository.php
│   │   └── CartRepository.php
│   │
│   ├── Services/                  # Business logic (only complex ones)
│   │   ├── CartService.php
│   │   ├── CheckoutService.php
│   │   ├── PaymentService.php
│   │   └── ShippingService.php
│   │
│   ├── Events/                    # Events
│   │   ├── OrderCreated.php
│   │   ├── PaymentReceived.php
│   │   └── LowStock.php
│   │
│   ├── Listeners/                 # Event listeners
│   │   ├── SendOrderEmail.php
│   │   ├── UpdateInventory.php
│   │   └── CreateInvoice.php
│   │
│   ├── Jobs/                      # Background jobs
│   │   ├── SendEmail.php
│   │   ├── ProcessImage.php
│   │   └── CleanupCart.php
│   │
│   └── Constants/                 # App constants
│       ├── OrderStatus.php
│       ├── PaymentStatus.php
│       └── General.php
│
├── config/
│   ├── minimoda.php              # App config
│   ├── payment.php               # Payment config
│   └── shipping.php              # Shipping config
│
├── database/
│   ├── migrations/
│   ├── seeders/
│   └── factories/
│
├── routes/
│   ├── api.php
│   ├── admin.php
│   └── webhook.php
│
└── tests/
    ├── Feature/
    └── Unit/
```

---

## 2. CONTROLLER EXAMPLES

### API Controller Pattern
```php
// app/Http/Controllers/Api/ProductController.php
class ProductController extends Controller
{
    protected $productRepo;
    
    public function __construct(ProductRepository $productRepo)
    {
        $this->productRepo = $productRepo;
    }
    
    public function index(Request $request)
    {
        $products = $this->productRepo->getWithFilters($request->all());
        return ApiResponse::success(ProductResource::collection($products));
    }
    
    public function show($slug)
    {
        $product = $this->productRepo->findBySlug($slug);
        if (!$product) {
            return ApiResponse::error('Product not found', 404);
        }
        return ApiResponse::success(new ProductResource($product));
    }
}
```

### Admin Controller Pattern
```php
// app/Http/Controllers/Admin/OrderController.php
class OrderController extends Controller
{
    protected $orderRepo;
    
    public function index()
    {
        $orders = $this->orderRepo->getLatestWithRelations();
        return view('admin.orders.index', compact('orders'));
    }
    
    public function updateStatus(Request $request, $id)
    {
        $order = $this->orderRepo->updateStatus($id, $request->status);
        event(new OrderStatusUpdated($order));
        return redirect()->back()->with('success', 'Status updated');
    }
}
```

---

## 3. REPOSITORY PATTERN (SIMPLIFIED)

### Base Repository
```php
// app/Repositories/BaseRepository.php
class BaseRepository
{
    protected $model;
    
    public function find($id)
    public function all()
    public function create(array $data)
    public function update($id, array $data)
    public function delete($id)
    public function paginate($perPage = 15)
}
```

### Product Repository
```php
// app/Repositories/ProductRepository.php
class ProductRepository extends BaseRepository
{
    public function __construct(Product $model)
    
    public function getActive($perPage = 20)
    public function findBySlug($slug)
    public function getWithFilters($filters)
    public function updateStock($variantId, $quantity)
    public function getFeatured($limit = 10)
    public function searchProducts($query)
}
```

### Order Repository
```php
// app/Repositories/OrderRepository.php
class OrderRepository extends BaseRepository
{
    public function createOrder($cartData, $customerData)
    public function getByUser($userId)
    public function getByCode($code)
    public function updateStatus($id, $status)
    public function getTodayOrders()
    public function getRevenue($startDate, $endDate)
}
```

---

## 4. SERVICE LAYER (ONLY FOR COMPLEX LOGIC)

### Cart Service
```php
// app/Services/CartService.php
class CartService
{
    protected $cartRepo;
    protected $productRepo;
    
    public function addItem($cartId, $variantId, $qty)
    public function updateItem($itemId, $qty)
    public function removeItem($itemId)
    public function calculateTotals($cart)
    public function applyCoupon($cart, $couponCode)
    public function clearCart($cartId)
    public function getOrCreateCart($userId = null)
}
```

### Checkout Service
```php
// app/Services/CheckoutService.php
class CheckoutService
{
    protected $orderRepo;
    protected $cartService;
    protected $paymentService;
    protected $shippingService;
    
    public function process($cart, $checkoutData)
    {
        // 1. Validate cart
        // 2. Calculate shipping
        // 3. Create order
        // 4. Process payment
        // 5. Clear cart
        // 6. Send notifications
    }
}
```

### Payment Service
```php
// app/Services/PaymentService.php
class PaymentService
{
    public function createPayment($order)
    public function handleWebhook($payload)
    public function verifySignature($payload, $signature)
    public function updatePaymentStatus($orderId, $status)
    public function processRefund($paymentId, $amount)
}
```

---

## 5. HELPERS (PRACTICAL)

### API Response Helper
```php
// app/Helpers/ApiResponse.php
class ApiResponse
{
    public static function success($data = null, $message = 'Success', $code = 200)
    public static function error($message = 'Error', $code = 400, $errors = null)
    public static function paginated($data, $message = 'Success')
}

// Usage
return ApiResponse::success($data, 'Product created');
return ApiResponse::error('Validation failed', 422, $errors);
```

### Format Helper
```php
// app/Helpers/Format.php
class Format
{
    public static function currency($amount)  // Rp 100.000
    public static function phone($phone)      // +62 812-3456-7890
    public static function date($date)        // 01 Jan 2024
    public static function slug($string)      // product-name
    public static function orderCode()        // ORD-20241201-001
}
```

### Upload Helper
```php
// app/Helpers/Upload.php
class Upload
{
    public static function image($file, $path = 'products')
    public static function multiple($files, $path)
    public static function delete($path)
    public static function resize($file, $width, $height)
}
```

### General Helper
```php
// app/Helpers/General.php
class General
{
    public static function generateOTP()
    public static function calculateDiscount($price, $discount, $type)
    public static function getShippingOrigin()
    public static function isProduction()
}
```

---

## 6. MODELS WITH TRAITS

### HasUuid Trait
```php
// app/Models/Traits/HasUuid.php
trait HasUuid
{
    public static function bootHasUuid()
    {
        static::creating(function ($model) {
            $model->id = Str::uuid();
        });
    }
}
```

### HasSlug Trait
```php
// app/Models/Traits/HasSlug.php
trait HasSlug
{
    public static function bootHasSlug()
    {
        static::creating(function ($model) {
            $model->slug = Str::slug($model->name);
        });
    }
}
```

### Product Model
```php
// app/Models/Product.php
class Product extends Model
{
    use HasUuid, HasSlug;
    
    // Relationships
    public function categories()
    public function variants()
    public function images()
    
    // Scopes
    public function scopeActive($query)
    public function scopeFeatured($query)
    
    // Accessors
    public function getPriceFormattedAttribute()
    public function getImageUrlAttribute()
}
```

---

## 7. EVENTS & LISTENERS

### Events
```php
// app/Events/OrderCreated.php
class OrderCreated
{
    public $order;
    public function __construct(Order $order)
}

// app/Events/PaymentReceived.php
class PaymentReceived
{
    public $payment;
    public function __construct(Payment $payment)
}

// app/Events/LowStock.php
class LowStock
{
    public $product;
    public $variant;
    public function __construct(Product $product, $variant)
}
```

### Listeners
```php
// app/Listeners/SendOrderEmail.php
class SendOrderEmail
{
    public function handle(OrderCreated $event)
    {
        // Send email to customer
        Mail::to($event->order->customer_email)
            ->queue(new OrderConfirmation($event->order));
    }
}

// app/Listeners/UpdateInventory.php
class UpdateInventory
{
    public function handle(OrderCreated $event)
    {
        // Update stock for each item
    }
}
```

### Event Service Provider
```php
// app/Providers/EventServiceProvider.php
protected $listen = [
    OrderCreated::class => [
        SendOrderEmail::class,
        UpdateInventory::class,
        CreateInvoice::class,
    ],
    PaymentReceived::class => [
        UpdateOrderStatus::class,
        SendPaymentNotification::class,
    ],
];
```

---

## 8. CONSTANTS

### Order Status
```php
// app/Constants/OrderStatus.php
class OrderStatus
{
    const PENDING = 'pending';
    const PROCESSING = 'processing';
    const SHIPPED = 'shipped';
    const DELIVERED = 'delivered';
    const CANCELLED = 'cancelled';
    
    public static function all()
    {
        return [
            self::PENDING,
            self::PROCESSING,
            self::SHIPPED,
            self::DELIVERED,
            self::CANCELLED,
        ];
    }
}
```

### Payment Status
```php
// app/Constants/PaymentStatus.php
class PaymentStatus
{
    const PENDING = 'pending';
    const SUCCESS = 'success';
    const FAILED = 'failed';
    const REFUNDED = 'refunded';
}
```

---

## 9. ROUTES STRUCTURE

### API Routes
```php
// routes/api.php
Route::prefix('v1')->group(function () {
    // Public routes
    Route::post('login', [AuthController::class, 'login']);
    Route::post('register', [AuthController::class, 'register']);
    Route::get('products', [ProductController::class, 'index']);
    Route::get('products/{slug}', [ProductController::class, 'show']);
    
    // Protected routes
    Route::middleware('auth:sanctum')->group(function () {
        Route::get('profile', [ProfileController::class, 'show']);
        Route::post('cart/add', [CartController::class, 'add']);
        Route::post('checkout', [CheckoutController::class, 'process']);
        Route::get('orders', [OrderController::class, 'index']);
    });
});
```

### Admin Routes
```php
// routes/admin.php
Route::prefix('admin')->name('admin.')->group(function () {
    Route::get('login', [LoginController::class, 'showLoginForm'])->name('login');
    Route::post('login', [LoginController::class, 'login']);
    
    Route::middleware('admin')->group(function () {
        Route::get('dashboard', [DashboardController::class, 'index'])->name('dashboard');
        Route::resource('products', ProductController::class);
        Route::resource('orders', OrderController::class);
        Route::put('orders/{id}/status', [OrderController::class, 'updateStatus']);
    });
});
```

---

## 10. MIDDLEWARE

### API Authentication
```php
// app/Http/Middleware/ApiAuth.php
class ApiAuth
{
    public function handle($request, Closure $next)
    {
        if (!auth()->check()) {
            return ApiResponse::error('Unauthenticated', 401);
        }
        return $next($request);
    }
}
```

### Admin Authentication
```php
// app/Http/Middleware/AdminAuth.php
class AdminAuth
{
    public function handle($request, Closure $next)
    {
        if (!auth()->check() || auth()->user()->role !== 'admin') {
            return redirect()->route('admin.login');
        }
        return $next($request);
    }
}
```

---

## 11. FORM REQUESTS

### Store Product Request
```php
// app/Http/Requests/StoreProductRequest.php
class StoreProductRequest extends FormRequest
{
    public function rules()
    {
        return [
            'name' => 'required|string|max:255',
            'description' => 'required|string',
            'category_id' => 'required|exists:categories,id',
            'variants' => 'required|array|min:1',
            'variants.*.size' => 'required|string',
            'variants.*.price' => 'required|numeric|min:0',
            'variants.*.stock' => 'required|integer|min:0',
            'images' => 'required|array|min:1',
            'images.*' => 'image|max:2048'
        ];
    }
}
```

### Checkout Request
```php
// app/Http/Requests/CheckoutRequest.php
class CheckoutRequest extends FormRequest
{
    public function rules()
    {
        return [
            'shipping_address' => 'required|array',
            'shipping_address.name' => 'required|string',
            'shipping_address.phone' => 'required|string',
            'shipping_address.address' => 'required|string',
            'shipping_address.city_id' => 'required|integer',
            'courier' => 'required|in:jne,tiki,pos',
            'service' => 'required|string',
            'payment_method' => 'required|string'
        ];
    }
}
```

---

## 12. API RESOURCES

### Product Resource
```php
// app/Http/Resources/ProductResource.php
class ProductResource extends JsonResource
{
    public function toArray($request)
    {
        return [
            'id' => $this->id,
            'name' => $this->name,
            'slug' => $this->slug,
            'price' => $this->price,
            'formatted_price' => Format::currency($this->price),
            'image' => $this->image_url,
            'category' => $this->category->name,
            'stock' => $this->stock,
            'variants' => VariantResource::collection($this->whenLoaded('variants')),
        ];
    }
}
```

### Order Resource
```php
// app/Http/Resources/OrderResource.php
class OrderResource extends JsonResource
{
    public function toArray($request)
    {
        return [
            'order_code' => $this->order_code,
            'status' => $this->status,
            'total' => Format::currency($this->grand_total),
            'items' => OrderItemResource::collection($this->items),
            'shipping' => [
                'courier' => $this->courier,
                'service' => $this->service,
                'tracking' => $this->tracking_code
            ],
            'created_at' => $this->created_at->format('d M Y H:i')
        ];
    }
}
```

---

## 13. CONFIG FILES

### App Config
```php
// config/minimoda.php
return [
    'app_name' => env('APP_NAME', 'Minimoda'),
    'shipping_origin' => [
        'city_id' => env('SHIPPING_ORIGIN_CITY', 155), // Jakarta
        'address' => env('SHIPPING_ORIGIN_ADDRESS'),
    ],
    'order' => [
        'prefix' => 'ORD',
        'auto_cancel_hours' => 24,
        'minimum_amount' => 50000,
    ],
    'product' => [
        'image_sizes' => [
            'thumbnail' => [150, 150],
            'medium' => [500, 500],
            'large' => [1000, 1000],
        ],
        'per_page' => 20,
    ],
];
```

### Payment Config
```php
// config/payment.php
return [
    'midtrans' => [
        'server_key' => env('MIDTRANS_SERVER_KEY'),
        'client_key' => env('MIDTRANS_CLIENT_KEY'),
        'is_production' => env('MIDTRANS_IS_PRODUCTION', false),
        'is_sanitized' => true,
        'is_3ds' => true,
    ],
    'payment_methods' => [
        'bank_transfer',
        'credit_card',
        'e_wallet',
        'convenience_store',
    ],
];
```

---

## 14. JOBS (BACKGROUND TASKS)

### Send Email Job
```php
// app/Jobs/SendEmail.php
class SendEmail implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;
    
    protected $order;
    
    public function __construct(Order $order)
    public function handle()
    {
        Mail::to($this->order->customer_email)
            ->send(new OrderConfirmation($this->order));
    }
}
```

### Process Image Job
```php
// app/Jobs/ProcessImage.php
class ProcessImage implements ShouldQueue
{
    protected $imagePath;
    
    public function handle()
    {
        // Resize image
        // Create thumbnails
        // Optimize file size
    }
}
```

---

## 15. TESTING STRUCTURE

### Feature Test
```php
// tests/Feature/OrderTest.php
class OrderTest extends TestCase
{
    public function test_can_create_order()
    public function test_cannot_checkout_empty_cart()
    public function test_payment_webhook_updates_order()
}
```

### Unit Test
```php
// tests/Unit/CartServiceTest.php
class CartServiceTest extends TestCase
{
    public function test_can_calculate_cart_total()
    public function test_can_apply_discount()
    public function test_validates_stock_availability()
}
```

---

## WHEN TO USE WHAT?

### Use Repository When:
- Need database operations
- Want to abstract database queries
- Multiple controllers need same data

### Use Service When:
- Complex business logic
- Multiple steps/transactions
- Integration with external services
- Orchestration between repositories

### Use Helper When:
- Simple utility functions
- Formatting/parsing
- Reusable across application

### Use Event When:
- Decoupled actions needed
- Multiple things happen after an action
- Background processing needed

### Use Job When:
- Time-consuming tasks
- External API calls
- Email sending
- Image processing

---

## PRACTICAL TIPS

1. **Don't over-engineer**
   - Start simple, refactor when needed
   - Not everything needs a repository
   - Services only for complex logic

2. **Focus on getting MVP done**
   - Basic features first
   - Polish later
   - Technical debt is OK initially

3. **Copy-paste is OK initially**
   - DRY can come later
   - Speed over perfection for MVP
   - Refactor in Phase 2

4. **Test critical paths only**
   - Checkout process
   - Payment handling
   - User authentication

5. **Use Laravel features**
   - Eloquent relationships
   - Query scopes
   - Model events
   - Form requests

---

**END OF PRACTICAL ARCHITECTURE GUIDE**

*This approach balances clean code with development speed - perfect for 2-person team building MVP*