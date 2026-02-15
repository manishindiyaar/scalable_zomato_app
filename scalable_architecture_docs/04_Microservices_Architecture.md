# 04 - Microservices Architecture: Breaking the Monolith

## The Problem with Our Growing Monolith

Remember our single Express server? Let's see what happens as we scale:

```javascript
// app.js - Our monolithic server
const express = require('express');
const app = express();

// Auth routes
app.post('/api/auth/login', authController.login);
app.post('/api/auth/register', authController.register);

// Restaurant routes
app.get('/api/restaurants', restaurantController.getAll);
app.post('/api/restaurants', restaurantController.create);

// Order routes
app.post('/api/orders', orderController.create);
app.get('/api/orders/:id', orderController.getById);

// Rider routes
app.get('/api/riders/nearby', riderController.findNearby);
app.post('/api/riders/assign', riderController.assignToOrder);

// Payment routes
app.post('/api/payments/process', paymentController.process);

app.listen(3000);
```

### Problems That Emerge at Scale

#### 1. **Single Point of Failure**
```
User places order → Server crashes → Everything stops
- Can't login
- Can't browse restaurants
- Can't track existing orders
- Can't process payments
```

**Real-world scenario:**
```
Black Friday Sale
├─ 10,000 users trying to place orders
├─ Payment processing takes 5 seconds each
├─ Server has 4 CPU cores
└─ Result: Server overloaded, entire app goes down
    Even users just browsing restaurants are affected!
```

#### 2. **Scaling Inefficiency**

```
Problem: Rider location updates happen every 5 seconds
        But restaurant browsing happens once per session

Current solution: Scale entire server
├─ Deploy 10 instances of the ENTIRE app
├─ Each instance has:
│   ├─ Auth logic (rarely used)
│   ├─ Restaurant logic (moderate use)
│   ├─ Order logic (heavy use)
│   ├─ Rider logic (VERY heavy use - constant updates)
│   └─ Payment logic (moderate use)
└─ Result: Wasting resources on underutilized components
```

**Cost Analysis:**
```
Monolith Scaling:
- 1 server handles 1000 requests/sec
- Need 10,000 requests/sec capacity
- Deploy 10 identical servers
- Cost: 10 × $200/month = $2000/month

But breakdown shows:
- Rider updates: 7000 req/sec (70%)
- Orders: 2000 req/sec (20%)
- Restaurants: 800 req/sec (8%)
- Auth: 200 req/sec (2%)

We're paying for 10× auth capacity when we only need 1×!
```

#### 3. **Deployment Risk**

```javascript
// Developer fixes a small bug in restaurant search
function searchRestaurants(query) {
  // Fixed: typo in search logic
  return Restaurant.find({ name: { $regex: query, $options: 'i' } });
}

// Deploy to production
// Result: ENTIRE app restarts
// Impact:
// ├─ All active WebSocket connections drop
// ├─ In-progress payments fail
// ├─ Riders lose connection
// └─ Users get logged out
```

#### 4. **Technology Lock-in**

```
Current stack: Node.js + Express + MongoDB

New requirements:
├─ ML team wants to add restaurant recommendations
│   └─ Best done in Python with TensorFlow
│
├─ Payment team wants to use PostgreSQL
│   └─ Better for financial transactions (ACID compliance)
│
└─ Real-time team wants to use Go
    └─ Better WebSocket performance

Problem: Can't mix technologies in a monolith!
```

#### 5. **Team Coordination Nightmare**

```
Team Structure:
├─ Auth Team (2 developers)
├─ Restaurant Team (3 developers)
├─ Order Team (4 developers)
├─ Rider Team (3 developers)
└─ Payment Team (2 developers)

All working on the SAME codebase!

Conflicts:
├─ Merge conflicts every day
├─ One team's bug breaks another team's feature
├─ Can't deploy independently
└─ Testing takes hours (must test entire app)
```

---

## The Microservices Solution

### Core Principle: Decomposition by Business Capability

Instead of one big app, create **small, independent services** that each do ONE thing well.

