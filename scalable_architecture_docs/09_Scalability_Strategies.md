# 09. Scalability Strategies: From 1K to 1M Users

## The Scaling Journey

### Phase 1: 0-10K Users (Single Server)
**Architecture:**
```
┌─────────────────┐
│   Single VPS    │
│  ┌──────────┐   │
│  │ Node.js  │   │
│  │ MongoDB  │   │
│  │  Redis   │   │
│  └──────────┘   │
└─────────────────┘
```

**Characteristics:**
- Everything on one machine
- Vertical scaling (upgrade RAM/CPU)
- Cost: $20-50/month
- Response time: 50-200ms

**When to Move On:**
- CPU consistently > 70%
- Memory usage > 80%
- Response times > 500ms
- Database queries slowing down

---

### Phase 2: 10K-50K Users (Separation of Concerns)
**Architecture:**
```
┌──────────────┐     ┌──────────────┐
│  App Server  │────▶│   Database   │
│   Node.js    │     │   MongoDB    │
└──────────────┘     └──────────────┘
       │
       ▼
┌──────────────┐
│    Redis     │
│    Cache     │
└──────────────┘
```

**Key Changes:**
1. **Separate Database Server**
   - Dedicated resources for DB
   - Better backup/recovery
   - Independent scaling

2. **Redis for Caching**
   ```javascript
   // Cache frequently accessed data
   async function getRestaurant(id) {
     const cached = await redis.get(`restaurant:${id}`);
     if (cached) return JSON.parse(cached);

     const restaurant = await Restaurant.findById(id);
     await redis.setex(`restaurant:${id}`, 3600, JSON.stringify(restaurant));
     return restaurant;
   }
   ```

3. **CDN for Static Assets**
   - Images, CSS, JS on CloudFront/Cloudflare
   - 90% reduction in bandwidth costs

**Cost:** $200-500/month

---

### Phase 3: 50K-200K Users (Horizontal Scaling)
**Architecture:**
```
                ┌──────────────┐
                │ Load Balancer│
                └──────┬───────┘
                       │
        ┌──────────────┼──────────────┐
        ▼              ▼              ▼
   ┌────────┐     ┌────────┐     ┌────────┐
   │ App 1  │     │ App 2  │     │ App 3  │
   └────────┘     └────────┘     └────────┘
        │              │              │
        └──────────────┼──────────────┘
                       ▼
              ┌─────────────────┐
              │  Database Pool  │
              │  Primary + Read │
              │    Replicas     │
              └─────────────────┘
```

**Implementation:**

1. **Load Balancer (Nginx)**
   ```nginx
   upstream app_servers {
     least_conn;
     server app1.internal:3000;
     server app2.internal:3000;
     server app3.internal:3000;
   }

   server {
     listen 80;
     location / {
       proxy_pass http://app_servers;
       proxy_set_header X-Real-IP $remote_addr;
     }
   }
   ```

2. **Database Read Replicas**
   ```javascript
   // Write to primary
   await Restaurant.create(data);

   // Read from replica
   const restaurants = await Restaurant.find()
     .read('secondary')
     .exec();
   ```

3. **Session Management**
   ```javascript
   // Store sessions in Redis (not in-memory)
   app.use(session({
     store: new RedisStore({ client: redisClient }),
     secret: process.env.SESSION_SECRET,
     resave: false,
     saveUninitialized: false
   }));
   ```

**Cost:** $1,000-2,000/month

---

### Phase 4: 200K-500K Users (Microservices)
**Architecture:**
```
                    ┌─────────────┐
                    │  API Gateway│
                    └──────┬──────┘
                           │
        ┌──────────────────┼──────────────────┐
        ▼                  ▼                  ▼
   ┌─────────┐       ┌─────────┐       ┌─────────┐
   │  User   │       │  Order  │       │Restaurant│
   │ Service │       │ Service │       │ Service │
   └────┬────┘       └────┬────┘       └────┬────┘
        │                 │                  │
        ▼                 ▼                  ▼
   ┌─────────┐       ┌─────────┐       ┌─────────┐
   │ User DB │       │Order DB │       │Rest. DB │
   └─────────┘       └─────────┘       └─────────┘
```

**Benefits:**
- Independent scaling per service
- Team autonomy
- Technology flexibility
- Fault isolation

**Challenges:**
- Distributed transactions
- Service discovery
- Monitoring complexity

**Example: Order Service**
```javascript
// orders-service/index.js
const express = require('express');
const app = express();

app.post('/orders', async (req, res) => {
  const { userId, restaurantId, items } = req.body;

  // Call other services
  const user = await fetch(`http://user-service/users/${userId}`);
  const restaurant = await fetch(`http://restaurant-service/restaurants/${restaurantId}`);

  // Create order
  const order = await Order.create({ userId, restaurantId, items });

  // Publish event
  await eventBus.publish('order.created', order);

  res.json(order);
});

app.listen(3001);
```

**Cost:** $3,000-5,000/month

---

### Phase 5: 500K-1M Users (Full Distribution)
**Architecture:**
```
┌─────────────────────────────────────────────┐
│              Global CDN                     │
└─────────────────┬───────────────────────────┘
                  │
         ┌────────┴────────┐
         ▼                 ▼
    ┌─────────┐       ┌─────────┐
    │ Region  │       │ Region  │
    │   US    │       │   EU    │
    └────┬────┘       └────┬────┘
         │                 │
    ┌────┴────┐       ┌────┴────┐
    │ Service │       │ Service │
    │  Mesh   │       │  Mesh   │
    └────┬────┘       └────┬────┘
         │                 │
    ┌────┴────┐       ┌────┴────┐
    │Sharded  │       │Sharded  │
    │Database │       │Database │
    └─────────┘       └─────────┘
