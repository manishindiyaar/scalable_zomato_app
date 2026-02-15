# 05. Event-Driven Architecture (EDA)

## The Problem: Tight Coupling Between Services

After splitting into microservices, we face a new problem:

### Scenario: Order Ready for Delivery

```javascript
// ❌ BAD: Restaurant Service directly calling Rider Service
app.put('/order/:id/ready', async (req, res) => {
  const order = await Order.findByIdAndUpdate(req.params.id, {
    status: 'ready_for_rider'
  });

  // Direct HTTP call to Rider Service
  const response = await axios.post('http://rider-service:5002/api/find-riders', {
    orderId: order._id,
    location: order.restaurantLocation
  });

  res.json({ success: true });
});
```

### Problems with Direct Service-to-Service Calls

**1. Tight Coupling**
```
Restaurant Service ──────> Rider Service
                    (knows URL, port, API contract)
```
- Restaurant service must know Rider service exists
- Must know its URL, port, and API structure
- If Rider service changes API, Restaurant service breaks

**2. Synchronous Blocking**
```
User Request → Restaurant Service → [WAITING...] → Rider Service
                                    ↓
                              Takes 5 seconds
                                    ↓
                              User waits 5 seconds
```
- User's request is blocked until Rider service responds
- If Rider service is slow, everything is slow

**3. Cascading Failures**
```
Rider Service DOWN
        ↓
Restaurant Service can't complete request
        ↓
User sees error: "Failed to place order"
        ↓
Order is stuck in limbo
```

**4. No Retry Mechanism**
```
Restaurant Service → [Network Error] → Rider Service
                     ↓
                Message lost forever
```

**5. Can't Scale Independently**
```
Black Friday Sale: 10,000 orders ready for delivery
        ↓
Restaurant Service makes 10,000 HTTP calls
        ↓
Rider Service overwhelmed and crashes
```

---

## The Solution: Message Queue (RabbitMQ)

### What is a Message Queue?

Think of it like a **post office**:

```
┌─────────────┐         ┌──────────────┐         ┌─────────────┐
│  Publisher  │ ──────> │ Message Queue│ ──────> │  Consumer   │
│ (Restaurant)│  Drops  │  (RabbitMQ)  │  Picks  │   (Rider)   │
│  Service    │  letter │              │  up     │  Service    │
└─────────────┘         └──────────────┘         └─────────────┘
```

**Key Concepts:**

