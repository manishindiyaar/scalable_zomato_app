# 03 - Database Evolution: From Simple CRUD to Production-Grade

## The Simple Beginning: Basic MongoDB CRUD

When we started, our database operations looked like this:

```javascript
// Create a restaurant
app.post('/api/restaurant', async (req, res) => {
  const restaurant = await Restaurant.create({
    name: req.body.name,
    address: req.body.address,
    phone: req.body.phone
  });
  res.json(restaurant);
});

// Get all restaurants
app.get('/api/restaurants', async (req, res) => {
  const restaurants = await Restaurant.find();
  res.json(restaurants);
});

// Update restaurant
app.put('/api/restaurant/:id', async (req, res) => {
  const restaurant = await Restaurant.findByIdAndUpdate(
    req.params.id,
    req.body,
    { new: true }
  );
  res.json(restaurant);
});

// Delete restaurant
app.delete('/api/restaurant/:id', async (req, res) => {
  await Restaurant.findByIdAndDelete(req.params.id);
  res.json({ message: 'Deleted' });
});
```

**This works fine for:**
- Small user base (< 1000 users)
- Simple queries
- No real-time requirements
- Single server deployment

---

## Problem 1: "Show me restaurants near me"

### The Naive Approach (DOESN'T SCALE)

```javascript
app.get('/api/restaurants/nearby', async (req, res) => {
  const { latitude, longitude } = req.query;

  // Get ALL restaurants from database
  const allRestaurants = await Restaurant.find();

  // Calculate distance for EACH restaurant in JavaScript
  const nearby = allRestaurants.filter(restaurant => {
    const distance = calculateDistance(
      latitude, longitude,
      restaurant.latitude, restaurant.longitude
    );
    return distance < 5; // 5km radius
  });

  res.json(nearby);
});

function calculateDistance(lat1, lon1, lat2, lon2) {
  const R = 6371; // Earth's radius in km
  const dLat = (lat2 - lat1) * Math.PI / 180;
  const dLon = (lon2 - lon1) * Math.PI / 180;

  const a = Math.sin(dLat/2) * Math.sin(dLat/2) +
            Math.cos(lat1 * Math.PI / 180) * Math.cos(lat2 * Math.PI / 180) *
            Math.sin(dLon/2) * Math.sin(dLon/2);

  const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1-a));
  return R * c;
}
```

### Why This FAILS at Scale:

**Performance Breakdown:**
```
10 restaurants    → 10 distance calculations   → 5ms
100 restaurants   → 100 distance calculations  → 50ms
1,000 restaurants → 1,000 calculations         → 500ms
10,000 restaurants → 10,000 calculations       → 5 seconds ❌
```

**Problems:**
1. **Fetches ALL restaurants** from database (network overhead)
2. **Calculates distance in JavaScript** (CPU intensive)
3. **No indexing** - can't optimize the query
4. **Memory intensive** - loads entire dataset into RAM
5. **Blocks event loop** - Node.js single-threaded nature suffers

**Real-world impact:**
- User opens app → waits 5 seconds → closes app
- 100 concurrent users → server crashes
- Database connection pool exhausted

---

## Solution 1: MongoDB Geospatial Indexing

### First Principles: Let the Database Do the Work

**Key Insight:** Databases are optimized for searching. Don't bring data to code; bring code to data.

### Step 1: Change Data Structure (GeoJSON)

```javascript
// OLD Schema (Wrong)
const RestaurantSchema = new Schema({
  name: String,
  latitude: Number,
  longitude: Number
});

// NEW Schema (Correct)
const RestaurantSchema = new Schema({
  name: String,
  location: {
    type: {
      type: String,
      enum: ['Point'],
      required: true
    },
    coordinates: {
      type: [Number],  // [longitude, latitude] - ORDER MATTERS!
      required: true
    }
  }
});

// Create 2dsphere index
RestaurantSchema.index({ location: '2dsphere' });
```

