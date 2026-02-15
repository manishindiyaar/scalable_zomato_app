# 07. API Design Patterns

## The Problem: API Chaos at Scale

### What Happens with Simple REST APIs?

```javascript
// Simple approach - Everything in one server
app.post('/api/order', async (req, res) => {
  // Create order
  // Process payment
  // Notify restaurant
  // Find rider
  // Send emails
  // Update inventory
  // ... 500 lines of code
});
```

**Problems:**
1. **Tight Coupling**: One endpoint does everything
2. **No Separation**: Payment logic mixed with order logic
3. **Hard to Scale**: Can't scale payment processing independently
4. **Security Risk**: All services share same authentication
5. **Deployment Hell**: One bug breaks everything

---

## The Solution: Layered API Architecture

### Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    CLIENT (React Frontend)                   │
│  - Axios HTTP client                                         │
│  - JWT token in Authorization header                         │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                   PUBLIC APIs (External)                     │
│  - Exposed to internet                                       │
│  - JWT authentication required                               │
│  - Rate limiting applied                                     │
│  - CORS enabled                                              │
└─────────────────────────────────────────────────────────────┘
                              ↓
        ┌─────────────────────┴─────────────────────┐
        ↓                     ↓                      ↓
┌──────────────┐    ┌──────────────────┐    ┌──────────────┐
│ Auth Service │    │ Restaurant Svc   │    │ Rider Service│
│   Port 5000  │    │   Port 5001      │    │   Port 5002  │
└──────────────┘    └──────────────────┘    └──────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                  INTERNAL APIs (Private)                     │
│  - Not exposed to internet                                   │
│  - x-internal-key authentication                             │
│  - Service-to-service communication                          │
│  - No rate limiting needed                                   │
└─────────────────────────────────────────────────────────────┘
```

---

## Pattern 1: Public REST APIs

### Design Principles

```typescript
// Restaurant Service - Public API
import express from "express";
import { isAuth, isSeller } from "../middlewares/isAuth.js";

const router = express.Router();

// Public endpoints (no auth)
router.get("/all", getAllRestaurants);           // Anyone can browse

// Protected endpoints (auth required)
router.get("/:id", isAuth, getRestaurantDetails); // Must be logged in

// Role-based endpoints (seller only)
router.post("/new", isAuth, isSeller, createRestaurant);
router.put("/:id", isAuth, isSeller, updateRestaurant);
```

### RESTful Conventions

```
Resource: Restaurant
├─ GET    /api/restaurant/all          → List all restaurants
├─ GET    /api/restaurant/:id          → Get single restaurant
├─ POST   /api/restaurant/new          → Create restaurant
├─ PUT    /api/restaurant/:id          → Update restaurant
└─ DELETE /api/restaurant/:id          → Delete restaurant

Resource: Order
├─ GET    /api/order/myorder            → Get user's orders
├─ GET    /api/order/:id                → Get single order
├─ POST   /api/order/new                → Create order
└─ PUT    /api/order/:orderId           → Update order status
```

**Why This Structure?**
- **Predictable**: Developers know where to find endpoints
- **Cacheable**: GET requests can be cached by browsers
- **Idempotent**: PUT/DELETE can be retried safely
- **Stateless**: Each request contains all needed info

---

## Pattern 2: Internal APIs (Service-to-Service)

### The Problem: How Do Services Talk?

```
Scenario: Rider accepts an order

┌──────────────┐                    ┌──────────────────┐
│ Rider Service│                    │ Restaurant Svc   │
│              │                    │                  │
│ Rider clicks │                    │ Needs to update  │
│ "Accept"     │ ──────────────────>│ order.riderId    │
│              │   How to call?     │                  │
└──────────────┘                    └──────────────────┘
```

**Options:**
1. ❌ **Direct DB Access**: Rider service writes to Restaurant DB
   - Violates database-per-service pattern
   - Creates tight coupling

2. ❌ **Public API**: Use public REST endpoint
   - Security risk (anyone could call it)
   - Requires user authentication (but this is service-to-service)

3. ✅ **Internal API**: Special endpoints for services only

### Implementation: Internal API Key

```typescript
// Restaurant Service - Internal endpoint
export const assignRiderToOrder = TryCatch(async (req, res) => {
  // Step 1: Verify this is a service calling, not a user
  if (req.headers["x-internal-key"] !== process.env.INTERNAL_SERVICE_KEY) {
    return res.status(403).json({
      message: "Forbidden"
    });
  }

  // Step 2: Business logic (no user auth needed)
  const { orderId, riderId, riderName, riderPhone } = req.body;

  // Step 3: Atomic update (prevent race conditions)
  const order = await Order.findOneAndUpdate(
    { _id: orderId, riderId: null },  // Only if not assigned yet
    {
      riderId,
      riderName,
      riderPhone,
      status: "rider_assigned"
    },
    { new: true }
  );

  if (!order) {
    return res.status(400).json({
      message: "Order already taken"
    });
  }

  res.json({ success: true, order });
});

