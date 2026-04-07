# Introduction to Express Middleware — Presentation Notes

---

## Slide 01 — Title

**Introduction to Express Middleware**

The Backbone of Every Express Application

19 slides · pipeline, auth, validation, composition, testing · 2026

---

## Slide 02 — Agenda

### Foundations
- What Is Middleware?
- How the Middleware Stack Works
- Application-Level Middleware
- Router-Level Middleware
- Built-In Middleware

### Error Handling & Third-Party
- Error-Handling Middleware
- Third-Party Middleware
- Writing Custom Middleware

### Security & Validation
- Authentication Middleware
- Authorisation Middleware
- Validation Middleware
- Request Processing Pipeline

### Advanced Patterns
- Async Middleware & Error Propagation
- Middleware Composition Patterns
- Testing Middleware
- Performance Considerations
- Summary & Next Steps

---

## Slide 03 — What Is Middleware?

### The Signature

A middleware function is any function with access to the **request object**, **response object**, and the **next function** in the application's request-response cycle. Middleware functions can execute code, modify `req`/`res`, end the cycle, or call `next()` to pass control forward.

```javascript
// Every middleware has the same shape
function myMiddleware(req, res, next) {
  // 1. Execute any code
  console.log(`${req.method} ${req.url}`);

  // 2. Modify request or response
  req.requestTime = Date.now();

  // 3. End the cycle...
  // res.send('done');

  // 4. ...or pass control to the next middleware
  next();
}
```

### The Pipeline Concept

- **Request** enters the top of the stack
- **Middleware 1** → processes → calls `next()`
- **Middleware 2** → processes → calls `next()`
- **Route handler** → sends response
- **Response** flows back to client

**Key insight:** Express is essentially a series of middleware function calls. The route handler itself is just the final middleware in the chain.

---

## Slide 04 — How the Middleware Stack Works

Middleware executes in the **order it is registered**. Each function must either call `next()` or send a response — otherwise the request hangs indefinitely.

### Execution Order

```javascript
const app = express();

// 1st — runs on every request
app.use((req, res, next) => {
  console.log('A: global middleware');
  next();
});

// 2nd — only GET /api/users
app.get('/api/users', (req, res, next) => {
  console.log('B: route middleware');
  next();
});

// 3rd — the handler
app.get('/api/users', (req, res) => {
  console.log('C: route handler');
  res.json({ users: [] });
});

// Output for GET /api/users:
// A: global middleware
// B: route middleware
// C: route handler
```

### Request Lifecycle Diagram

Incoming Request → body parser (json) → cors() → authMiddleware → validateInput → Route Handler → res.json() / send()

**If you forget `next()`:** the request will hang until the client times out. Express does not emit a warning.

---

## Slide 05 — Application-Level Middleware

Bound to an instance of `express()` using `app.use()` or `app.METHOD()`. This is where you attach global behaviour to every incoming request.

### app.use() — All Routes

```javascript
// Runs on every request, any HTTP method
app.use((req, res, next) => {
  console.log(`[${new Date().toISOString()}] ${req.method} ${req.url}`);
  next();
});

// Mount on a specific path prefix
// Runs for ANY request starting with /api
app.use('/api', (req, res, next) => {
  res.set('X-API-Version', '2.1');
  next();
});
```

### app.METHOD() — Specific Routes

```javascript
// Only GET requests to /health
app.get('/health', (req, res) => {
  res.json({ status: 'ok', uptime: process.uptime() });
});

// Multiple middleware on one route
app.post('/api/orders',
  authenticate,
  validateOrder,
  (req, res) => {
    // handler only runs if both middleware called next()
    res.status(201).json({ orderId: '...' });
  }
);
```

### Mounting Paths

- `app.use('/')` — matches **all** paths (default when no path given)
- `app.use('/api')` — matches `/api`, `/api/users`, `/api/orders/123`
- `app.use('/api/v2')` — matches any path starting with `/api/v2`

### Order Matters

