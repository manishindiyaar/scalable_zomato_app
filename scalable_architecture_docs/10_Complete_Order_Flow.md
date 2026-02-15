# Chapter 10: Complete Order Flow - Putting It All Together

## Introduction

We've covered individual components - databases, microservices, events, real-time communication, APIs, security, and scalability strategies. Now let's see how they all work together in a complete order flow from start to finish.

This chapter traces a single order through our entire system, showing how every architectural decision we've made contributes to handling 1M+ users.

---

## The Complete Journey of an Order

### Step 1: User Places Order (Frontend → API Gateway)

```javascript
// User clicks "Place Order" button
async function placeOrder(cartItems, deliveryAddress) {
  const response = await fetch('https://api.tomato.com/orders', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${accessToken}`,
      'Content-Type': 'application/json',
      'X-Request-ID': generateRequestId(),
      'X-Idempotency-Key': generateIdempotencyKey()
    },
    body: JSON.stringify({
      items: cartItems,
      delivery_address: deliveryAddress,
      payment_method: 'card_ending_1234'
    })
  });

  return response.json();
}
```

**What Happens:**
- Request hits API Gateway (Kong/AWS API Gateway)
- Rate limiting checked (100 requests/minute per user)
- JWT token validated
- Request routed to Order Service

---

### Step 2: Order Service Validates Request

```python
# Order Service - POST /orders endpoint
@app.post("/orders")
async def create_order(
    order_data: OrderCreate,
    user_id: str = Depends(get_current_user),
    idempotency_key: str = Header(...)
):
    # Check idempotency - prevent duplicate orders
    existing_order = await check_idempotency(idempotency_key)
    if existing_order:
        return existing_order

    # Validate cart items exist and are available
    items_valid = await validate_items(order_data.items)
    if not items_valid:
        raise HTTPException(400, "Invalid items")

    # Check delivery address is in serviceable area
    serviceable = await check_serviceability(order_data.delivery_address)
    if not serviceable:
        raise HTTPException(400, "Area not serviceable")

    # Create order in database (status: PENDING)
    order = await create_order_record(user_id, order_data)

    # Publish OrderCreated event
    await event_bus.publish("order.created", {
        "order_id": order.id,
        "user_id": user_id,
        "items": order.items,
        "total_amount": order.total_amount,
        "timestamp": datetime.utcnow()
    })

    return {"order_id": order.id, "status": "PENDING"}
```

**Database Write:**
```sql
-- Orders table (sharded by user_id)
INSERT INTO orders (
    id, user_id, status, total_amount,
    delivery_address, created_at
) VALUES (
    'ord_123', 'usr_456', 'PENDING', 599.00,
    '123 Main St', NOW()
);

-- Order items table
INSERT INTO order_items (order_id, item_id, quantity, price)
VALUES
    ('ord_123', 'item_1', 2, 299.00),
    ('ord_123', 'item_2', 1, 300.00);
```

---

### Step 3: Inventory Service Reserves Items

```python
# Inventory Service - Listens to order.created event
@event_handler("order.created")
async def handle_order_created(event_data):
    order_id = event_data['order_id']
    items = event_data['items']

    try:
        # Reserve inventory using Redis for fast locking
        for item in items:
            reserved = await reserve_inventory(
                item_id=item['id'],
                quantity=item['quantity'],
                order_id=order_id,
                ttl=600  # 10 minute reservation
            )

            if not reserved:
                # Rollback previous reservations
                await rollback_reservations(order_id)

                # Publish failure event
                await event_bus.publish("inventory.reservation.failed", {
                    "order_id": order_id,
                    "reason": "insufficient_stock"
                })
                return

        # All items reserved successfully
        await event_bus.publish("inventory.reserved", {
            "order_id": order_id,
            "items": items
        })

    except Exception as e:
        logger.error(f"Inventory reservation failed: {e}")
        await event_bus.publish("inventory.reservation.failed", {
            "order_id": order_id,
            "reason": str(e)
        })
