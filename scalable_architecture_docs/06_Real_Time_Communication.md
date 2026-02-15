# 06. Real-Time Communication with WebSockets

## The Problem: HTTP Polling is Inefficient

### What We Had Before (HTTP Polling)

```javascript
// Client keeps asking "Any updates?" every 2 seconds
setInterval(async () => {
  const response = await fetch('/api/order/status/123');
  const data = await response.json();

  if (data.status !== currentStatus) {
    updateUI(data.status);
  }
}, 2000);
```

### Why This Breaks at Scale

**For 10,000 active orders:**
- 10,000 users × 30 requests/minute = **300,000 requests/minute**
- 99% of these requests return "No change"
- Wasted bandwidth, server CPU, database queries
- Battery drain on mobile devices
- Delayed updates (up to 2 seconds lag)

**The Fundamental Problem:**
HTTP is **request-response** - the server can't "push" updates to the client.

---

## The Solution: WebSocket Protocol

### What is WebSocket?

WebSocket is a **persistent, bidirectional** communication channel between client and server.

```
HTTP (Request-Response):
Client  ──request──>  Server
Client  <──response── Server
[Connection closes]

WebSocket (Persistent Connection):
Client  ══════════════  Server
        ↕ bidirectional ↕
        stays open forever
```

### The Handshake

```
1. Client sends HTTP upgrade request:
   GET /socket.io/ HTTP/1.1
   Upgrade: websocket
   Connection: Upgrade

2. Server responds:
   HTTP/1.1 101 Switching Protocols
   Upgrade: websocket
   Connection: Upgrade

3. Connection stays open - now both can send messages anytime
```

---

## Socket.io: WebSocket Made Easy

### Why Socket.io Over Raw WebSocket?

Socket.io adds:
1. **Automatic reconnection** - handles network drops
2. **Rooms** - group users for targeted broadcasts
3. **Fallback to polling** - works even if WebSocket blocked
4. **Acknowledgments** - confirm message delivery

### Architecture in Tomato-Code

```
┌─────────────────────────────────────────────────────────────┐
│                    Realtime Service                          │
│                   (Socket.io Server)                         │
│                                                              │
│  ┌────────────────────────────────────────────────────┐    │
│  │              Connection Manager                     │    │
│  │                                                     │    │
│  │  socket.on('connection', (socket) => {             │    │
│  │    const userId = verifyJWT(socket.handshake.auth) │    │
│  │    socket.join(`user:${userId}`)                   │    │
│  │  })                                                 │    │
│  └────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌────────────────────────────────────────────────────┐    │
│  │                Room Structure                       │    │
│  │                                                     │    │
│  │  user:123        → [socket_abc, socket_def]        │    │
│  │  user:456        → [socket_ghi]                    │    │
│  │  restaurant:789  → [socket_jkl, socket_mno]        │    │
│  │  restaurant:101  → [socket_pqr]                    │    │
│  └────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌────────────────────────────────────────────────────┐    │
│  │            Internal Emit API                        │    │
│  │                                                     │    │
│  │  POST /api/v1/internal/emit                        │    │
│  │  {                                                  │    │
│  │    event: "order:update",                          │    │
│  │    room: "user:123",                               │    │
│  │    payload: { orderId, status }                    │    │
│  │  }                                                  │    │
│  │                                                     │    │
│  │  → io.to(room).emit(event, payload)                │    │
│  └────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

---

## Implementation Deep Dive

### 1. Client-Side Setup (React)

```typescript
// frontend/src/context/SocketContext.tsx

import { io, Socket } from "socket.io-client";