```javascript
// WRONG — 404 handler before routes
app.use((req, res) => res.status(404).send('Not found'));
app.get('/users', getUsers); // never reached

// CORRECT — routes first, then 404
app.get('/users', getUsers);
app.use((req, res) => res.status(404).send('Not found'));
```

---

## Slide 06 — Router-Level Middleware

Works identically to application-level middleware, but is bound to an instance of `express.Router()`. This lets you create **modular, mountable route handlers** with their own middleware stacks.

### Creating a Router Module

```javascript
// routes/users.js
const router = require('express').Router();

// Router-level middleware — only runs for /users/*
router.use((req, res, next) => {
  console.log(`Users route: ${req.method} ${req.url}`);
  next();
});

// Param middleware — runs when :id is present
router.param('id', async (req, res, next, id) => {
  const user = await User.findById(id);
  if (!user) return res.status(404).json({ error: 'User not found' });
  req.user = user;
  next();
});

router.get('/', async (req, res) => {
  const users = await User.find();
  res.json(users);
});

router.get('/:id', (req, res) => {
  res.json(req.user); // set by param middleware
});

router.delete('/:id', authorize('admin'), async (req, res) => {
  await req.user.remove();
  res.status(204).end();
});

module.exports = router;
```

### Mounting in app.js

```javascript
// app.js
const express = require('express');
const app = express();

const usersRouter  = require('./routes/users');
const ordersRouter = require('./routes/orders');
const authRouter   = require('./routes/auth');

// Mount each router at its prefix
app.use('/api/users',  usersRouter);
app.use('/api/orders', ordersRouter);
app.use('/auth',       authRouter);
```

### Benefits of Router Isolation

- **Separation of concerns** — each file owns its routes
- **Independent middleware** — auth on `/api`, none on `/public`
- **Testable** — mount router in test app, no side effects
- **Reusable** — same router can be mounted at different paths
- **Scalable** — team members work on separate route files

### Project Structure

```
routes/
├── auth.js        # login, register, logout
├── users.js       # CRUD for users
├── orders.js      # CRUD for orders
└── index.js       # re-exports all routers
```

---

## Slide 07 — Built-In Middleware

Express ships with three built-in middleware functions. Since Express 4.16+, you no longer need `body-parser` as a separate dependency.

### express.json()

Parses incoming JSON payloads. Populates `req.body`.

```javascript
app.use(express.json({
  limit: '10kb',     // max payload size
  strict: true,      // only objects/arrays
  type: 'application/json'
}));

// Now req.body is parsed
app.post('/api/users', (req, res) => {
  console.log(req.body.name);
  // "Alice"
});
```

### express.urlencoded()

Parses URL-encoded form bodies (`application/x-www-form-urlencoded`).

```javascript
app.use(express.urlencoded({
  extended: true,  // use qs library
  limit: '10kb',
  parameterLimit: 1000
}));
```

### express.static()

Serves static files from a directory. The only built-in middleware that does **not** call `next()` on match.

```javascript
const path = require('path');

app.use(express.static(
  path.join(__dirname, 'public'),
  {
    maxAge: '1d',       // cache 1 day
    etag: true,         // ETag headers
    index: 'index.html',
    dotfiles: 'ignore'
  }
));

// GET /css/style.css  → public/css/style.css
// GET /img/logo.png   → public/img/logo.png
```

**Tip:** Always place `express.static()` *before* your routes to serve assets without unnecessary middleware overhead. Use `express.json()` and `express.urlencoded()` only on routes that need body parsing.

---

## Slide 08 — Error-Handling Middleware

Express identifies error handlers by their **four-argument signature**: `(err, req, res, next)`. They must be registered **after** all routes and other middleware.

### Basic Error Handler

```javascript
// Must have exactly 4 parameters
app.use((err, req, res, next) => {
  console.error(err.stack);

  const status = err.status || 500;
  res.status(status).json({
    error: {
      message: err.message,
      // Only include stack in development
      ...(process.env.NODE_ENV === 'development' && {
        stack: err.stack
      })
    }
  });
});
```

### Triggering Errors