```

**Redis Operations:**
```python
# Reserve inventory in Redis
async def reserve_inventory(item_id, quantity, order_id, ttl):
    # Use Lua script for atomic operation
    lua_script = """
    local current = redis.call('GET', KEYS[1])
    if not current or tonumber(current) < tonumber(ARGV[1]) then
        return 0
    end
    redis.call('DECRBY', KEYS[1], ARGV[1])
    redis.call('SETEX', KEYS[2], ARGV[3], ARGV[1])
    return 1
    """

    result = await redis.eval(
        lua_script,
        keys=[f"inventory:{item_id}", f"reservation:{order_id}:{item_id}"],
        args=[quantity, quantity, ttl]
    )

    return result == 1
```

---

### Step 4: Payment Service Processes Payment

```python
# Payment Service - Listens to inventory.reserved event
@event_handler("inventory.reserved")
async def handle_inventory_reserved(event_data):
    order_id = event_data['order_id']

    # Get order details
    order = await get_order(order_id)

    try:
        # Call payment gateway (Stripe/Razorpay)
        payment_result = await payment_gateway.charge(
            amount=order.total_amount,
            currency="USD",
            payment_method=order.payment_method,
            idempotency_key=f"payment_{order_id}"
        )

        if payment_result.status == "succeeded":
            # Store payment record
            await store_payment(
                order_id=order_id,
                transaction_id=payment_result.id,
                amount=order.total_amount,
                status="SUCCESS"
            )

            # Publish success event
            await event_bus.publish("payment.completed", {
                "order_id": order_id,
                "transaction_id": payment_result.id,
                "amount": order.total_amount
            })
        else:
            raise PaymentFailedException(payment_result.error)

    except Exception as e:
        logger.error(f"Payment failed: {e}")

        # Publish failure event
        await event_bus.publish("payment.failed", {
            "order_id": order_id,
            "reason": str(e)
        })
```

---

### Step 5: Order Service Updates Order Status

```python
# Order Service - Listens to payment.completed event
@event_handler("payment.completed")
async def handle_payment_completed(event_data):
    order_id = event_data['order_id']

    # Update order status to CONFIRMED
    await update_order_status(order_id, "CONFIRMED")

    # Publish order confirmed event
    await event_bus.publish("order.confirmed", {
        "order_id": order_id,
        "timestamp": datetime.utcnow()
    })

    # Send real-time notification to user
    await websocket_manager.send_to_user(
        user_id=order.user_id,
        message={
            "type": "ORDER_CONFIRMED",
            "order_id": order_id,
            "estimated_delivery": calculate_delivery_time()
        }
    )
```

---

### Step 6: Restaurant Service Assigns Restaurant

```python
# Restaurant Service - Listens to order.confirmed event
@event_handler("order.confirmed")
async def handle_order_confirmed(event_data):
    order_id = event_data['order_id']
    order = await get_order(order_id)

    # Find nearest available restaurant
    restaurant = await find_nearest_restaurant(
        location=order.delivery_address,
        items=order.items,
        max_distance_km=5
    )

    if not restaurant:
        await event_bus.publish("restaurant.assignment.failed", {
            "order_id": order_id,
            "reason": "no_restaurant_available"
        })
        return

    # Assign order to restaurant
    await assign_order_to_restaurant(order_id, restaurant.id)

    # Publish event
    await event_bus.publish("restaurant.assigned", {
        "order_id": order_id,
        "restaurant_id": restaurant.id,
        "restaurant_name": restaurant.name
    })

    # Notify restaurant via WebSocket
    await websocket_manager.send_to_restaurant(
        restaurant_id=restaurant.id,
        message={
            "type": "NEW_ORDER",
            "order_id": order_id,
            "items": order.items
        }
    )