export const SocketProvider = ({ children }) => {
  const { isAuth } = useAppData();
  const socketRef = useRef<Socket | null>(null);

  useEffect(() => {
    if (!isAuth) {
      socketRef.current?.disconnect();
      return;
    }

    // Create WebSocket connection
    const socket = io(REALTIME_SERVICE_URL, {
      auth: {
        token: localStorage.getItem("token")  // JWT for authentication
      },
      transports: ["websocket"],  // Force WebSocket (no polling fallback)
      reconnection: true,          // Auto-reconnect on disconnect
      reconnectionDelay: 1000,     // Wait 1s before reconnecting
      reconnectionAttempts: 5      // Try 5 times
    });

    socketRef.current = socket;

    socket.on("connect", () => {
      console.log("Connected:", socket.id);
    });

    socket.on("disconnect", (reason) => {
      console.log("Disconnected:", reason);
      // Reasons: "io server disconnect", "transport close", etc.
    });

    return () => socket.disconnect();
  }, [isAuth]);

  return (
    <SocketContext.Provider value={{ socket: socketRef.current }}>
      {children}
    </SocketContext.Provider>
  );
};
```

### 2. Server-Side Setup (Realtime Service)

```typescript
// services/realtime/src/index.ts

import { Server } from "socket.io";
import jwt from "jsonwebtoken";

const io = new Server(server, {
  cors: {
    origin: process.env.FRONTEND_URL,
    credentials: true
  },
  transports: ["websocket", "polling"]
});

// Authentication Middleware
io.use((socket, next) => {
  const token = socket.handshake.auth.token;

  if (!token) {
    return next(new Error("Authentication error"));
  }

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    socket.userId = decoded._id;  // Attach userId to socket
    next();
  } catch (err) {
    next(new Error("Invalid token"));
  }
});

// Connection Handler
io.on("connection", (socket) => {
  console.log(`User ${socket.userId} connected`);

  // Join user-specific room
  socket.join(`user:${socket.userId}`);

  // If user is a restaurant owner, join restaurant rooms
  if (socket.restaurantId) {
    socket.join(`restaurant:${socket.restaurantId}`);
  }

  socket.on("disconnect", () => {
    console.log(`User ${socket.userId} disconnected`);
  });
});
```

### 3. Internal Emit API (Service-to-Service Communication)

```typescript
// services/realtime/src/routes/internal.ts

router.post("/emit", (req, res) => {
  // Verify internal service key
  if (req.headers["x-internal-key"] !== process.env.INTERNAL_SERVICE_KEY) {
    return res.status(403).json({ message: "Forbidden" });
  }

  const { event, room, payload } = req.body;

  // Emit to specific room
  io.to(room).emit(event, payload);

  res.json({ success: true });
});
```

### 4. Emitting Events from Other Services

```typescript
// services/restaurant/src/controllers/order.ts

import axios from "axios";

export const updateOrderStatus = async (req, res) => {
  const order = await Order.findById(req.params.orderId);
  order.status = req.body.status;
  await order.save();

  // Notify user via WebSocket
  await axios.post(
    `${process.env.REALTIME_SERVICE}/api/v1/internal/emit`,
    {
      event: "order:update",
      room: `user:${order.userId}`,
      payload: {
        orderId: order._id,
        status: order.status,
        timestamp: Date.now()
      }
    },
    {
      headers: {
        "x-internal-key": process.env.INTERNAL_SERVICE_KEY
      }
    }
  );

  res.json({ message: "Order updated" });
};
```

### 5. Listening to Events on Client

```typescript
// frontend/src/pages/OrderPage.tsx

import { useSocket } from "../context/SocketContext";

export const OrderPage = () => {
  const { socket } = useSocket();
  const [orderStatus, setOrderStatus] = useState("placed");

  useEffect(() => {
    if (!socket) return;

    // Listen for order updates
    socket.on("order:update", (data) => {
      console.log("Order updated:", data);
      setOrderStatus(data.status);

      // Show notification
      toast.success(`Order is now ${data.status}`);
    });

    // Cleanup listener on unmount
    return () => {
      socket.off("order:update");
    };
  }, [socket]);

  return (
    <div>
      <h1>Order Status: {orderStatus}</h1>
    </div>
  );
};
```

---

## Room-Based Broadcasting

### The Power of Rooms

Rooms allow **targeted broadcasting** - send messages only to relevant users.

```typescript
// Without rooms (BAD - broadcasts to everyone)
io.emit("order:update", { orderId: 123, status: "preparing" });
// All 10,000 connected users receive this!