```javascript
// Pass error to next()
app.get('/api/users/:id', async (req, res, next) => {
  try {
    const user = await User.findById(req.params.id);
    if (!user) {
      const err = new Error('User not found');
      err.status = 404;
      return next(err); // skip to error handler
    }
    res.json(user);
  } catch (err) {
    next(err); // pass DB errors, etc.
  }
});
```

### Centralised Error Handling Pattern

```javascript
// Custom AppError class
class AppError extends Error {
  constructor(message, status = 500, code) {
    super(message);
    this.status = status;
    this.code = code;
    this.isOperational = true;
  }
}

// Specific error types
class NotFoundError extends AppError {
  constructor(resource = 'Resource') {
    super(`${resource} not found`, 404, 'NOT_FOUND');
  }
}

class ValidationError extends AppError {
  constructor(details) {
    super('Validation failed', 422, 'VALIDATION_ERROR');
    this.details = details;
  }
}

// In routes — throw clean errors
app.get('/api/users/:id', async (req, res, next) => {
  const user = await User.findById(req.params.id);
  if (!user) throw new NotFoundError('User');
  res.json(user);
});
```

### Multiple Error Handlers

```javascript
// 1st — log the error
app.use((err, req, res, next) => {
  logger.error({ err, req });
  next(err); // forward to next error handler
});

// 2nd — send response
app.use((err, req, res, next) => {
  res.status(err.status || 500).json({
    error: err.message
  });
});
```

---

## Slide 09 — Third-Party Middleware

The Express ecosystem relies on third-party middleware for cross-cutting concerns. These are installed via npm and registered with `app.use()`.

| Package | Purpose | Usage |
|---------|---------|-------|
| **helmet** | Sets security HTTP headers (CSP, HSTS, X-Frame-Options, etc.) | `app.use(helmet())` |
| **cors** | Configures Cross-Origin Resource Sharing headers | `app.use(cors({ origin: 'https://myapp.com' }))` |
| **morgan** | HTTP request logger (dev, combined, custom formats) | `app.use(morgan('combined'))` |
| **compression** | Gzip/Brotli response compression | `app.use(compression())` |
| **express-rate-limit** | Rate limiting (DDoS protection, API quotas) | `app.use(rateLimit({ windowMs: 15*60*1000, max: 100 }))` |
| **cookie-parser** | Parses Cookie header, populates `req.cookies` | `app.use(cookieParser('secret'))` |
| **express-session** | Server-side session with pluggable stores | `app.use(session({ store, secret, ... }))` |
| **multer** | Multipart form data (file uploads) | `upload.single('avatar')` |

### Recommended Registration Order

```javascript
app.use(helmet());                    // security headers first
app.use(cors({ origin: allowList })); // CORS before routes
app.use(compression());               // compress responses
app.use(morgan('combined'));           // log every request
app.use(express.json());              // parse JSON bodies
app.use(express.urlencoded({ extended: true }));
app.use(cookieParser());              // parse cookies
app.use(express.static('public'));    // static assets
// ... routes ...
// ... error handlers last ...
```

---

## Slide 10 — Writing Custom Middleware

Custom middleware lets you extract **cross-cutting concerns** into reusable functions. The pattern is always the same: receive `req`, `res`, `next`, do work, call `next()`.

### Request Logger

```javascript
function requestLogger(req, res, next) {
  const start = Date.now();

  // Hook into response finish event
  res.on('finish', () => {
    const duration = Date.now() - start;
    console.log(
      `${req.method} ${req.originalUrl} ` +
      `${res.statusCode} ${duration}ms`
    );
  });

  next();
}

app.use(requestLogger);
```

### Request ID

```javascript
const { randomUUID } = require('crypto');

function requestId(req, res, next) {
  req.id = req.headers['x-request-id'] || randomUUID();
  res.set('X-Request-Id', req.id);
  next();
}

app.use(requestId);
```

### Response Timing

```javascript
function responseTime(req, res, next) {
  const start = process.hrtime.bigint();

  res.on('finish', () => {
    const ns = process.hrtime.bigint() - start;
    const ms = Number(ns) / 1_000_000;
    res.set('X-Response-Time', `${ms.toFixed(2)}ms`);
  });

  next();
}
```