// Route definition
router.put("/assign/rider", assignRiderToOrder);  // No isAuth middleware!
```

### Calling Internal APIs

```typescript
// Rider Service - Calling Restaurant Service
import axios from "axios";

export const acceptOrder = async (req, res) => {
  const { orderId } = req.body;
  const rider = req.user;

  try {
    // Call internal API of Restaurant Service
    const response = await axios.put(
      `${process.env.RESTAURANT_SERVICE}/api/order/assign/rider`,
      {
        orderId,
        riderId: rider._id,
        riderName: rider.name,
        riderPhone: rider.phoneNumber
      },
      {
        headers: {
          "x-internal-key": process.env.INTERNAL_SERVICE_KEY
        }
      }
    );

    res.json({ message: "Order accepted", order: response.data.order });
  } catch (error) {
    res.status(400).json({ message: "Failed to accept order" });
  }
};
```

**Security Model:**
```
┌─────────────────────────────────────────────────────────────┐
│ Public API                                                   │
│ Authorization: Bearer <JWT>                                  │
│ ✓ User authentication                                        │
│ ✓ Rate limiting                                              │
│ ✓ CORS restrictions                                          │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ Internal API                                                 │
│ x-internal-key: <shared-secret>                              │
│ ✓ Service authentication                                     │
│ ✗ No rate limiting (trusted services)                        │
│ ✗ No CORS (not exposed to browsers)                          │
└─────────────────────────────────────────────────────────────┘
```

---

## Pattern 3: API Gateway (Future Enhancement)

### Current Architecture (Direct Service Calls)

```
Frontend
├─ http://localhost:5000/api/auth/login
├─ http://localhost:5001/api/restaurant/all
├─ http://localhost:5002/api/rider/me
└─ ws://localhost:5003/  (WebSocket)
```

**Problems:**
- Frontend needs to know all service URLs
- CORS configuration on every service
- No centralized rate limiting
- Hard to add API versioning

### Future: API Gateway Pattern

```
┌─────────────┐
│  Frontend   │
└──────┬──────┘
       │ All requests go to one URL
       ↓