```

**Key Strategies:**

1. **Database Sharding**
   ```javascript
   // Shard by user ID
   function getShardForUser(userId) {
     const hash = crypto.createHash('md5').update(userId).digest('hex');
     const shardNum = parseInt(hash.substring(0, 8), 16) % NUM_SHARDS;
     return shardConnections[shardNum];
   }

   async function getUserOrders(userId) {
     const db = getShardForUser(userId);
     return db.collection('orders').find({ userId }).toArray();
   }
   ```

2. **Caching Layers**
   ```
   Browser Cache (5 min)
        ↓
   CDN Cache (1 hour)
        ↓
   Redis Cache (15 min)
        ↓
   Database
   ```

3. **Async Processing**
   ```javascript
   // Don't wait for email
   app.post('/orders', async (req, res) => {
     const order = await Order.create(req.body);

     // Queue background jobs
     await queue.add('send-confirmation-email', { orderId: order.id });
     await queue.add('notify-restaurant', { orderId: order.id });
     await queue.add('update-analytics', { orderId: order.id });

     res.json(order); // Respond immediately
   });
   ```

4. **Circuit Breakers**
   ```javascript
   const circuitBreaker = require('opossum');

   const options = {
     timeout: 3000,
     errorThresholdPercentage: 50,
     resetTimeout: 30000
   };

   const breaker = circuitBreaker(callExternalService, options);

   breaker.fallback(() => ({
     status: 'degraded',
     message: 'Using cached data'
   }));
   ```

**Cost:** $10,000-20,000/month

---

## Performance Optimization Techniques

### 1. Database Indexing
```javascript
// Before: 2000ms query
db.orders.find({ userId: "123", status: "pending" })

// After: 20ms query
db.orders.createIndex({ userId: 1, status: 1 })
```

### 2. Query Optimization
```javascript
// Bad: N+1 queries
const orders = await Order.find({ userId });
for (let order of orders) {
  order.restaurant = await Restaurant.findById(order.restaurantId);
}

// Good: Single query with populate
const orders = await Order.find({ userId })
  .populate('restaurant')
  .exec();
```

### 3. Connection Pooling
```javascript
const mongoose = require('mongoose');

mongoose.connect(process.env.MONGODB_URI, {
  maxPoolSize: 50,
  minPoolSize: 10,
  socketTimeoutMS: 45000,
});
```

### 4. Compression
```javascript
const compression = require('compression');
app.use(compression());

// Reduces response size by 70-90%
```

### 5. Rate Limiting
```javascript
const rateLimit = require('express-rate-limit');

const limiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100 // limit each IP to 100 requests per windowMs
});

app.use('/api/', limiter);
```

---

## Monitoring & Metrics

### Key Metrics to Track

1. **Application Metrics**
   - Request rate (req/sec)
   - Response time (p50, p95, p99)
   - Error rate (%)
   - Active connections

2. **Database Metrics**
   - Query time
   - Connection pool usage
   - Cache hit rate
   - Replication lag

3. **Infrastructure Metrics**
   - CPU usage
   - Memory usage
   - Disk I/O
   - Network throughput

### Implementation
```javascript
const prometheus = require('prom-client');

// Request duration histogram
const httpRequestDuration = new prometheus.Histogram({
  name: 'http_request_duration_seconds',
  help: 'Duration of HTTP requests in seconds',
  labelNames: ['method', 'route', 'status_code']
});

app.use((req, res, next) => {
  const start = Date.now();
  res.on('finish', () => {
    const duration = (Date.now() - start) / 1000;
    httpRequestDuration
      .labels(req.method, req.route?.path, res.statusCode)
      .observe(duration);
  });
  next();
});
```

---

## Cost Optimization

### Scaling Cost Comparison

| Users | Architecture | Monthly Cost | Cost per User |
|-------|-------------|--------------|---------------|
| 1K | Single server | $50 | $0.05 |
| 10K | Separated | $500 | $0.05 |
| 50K | Horizontal | $2,000 | $0.04 |
| 200K | Microservices | $5,000 | $0.025 |
| 1M | Distributed | $20,000 | $0.02 |

### Cost Saving Tips

1. **Use Spot Instances** (AWS/GCP)
   - 70% cheaper than on-demand
   - Good for stateless services

2. **Auto-scaling**
   - Scale down during off-peak hours
   - Save 30-40% on compute costs

3. **Reserved Instances**
   - 40-60% discount for 1-3 year commitment
   - Use for baseline capacity

4. **Optimize Data Transfer**
   - Use CDN for static assets
   - Compress responses
   - Reduce API payload sizes

---

## Next Steps

Now that you understand scaling strategies, let's see how everything comes together in [10. Complete Order Flow](./10_Complete_Order_Flow.md) - a real-world example showing all these concepts in action.

---

## Quick Reference

### When to Scale What

| Bottleneck | Solution | When |
|------------|----------|------|
| Slow API | Add caching | Response > 500ms |
| High CPU | Horizontal scaling | CPU > 70% |
| Slow queries | Add indexes | Query > 100ms |
| Memory issues | Vertical scaling | Memory > 80% |
| Database load | Read replicas | DB CPU > 60% |
| Global latency | Multi-region | Users worldwide |

### Scaling Checklist

- [ ] Monitor key metrics
- [ ] Set up alerts
- [ ] Implement caching
- [ ] Optimize database queries
- [ ] Add load balancing
- [ ] Use CDN for static assets
- [ ] Implement rate limiting
- [ ] Set up auto-scaling
- [ ] Plan for disaster recovery
- [ ] Document architecture

---

**Remember:** Premature optimization is the root of all evil. Scale when you need to, not before. Start simple, measure everything, and scale based on data, not assumptions.