**Why GeoJSON?**
- Industry standard (used by Google Maps, Mapbox, etc.)
- MongoDB has built-in operators: `$near`, `$geoWithin`, `$geoIntersects`
- Uses R-tree indexing (O(log n) search time)

### Step 2: Use Geospatial Query

```javascript
app.get('/api/restaurants/nearby', async (req, res) => {
  const { latitude, longitude } = req.query;

  const restaurants = await Restaurant.find({
    location: {
      $near: {
        $geometry: {
          type: 'Point',
          coordinates: [parseFloat(longitude), parseFloat(latitude)]
        },
        $maxDistance: 5000  // 5km in meters
      }
    }
  }).limit(20);  // Only return top 20

  res.json(restaurants);
});
```

### Performance Comparison:

```
Naive Approach (10,000 restaurants):
├─ Fetch all: 2000ms
├─ Calculate distances: 3000ms
└─ Total: 5000ms ❌

Geospatial Index (10,000 restaurants):
├─ Index lookup: 5ms
├─ Fetch results: 10ms
└─ Total: 15ms ✅

Speed improvement: 333x faster!
```

### How 2dsphere Index Works:

```
Without Index:
┌─────────────────────────────────────┐
│ Scan ALL 10,000 restaurants         │
│ Calculate distance for each         │
│ Sort by distance                    │
│ Return top 20                       │
└─────────────────────────────────────┘
Time: O(n) where n = total restaurants

With 2dsphere Index (R-tree):
┌─────────────────────────────────────┐
│         Root Node                   │
│    /              \                 │
│  NW Quadrant    SE Quadrant         │
│  /      \        /      \           │
│ Cell1  Cell2  Cell3  Cell4          │
│  |      |      |      |             │
│ [Restaurants in each cell]          │
└─────────────────────────────────────┘
Time: O(log n) - only searches relevant cells
```

---

## Problem 2: Order Expiration (Unpaid Orders)

### The Naive Approach (DOESN'T SCALE)

```javascript
// Create order with pending payment
app.post('/api/order', async (req, res) => {
  const order = await Order.create({
    userId: req.user._id,
    items: req.body.items,
    paymentStatus: 'pending',
    createdAt: new Date()
  });

  // Set timeout to delete after 15 minutes
  setTimeout(async () => {
    const order = await Order.findById(order._id);
    if (order.paymentStatus === 'pending') {
      await Order.findByIdAndDelete(order._id);
      console.log('Order expired:', order._id);
    }
  }, 15 * 60 * 1000);  // 15 minutes

  res.json(order);
});
```

### Why This FAILS:

**Problems:**
1. **Server restart = lost timers** - All setTimeout cleared
2. **Memory leak** - 10,000 pending orders = 10,000 timers in RAM
3. **Not distributed** - Only works on single server
4. **Race conditions** - Payment might complete while deleting

**Real-world scenario:**
```
10:00 AM - User creates order (timer starts)
10:05 AM - Server crashes (timer lost)
10:10 AM - Server restarts
10:15 AM - Order should expire, but timer is gone
Result: Database filled with expired orders ❌
```

---

## Solution 2: MongoDB TTL Index

### First Principles: Database Manages Its Own Data

**Key Insight:** Databases have background processes. Use them for time-based operations.

```javascript
const OrderSchema = new Schema({
  userId: String,
  items: Array,
  paymentStatus: {
    type: String,
    enum: ['pending', 'paid', 'failed'],
    default: 'pending'
  },
  expiresAt: {
    type: Date,
    index: { expireAfterSeconds: 0 }  // TTL index
  }
});

// Create order
app.post('/api/order', async (req, res) => {
  const order = await Order.create({
    userId: req.user._id,
    items: req.body.items,
    paymentStatus: 'pending',
    expiresAt: new Date(Date.now() + 15 * 60 * 1000)  // 15 minutes
  });

  res.json(order);
});

// When payment succeeds
app.post('/api/payment/success', async (req, res) => {
  await Order.findByIdAndUpdate(req.body.orderId, {
    paymentStatus: 'paid',
    $unset: { expiresAt: 1 }  // Remove expiration
  });

  res.json({ success: true });
});
```