// With rooms (GOOD - only relevant user)
io.to(`user:${userId}`).emit("order:update", { orderId: 123, status: "preparing" });
// Only the user who placed the order receives this
```

### Room Patterns in Tomato-Code

```typescript
// 1. User-specific room
socket.join(`user:${userId}`);
// Use case: Order updates, rider assignment

// 2. Restaurant-specific room
socket.join(`restaurant:${restaurantId}`);
// Use case: New orders, rider pickup notifications

// 3. Rider-specific room (same as user room)
socket.join(`user:${riderId}`);
// Use case: Available orders nearby

// 4. Order-specific room (for live tracking)
socket.join(`order:${orderId}`);
// Use case: Real-time location updates during delivery
```

### Multi-Room Broadcasting

```typescript
// Notify both user and restaurant
const rooms = [`user:${order.userId}`, `restaurant:${order.restaurantId}`];

rooms.forEach(room => {
  io.to(room).emit("order:rider_assigned", {
    orderId: order._id,
    riderName: rider.name,
    riderPhone: rider.phone
  });
});
```

---

## Real-World Use Cases

### 1. New Order Notification (Restaurant)

```typescript
// Restaurant Service: After payment success
await axios.post(`${REALTIME_SERVICE}/api/v1/internal/emit`, {
  event: "order:new",
  room: `restaurant:${order.restaurantId}`,
  payload: {
    orderId: order._id,
    items: order.items,
    totalAmount: order.totalAmount,
    deliveryAddress: order.deliveryAddress
  }
});

// Restaurant Dashboard: Listen for new orders
socket.on("order:new", (data) => {
  playNotificationSound();
  showOrderPopup(data);
  addToOrderList(data);
});
```

### 2. Rider Assignment Notification (User)

```typescript
// Restaurant Service: When rider accepts order
await axios.post(`${REALTIME_SERVICE}/api/v1/internal/emit`, {
  event: "order:rider_assigned",
  room: `user:${order.userId}`,
  payload: {
    orderId: order._id,
    riderName: order.riderName,
    riderPhone: order.riderPhone,
    estimatedTime: 30  // minutes
  }
});

// User App: Show rider details
socket.on("order:rider_assigned", (data) => {
  setRiderInfo(data);
  showMap(data.riderLocation);
});
```

### 3. Available Order Notification (Riders)

```typescript
// Rider Service: When order is ready for pickup
const nearbyRiders = await Rider.find({
  isAvailable: true,
  location: {
    $near: {
      $geometry: restaurantLocation,
      $maxDistance: 5000
    }
  }
});

// Notify all nearby riders
for (const rider of nearbyRiders) {
  await axios.post(`${REALTIME_SERVICE}/api/v1/internal/emit`, {
    event: "order:available",
    room: `user:${rider.userId}`,
    payload: {
      orderId: order._id,
      restaurantName: order.restaurantName,
      pickupAddress: order.restaurantAddress,
      deliveryAddress: order.deliveryAddress,
      distance: calculateDistance(rider.location, restaurantLocation),
      amount: order.riderAmount
    }
  });
}

// Rider App: Show available orders
socket.on("order:available", (data) => {
  showOrderRequest(data);
  playAlertSound();
});
```

---

## Handling Edge Cases

### 1. Reconnection Logic

```typescript
// Client-side reconnection handling
socket.on("connect", () => {
  console.log("Reconnected!");

  // Re-fetch current state after reconnection
  fetchCurrentOrderStatus();
});

socket.on("disconnect", (reason) => {
  if (reason === "io server disconnect") {
    // Server forcefully disconnected - manual reconnect
    socket.connect();
  }
  // Otherwise, Socket.io auto-reconnects
});
```

### 2. Message Acknowledgments

```typescript
// Server: Emit with callback
socket.emit("order:update", data, (ack) => {
  if (ack.success) {
    console.log("Client received message");
  }
});

// Client: Acknowledge receipt
socket.on("order:update", (data, callback) => {
  updateUI(data);
  callback({ success: true });
});
```

### 3. Duplicate Connection Prevention

```typescript
// Server: Track active connections per user
const activeConnections = new Map();