1. **Publisher**: Service that sends messages (doesn't care who receives)
2. **Queue**: Holds messages until someone picks them up
3. **Consumer**: Service that processes messages (doesn't care who sent)
4. **Message**: Data packet with event type and payload

---

## How RabbitMQ Works in Tomato-Code

### Step 1: Setup Connection

```typescript
// services/restaurant/src/config/rabbitmq.ts
import amqp from "amqplib";

let channel: amqp.Channel;

export const connectRabbitMQ = async () => {
  // Connect to RabbitMQ server
  const connection = await amqp.connect(process.env.RABBITMQ_URL);

  // Create a channel (like opening a mailbox)
  channel = await connection.createChannel();

  // Declare queues (create mailboxes if they don't exist)
  await channel.assertQueue('payment_queue', { durable: true });
  await channel.assertQueue('order_ready_queue', { durable: true });

  console.log("🐇 Connected to RabbitMQ");
};

export const getChannel = () => channel;
```

**What is `durable: true`?**
- Queue survives RabbitMQ server restart
- Messages are saved to disk
- No data loss if server crashes

---

### Step 2: Publishing Events (Sending Messages)

```typescript
// services/restaurant/src/config/order.publisher.ts
import { getChannel } from "./rabbitmq.js";

export const publishEvent = async (type: string, data: any) => {
  const channel = getChannel();

  const message = {
    type: type,
    data: data,
    timestamp: new Date().toISOString()
  };

  // Send message to queue
  channel.sendToQueue(
    'order_ready_queue',
    Buffer.from(JSON.stringify(message)),
    { persistent: true }  // Save message to disk
  );

  console.log(`📤 Published: ${type}`);
};
```

**Usage in Order Controller:**

```typescript
// When restaurant marks order as ready
if (status === 'ready_for_rider') {
  await publishEvent('ORDER_READY_FOR_RIDER', {
    orderId: order._id.toString(),
    restaurantId: restaurant._id.toString(),
    location: restaurant.autoLocation
  });

  // ✅ Request completes immediately
  // ✅ User gets instant response
  // ✅ Rider service processes in background
}
```

---

### Step 3: Consuming Events (Receiving Messages)

```typescript
// services/rider/src/config/orderReady.consumer.ts
import { getChannel } from "./rabbitmq.js";
import { Rider } from "../model/Rider.js";

export const startOrderReadyConsumer = async () => {
  const channel = getChannel();

  console.log("👂 Listening for order_ready events...");

  // Start consuming messages from queue
  channel.consume('order_ready_queue', async (msg) => {
    if (!msg) return;

    try {
      // Parse message
      const event = JSON.parse(msg.content.toString());

      console.log(`📥 Received: ${event.type}`);

      // Filter: Only process ORDER_READY_FOR_RIDER events
      if (event.type !== 'ORDER_READY_FOR_RIDER') {
        channel.ack(msg);  // Acknowledge and skip
        return;
      }

      const { orderId, restaurantId, location } = event.data;

      // Find nearby riders using geospatial query
      const riders = await Rider.find({
        isAvailable: true,
        isVerified: true,
        location: {
          $near: {
            $geometry: location,
            $maxDistance: 5000  // 5km radius
          }
        }
      });

      console.log(`Found ${riders.length} nearby riders`);

      // Notify all nearby riders via WebSocket
      for (const rider of riders) {
        await axios.post(`${REALTIME_SERVICE}/api/v1/internal/emit`, {
          event: 'order:available',
          room: `user:${rider.userId}`,
          payload: { orderId, restaurantId }
        });
      }

      // ✅ Acknowledge message (remove from queue)
      channel.ack(msg);

    } catch (error) {
      console.error("❌ Error processing message:", error);
      // ❌ Don't acknowledge - message stays in queue for retry
    }
  });
};
```

---

## Message Flow Diagram

```
┌──────────────────────────────────────────────────────────────┐
│ PHASE 1: Restaurant marks order ready                        │
└──────────────────────────────────────────────────────────────┘

Restaurant Service
    │
    │ 1. Update order status to "ready_for_rider"
    │
    ├─> publishEvent('ORDER_READY_FOR_RIDER', {
    │     orderId: "123",
    │     location: { type: "Point", coordinates: [77.5, 12.9] }
    │   })
    │
    ↓
┌─────────────────────────────────────┐
│      RabbitMQ (order_ready_queue)   │
│                                     │
│  [Message 1] [Message 2] [Message 3]│
│      ↑                              │
│      │ Durable storage (disk)       │
└─────────────────────────────────────┘
    │
    │ 2. Rider Service pulls message
    │
    ↓
Rider Service
    │
    │ 3. Parse message
    │ 4. Find nearby riders (MongoDB $near query)
    │ 5. Notify riders via WebSocket
    │ 6. Acknowledge message (remove from queue)
    │
    ✅ Done


┌──────────────────────────────────────────────────────────────┐
│ PHASE 2: Payment success event                               │
└──────────────────────────────────────────────────────────────┘

Utils Service (Payment)
    │
    │ 1. Razorpay webhook received
    │ 2. Verify payment signature
    │
    ├─> publishEvent('PAYMENT_SUCCESS', {
    │     orderId: "123",
    │     paymentId: "pay_xyz",
    │     amount: 450
    │   })
    │
    ↓
┌─────────────────────────────────────┐
│   RabbitMQ (payment_queue)          │
│                                     │
│  [Message 1]                        │
└─────────────────────────────────────┘
    │
    ↓
Restaurant Service
    │
    │ 3. Consume payment_queue
    │ 4. Update order: paymentStatus = "paid"
    │ 5. Remove TTL expiration
    │ 6. Notify restaurant via WebSocket
    │
    ✅ Done
```

---

## Benefits of Event-Driven Architecture

### 1. **Loose Coupling**

**Before (Tight Coupling):**
```javascript
// Restaurant Service knows about Rider Service
const riders = await axios.post('http://rider-service:5002/find-riders', data);
```

**After (Loose Coupling):**
```javascript
// Restaurant Service just publishes event
await publishEvent('ORDER_READY_FOR_RIDER', data);
// Doesn't know who consumes it
// Doesn't care if Rider Service exists
```

### 2. **Asynchronous Processing**

```
User Request → Restaurant Service → Publish Event → Return Response
                                         ↓
                                    (Background)
                                         ↓
                                   Rider Service processes
```

**Response Time:**
- Before: 5 seconds (waiting for Rider Service)
- After: 200ms (just publish and return)

### 3. **Fault Tolerance**

```
Scenario: Rider Service is DOWN

Before:
  Restaurant Service → [ERROR] → Rider Service
  ↓
  User sees error
  ↓
  Order fails

After:
  Restaurant Service → Publish Event → RabbitMQ Queue
  ↓
  User gets success response
  ↓
  Message waits in queue
  ↓
  When Rider Service comes back online
  ↓
  Processes all pending messages
```

### 4. **Automatic Retry**

```javascript
channel.consume('order_ready_queue', async (msg) => {
  try {
    // Process message
    await processOrder(msg);
    channel.ack(msg);  // ✅ Success - remove from queue
  } catch (error) {
    // ❌ Error - don't acknowledge
    // Message stays in queue
    // RabbitMQ will redeliver it
  }
});
```

### 5. **Scalability**

```
Single Consumer:
┌─────────────┐
│ RabbitMQ    │ ──────> Rider Service Instance 1
│   Queue     │         (processes 100 msg/sec)
└─────────────┘

Multiple Consumers (Load Balancing):
┌─────────────┐
│ RabbitMQ    │ ──────> Rider Service Instance 1 (100 msg/sec)
│   Queue     │ ──────> Rider Service Instance 2 (100 msg/sec)
│             │ ──────> Rider Service Instance 3 (100 msg/sec)
└─────────────┘
                Total: 300 msg/sec
```

RabbitMQ automatically distributes messages across consumers (round-robin).

---

## Message Acknowledgment Patterns

### 1. **Auto-Ack (Dangerous)**

```javascript
channel.consume('queue', async (msg) => {
  // Message removed from queue immediately
  await processOrder(msg);  // If this fails, message is lost!
}, { noAck: true });
```

❌ **Problem:** If processing fails, message is gone forever.

### 2. **Manual-Ack (Safe)**

```javascript
channel.consume('queue', async (msg) => {
  try {
    await processOrder(msg);
    channel.ack(msg);  // ✅ Only remove if successful
  } catch (error) {
    // Message stays in queue for retry
  }
});
```

✅ **Benefit:** Message is only removed after successful processing.

### 3. **Negative-Ack (Requeue)**

```javascript
channel.consume('queue', async (msg) => {
  try {
    await processOrder(msg);
    channel.ack(msg);
  } catch (error) {
    // Put message back at front of queue
    channel.nack(msg, false, true);
  }
});
```

---

## Dead Letter Queue (DLQ)

What if a message keeps failing?

```javascript
// Create queue with DLQ
await channel.assertQueue('order_ready_queue', {
  durable: true,
  arguments: {
    'x-dead-letter-exchange': 'dlx',
    'x-message-ttl': 60000,  // 1 minute
    'x-max-retries': 3
  }
});

// After 3 failed attempts, message goes to DLQ
await channel.assertQueue('order_ready_queue_dlq', {
  durable: true
});
```

**Flow:**
```
order_ready_queue
    ↓
  Fails 3 times
    ↓
order_ready_queue_dlq (Dead Letter Queue)
    ↓
  Manual investigation
```

---

## Event Types in Tomato-Code

### 1. **PAYMENT_SUCCESS**

**Publisher:** Utils Service (Payment)
**Consumer:** Restaurant Service
**Payload:**
```json
{
  "type": "PAYMENT_SUCCESS",
  "data": {
    "orderId": "507f1f77bcf86cd799439011",
    "paymentId": "pay_xyz123",
    "amount": 450
  }
}
```

**Action:**
- Update order: `paymentStatus = "paid"`
- Remove TTL expiration
- Notify restaurant via WebSocket

---

### 2. **ORDER_READY_FOR_RIDER**

**Publisher:** Restaurant Service
**Consumer:** Rider Service
**Payload:**
```json
{
  "type": "ORDER_READY_FOR_RIDER",
  "data": {
    "orderId": "507f1f77bcf86cd799439011",
    "restaurantId": "507f1f77bcf86cd799439012",
    "location": {
      "type": "Point",
      "coordinates": [77.5946, 12.9716]
    }
  }
}
```

**Action:**
- Find nearby riders (MongoDB $near query)
- Notify riders via WebSocket
- First rider to accept gets assigned

---

## Comparison: Before vs After

### Scenario: 1000 Orders Ready for Delivery

**Before (Direct HTTP Calls):**
```
Restaurant Service makes 1000 HTTP calls to Rider Service
    ↓
Each call takes 500ms
    ↓
Total time: 1000 × 500ms = 500 seconds (8.3 minutes)
    ↓
If Rider Service crashes at call #500
    ↓
500 orders lost
```

**After (RabbitMQ):**
```
Restaurant Service publishes 1000 messages
    ↓
Each publish takes 5ms
    ↓
Total time: 1000 × 5ms = 5 seconds
    ↓
Rider Service processes at its own pace
    ↓
If Rider Service crashes
    ↓
Messages wait in queue
    ↓
When it restarts, processes remaining messages
```

---

## When to Use Event-Driven Architecture

✅ **Use EDA when:**
- Services need to communicate asynchronously
- You need fault tolerance and retry logic
- You want to decouple services
- You need to scale consumers independently
- Processing can happen in background

❌ **Don't use EDA when:**
- You need immediate response (use HTTP)
- Simple CRUD operations
- Single service application
- Real-time synchronous communication required

---

## Next Steps

Now that we understand Event-Driven Architecture, let's explore:

**[06. Real-Time Communication (WebSocket)](./06_Real_Time_Communication.md)**
- How WebSocket differs from HTTP
- Socket.io room-based broadcasting
- Keeping users, restaurants, and riders synchronized in real-time

---

## Key Takeaways

1. **Message Queues decouple services** - Publisher doesn't know consumer
2. **Asynchronous processing** - Don't block user requests
3. **Fault tolerance** - Messages survive service crashes
4. **Automatic retry** - Failed messages stay in queue
5. **Scalability** - Add more consumers to process faster
6. **RabbitMQ is the post office** - Reliable message delivery

Event-Driven Architecture is the backbone of scalable, resilient systems. It allows services to communicate without tight coupling, enabling independent scaling and fault tolerance.