### How TTL Index Works:

```
MongoDB Background Thread (runs every 60 seconds):
┌─────────────────────────────────────────────┐
│ 1. Find documents where:                    │
│    expiresAt < Date.now()                   │
│                                             │
│ 2. Delete matching documents                │
│                                             │
│ 3. Sleep for 60 seconds                     │
│                                             │
│ 4. Repeat                                   │
└─────────────────────────────────────────────┘

Benefits:
✅ Survives server restarts
✅ Works in distributed systems
✅ No memory overhead
✅ Automatic cleanup
```

---

## Problem 3: Race Conditions (Rider Assignment)

### The Naive Approach (DOESN'T SCALE)

```javascript
app.post('/api/order/assign-rider', async (req, res) => {
  const { orderId, riderId } = req.body;

  // Check if order already has a rider
  const order = await Order.findById(orderId);

  if (order.riderId) {
    return res.status(400).json({ error: 'Order already assigned' });
  }

  // Assign rider
  order.riderId = riderId;
  await order.save();

  res.json({ success: true });
});
```

### Why This FAILS:

**Race Condition Scenario:**
```
Time    Rider A                    Rider B
10:00   GET order (riderId: null)
10:01                              GET order (riderId: null)
10:02   SET riderId = A
10:03                              SET riderId = B (overwrites!)
10:04   Both riders think they got the order ❌
```

**Real-world impact:**
- Two riders show up at restaurant
- Customer confusion
- Wasted rider time
- Bad user experience

---

## Solution 3: Atomic Operations

### First Principles: Database Guarantees Atomicity

**Key Insight:** Use database's atomic operations to prevent race conditions.

```javascript
app.post('/api/order/assign-rider', async (req, res) => {
  const { orderId, riderId, riderName, riderPhone } = req.body;

  // Atomic operation: only update if riderId is null
  const order = await Order.findOneAndUpdate(
    {
      _id: orderId,
      riderId: null  // Condition: must be unassigned
    },
    {
      riderId,
      riderName,
      riderPhone,
      status: 'rider_assigned'
    },
    { new: true }  // Return updated document
  );

  if (!order) {
    return res.status(400).json({
      error: 'Order already assigned or not found'
    });
  }

  res.json({ success: true, order });
});
```

### How Atomic Operations Work:

```
MongoDB Internal Lock:
┌─────────────────────────────────────────────┐
│ Rider A: findOneAndUpdate(riderId: null)    │
│ ├─ Acquire lock on document                 │
│ ├─ Check condition (riderId === null) ✅    │
│ ├─ Update riderId = A                       │
│ └─ Release lock                             │
│                                             │
│ Rider B: findOneAndUpdate(riderId: null)    │
│ ├─ Acquire lock on document                 │
│ ├─ Check condition (riderId === null) ❌    │
│ │   (Already set to A)                      │
│ └─ Return null (no update)                  │
└─────────────────────────────────────────────┘

Result: Only Rider A gets the order ✅
```

---

## Problem 4: Embedded vs Referenced Data

### The Dilemma: Order History

**Scenario:** User places order with delivery address. Later, user changes their address. Should order history show old or new address?

### Approach 1: Reference (WRONG for this case)

```javascript
const OrderSchema = new Schema({
  userId: String,
  items: Array,
  addressId: { type: Schema.Types.ObjectId, ref: 'Address' }
});

const AddressSchema = new Schema({
  userId: String,
  street: String,
  city: String
});

// Get order with address
const order = await Order.findById(orderId).populate('addressId');
console.log(order.addressId.street);  // Shows CURRENT address ❌
```

**Problem:** If user updates address, order history changes retroactively!