### Feature Flags

```javascript
function featureFlags(flagService) {
  return async (req, res, next) => {
    try {
      req.flags = await flagService.getFlags(
        req.user?.id,
        req.headers['x-region']
      );
      next();
    } catch (err) {
      // Default flags on failure — don't block the request
      req.flags = flagService.defaults;
      next();
    }
  };
}

app.use(featureFlags(launchDarkly));
```

### Maintenance Mode

```javascript
function maintenanceMode(req, res, next) {
  if (process.env.MAINTENANCE === 'true') {
    return res.status(503).json({
      error: 'Service under maintenance',
      retryAfter: 300
    });
  }
  next();
}
```

---

## Slide 11 — Authentication Middleware

Authentication middleware verifies **who the user is**. It typically extracts credentials from headers or cookies, validates them, and attaches the user object to `req.user`.

### JWT Verification

```javascript
const jwt = require('jsonwebtoken');

function authenticate(req, res, next) {
  const header = req.headers.authorization;
  if (!header?.startsWith('Bearer ')) {
    return res.status(401).json({
      error: 'Missing or malformed token'
    });
  }

  const token = header.slice(7);
  try {
    const payload = jwt.verify(token, process.env.JWT_SECRET, {
      algorithms: ['HS256'],
      issuer: 'my-app',
    });
    req.user = payload;  // { id, email, role, iat, exp }
    next();
  } catch (err) {
    if (err.name === 'TokenExpiredError') {
      return res.status(401).json({ error: 'Token expired' });
    }
    return res.status(401).json({ error: 'Invalid token' });
  }
}

// Protect all /api routes
app.use('/api', authenticate);
```

### Session-Based Auth

```javascript
function requireSession(req, res, next) {
  if (!req.session?.userId) {
    return res.status(401).json({
      error: 'Not authenticated'
    });
  }
  next();
}

// Login route — creates session
app.post('/login', async (req, res) => {
  const user = await User.findByEmail(req.body.email);
  if (!user || !await user.verifyPassword(req.body.password)) {
    return res.status(401).json({ error: 'Invalid credentials' });
  }
  req.session.userId = user.id;
  req.session.role = user.role;
  res.json({ message: 'Logged in' });
});
```

### Passport.js Strategy

```javascript
const passport = require('passport');
const { Strategy: JwtStrategy, ExtractJwt } = require('passport-jwt');

passport.use(new JwtStrategy({
  jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
  secretOrKey: process.env.JWT_SECRET,
}, async (payload, done) => {
  const user = await User.findById(payload.id);
  return done(null, user || false);
}));

// Use as middleware
app.use('/api', passport.authenticate('jwt', { session: false }));
```

---

## Slide 12 — Authorisation Middleware

Authorisation determines **what an authenticated user is allowed to do**. It runs **after** authentication and checks roles, permissions, or resource ownership.

### Role-Based Access Control

```javascript
// Factory function — returns middleware
function requireRole(...allowedRoles) {
  return (req, res, next) => {
    if (!req.user) {
      return res.status(401).json({ error: 'Not authenticated' });
    }
    if (!allowedRoles.includes(req.user.role)) {
      return res.status(403).json({
        error: 'Insufficient permissions',
        required: allowedRoles,
        current: req.user.role,
      });
    }
    next();
  };
}

// Usage on routes
app.delete('/api/users/:id',
  authenticate,
  requireRole('admin'),
  deleteUser
);

app.put('/api/posts/:id',
  authenticate,
  requireRole('admin', 'editor'),
  updatePost
);
```

### Permission Guards

```javascript
function requirePermission(...perms) {
  return (req, res, next) => {
    const userPerms = req.user.permissions || [];
    const has = perms.every(p => userPerms.includes(p));
    if (!has) {
      return res.status(403).json({
        error: 'Missing permissions',
        required: perms,
      });
    }
    next();
  };
}

app.post('/api/reports/export',
  authenticate,
  requirePermission('reports:read', 'reports:export'),
  exportReport
);
```

### Resource Ownership

