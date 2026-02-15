# 08. Security Architecture: Protecting Your Scale

## The Security Challenge at Scale

As your food delivery platform grows to 1M users, security becomes exponentially more critical. More users mean more attack vectors, more sensitive data, and higher stakes.

## Authentication & Authorization Evolution

### Phase 1: Basic JWT Authentication
```javascript
// Simple JWT auth
const jwt = require('jsonwebtoken');

function generateToken(user) {
  return jwt.sign(
    { userId: user.id, role: user.role },
    process.env.JWT_SECRET,
    { expiresIn: '24h' }
  );
}

function verifyToken(req, res, next) {
  const token = req.headers.authorization?.split(' ')[1];

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (error) {
    res.status(401).json({ error: 'Invalid token' });
  }
}
```

### Phase 2: Refresh Token Pattern
```javascript
// Separate access and refresh tokens
function generateTokenPair(user) {
  const accessToken = jwt.sign(
    { userId: user.id, role: user.role },
    process.env.JWT_SECRET,
    { expiresIn: '15m' }  // Short-lived
  );

  const refreshToken = jwt.sign(
    { userId: user.id, tokenVersion: user.tokenVersion },
    process.env.REFRESH_SECRET,
    { expiresIn: '7d' }  // Long-lived
  );

  return { accessToken, refreshToken };
}

// Refresh endpoint
app.post('/auth/refresh', async (req, res) => {
  const { refreshToken } = req.body;

  try {
    const decoded = jwt.verify(refreshToken, process.env.REFRESH_SECRET);
    const user = await User.findById(decoded.userId);

    // Check token version (allows invalidation)
    if (user.tokenVersion !== decoded.tokenVersion) {
      throw new Error('Token revoked');
    }

    const tokens = generateTokenPair(user);
    res.json(tokens);
  } catch (error) {
    res.status(401).json({ error: 'Invalid refresh token' });
  }
});
```

### Phase 3: OAuth2 + Social Login
```javascript
// Google OAuth integration
const { OAuth2Client } = require('google-auth-library');
const client = new OAuth2Client(process.env.GOOGLE_CLIENT_ID);

app.post('/auth/google', async (req, res) => {
  const { token } = req.body;

  try {
    const ticket = await client.verifyIdToken({
      idToken: token,
      audience: process.env.GOOGLE_CLIENT_ID
    });

    const payload = ticket.getPayload();
    const { email, name, picture } = payload;

    // Find or create user
    let user = await User.findOne({ email });
    if (!user) {
      user = await User.create({
        email,
        name,
        avatar: picture,
        provider: 'google'
      });
    }

    const tokens = generateTokenPair(user);
    res.json({ user, tokens });
  } catch (error) {
    res.status(401).json({ error: 'Invalid Google token' });
  }
});
```

## Role-Based Access Control (RBAC)

### Permission System
```javascript
// Define permissions
const PERMISSIONS = {
  // Order permissions
  'order:create': ['customer'],
  'order:view': ['customer', 'restaurant', 'driver', 'admin'],
  'order:update': ['restaurant', 'driver', 'admin'],
  'order:cancel': ['customer', 'admin'],

  // Restaurant permissions
  'restaurant:create': ['admin'],
  'restaurant:update': ['restaurant', 'admin'],
  'restaurant:delete': ['admin'],

  // User permissions
  'user:view': ['admin'],
  'user:update': ['admin'],
  'user:delete': ['admin']
};

// Permission middleware
function requirePermission(permission) {
  return (req, res, next) => {
    const userRole = req.user.role;

    if (!PERMISSIONS[permission]?.includes(userRole)) {
      return res.status(403).json({
        error: 'Insufficient permissions'
      });
    }

    next();
  };
}

// Usage
app.post('/orders',
  verifyToken,
  requirePermission('order:create'),
  createOrder
);

app.delete('/restaurants/:id',
  verifyToken,
  requirePermission('restaurant:delete'),
  deleteRestaurant
);
```

### Resource-Level Authorization
```javascript
// Check if user owns the resource
async function authorizeOrderAccess(req, res, next) {
  const orderId = req.params.id;
  const userId = req.user.userId;
  const userRole = req.user.role;

  const order = await Order.findById(orderId);

  if (!order) {
    return res.status(404).json({ error: 'Order not found' });
  }

  // Admin can access all orders
  if (userRole === 'admin') {
    req.order = order;
    return next();
  }

  // Customer can only access their own orders
  if (userRole === 'customer' && order.customerId !== userId) {
    return res.status(403).json({ error: 'Access denied' });
  }

  // Restaurant can access orders for their restaurant
  if (userRole === 'restaurant' && order.restaurantId !== userId) {
    return res.status(403).json({ error: 'Access denied' });
  }

  // Driver can access assigned orders
  if (userRole === 'driver' && order.driverId !== userId) {
    return res.status(403).json({ error: 'Access denied' });
  }

  req.order = order;
  next();
}

app.get('/orders/:id',
  verifyToken,
  authorizeOrderAccess,
  (req, res) => {
    res.json(req.order);
  }
);
```