```
Timeline:
10:00 AM - Order placed to "123 Old Street"
11:00 AM - User updates address to "456 New Street"
12:00 PM - View order history → shows "456 New Street" ❌

This is wrong! Order was delivered to old address.
```

### Approach 2: Embed (CORRECT for this case)

```javascript
const OrderSchema = new Schema({
  userId: String,
  items: Array,
  addressId: String,  // Reference for lookup
  deliveryAddress: {  // Snapshot at order time
    formattedAddress: String,
    mobile: Number,
    latitude: Number,
    longitude: Number
  }
});

// Create order
app.post('/api/order', async (req, res) => {
  const address = await Address.findById(req.body.addressId);

  const order = await Order.create({
    userId: req.user._id,
    items: req.body.items,
    addressId: address._id,
    deliveryAddress: {  // Snapshot
      formattedAddress: address.formattedAddress,
      mobile: address.mobile,
      latitude: address.location.coordinates[1],
      longitude: address.location.coordinates[0]
    }
  });

  res.json(order);
});
```

**Result:** Order history always shows address at time of order ✅

---

## Decision Matrix: Embed vs Reference

| Criteria | Embed | Reference |
|----------|-------|-----------|
| Data changes frequently | ❌ | ✅ |
| Need historical snapshot | ✅ | ❌ |
| Data is large (>16MB) | ❌ | ✅ |
| Need to query independently | ❌ | ✅ |
| One-to-few relationship | ✅ | ❌ |
| One-to-many relationship | ❌ | ✅ |

**Examples:**
- **Embed:** Order items, delivery address, payment details
- **Reference:** User profile, restaurant details, rider info

---

## Database Performance Optimization

### 1. Indexing Strategy

```javascript
// Single field index
RestaurantSchema.index({ ownerId: 1 });

// Compound index (order matters!)
OrderSchema.index({ userId: 1, createdAt: -1 });

// Geospatial index
RestaurantSchema.index({ location: '2dsphere' });

// Text search index
RestaurantSchema.index({ name: 'text', description: 'text' });

// TTL index
OrderSchema.index({ expiresAt: 1 }, { expireAfterSeconds: 0 });
```

### 2. Query Optimization

```javascript
// BAD: Fetches all fields
const orders = await Order.find({ userId });

// GOOD: Only fetch needed fields
const orders = await Order.find({ userId })
  .select('items totalAmount status createdAt')
  .lean();  // Returns plain JS object (faster)

// BAD: N+1 query problem
const orders = await Order.find({ userId });
for (const order of orders) {
  const restaurant = await Restaurant.findById(order.restaurantId);
  order.restaurant = restaurant;
}

// GOOD: Use populate
const orders = await Order.find({ userId })
  .populate('restaurantId', 'name image');
```

### 3. Connection Pooling

```javascript
// BAD: New connection per request
mongoose.connect(MONGO_URI);

// GOOD: Connection pool
mongoose.connect(MONGO_URI, {
  maxPoolSize: 10,  // Max 10 concurrent connections
  minPoolSize: 2,   // Keep 2 connections alive
  socketTimeoutMS: 45000,
  serverSelectionTimeoutMS: 5000
});
```

---

## Summary: Database Evolution

```
Simple CRUD
    ↓
Problem: Geospatial queries slow
    ↓
Solution: 2dsphere indexing
    ↓
Problem: Order expiration management
    ↓
Solution: TTL indexes
    ↓
Problem: Race conditions
    ↓
Solution: Atomic operations
    ↓
Problem: Data consistency
    ↓
Solution: Embed vs Reference strategy
    ↓
Production-Ready Database
```

**Key Takeaways:**
1. **Let the database do the work** - Use built-in features
2. **Indexes are critical** - But don't over-index
3. **Atomic operations prevent bugs** - Use findOneAndUpdate
4. **Embed for snapshots** - Reference for live data
5. **Monitor query performance** - Use explain() to analyze

Next: [04 - API Architecture Evolution](./04_API_Architecture_Evolution.md)