```
┌─────────────────────────────────────────────────────────────┐
│                         MONOLITH                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ Auth + Restaurant + Order + Rider + Payment          │  │
│  │ All in one process, one database, one deployment     │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                            ↓
                    DECOMPOSE BY DOMAIN
                            ↓
┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
│  Auth    │  │Restaurant│  │  Order   │  │  Rider   │  │ Payment  │
│ Service  │  │ Service  │  │ Service  │  │ Service  │  │ Service  │
│          │  │          │  │          │  │          │  │          │
│ Port     │  │ Port     │  │ Port     │  │ Port     │  │ Port     │
│ 5000     │  │ 5001     │  │ 5002     │  │ 5003     │  │ 5004     │
│          │  │          │  │          │  │          │  │          │
│ Own DB   │  │ Own DB   │  │ Own DB   │  │ Own DB   │  │ Own DB   │
└──────────┘  └──────────┘  └──────────┘  └──────────┘  └──────────┘
```

---

## Tomato-Code Microservices Architecture

### Service Breakdown

#### **1. Auth Service (Port 5000)**

**Responsibility:** User authentication and authorization

```javascript
// services/auth/src/index.js
import express from 'express';
import connectDB from './config/db.js';
import authRoute from './routes/auth.js';

const app = express();
app.use(express.json());
app.use('/api/auth', authRoute);

app.listen(5000, () => {
  console.log('Auth service running on port 5000');
  connectDB(); // Connects to auth_db
});
```

**API Endpoints:**
```
POST   /api/auth/google     - Google OAuth login
GET    /api/auth/me         - Get current user
POST   /api/auth/logout     - Logout user
```

**Database:** `auth_db`
```javascript
// models/User.js
{
  name: String,
  email: String,
  image: String,
  role: String  // 'customer', 'seller', 'rider', 'admin'
}
```