```javascript
function requireOwnership(resourceLoader) {
  return async (req, res, next) => {
    const resource = await resourceLoader(req);
    if (!resource) {
      return res.status(404).json({ error: 'Not found' });
    }
    if (resource.userId !== req.user.id
        && req.user.role !== 'admin') {
      return res.status(403).json({ error: 'Forbidden' });
    }
    req.resource = resource;
    next();
  };
}

app.put('/api/posts/:id',
  authenticate,
  requireOwnership((req) => Post.findById(req.params.id)),
  updatePost
);
```

---

## Slide 13 — Validation Middleware

Input validation ensures **request data conforms to expected shapes** before it reaches your business logic. Never trust client input.

### express-validator

```javascript
const { body, validationResult } = require('express-validator');

const validateUser = [
  body('email')
    .isEmail()
    .normalizeEmail()
    .withMessage('Valid email required'),
  body('password')
    .isLength({ min: 8 })
    .matches(/\d/)
    .withMessage('Min 8 chars with a number'),
  body('name')
    .trim()
    .notEmpty()
    .escape(),

  // Check result
  (req, res, next) => {
    const errors = validationResult(req);
    if (!errors.isEmpty()) {
      return res.status(422).json({
        errors: errors.array()
      });
    }
    next();
  }
];

app.post('/api/users', validateUser, createUser);
```

### Zod Schema Validation

```javascript
const { z } = require('zod');

const CreateOrderSchema = z.object({
  items: z.array(z.object({
    productId: z.string().uuid(),
    quantity: z.number().int().positive(),
  })).min(1),
  shippingAddress: z.object({
    line1: z.string().min(1),
    city: z.string().min(1),
    postcode: z.string().regex(/^[A-Z]{1,2}\d/),
    country: z.string().length(2),
  }),
});

function validate(schema) {
  return (req, res, next) => {
    const result = schema.safeParse(req.body);
    if (!result.success) {
      return res.status(422).json({
        errors: result.error.issues,
      });
    }
    req.validated = result.data;
    next();
  };
}

app.post('/api/orders',
  validate(CreateOrderSchema),
  createOrder
);
```

### Joi Validation

```javascript
const Joi = require('joi');

const productSchema = Joi.object({
  name: Joi.string().min(2).max(100).required(),
  price: Joi.number().positive().precision(2).required(),
  category: Joi.string().valid(
    'electronics', 'books', 'clothing'
  ).required(),
  tags: Joi.array().items(Joi.string()).max(10),
  inStock: Joi.boolean().default(true),
});

function validateBody(schema) {
  return (req, res, next) => {
    const { error, value } = schema.validate(
      req.body,
      { abortEarly: false, stripUnknown: true }
    );
    if (error) {
      return res.status(422).json({
        errors: error.details.map(d => ({
          field: d.path.join('.'),
          message: d.message,
        })),
      });
    }
    req.body = value; // sanitised
    next();
  };
}

app.post('/api/products',
  validateBody(productSchema),
  createProduct
);
```

---

## Slide 14 — Request Processing Pipeline

A typical production Express app processes requests through a **layered pipeline**. Each layer has a single responsibility and can short-circuit the chain by sending a response.

### Complete Pipeline Example

```javascript
const express = require('express');
const app = express();

// ── Layer 1: Security & Parsing ──────────────
app.use(helmet());
app.use(cors({ origin: process.env.CLIENT_URL }));
app.use(compression());
app.use(express.json({ limit: '10kb' }));
app.use(express.urlencoded({ extended: true }));
app.use(cookieParser());

// ── Layer 2: Logging & Request ID ────────────
app.use(requestId);
app.use(morgan(':method :url :status :response-time ms'));

// ── Layer 3: Rate Limiting ───────────────────
app.use('/api', rateLimit({
  windowMs: 15 * 60 * 1000,  // 15 minutes
  max: 100,
  standardHeaders: true,
}));

// ── Layer 4: Static Assets ───────────────────
app.use(express.static('public', { maxAge: '1d' }));

// ── Layer 5: Authentication ──────────────────
app.use('/api', authenticate);

// ── Layer 6: Routes (validation inside) ──────
app.use('/api/users',  usersRouter);
app.use('/api/orders', ordersRouter);

// ── Layer 7: 404 Handler ─────────────────────
app.use((req, res) => {
  res.status(404).json({ error: 'Route not found' });
});

// ── Layer 8: Error Handler ───────────────────
app.use((err, req, res, next) => {
  logger.error({ err, requestId: req.id });
  res.status(err.status || 500).json({
    error: err.message,
    requestId: req.id,
  });
});
```