io.on("connection", (socket) => {
  const userId = socket.userId;

  // Disconnect previous connection
  if (activeConnections.has(userId)) {
    const oldSocket = activeConnections.get(userId);
    oldSocket.disconnect(true);
  }

  activeConnections.set(userId, socket);

  socket.on("disconnect", () => {
    activeConnections.delete(userId);
  });
});
```

---

## Performance Optimization

### 1. Connection Pooling

```typescript
// Use Redis adapter for horizontal scaling
import { createAdapter } from "@socket.io/redis-adapter";
import { createClient } from "redis";

const pubClient = createClient({ url: process.env.REDIS_URL });
const subClient = pubClient.duplicate();

await Promise.all([pubClient.connect(), subClient.connect()]);

io.adapter(createAdapter(pubClient, subClient));
```

**Why Redis Adapter?**
```
Without Redis:
┌─────────────┐     ┌─────────────┐
│ Socket.io   │     │ Socket.io   │
│ Instance 1  │     │ Instance 2  │
│             │     │             │
│ User A ✓    │     │ User B ✓    │
└─────────────┘     └─────────────┘

Problem: If we emit to "user:A" from Instance 2, it won't reach User A!

With Redis:
┌─────────────┐     ┌─────────────┐
│ Socket.io   │     │ Socket.io   │
│ Instance 1  │     │ Instance 2  │
│     ↓       │     │     ↓       │
└─────┼───────┘     └─────┼───────┘
      └──────┬──────┬──────┘
             ↓      ↓
        ┌─────────────┐
        │    Redis    │
        │  Pub/Sub    │
        └─────────────┘

Solution: Redis broadcasts events across all instances!
```

### 2. Namespace Separation

```typescript
// Separate namespaces for different features
const orderNamespace = io.of("/orders");
const chatNamespace = io.of("/chat");

orderNamespace.on("connection", (socket) => {
  // Handle order-related events
});

chatNamespace.on("connection", (socket) => {
  // Handle chat-related events
});
```

### 3. Binary Data Optimization

```typescript
// Send binary data (images, files) efficiently
socket.emit("rider:location", {
  orderId: "123",
  location: Buffer.from([lat, lon])  // Binary instead of JSON
});
```

---

## Monitoring and Debugging

### 1. Connection Metrics

```typescript
// Track active connections
setInterval(() => {
  const sockets = await io.fetchSockets();
  console.log(`Active connections: ${sockets.length}`);

  // Send to monitoring service
  metrics.gauge("websocket.connections", sockets.length);
}, 60000);
```

### 2. Event Logging

```typescript
// Log all emitted events
io.on("connection", (socket) => {
  socket.onAny((event, ...args) => {
    console.log(`Event: ${event}`, args);
  });
});
```

### 3. Health Check Endpoint

```typescript
app.get("/health", (req, res) => {
  const sockets = io.sockets.sockets.size;
  res.json({
    status: "healthy",
    activeConnections: sockets,
    uptime: process.uptime()
  });
});
```

---

## Comparison: Before vs After

### Before (HTTP Polling)

```
10,000 users × 30 requests/min = 300,000 requests/min
- Database queries: 300,000/min
- Network bandwidth: ~300 MB/min
- Average latency: 1-2 seconds
- Battery drain: High
```

### After (WebSocket)

```
10,000 users × 1 connection = 10,000 persistent connections
- Database queries: Only when data changes
- Network bandwidth: ~10 MB/min (only actual updates)
- Average latency: <100ms
- Battery drain: Low
```

**Result: 30x reduction in server load, 20x faster updates!**

---

## Key Takeaways

1. **WebSocket = Persistent Connection**: Server can push updates instantly
2. **Rooms = Targeted Broadcasting**: Send messages only to relevant users
3. **Socket.io = Production-Ready**: Handles reconnection, fallbacks, acknowledgments
4. **Redis Adapter = Horizontal Scaling**: Multiple server instances work together
5. **Internal API = Service Integration**: Other services can trigger WebSocket events

**Next:** [07. API Design Patterns](./07_API_Design_Patterns.md) - How services communicate via REST APIs