## API Security Best Practices

### Rate Limiting
```javascript
const rateLimit = require('express-rate-limit');
const RedisStore = require('rate-limit-redis');
const Redis = require('ioredis');

const redis = new Redis(process.env.REDIS_URL);

// General API rate limit
const apiLimiter = rateLimit({
  store: new RedisStore({
    client: redis,
    prefix: 'rl:api:'
  }),
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100, // 100 requests per window
  message: 'Too many requests, please try again later'
});

// Stricter limit for auth endpoints
const authLimiter = rateLimit({
  store: new RedisStore({
    client: redis,
    prefix: 'rl:auth:'
  }),
  windowMs: 15 * 60 * 1000,
  max: 5, // Only 5 login attempts per 15 minutes
  message: 'Too many login attempts, please try again later'
});

app.use('/api/', apiLimiter);
app.use('/auth/', authLimiter);
```

### Input Validation & Sanitization
```javascript
const { body, param, validationResult } = require('express-validator');
const sanitizeHtml = require('sanitize-html');

// Validation middleware
const validateOrder = [
  body('restaurantId').isMongoId(),
  body('items').isArray({ min: 1 }),
  body('items.*.menuItemId').isMongoId(),
  body('items.*.quantity').isInt({ min: 1, max: 99 }),
  body('deliveryAddress.street').trim().notEmpty(),
  body('deliveryAddress.city').trim().notEmpty(),
  body('deliveryAddress.zipCode').matches(/^\d{5}$/),
  body('specialInstructions')
    .optional()
    .customSanitizer(value => sanitizeHtml(value, {
      allowedTags: [],
      allowedAttributes: {}
    }))
];

app.post('/orders',
  verifyToken,
  validateOrder,
  (req, res, next) => {
    const errors = validationResult(req);
    if (!errors.isEmpty()) {
      return res.status(400).json({ errors: errors.array() });
    }
    next();
  },
  createOrder
);
```

### SQL Injection Prevention
```javascript
// BAD: Vulnerable to SQL injection
app.get('/users', async (req, res) => {
  const { email } = req.query;
  const query = `SELECT * FROM users WHERE email = '${email}'`;
  const users = await db.query(query);
  res.json(users);
});

// GOOD: Using parameterized queries
app.get('/users', async (req, res) => {
  const { email } = req.query;
  const query = 'SELECT * FROM users WHERE email = ?';
  const users = await db.query(query, [email]);
  res.json(users);
});

// GOOD: Using ORM (Mongoose)
app.get('/users', async (req, res) => {
  const { email } = req.query;
  const users = await User.find({ email });
  res.json(users);
});
```

## Data Encryption

### Encryption at Rest
```javascript
const crypto = require('crypto');

class EncryptionService {
  constructor() {
    this.algorithm = 'aes-256-gcm';
    this.key = Buffer.from(process.env.ENCRYPTION_KEY, 'hex');
  }

  encrypt(text) {
    const iv = crypto.randomBytes(16);
    const cipher = crypto.createCipheriv(this.algorithm, this.key, iv);

    let encrypted = cipher.update(text, 'utf8', 'hex');
    encrypted += cipher.final('hex');

    const authTag = cipher.getAuthTag();

    return {
      encrypted,
      iv: iv.toString('hex'),
      authTag: authTag.toString('hex')
    };
  }

  decrypt(encrypted, iv, authTag) {
    const decipher = crypto.createDecipheriv(
      this.algorithm,
      this.key,
      Buffer.from(iv, 'hex')
    );

    decipher.setAuthTag(Buffer.from(authTag, 'hex'));

    let decrypted = decipher.update(encrypted, 'hex', 'utf8');
    decrypted += decipher.final('utf8');

    return decrypted;
  }
}

// Store sensitive data encrypted
const encryptionService = new EncryptionService();

const paymentMethodSchema = new mongoose.Schema({
  userId: { type: mongoose.Schema.Types.ObjectId, required: true },
  cardNumberEncrypted: String,
  cardNumberIv: String,
  cardNumberAuthTag: String,
  cardholderName: String,
  expiryMonth: Number,
  expiryYear: Number
});

paymentMethodSchema.methods.setCardNumber = function(cardNumber) {
  const { encrypted, iv, authTag } = encryptionService.encrypt(cardNumber);
  this.cardNumberEncrypted = encrypted;
  this.cardNumberIv = iv;
  this.cardNumberAuthTag = authTag;
};

paymentMethodSchema.methods.getCardNumber = function() {
  return encryptionService.decrypt(
    this.cardNumberEncrypted,
    this.cardNumberIv,
    this.cardNumberAuthTag
  );
};
```