### Pipeline Layers

1. **Layer 1** — Security headers, CORS, body parsing
2. **Layer 2** — Request ID, logging
3. **Layer 3** — Rate limiting
4. **Layer 4** — Static assets (short-circuits)
5. **Layer 5** — Authentication (rejects 401)
6. **Layer 6** — Validate → Authorise → Handle
7. **Layer 7–8** — 404 catch-all → Error handler

### Design Principles

- **Fail fast** — reject bad requests early
- **Cheapest checks first** — rate limit before DB queries
- **Single responsibility** — one concern per middleware
- **Never skip error handlers** — always register them last

---

## Slide 15 — Async Middleware & Error Propagation

Express 4.x does **not** catch rejected promises in `async` middleware. Unhandled rejections crash the process or silently hang. You must wrap async handlers or upgrade to Express 5.

### The Problem

```javascript
// BROKEN — unhandled rejection if DB throws
app.get('/api/users', async (req, res) => {
  const users = await User.find(); // throws!
  res.json(users);
});
// Express never calls the error handler.
// Request hangs or process crashes.
```

### Solution 1: try/catch

```javascript
app.get('/api/users', async (req, res, next) => {
  try {
    const users = await User.find();
    res.json(users);
  } catch (err) {
    next(err); // forwards to error handler
  }
});
```

### Solution 2: Wrapper Function

```javascript
// asyncHandler wraps every async route
function asyncHandler(fn) {
  return (req, res, next) => {
    Promise.resolve(fn(req, res, next)).catch(next);
  };
}

// Clean — no try/catch in every route
app.get('/api/users', asyncHandler(async (req, res) => {
  const users = await User.find();
  res.json(users);
}));

app.get('/api/users/:id', asyncHandler(async (req, res) => {
  const user = await User.findByIdOrFail(req.params.id);
  res.json(user);
}));
```

### Solution 3: express-async-errors

```javascript
// Monkey-patches Express to catch async rejections
require('express-async-errors');

// Now this just works — no wrapper needed
app.get('/api/users', async (req, res) => {
  const users = await User.find();
  res.json(users);
});

// Errors automatically forwarded to error handler
```

### Solution 4: Express 5 (Built-In)

```javascript
// Express 5 natively catches async rejections
// npm install express@5
const express = require('express');

// No wrapper, no monkey-patch — it just works
app.get('/api/users', async (req, res) => {
  const users = await User.find();
  res.json(users);
});
// Rejected promise → error handler automatically
```

### Async Middleware Example

```javascript
// Async middleware also needs wrapping (Express 4)
const loadTenant = asyncHandler(async (req, res, next) => {
  const tenant = await Tenant.findByDomain(req.hostname);
  if (!tenant) throw new NotFoundError('Tenant');
  req.tenant = tenant;
  next(); // don't forget next() in middleware!
});

app.use('/api', loadTenant);
```

**Express 5 recommendation:** If starting a new project, use Express 5. It eliminates the async error gap entirely.

---

## Slide 16 — Middleware Composition Patterns

Real applications compose middleware using **factory functions**, **conditional logic**, and **chaining arrays** to keep code DRY and configurable.

### Factory Functions (Configurable)

```javascript
// Middleware that accepts options
function rateLimit({ windowMs, max, keyGenerator }) {
  const hits = new Map();

  return (req, res, next) => {
    const key = keyGenerator
      ? keyGenerator(req)
      : req.ip;
    const now = Date.now();
    const record = hits.get(key) || { count: 0, start: now };

    if (now - record.start > windowMs) {
      record.count = 0;
      record.start = now;
    }

    record.count++;
    hits.set(key, record);

    if (record.count > max) {
      return res.status(429).json({ error: 'Too many requests' });
    }
    next();
  };
}

// Different limits for different routes
app.use('/api', rateLimit({ windowMs: 60000, max: 100 }));
app.use('/auth/login', rateLimit({ windowMs: 60000, max: 5 }));
```