```

---

### Step 7: Restaurant Prepares Food

```python
# Restaurant accepts order via mobile app
@app.post("/restaurants/{restaurant_id}/orders/{order_id}/accept")
async def accept_order(restaurant_id: str, order_id: str):
    await update_order_status(order_id, "PREPARING")

    await event_bus.publish("order.preparing", {
        "order_id": order_id,
        "restaurant_id": restaurant_id
    })

    # Notify user
    await websocket_manager.send_to_user(
        user_id=order.user_id,
        message={
            "type": "ORDER_PREPARING",
            "order_id": order_id,
            "restaurant_name": restaurant.name
        }
    )

    return {"status": "accepted"}

# Restaurant marks food ready
@app.post("/restaurants/{restaurant_id}/orders/{order_id}/ready")
async def mark_order_ready(restaurant_id: str, order_id: str):
    await update_order_status(order_id, "READY_FOR_PICKUP")

    await event_bus.publish("order.ready", {
        "order_id": order_id,
        "restaurant_id": restaurant_id
    })

    return {"status": "ready"}
```

---

### Step 8: Delivery Service Assigns Driver

```python
# Delivery Service - Listens to order.ready event
@event_handler("order.ready")
async def handle_order_ready(event_data):
    order_id = event_data['order_id']
    restaurant_id = event_data['restaurant_id']

    order = await get_order(order_id)
    restaurant = await get_restaurant(restaurant_id)

    # Find nearest available driver using geospatial query
    driver = await find_nearest_driver(
        location=restaurant.location,
        max_distance_km=3
    )

    if not driver:
        # Add to queue, retry in 30 seconds
        await add_to_assignment_queue(order_id)
        return

    # Assign driver
    await assign_driver(order_id, driver.id)

    await event_bus.publish("driver.assigned", {
        "order_id": order_id,
        "driver_id": driver.id,
        "driver_name": driver.name,
        "driver_phone": driver.phone
    })

    # Notify user and driver
    await notify_driver_assignment(order, driver)
```

**Geospatial Query (PostgreSQL with PostGIS):**
```sql
-- Find nearest available drivers
SELECT
    id, name, phone,
    ST_Distance(
        location::geography,
        ST_SetSRID(ST_MakePoint($1, $2), 4326)::geography
    ) as distance
FROM drivers
WHERE
    status = 'AVAILABLE'
    AND ST_DWithin(
        location::geography,
        ST_SetSRID(ST_MakePoint($1, $2), 4326)::geography,
        3000  -- 3km radius
    )
ORDER BY distance
LIMIT 1;
```

---

### Step 9: Real-Time Tracking

```python
# Driver's app sends location updates every 5 seconds
@app.post("/drivers/{driver_id}/location")
async def update_driver_location(
    driver_id: str,
    location: LocationUpdate
):
    # Update location in Redis (fast writes)
    await redis.geoadd(
        "driver_locations",
        location.longitude,
        location.latitude,
        driver_id
    )

    # Get active delivery for this driver
    delivery = await get_active_delivery(driver_id)

    if delivery:
        # Publish location update event
        await event_bus.publish("driver.location.updated", {
            "order_id": delivery.order_id,
            "driver_id": driver_id,
            "location": {
                "lat": location.latitude,
                "lng": location.longitude
            },
            "timestamp": datetime.utcnow()
        })

        # Send to user via WebSocket
        await websocket_manager.send_to_user(
            user_id=delivery.user_id,
            message={
                "type": "DRIVER_LOCATION",
                "order_id": delivery.order_id,
                "location": {
                    "lat": location.latitude,
                    "lng": location.longitude
                }
            }
        )

    return {"status": "updated"}