### Password Hashing
```javascript
const bcrypt = require('bcrypt');

// Hash password before saving
userSchema.pre('save', async function(next) {
  if (!this.isModified('password')) return next();

  const salt = await bcrypt.genSalt(12);
  this.password = await bcrypt.hash(this.password, salt);
  next();
});

// Compare password
userSchema.methods.comparePassword = async function(candidatePassword) {
  return bcrypt.compare(candidatePassword, this.password);
};

// Usage
app.post('/auth/login', async (req, res) => {
  const { email, password } = req.body;

  const user = await User.findOne({ email }).select('+password');
  if (!user) {
    return res.status(401).json({ error: 'Invalid credentials' });
  }

  const isMatch = await user.comparePassword(password);
  if (!isMatch) {
    return res.status(401).json({ error: 'Invalid credentials' });
  }

  const tokens = generateTokenPair(user);
  res.json({ user: user.toPublicJSON(), tokens });
});
```

## Security Headers

```javascript
const helmet = require('helmet');
const cors = require('cors');

app.use(helmet({
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      styleSrc: ["'self'", "'unsafe-inline'"],
      scriptSrc: ["'self'"],
      imgSrc: ["'self'", 'data:', 'https:'],
    }
  },
  hsts: {
    maxAge: 31536000,
    includeSubDomains: true,
    preload: true
  }
}));

// CORS configuration
app.use(cors({
  origin: process.env.ALLOWED_ORIGINS.split(','),
  credentials: true,
  methods: ['GET', 'POST', 'PUT', 'DELETE', 'PATCH'],
  allowedHeaders: ['Content-Type', 'Authorization']
}));

// Additional security headers
app.use((req, res, next) => {
  res.setHeader('X-Content-Type-Options', 'nosniff');
  res.setHeader('X-Frame-Options', 'DENY');
  res.setHeader('X-XSS-Protection', '1; mode=block');
  next();
});
```

## Audit Logging

```javascript
// Audit log schema
const auditLogSchema = new mongoose.Schema({
  userId: mongoose.Schema.Types.ObjectId,
  action: String,
  resource: String,
  resourceId: String,
  changes: mongoose.Schema.Types.Mixed,
  ipAddress: String,
  userAgent: String,
  timestamp: { type: Date, default: Date.now }
});

// Audit middleware
async function auditLog(action, resource) {
  return async (req, res, next) => {
    const originalJson = res.json.bind(res);

    res.json = function(data) {
      // Log after successful response
      if (res.statusCode < 400) {
        AuditLog.create({
          userId: req.user?.userId,
          action,
          resource,
          resourceId: req.params.id || data?.id,
          changes: req.body,
          ipAddress: req.ip,
          userAgent: req.get('user-agent')
        }).catch(err => console.error('Audit log failed:', err));
      }

      return originalJson(data);
    };

    next();
  };
}

// Usage
app.put('/restaurants/:id',
  verifyToken,
  requirePermission('restaurant:update'),
  auditLog('UPDATE', 'restaurant'),
  updateRestaurant
);

app.delete('/users/:id',
  verifyToken,
  requirePermission('user:delete'),
  auditLog('DELETE', 'user'),
  deleteUser
);
```

## Key Takeaways

1. **Defense in Depth**: Multiple layers of security (authentication, authorization, encryption, validation)
2. **Least Privilege**: Users only get permissions they need
3. **Secure by Default**: Security built into every endpoint, not added later
4. **Audit Everything**: Track sensitive operations for compliance and debugging
5. **Never Trust Input**: Validate and sanitize all user input
6. **Encrypt Sensitive Data**: Both in transit (HTTPS) and at rest (database encryption)

Security isn't optional at scale—it's the foundation everything else is built on.

---
**Next**: [09. Scalability Strategies](./09_Scalability_Strategies.md) - Techniques to handle millions of users