### Conditional Middleware

```javascript
function unless(middleware, ...paths) {
  return (req, res, next) => {
    if (paths.some(p => req.path.startsWith(p))) {
      return next(); // skip this middleware
    }
    middleware(req, res, next);
  };
}

// Authenticate everything except /auth and /health
app.use(unless(authenticate, '/auth', '/health', '/public'));
```

### Chaining Arrays

```javascript
// Group related middleware into reusable arrays
const apiDefaults = [
  helmet(),
  cors({ origin: process.env.CLIENT_URL }),
  express.json({ limit: '10kb' }),
  requestId,
  authenticate,
];

// Apply to all API routes
app.use('/api', ...apiDefaults);

// Route-specific chains
const createUserPipeline = [
  ...apiDefaults,
  requireRole('admin'),
  validateUser,
  createUser,
];

app.post('/api/users', createUserPipeline);
```

### Compose Utility

```javascript
// Combine multiple middleware into one
function compose(...middlewares) {
  return middlewares.flat();
}

const secured = compose(authenticate, requireRole('admin'));
const validated = (schema) => compose(authenticate, validate(schema));

app.get('/admin/users', ...secured, listUsers);
app.post('/api/orders', ...validated(orderSchema), createOrder);
```

### Environment-Based Middleware

```javascript
if (process.env.NODE_ENV === 'development') {
  app.use(morgan('dev'));
  app.use(require('errorhandler')());
}

if (process.env.NODE_ENV === 'production') {
  app.use(morgan('combined'));
  app.use(compression());
  app.use(helmet());
}
```

---

## Slide 17 — Testing Middleware

Middleware functions are plain functions — they can be **unit tested with mocks** or **integration tested with supertest**.

### Unit Testing with Mocks

```javascript
// middleware/requestId.js
const { randomUUID } = require('crypto');

function requestId(req, res, next) {
  req.id = req.headers['x-request-id'] || randomUUID();
  res.set('X-Request-Id', req.id);
  next();
}

// __tests__/requestId.test.js
const requestId = require('../middleware/requestId');

describe('requestId middleware', () => {
  let req, res, next;

  beforeEach(() => {
    req = { headers: {} };
    res = { set: jest.fn() };
    next = jest.fn();
  });

  it('generates a UUID when no header present', () => {
    requestId(req, res, next);
    expect(req.id).toMatch(
      /^[0-9a-f]{8}-[0-9a-f]{4}-/
    );
    expect(res.set).toHaveBeenCalledWith(
      'X-Request-Id', req.id
    );
    expect(next).toHaveBeenCalledTimes(1);
  });

  it('uses existing x-request-id header', () => {
    req.headers['x-request-id'] = 'abc-123';
    requestId(req, res, next);
    expect(req.id).toBe('abc-123');
    expect(next).toHaveBeenCalled();
  });
});
```

### Integration Testing with supertest

```javascript
const request = require('supertest');
const express = require('express');
const authenticate = require('../middleware/authenticate');

function createTestApp() {
  const app = express();
  app.use(express.json());
  app.use(authenticate);
  app.get('/protected', (req, res) => {
    res.json({ userId: req.user.id });
  });
  app.use((err, req, res, next) => {
    res.status(err.status || 500).json({ error: err.message });
  });
  return app;
}

describe('authenticate middleware', () => {
  const app = createTestApp();

  it('returns 401 without a token', async () => {
    const res = await request(app)
      .get('/protected')
      .expect(401);
    expect(res.body.error).toMatch(/missing/i);
  });

  it('returns 401 with expired token', async () => {
    const expired = createToken({ id: '1' }, { expiresIn: '0s' });
    await request(app)
      .get('/protected')
      .set('Authorization', `Bearer ${expired}`)
      .expect(401);
  });

  it('attaches user to req with valid token', async () => {
    const token = createToken({ id: '42', role: 'user' });
    const res = await request(app)
      .get('/protected')
      .set('Authorization', `Bearer ${token}`)
      .expect(200);
    expect(res.body.userId).toBe('42');
  });
});
```