```

---

### Step 10: Order Delivered

```python
# Driver marks order as delivered
@app.post("/deliveries/{delivery_id}/complete")
async def complete_delivery(
    delivery_id: str,
    completion_data: DeliveryCompletion
):
    delivery = await get_delivery(delivery_id)

    # Update order status
    await update_order_status(delivery.order_id, "DELIVERED")

    # Publish event
    await event_bus.publish("order.delivered", {
        "order_id": delivery.order_id,
        "delivery_id": delivery_id,
        "delivered_at": datetime.utcnow(),
        "proof_of_delivery": completion_data.photo_url
    })

    # Notify user
    await websocket_manager.send_to_user(
        user_id=delivery.user_id,
        message={
            "type": "ORDER_DELIVERED",
            "order_id": delivery.order_id
        }
    )

    # Release inventory reservation
    await release_inventory_reservation(delivery.order_id)

    # Trigger analytics event
    await analytics.track("order_completed", {
        "order_id": delivery.order_id,
        "total_time_minutes": calculate_total_time(delivery.order_id)
    })

    return {"status": "completed"}
```

---

### Step 11: Post-Delivery Processing

```python
# Analytics Service - Listens to order.delivered event
@event_handler("order.delivered")
async def handle_order_delivered(event_data):
    order_id = event_data['order_id']

    # Calculate metrics
    metrics = await calculate_order_metrics(order_id)

    # Store in analytics database (ClickHouse)
    await analytics_db.insert("order_metrics", {
        "order_id": order_id,
        "total_time_minutes": metrics.total_time,
        "preparation_time_minutes": metrics.prep_time,
        "delivery_time_minutes": metrics.delivery_time,
        "delivered_at": event_data['delivered_at']
    })

# Notification Service - Send rating request
@event_handler("order.delivered")
async def send_rating_request(event_data):
    order_id = event_data['order_id']
    order = await get_order(order_id)

    # Wait 5 minutes, then send rating request
    await schedule_task(
        delay_seconds=300,
        task="send_rating_notification",
        params={"order_id": order_id, "user_id": order.user_id}
    )
```

---

## Error Handling & Rollback Scenarios

### Scenario 1: Payment Fails

```python
# Payment Service publishes payment.failed event
@event_handler("payment.failed")
async def handle_payment_failed(event_data):
    order_id = event_data['order_id']

    # Update order status
    await update_order_status(order_id, "PAYMENT_FAILED")

    # Release inventory reservations
    await release_inventory_reservation(order_id)

    # Notify user
    await websocket_manager.send_to_user(
        user_id=order.user_id,
        message={
            "type": "PAYMENT_FAILED",
            "order_id": order_id,
            "reason": event_data['reason']
        }
    )
```

### Scenario 2: No Restaurant Available

```python
@event_handler("restaurant.assignment.failed")
async def handle_no_restaurant(event_data):
    order_id = event_data['order_id']

    # Refund payment
    await initiate_refund(order_id)

    # Update order status
    await update_order_status(order_id, "CANCELLED")

    # Release inventory
    await release_inventory_reservation(order_id)

    # Notify user
    await notify_user_cancellation(order_id, "No restaurant available")
```

### Scenario 3: Driver Cancels

```python
@event_handler("driver.cancelled")
async def handle_driver_cancellation(event_data):
    order_id = event_data['order_id']

    # Try to find another driver
    await reassign_driver(order_id)

    # If no driver found after 3 attempts, cancel order
    attempts = await get_assignment_attempts(order_id)
    if attempts >= 3:
        await cancel_order_and_refund(order_id)
```

---

## System-Wide View

### Data Flow Diagram

```
User App
   ↓ (HTTPS)
API Gateway (Rate Limiting, Auth)
   ↓
Order Service → PostgreSQL (Write)
   ↓ (Event: order.created)
Kafka/RabbitMQ Event Bus
   ↓ (Fan-out to multiple services)
   ├→ Inventory Service → Redis (Reserve)
   ├→ Payment Service → Stripe API
   ├→ Notification Service → FCM/APNS
   └→ Analytics Service → ClickHouse

   ↓ (Event: payment.completed)
Order Service → Update Status
   ↓ (Event: order.confirmed)
Restaurant Service → Assign Restaurant
   ↓ (WebSocket)
Restaurant App

   ↓ (Event: order.ready)
Delivery Service → Find Driver (PostGIS)
   ↓ (WebSocket)
Driver App → Location Updates (Redis)
   ↓ (WebSocket)