┌─────────────────────────────────────────┐
│         API Gateway (Port 80)           │
│  - Single entry point                   │
│  - Route to correct service             │
│  - Centralized auth                     │
│  - Rate limiting                        │
│  - Request logging                      │
└─────────────────────────────────────────┘
       │
       ├─ /api/auth/*      → Auth Service (5000)
       ├─ /api/restaurant/* → Restaurant Service (5001)
       ├─ /api/rider/*     → Rider Service (5002)
       └─ /ws              → Realtime Service (5003)
```

**Implementation (Nginx or Node.js):**

```nginx
# Nginx API Gateway
server {
  listen 80;

  # Auth Service
  location /api/auth/ {
    proxy_pass http://localhost:5000;
  }

  # Restaurant Service
  location /api/restaurant/ {
    proxy_pass http://localhost:5001;
  }

  # Rider Service
  location /api/rider/ {
    proxy_pass http://localhost:5002;
  }

  # WebSocket
  location /ws {
    proxy_pass http://localhost:5003;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
  }
}
```

---

## Pattern 4: Request/Response Flow

### Complete HTTP Request Lifecycle

```
┌─────────────────────────────────────────────────────────────┐
│ 1. CLIENT SENDS REQUEST                                      │
└─────────────────────────────────────────────────────────────┘

POST /api/order/new HTTP/1.1
Host: localhost:5001
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
Content-Type: application/json

{
  "paymentMethod": "razorpay",
  "addressId": "507f1f77bcf86cd799439011"
}

┌─────────────────────────────────────────────────────────────┐
│ 2. MIDDLEWARE CHAIN                                          │
└─────────────────────────────────────────────────────────────┘

app.use(cors());              // ✓ Check CORS headers
app.use(express.json());      // ✓ Parse JSON body
app.use("/api/order", orderRoutes);

router.post("/new", isAuth, createOrder);
                    ↓
                isAuth middleware:
                ├─ Extract JWT from header
                ├─ Verify signature
                ├─ Decode payload
                ├─ Fetch user from DB
                ├─ Attach to req.user
                └─ Call next()

┌─────────────────────────────────────────────────────────────┐
│ 3. CONTROLLER LOGIC                                          │
└─────────────────────────────────────────────────────────────┘

export const createOrder = TryCatch(async (req, res) => {
  // Validate input
  const { paymentMethod, addressId } = req.body;
  if (!addressId) {
    return res.status(400).json({ message: "Address required" });
  }

  // Business logic
  const address = await Address.findOne({ _id: addressId, userId: req.user._id });
  const cartItems = await Cart.find({ userId: req.user._id }).populate("itemId");

  // Calculate pricing
  const subtotal = cartItems.reduce((sum, item) => sum + item.price * item.quantity, 0);
  const deliveryFee = subtotal < 250 ? 49 : 0;
  const totalAmount = subtotal + deliveryFee + 7;

  // Create order
  const order = await Order.create({
    userId: req.user._id,
    items: cartItems,
    totalAmount,
    paymentStatus: "pending",
    expiresAt: new Date(Date.now() + 15 * 60 * 1000)
  });

  // Clear cart
  await Cart.deleteMany({ userId: req.user._id });

  // Return response
  res.json({
    message: "Order created",
    orderId: order._id,
    amount: totalAmount
  });
});

┌─────────────────────────────────────────────────────────────┐
│ 4. SERVER SENDS RESPONSE                                     │
└─────────────────────────────────────────────────────────────┘

HTTP/1.1 200 OK
Content-Type: application/json
Access-Control-Allow-Origin: http://localhost:5173

{
  "message": "Order created",
  "orderId": "507f1f77bcf86cd799439011",
  "amount": 456
}

┌─────────────────────────────────────────────────────────────┐
│ 5. CLIENT PROCESSES RESPONSE                                 │
└─────────────────────────────────────────────────────────────┘

const response = await axios.post('/api/order/new', {
  paymentMethod: 'razorpay',
  addressId: selectedAddress._id
}, {
  headers: {
    Authorization: `Bearer ${localStorage.getItem('token')}`
  }
});

// Redirect to payment
navigate(`/checkout/${response.data.orderId}`);
```

---

## Pattern 5: Error Handling

### Centralized Error Handler

```typescript
// TryCatch wrapper
export const TryCatch = (fn) => (req, res, next) => {
  Promise.resolve(fn(req, res, next)).catch(next);
};

// Global error handler
app.use((err, req, res, next) => {
  console.error(err.stack);

  // Mongoose validation error
  if (err.name === 'ValidationError') {
    return res.status(400).json({
      message: "Validation failed",
      errors: Object.values(err.errors).map(e => e.message)
    });
  }

  // JWT error
  if (err.name === 'JsonWebTokenError') {
    return res.status(401).json({
      message: "Invalid token"
    });
  }

  // Default error
  res.status(500).json({
    message: "Internal server error"
  });
});
```

### Consistent Error Responses

```typescript
// Success response
{
  "success": true,
  "data": { ... },
  "message": "Order created successfully"
}

// Error response
{
  "success": false,
  "message": "Address not found",
  "error": "NOT_FOUND"
}
```

---

## Pattern 6: API Versioning

### Why Version APIs?

```
Problem: You want to change order response format

Old format:
{
  "orderId": "123",
  "items": [...]
}

New format:
{
  "id": "123",
  "orderItems": [...],
  "metadata": { ... }
}

If you change it, old mobile apps break!
```

### Solution: Version in URL

```typescript
// v1 API (old clients)
app.use("/api/v1/order", orderRoutesV1);

// v2 API (new clients)
app.use("/api/v2/order", orderRoutesV2);

// Both can coexist!
```

---

## Real-World Example: Complete API Flow

### Scenario: User Places Order

```typescript
// 1. Frontend makes request
const placeOrder = async () => {
  try {
    // Step 1: Create order
    const orderResponse = await axios.post(
      `${restaurantService}/api/order/new`,
      { paymentMethod: 'razorpay', addressId: address._id },
      { headers: { Authorization: `Bearer ${token}` } }
    );

    // Step 2: Initiate payment
    const paymentResponse = await axios.post(
      `${utilsService}/api/payment/create`,
      { orderId: orderResponse.data.orderId },
      { headers: { Authorization: `Bearer ${token}` } }
    );

    // Step 3: Open Razorpay modal
    const razorpay = new Razorpay({
      key: paymentResponse.data.key,
      order_id: paymentResponse.data.razorpayOrderId,
      handler: async (response) => {
        // Step 4: Verify payment
        await axios.post(
          `${utilsService}/api/payment/verify`,
          { ...response },
          { headers: { Authorization: `Bearer ${token}` } }
        );

        // Step 5: Redirect to success page
        navigate(`/order-success/${orderResponse.data.orderId}`);
      }
    });

    razorpay.open();
  } catch (error) {
    toast.error(error.response?.data?.message || "Order failed");
  }
};
```

---

## Key Takeaways

### From Simple to Scalable

**Simple Approach:**
```javascript
// One endpoint does everything
app.post('/api/order', (req, res) => {
  // 500 lines of code
});
```

**Scalable Approach:**
```javascript
// Public API (user-facing)
router.post("/new", isAuth, createOrder);

// Internal API (service-to-service)
router.put("/assign/rider", assignRiderToOrder);

// Each service handles its domain
// Services communicate via internal APIs
// Clear separation of concerns
```

### Design Principles

1. **RESTful Conventions**: Predictable URL structure
2. **Layered Security**: Public vs Internal APIs
3. **Middleware Chain**: Reusable authentication/validation
4. **Error Handling**: Consistent error responses
5. **Versioning**: Support old clients while evolving

---

**Next:** [08. Security Architecture](./08_Security_Architecture.md) - How JWT, OAuth, and internal keys protect the system