**Why separate?**
- Authentication logic is critical and rarely changes
- Can scale independently (auth is low-traffic)
- Security isolation (if compromised, doesn't expose order data)

---

#### **2. Restaurant Service (Port 5001)**

**Responsibility:** Restaurants, menu items, cart, orders, addresses

```javascript
// services/restaurant/src/index.js
import express from 'express';
import connectDB from './config/db.js';
import restaurantRoutes from './routes/restaurant.js';
import itemRoutes from './routes/menuitem.js';
import cartRoutes from './routes/cart.js';
import orderRoutes from './routes/order.js';
import addressRoutes from './routes/address.js';
import { connectRabbitMQ } from './config/rabbitmq.js';
import { startPaymentConsumer } from './config/payment.consumer.js';

const app = express();

// Initialize message queue
await connectRabbitMQ();
startPaymentConsumer();

app.use('/api/restaurant', restaurantRoutes);
app.use('/api/item', itemRoutes);
app.use('/api/cart', cartRoutes);
app.use('/api/order', orderRoutes);
app.use('/api/address', addressRoutes);

app.listen(5001, () => {
  console.log('Restaurant service running on port 5001');
  connectDB(); // Connects to restaurant_db
});
```

**API Endpoints:**
```
Restaurants:
GET    /api/restaurant/all
POST   /api/restaurant/new
GET    /api/restaurant/:id

Menu Items:
GET    /api/item/:restaurantId
POST   /api/item/new
PUT    /api/item/:id
DELETE /api/item/:id

Cart:
GET    /api/cart/all
POST   /api/cart/new
DELETE /api/cart/:id

Orders:
POST   /api/order/new
GET    /api/order/myorder
PUT    /api/order/:orderId
GET    /api/order/:id

Addresses:
GET    /api/address/all
POST   /api/address/new
DELETE /api/address/:id
```

**Database:** `restaurant_db`
```javascript
// Collections:
restaurants: {
  name, description, image, ownerId,
  autoLocation: { type: 'Point', coordinates: [lon, lat] },
  isOpen, isVerified
}

menu_items: {
  name, description, price, image,
  restaurantId, category, isAvailable
}

carts: {
  userId, itemId, restaurantId, quantity
}

orders: {
  userId, restaurantId, riderId,
  items: [{ itemId, name, price, quantity }],
  subtotal, deliveryFee, platformFee, totalAmount,
  deliveryAddress, status, paymentStatus, expiresAt
}

addresses: {
  userId, formattedAddress, mobile,
  location: { type: 'Point', coordinates: [lon, lat] }
}
```

**Why this grouping?**
- These entities are tightly coupled (order needs restaurant, cart, address)
- High cohesion: All related to the ordering flow
- Most frequently accessed together

---

#### **3. Rider Service (Port 5002)**

**Responsibility:** Rider management and order assignment

```javascript
// services/rider/src/index.js
import express from 'express';
import connectDB from './config/db.js';
import riderRoutes from './routes/rider.js';
import { connectRabbitMQ } from './config/rabbitmq.js';
import { startOrderReadyConsumer } from './config/orderReady.consumer.js';

const app = express();

// Initialize message queue
await connectRabbitMQ();
startOrderReadyConsumer(); // Listen for orders ready for pickup

app.use('/api/rider', riderRoutes);

app.listen(5002, () => {
  console.log('Rider service running on port 5002');
  connectDB(); // Connects to rider_db
});
```

**API Endpoints:**
```
POST   /api/rider/new              - Register as rider
GET    /api/rider/me               - Get rider profile
PUT    /api/rider/location         - Update location
PUT    /api/rider/availability     - Toggle availability
GET    /api/rider/orders/current   - Get current order
```

**Database:** `rider_db`
```javascript
riders: {
  userId, picture, phoneNumber,
  aadharNumber, drivingLicenseNumber,
  isVerified, isAvailable,
  location: { type: 'Point', coordinates: [lon, lat] },
  lastActiveAt
}
```

**Why separate?**
- Riders update location every 5 seconds (HIGH TRAFFIC)
- Needs independent scaling (10× more traffic than other services)
- Geospatial queries are resource-intensive
- Can optimize database specifically for location updates

---

#### **4. Utils Service (Port 5004)**

**Responsibility:** Shared utilities (Cloudinary, Razorpay, Stripe)

```javascript
// services/utils/src/index.js
import express from 'express';
import cloudinaryRoutes from './routes/cloudinary.js';
import paymentRoutes from './routes/payment.js';

const app = express();

app.use('/api/cloudinary', cloudinaryRoutes);
app.use('/api/payment', paymentRoutes);

app.listen(5004, () => {
  console.log('Utils service running on port 5004');
});
```

**API Endpoints:**
```
POST   /api/cloudinary/upload       - Upload image
POST   /api/payment/create-order    - Create Razorpay order
POST   /api/payment/verify          - Verify payment
```

**Why separate?**
- Reusable across all services
- Third-party integrations isolated
- Can swap payment providers without touching other services

---

#### **5. Realtime Service (Port 5005)**

**Responsibility:** WebSocket connections and real-time events

```javascript
// services/realtime/src/index.js
import express from 'express';
import { Server } from 'socket.io';
import { createServer } from 'http';

const app = express();
const httpServer = createServer(app);
const io = new Server(httpServer, {
  cors: { origin: process.env.FRONTEND_URL }
});

// WebSocket authentication
io.use((socket, next) => {
  const token = socket.handshake.auth.token;
  const user = verifyJWT(token);
  socket.userId = user._id;
  socket.join(`user:${user._id}`); // Join user-specific room
  next();
});

// Internal API for other services to emit events
app.post('/api/v1/internal/emit', (req, res) => {
  const { event, room, payload } = req.body;
  io.to(room).emit(event, payload);
  res.json({ success: true });
});

httpServer.listen(5005, () => {
  console.log('Realtime service running on port 5005');
});
```

**Why separate?**
- WebSocket connections are stateful (can't load balance easily)
- High memory usage (one connection per user)
- Needs sticky sessions
- Isolates real-time complexity from business logic

---

#### **6. Admin Service (Port 5006)**

**Responsibility:** Admin dashboard and verification

```javascript
// services/admin/src/index.js
import express from 'express';
import connectDB from './config/db.js';

const app = express();

app.put('/api/admin/restaurant/verify/:id', verifyRestaurant);
app.put('/api/admin/rider/verify/:id', verifyRider);
app.get('/api/admin/stats', getStats);

app.listen(5006, () => {
  console.log('Admin service running on port 5006');
  connectDB(); // Connects to admin_db
});
```

**Why separate?**
- Admin operations are low-traffic
- Security isolation (admin endpoints separate from public APIs)
- Can have different authentication/authorization rules

---

## Service Communication Patterns

### 1. **Synchronous Communication (HTTP/REST)**

Used when immediate response is needed.

```javascript
// Restaurant Service needs to verify user is authenticated
// Calls Auth Service synchronously

async function createOrder(req, res) {
  // Verify user with Auth Service
  const { data: user } = await axios.get(
    `${AUTH_SERVICE_URL}/api/auth/me`,
    {
      headers: { Authorization: req.headers.authorization }
    }
  );

  if (!user) {
    return res.status(401).json({ message: 'Unauthorized' });
  }

  // Continue with order creation
  const order = await Order.create({ userId: user._id, ... });
  res.json({ order });
}
```

**Problem with this approach:**
```
User Request → Restaurant Service → Auth Service
                     ↓                    ↓
                  Waiting...          Processing
                     ↓                    ↓
                  Waiting...          Response
                     ↓                    ↓
                  Response ←──────────────┘

Total latency = Restaurant processing + Auth processing + Network delay
If Auth Service is slow/down, Restaurant Service fails!
```

**Solution:** Use JWT tokens (no need to call Auth Service every time)

```javascript
// Better approach: Verify JWT locally
import jwt from 'jsonwebtoken';

async function createOrder(req, res) {
  const token = req.headers.authorization.split(' ')[1];
  const user = jwt.verify(token, JWT_SECRET); // No external call!

  const order = await Order.create({ userId: user._id, ... });
  res.json({ order });
}
```

---

### 2. **Asynchronous Communication (Message Queue)**

Used when immediate response is NOT needed.

```javascript
// Payment Service processes payment
// Needs to notify Restaurant Service
// But doesn't need immediate response

// ❌ BAD: Synchronous call
await axios.post(`${RESTAURANT_SERVICE_URL}/api/order/confirm`, {
  orderId: '123'
});
// Problem: If Restaurant Service is down, payment fails!

// ✅ GOOD: Publish to message queue
await publishToQueue('payment_queue', {
  type: 'PAYMENT_SUCCESS',
  data: { orderId: '123' }
});
// Payment Service doesn't care if Restaurant Service is down
// Message will be processed when it comes back online
```

**RabbitMQ Setup:**

```javascript
// services/restaurant/src/config/rabbitmq.js
import amqp from 'amqplib';

let channel;

export const connectRabbitMQ = async () => {
  const connection = await amqp.connect(process.env.RABBITMQ_URL);
  channel = await connection.createChannel();

  // Declare queues
  await channel.assertQueue('payment_queue', { durable: true });
  await channel.assertQueue('order_ready_queue', { durable: true });

  console.log('Connected to RabbitMQ');
};

export const getChannel = () => channel;
```

**Publisher (Restaurant Service):**

```javascript
// services/restaurant/src/config/order.publisher.js
import { getChannel } from './rabbitmq.js';

export const publishEvent = async (type, data) => {
  const channel = getChannel();

  channel.sendToQueue(
    'order_ready_queue',
    Buffer.from(JSON.stringify({ type, data })),
    { persistent: true } // Survive RabbitMQ restart
  );
};

// Usage:
await publishEvent('ORDER_READY_FOR_RIDER', {
  orderId: order._id,
  restaurantId: restaurant._id,
  location: restaurant.autoLocation
});
```

**Consumer (Rider Service):**

```javascript
// services/rider/src/config/orderReady.consumer.js
import { getChannel } from './rabbitmq.js';
import { Rider } from '../model/Rider.js';

export const startOrderReadyConsumer = async () => {
  const channel = getChannel();

  channel.consume('order_ready_queue', async (msg) => {
    if (!msg) return;

    const event = JSON.parse(msg.content.toString());

    if (event.type === 'ORDER_READY_FOR_RIDER') {
      const { orderId, location } = event.data;

      // Find nearby riders
      const riders = await Rider.find({
        isAvailable: true,
        location: {
          $near: {
            $geometry: location,
            $maxDistance: 5000
          }
        }
      });

      // Notify riders via WebSocket
      riders.forEach(rider => {
        notifyRider(rider.userId, { orderId });
      });

      channel.ack(msg); // Acknowledge message processed
    }
  });
};
```

---

### 3. **Event-Driven Communication (WebSocket)**

Used for real-time updates to clients.

```javascript
// Restaurant Service updates order status
// Needs to notify user in real-time

await axios.post(
  `${REALTIME_SERVICE_URL}/api/v1/internal/emit`,
  {
    event: 'order:update',
    room: `user:${order.userId}`,
    payload: {
      orderId: order._id,
      status: order.status
    }
  },
  {
    headers: {
      'x-internal-key': process.env.INTERNAL_SERVICE_KEY
    }
  }
);
```

**Frontend receives update:**

```javascript
// Frontend: src/context/SocketContext.tsx
const socket = io(REALTIME_SERVICE_URL, {
  auth: { token: localStorage.getItem('token') }
});

socket.on('order:update', (data) => {
  console.log('Order status changed:', data.status);
  // Update UI in real-time
  setOrderStatus(data.status);
});
```

---

## Benefits Achieved

### 1. **Independent Scaling**

```
Before (Monolith):
├─ 1 server: $200/month × 10 instances = $2000/month
└─ All components scaled equally (wasteful)

After (Microservices):
├─ Auth Service: 1 instance × $50/month = $50
├─ Restaurant Service: 3 instances × $100/month = $300
├─ Rider Service: 10 instances × $100/month = $1000
├─ Utils Service: 2 instances × $50/month = $100
├─ Realtime Service: 5 instances × $150/month = $750
└─ Admin Service: 1 instance × $50/month = $50
    Total: $2250/month

But now handling 5× more traffic!
Cost per request: 50% lower
```

### 2. **Fault Isolation**

```
Scenario: Rider Service crashes

Before (Monolith):
└─ Entire app down ❌

After (Microservices):
├─ Users can still browse restaurants ✅
├─ Users can still add to cart ✅
├─ Users can still place orders ✅
└─ Only rider assignment fails (graceful degradation) ⚠️
```

### 3. **Independent Deployment**

```
Before:
├─ Fix bug in restaurant search
├─ Deploy entire app
├─ All users disconnected
└─ Downtime: 2 minutes

After:
├─ Fix bug in restaurant search
├─ Deploy only Restaurant Service
├─ Other services unaffected
└─ Downtime: 0 seconds (rolling deployment)
```

### 4. **Technology Freedom**

```
Current:
├─ Auth Service: Node.js + MongoDB
├─ Restaurant Service: Node.js + MongoDB
├─ Rider Service: Go + PostgreSQL (future: better geospatial)
├─ ML Service: Python + TensorFlow (future: recommendations)
└─ Analytics Service: Scala + Spark (future: big data)
```

### 5. **Team Autonomy**

```
Team Structure:
├─ Auth Team → Owns Auth Service
│   └─ Can deploy independently
├─ Restaurant Team → Owns Restaurant Service
│   └─ Can deploy independently
├─ Rider Team → Owns Rider Service
│   └─ Can deploy independently
└─ No merge conflicts, no coordination overhead
```

---

## Challenges Introduced

### 1. **Distributed System Complexity**

```
Problem: Network calls can fail

Solution: Implement retry logic, circuit breakers, timeouts
```

### 2. **Data Consistency**

```
Problem: Order data in Restaurant Service, User data in Auth Service
         How to ensure consistency?

Solution: Event sourcing, eventual consistency, saga pattern
```

### 3. **Debugging Difficulty**

```
Problem: Request spans multiple services
         How to trace errors?

Solution: Distributed tracing (Jaeger, Zipkin)
         Correlation IDs in logs
```

### 4. **Increased Operational Overhead**

```
Before: Deploy 1 app
After: Deploy 6 services + RabbitMQ + Load Balancer

Solution: Docker + Kubernetes for orchestration
```

---

## Next Steps

In the next chapter, we'll explore:
- **Message Queue Deep Dive**: How RabbitMQ ensures reliable communication
- **Event-Driven Architecture**: Designing event flows
- **Service Discovery**: How services find each other
- **API Gateway**: Single entry point for all services

---

**Key Takeaway:** Microservices solve scalability, fault tolerance, and team autonomy problems, but introduce distributed system complexity. The trade-off is worth it at scale (100K+ users).
