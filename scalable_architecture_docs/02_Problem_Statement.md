# 02 - The Problem Statement

## The Food Delivery Coordination Challenge

A food delivery platform isn't just a CRUD app. It's a **real-time coordination system** between three independent actors:

```
┌──────────────┐         ┌──────────────┐         ┌──────────────┐
│   Customer   │         │  Restaurant  │         │    Rider     │
│              │         │              │         │              │
│ "I'm hungry" │         │ "I'll cook"  │         │ "I'll deliver"│
└──────────────┘         └──────────────┘         └──────────────┘
       ↓                        ↓                        ↓
       └────────────────────────┴────────────────────────┘
                    Must coordinate in REAL-TIME
```

---

## The 3 Fundamental Challenges

### Challenge 1: The Matching Problem

**Scenario**: User wants food from a restaurant 5km away.

```javascript
// Naive approach
app.get('/restaurants', async (req, res) => {
  const restaurants = await Restaurant.find();
  res.json(restaurants);
});
```

**Problems**:
1. Returns ALL restaurants (even 100km away)
2. No distance calculation
3. No availability check (is restaurant open?)
4. No delivery radius validation

**What You Actually Need**:
```javascript
// Find restaurants within 5km of user's location
// Sort by distance
// Filter by: isOpen, hasAvailableRiders, deliveryRadius
// Calculate delivery time based on:
//   - Restaurant preparation time
//   - Distance to user
//   - Current rider availability
```

**Why It's Hard**:
- Geospatial queries (latitude/longitude math)
- Real-time availability data
- Dynamic pricing based on distance
- Rider availability prediction

---

### Challenge 2: The State Synchronization Problem

**Scenario**: Order goes through 8 state transitions, and 3 different people need to know about each change.

```
Order Lifecycle:
placed → accepted → preparing → ready_for_rider →
rider_assigned → picked_up → out_for_delivery → delivered
```

**Who Needs to Know What?**

| State | Customer | Restaurant | Rider |
|-------|----------|------------|-------|
| placed | ✅ "Order confirmed" | ✅ "New order!" | ❌ |
| accepted | ✅ "Restaurant accepted" | ✅ "You accepted" | ❌ |
| preparing | ✅ "Being prepared" | ✅ "Update status" | ❌ |
| ready_for_rider | ✅ "Ready for pickup" | ✅ "Waiting for rider" | ✅ "New delivery available!" |
| rider_assigned | ✅ "Rider assigned" | ✅ "Rider coming" | ✅ "You accepted" |
| picked_up | ✅ "On the way!" | ✅ "Order picked up" | ✅ "Navigate to customer" |
| delivered | ✅ "Enjoy your meal!" | ✅ "Order completed" | ✅ "Payment received" |

**Naive Approach**:
```javascript
// Update order status
app.put('/orders/:id', async (req, res) => {
  const order = await Order.findByIdAndUpdate(req.params.id, {
    status: req.body.status
  });

  // Now what? How do customer and rider know?
  // ❌ They don't! They have to keep polling:
  //    GET /orders/:id every 5 seconds
});
```

**The Polling Nightmare**:
```
100 active orders × 3 people per order × 12 requests/minute
= 3,600 requests/minute
= 60 requests/second
Just to check for updates!
```