User App (Real-time tracking)
```

---

## Performance Characteristics

### Latency Breakdown (Target)

```
User clicks "Place Order"
├─ API Gateway: 5ms
├─ Order Service validation: 20ms
├─ Database write: 10ms
├─ Event publish: 5ms
└─ Response to user: 40ms total ✓

Background Processing (async):
├─ Inventory reservation: 50ms
├─ Payment processing: 500ms
├─ Restaurant assignment: 100ms
└─ User notification: 200ms
```

### Throughput Capacity

```
- Orders per second: 10,000
- Concurrent users: 1,000,000
- WebSocket connections: 500,000
- Database writes/sec: 50,000
- Event messages/sec: 100,000
- Cache hits: 95%
```

---

## Monitoring & Observability

### Key Metrics to Track

```python
# Order funnel metrics
metrics = {
    "orders_created": Counter,
    "orders_confirmed": Counter,
    "orders_delivered": Counter,
    "orders_cancelled": Counter,

    # Latency metrics
    "order_creation_latency": Histogram,
    "payment_processing_latency": Histogram,
    "total_order_time": Histogram,

    # Error rates
    "payment_failures": Counter,
    "inventory_failures": Counter,
    "restaurant_assignment_failures": Counter,

    # Business metrics
    "revenue_per_minute": Gauge,
    "active_orders": Gauge,
    "average_order_value": Gauge
}
```

### Distributed Tracing

```python
# Each request gets a trace ID
@app.post("/orders")
async def create_order(request: Request):
    trace_id = request.headers.get("X-Trace-ID") or generate_trace_id()

    with tracer.start_span("create_order", trace_id=trace_id) as span:
        span.set_tag("user_id", user_id)
        span.set_tag("order_value", order.total_amount)

        # All downstream calls include this trace_id
        await process_order(order, trace_id=trace_id)
```

---

## Scaling Considerations

### Current Load: 1M Users

```
- Database: 5 PostgreSQL shards (by user_id)
- Cache: 3-node Redis cluster
- Services: 20 instances per service (auto-scaling)
- Message queue: 5-node Kafka cluster
- WebSocket servers: 50 instances (10k connections each)
```

### Scaling to 10M Users

```
- Database: 20 shards + read replicas
- Cache: 10-node Redis cluster with sharding
- Services: 100+ instances (horizontal scaling)
- Message queue: 15-node Kafka cluster
- WebSocket: 200+ instances
- CDN: CloudFront/Cloudflare for static assets
- Multi-region deployment
```

---

## Key Takeaways

1. **Async is King**: Payment, inventory, notifications all happen asynchronously
2. **Events Enable Decoupling**: Services don't call each other directly
3. **Real-time Matters**: WebSockets provide instant updates
4. **Idempotency Prevents Duplicates**: Critical for payments and orders
5. **Graceful Degradation**: System handles failures without cascading
6. **Observability**: Trace every request end-to-end
7. **Horizontal Scaling**: Add more instances, not bigger servers
8. **Data Locality**: Shard data to keep related data together

---

## What We've Built

From a simple CRUD app to a system that can:
- Handle 1M+ concurrent users
- Process 10,000 orders/second
- Provide real-time tracking
- Gracefully handle failures
- Scale horizontally
- Maintain 99.9% uptime

**This is production-grade architecture.**

---

## Next Steps

1. **Practice**: Build a simplified version yourself
2. **Learn**: Study open-source projects (Uber, DoorDash engineering blogs)
3. **Experiment**: Try different databases, message queues, patterns
4. **Measure**: Always benchmark and profile
5. **Iterate**: Start simple, scale when needed

---

## Conclusion

You've now seen how every piece fits together - from the moment a user clicks "Place Order" to the food arriving at their door. Every architectural decision we made throughout this series contributes to this flow working smoothly at scale.

**Remember**: You don't build this on day one. You start simple and evolve as you grow. But now you know the destination and the path to get there.

Happy building! 🚀🍕