**Testing tip:** Create a minimal Express app in each test file. Mount only the middleware under test and a simple route. This isolates tests from your production app configuration.

---

## Slide 18 — Performance Considerations

Middleware runs on **every matching request**. Poorly ordered or unnecessary middleware adds latency at scale. A few microseconds per middleware compound across millions of requests.

### Middleware Ordering

| Position | Middleware | Why |
|----------|-----------|-----|
| 1st | Health check | Short-circuit for load balancers |
| 2nd | Static files | No auth/parsing needed |
| 3rd | Rate limiter | Reject abusers before heavy work |
| 4th | Body parser | Only parse when needed |
| 5th | Auth | After parsing, before business logic |
| 6th | Validation | Reject bad input early |
| Last | Error handler | Catches all errors from above |

### Short-Circuit Early

```javascript
// Health check — skip all other middleware
app.get('/health', (req, res) => {
  res.status(200).send('ok');
});

// Only parse JSON on API routes
app.use('/api', express.json());

// Don't apply auth to public routes
app.use('/api/private', authenticate);
```

### Avoiding Unnecessary Work

```javascript
// BAD — parses body for every request (even static)
app.use(express.json());
app.use(express.static('public'));

// GOOD — static first, then parse
app.use(express.static('public'));
app.use('/api', express.json());

// BAD — logging creates strings for every request
app.use((req, res, next) => {
  console.log(JSON.stringify({
    method: req.method,
    url: req.url,
    headers: req.headers, // huge object
  }));
  next();
});

// GOOD — only log what you need
app.use((req, res, next) => {
  console.log(`${req.method} ${req.url}`);
  next();
});
```

### Measuring Middleware Cost

```javascript
function measure(name, middleware) {
  return (req, res, next) => {
    const start = process.hrtime.bigint();
    const origNext = next;
    next = (...args) => {
      const ns = Number(process.hrtime.bigint() - start);
      console.log(`${name}: ${(ns / 1e6).toFixed(2)}ms`);
      origNext(...args);
    };
    middleware(req, res, next);
  };
}

app.use(measure('helmet', helmet()));
app.use(measure('cors', cors()));
app.use(measure('json', express.json()));
```

---

## Slide 19 — Summary & Next Steps

### Core Takeaways

- Middleware = `(req, res, next)` — the universal building block
- Execution order is **registration order**
- Always call `next()` or send a response — never hang
- Error handlers have 4 arguments: `(err, req, res, next)`
- Wrap async middleware in Express 4 (or use Express 5)
- Compose via factory functions, arrays, and conditionals

### Best Practices

- Register middleware in order: security → parsing → auth → routes → errors
- Scope middleware to only the routes that need it
- Use Router for modular, testable route groups
- Validate all input before processing
- Centralise error handling with custom error classes
- Test middleware in isolation with mock req/res/next

### Next Steps

- Build a REST API with layered middleware pipeline
- Add JWT auth + role-based authorisation
- Implement Zod/Joi validation middleware
- Write unit + integration tests with supertest
- Add rate limiting and security headers in production
- Migrate to Express 5 for native async support

### Essential Packages

| Package | Purpose |
|---------|---------|
| `express` | Web framework |
| `helmet` | Security headers |
| `cors` | Cross-origin resource sharing |
| `morgan` | HTTP request logging |
| `express-rate-limit` | Rate limiting |
| `jsonwebtoken` | JWT signing & verification |
| `zod` / `joi` | Schema validation |
| `supertest` | HTTP integration testing |

### Resources

- **expressjs.com/en/guide/using-middleware.html** — official middleware guide
- **expressjs.com/en/guide/error-handling.html** — error handling guide
- **github.com/helmetjs/helmet** — security headers
- **github.com/ladjs/supertest** — HTTP testing