**What You Actually Need**:
- Push notifications (not pull)
- Selective broadcasting (only notify relevant people)
- Guaranteed delivery (what if user's phone is offline?)
- Order of events (don't show "delivered" before "picked_up")

---

### Challenge 3: The Transaction Integrity Problem

**Scenario**: User places order, but payment fails. What happens?

```javascript
// Naive approach
app.post('/orders', async (req, res) => {
  // 1. Create order
  const order = await Order.create(req.body);

  // 2. Process payment
  const payment = await processPayment(order);

  // ❌ What if payment fails here?
  // Order already exists in database!
  // Restaurant might start preparing!

  if (!payment.success) {
    // Too late! Order already created!
    return res.status(400).json({ error: 'Payment failed' });
  }

  res.json({ order });
});
```

**The Race Conditions**:

```
Timeline:
00:00 - User places order
00:01 - Order created in DB (status: "placed")
00:02 - Restaurant sees new order, starts cooking
00:03 - Payment processing starts
00:05 - Payment FAILS
00:06 - Try to cancel order
00:07 - Restaurant already cooked the food! 💸
```

**What You Actually Need**:
```
1. Create order with status: "pending_payment"
2. Process payment (async, don't block user)
3. Only after payment succeeds:
   - Update order status to "placed"
   - Notify restaurant
4. If payment fails:
   - Auto-delete order after 15 minutes
   - Never notify restaurant
```

---

## Why REST APIs Aren't Enough

### Problem 1: Request-Response Model

```
REST API:
Client: "Hey server, what's the order status?"
Server: "It's 'preparing'"

[5 seconds later]
Client: "Hey server, what's the order status?"
Server: "Still 'preparing'"

[5 seconds later]
Client: "Hey server, what's the order status?"
Server: "Still 'preparing'"

[5 seconds later]
Client: "Hey server, what's the order status?"
Server: "It's 'ready_for_rider'"
```

**Inefficiency**:
- 3 wasted requests
- 5-10 second delay in updates
- Server processing unnecessary requests

**What You Need**:
```
WebSocket:
Server: "Order status changed to 'ready_for_rider'"
Client: [Instantly receives update]
```

### Problem 2: Synchronous Blocking

```javascript
// User clicks "Place Order"
app.post('/orders', async (req, res) => {
  const order = await Order.create(req.body);        // 50ms
  await processPayment(order);                       // 2000ms ⏳
  await notifyRestaurant(order);                     // 300ms
  await findNearbyRiders(order);                     // 500ms
  await sendConfirmationEmail(order);                // 400ms

  // User waits 3250ms before seeing "Order Placed"!
  res.json({ order });
});
```

**What You Need**:
```javascript
// User clicks "Place Order"
app.post('/orders', async (req, res) => {
  const order = await Order.create(req.body);        // 50ms

  // Immediately respond to user
  res.json({ order });  // User sees "Order Placed" in 50ms!

  // Everything else happens asynchronously
  publishEvent('ORDER_CREATED', order);  // RabbitMQ handles the rest
});
```

### Problem 3: Tight Coupling

```javascript
// All in one API endpoint
app.post('/orders', async (req, res) => {
  const order = await Order.create(req.body);

  // If payment service is down, entire endpoint fails
  const payment = await paymentService.charge(order);

  // If rider service is slow, user waits
  const rider = await riderService.findNearest(order);

  // If email service fails, order creation fails
  await emailService.send(order);

  res.json({ order });
});
```

**What You Need**:
```javascript
// Decoupled services
app.post('/orders', async (req, res) => {
  const order = await Order.create(req.body);
  res.json({ order });  // User gets response immediately
});

// Payment service listens for ORDER_CREATED event
paymentService.on('ORDER_CREATED', async (order) => {
  await processPayment(order);
  publishEvent('PAYMENT_SUCCESS', order);
});

// Rider service listens for PAYMENT_SUCCESS event
riderService.on('PAYMENT_SUCCESS', async (order) => {
  await findNearestRider(order);
});
```

---

## The Complexity Matrix

| Feature | Simple REST | What Actually Needed |
|---------|-------------|---------------------|
| Find nearby restaurants | `Restaurant.find()` | Geospatial queries with 2dsphere index |
| Real-time updates | Polling every 5s | WebSocket push notifications |
| Payment processing | Synchronous API call | Async event-driven with retry logic |
| Rider assignment | Find first available | Geospatial query + availability + load balancing |
| Order expiration | Manual cleanup | TTL index with automatic deletion |
| Service isolation | One server | Microservices with message queue |
| Scaling | Vertical (bigger server) | Horizontal (more instances) |

---

## The Real-World Constraints

### Constraint 1: Latency Requirements

```
User Expectations:
- Browse restaurants: < 500ms
- Add to cart: < 200ms
- Place order: < 1s
- Real-time updates: < 100ms

Reality with Simple Architecture:
- Browse restaurants: 2-5s (no geospatial index)
- Add to cart: 500ms (database contention)
- Place order: 3-5s (synchronous payment)
- Real-time updates: 5-10s (polling delay)
```

### Constraint 2: Concurrency

```
Peak Hour (8 PM Friday):
- 10,000 users browsing
- 1,000 active orders
- 500 riders online
- 200 restaurants receiving orders

Single Server Limits:
- Max connections: 1,000
- Max DB connections: 100
- Max memory: 8 GB
- Max CPU: 4 cores

Result: Server crashes at 2,000 concurrent users
```

### Constraint 3: Data Consistency

```
Scenario: Rider accepts order, but another rider accepted it 1ms earlier

Without Proper Locking:
Rider A: GET /orders/123  → { riderId: null }
Rider B: GET /orders/123  → { riderId: null }
Rider A: PUT /orders/123  → { riderId: "A" }
Rider B: PUT /orders/123  → { riderId: "B" }  // Overwrites A!

Result: Two riders show up at restaurant!
```

---

## The Questions We Must Answer

1. **How do we find nearby restaurants efficiently?**
   → Geospatial indexing (covered in document 08)

2. **How do we notify users in real-time without polling?**
   → WebSockets (covered in document 06)

3. **How do we process payments without blocking users?**
   → Event-driven architecture (covered in document 05)

4. **How do we prevent race conditions in rider assignment?**
   → Atomic database operations (covered in document 07)

5. **How do we scale different parts independently?**
   → Microservices (covered in document 03)

6. **How do we handle service failures gracefully?**
   → Message queues with retry logic (covered in document 05)

---

## The Architecture We Need

```
┌─────────────────────────────────────────────────────────────┐
│                         FRONTEND                             │
│  - WebSocket for real-time updates                          │
│  - Optimistic UI updates                                    │
│  - Offline support with queue                               │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                      MICROSERVICES                           │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │   Auth   │  │Restaurant│  │  Rider   │  │ Realtime │   │
│  │ Service  │  │ Service  │  │ Service  │  │ Service  │   │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘   │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                      MESSAGE QUEUE                           │
│  - Async event processing                                   │
│  - Decoupling services                                      │
│  - Retry logic                                              │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                   SPECIALIZED DATABASES                      │
│  - Geospatial indexes for location queries                 │
│  - TTL indexes for auto-expiration                          │
│  - Sharding for horizontal scaling                          │
└─────────────────────────────────────────────────────────────┘
```

---

## What's Next?

Now that we understand the problems, let's explore the solutions:

**Continue to**: [03_Why_Microservices.md](./03_Why_Microservices.md)

---

## Key Takeaways

- Food delivery is a **coordination problem**, not just CRUD
- REST APIs are **request-response**, but we need **push notifications**
- Synchronous processing **blocks users**, async processing **scales**
- Simple architecture **couples everything**, microservices **isolate failures**
- Geospatial queries need **specialized indexes**, not `find()`
- Race conditions are **real**, atomic operations are **necessary**

The complexity isn't artificial—it's inherent to the problem domain.
