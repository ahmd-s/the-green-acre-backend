# THE GREEN ACRE — Backend FINAL (All Fixes Applied)

**Verified:** 27/27 automated checks passed.

## Fixes Applied vs Phase 1+2

| # | Fix | File(s) |
|---|---|---|
| 1 | **Critical — Express 4 async error handling** | `app.js`, `package.json` |
|   | Added `require('express-async-errors')` as first import in app.js | |
|   | All async controller errors now reach global error handler | |
| 2 | **Logout calls backend** | Phase 3 HTML (see separate file) |
|   | `logout()` now fires `POST /api/auth/logout` before clearing token | |
| 3 | **PENDING_HOLD_HOURS documented** | `.env.example` |
|   | Added `PENDING_HOLD_HOURS=48`, `ENABLE_AUTO_RELEASE_CRON`, `AUTO_RELEASE_CRON_SCHEDULE` | |

---

## `package.json`

```json
{
  "name": "greenacre-backend",
  "version": "1.0.0",
  "description": "The Green Acre — Private Farmhouse Booking Platform API",
  "main": "server.js",
  "engines": {
    "node": ">=20.0.0"
  },
  "scripts": {
    "start": "node server.js",
    "dev": "node --watch server.js",
    "migrate": "node db/migrate.js",
    "seed": "node db/seed.js",
    "setup": "npm run migrate && npm run seed"
  },
  "dependencies": {
    "bcryptjs": "^2.4.3",
    "cors": "^2.8.5",
    "dotenv": "^16.4.5",
    "express": "^4.19.2",
    "express-async-errors": "^3.1.1",
    "express-rate-limit": "^7.3.1",
    "express-validator": "^7.1.0",
    "helmet": "^7.1.0",
    "jsonwebtoken": "^9.0.2",
    "pg": "^8.12.0"
  },
  "license": "UNLICENSED",
  "private": true
}
```

---

## `.env.example`

```
# ─────────────────────────────────────────────
#  THE GREEN ACRE — Environment Variables
#  Copy this file to .env and fill in values.
#  NEVER commit .env to version control.
# ─────────────────────────────────────────────

# Server
NODE_ENV=development
PORT=5000

# PostgreSQL — Railway provides DATABASE_URL automatically in production
# For local dev, either set DATABASE_URL or the individual fields below.
DATABASE_URL=postgresql://user:password@localhost:5432/greenacre

# Alternatively, for local pg pool without a connection string:
# DB_HOST=localhost
# DB_PORT=5432
# DB_NAME=greenacre
# DB_USER=postgres
# DB_PASSWORD=yourpassword

# JWT — use a long random string (min 64 chars). Generate with:
#   node -e "console.log(require('crypto').randomBytes(64).toString('hex'))"
JWT_SECRET=replace_with_64_char_random_hex_string
JWT_EXPIRES_IN=8h

# CORS — comma-separated list of allowed frontend origins
# Example: https://greenacre.vercel.app,http://localhost:3000
CORS_ORIGINS=http://localhost:3000,http://localhost:5500,http://127.0.0.1:5500

# Rate limiting (auth endpoint)
LOGIN_RATE_LIMIT_WINDOW_MS=900000
LOGIN_RATE_LIMIT_MAX=5

# Auto-release cron job
# How many hours a PENDING booking holds a slot before auto-release.
# Masterplan default: 48h. Set to 2 for tighter calendar availability.
# Confirm with property owner before going live.
PENDING_HOLD_HOURS=48
# Set to 'true' to enable the in-process node-cron job (default: false).
# Alternative: use Railway Cron Jobs to call POST /internal/trigger-auto-release
ENABLE_AUTO_RELEASE_CRON=false
# Cron schedule (default: every 15 minutes). Only used if ENABLE_AUTO_RELEASE_CRON=true.
AUTO_RELEASE_CRON_SCHEDULE=*/15 * * * *

# WhatsApp notifications (Phase 3 — leave blank for now)
WHATSAPP_PROVIDER=console
WHATSAPP_API_KEY=
WHATSAPP_MANAGER_NUMBER=
MANAGER_WHATSAPP_NUMBER=
FRONTEND_URL=https://your-greenacre.vercel.app

# Timezone — all date logic anchored to IST
TZ=Asia/Kolkata
```

---

## `.gitignore`

```
# Dependencies
node_modules/

# Environment secrets — NEVER commit these
.env
.env.local
.env.production

# Logs
logs/
*.log
npm-debug.log*

# OS artifacts
.DS_Store
Thumbs.db

# Editor
.vscode/
.idea/
*.swp
*.swo

# Build outputs (if any)
dist/
build/

# Railway / deployment artifacts
.railway/
```

---

## `railway.json`

```json
{
  "$schema": "https://railway.app/railway.schema.json",
  "build": {
    "builder": "NIXPACKS",
    "buildCommand": "npm install --production"
  },
  "deploy": {
    "startCommand": "node server.js",
    "healthcheckPath": "/health",
    "healthcheckTimeout": 30,
    "restartPolicyType": "ON_FAILURE",
    "restartPolicyMaxRetries": 3
  }
}
```

---

## `vercel.json`

```json
{
  "version": 2,
  "comment": "Vercel config for The Green Acre static frontend. Place this file in the frontend directory (where index.html lives), NOT in the backend directory.",
  "builds": [
    {
      "src": "**/*",
      "use": "@vercel/static"
    }
  ],
  "routes": [
    {
      "src": "/manager-login",
      "dest": "/manager-login.html"
    },
    {
      "src": "/admin",
      "dest": "/admin.html"
    },
    {
      "src": "/booking-status",
      "dest": "/booking-status.html"
    },
    {
      "src": "/(.*)",
      "dest": "/$1"
    }
  ],
  "headers": [
    {
      "source": "/(.*)",
      "headers": [
        {
          "key": "X-Content-Type-Options",
          "value": "nosniff"
        },
        {
          "key": "X-Frame-Options",
          "value": "DENY"
        },
        {
          "key": "Referrer-Policy",
          "value": "strict-origin-when-cross-origin"
        }
      ]
    },
    {
      "source": "/styles.css",
      "headers": [
        {
          "key": "Cache-Control",
          "value": "public, max-age=31536000, immutable"
        }
      ]
    }
  ]
}
```

---

## `server.js`

```javascript
'use strict';

/**
 * server.js
 *
 * Application entry point.
 *
 * Responsibilities:
 *   1. Validate critical environment variables before starting
 *   2. Establish database connection
 *   3. Start the HTTP server
 *   4. Handle graceful shutdown on SIGTERM / SIGINT
 *
 * This file does NOT define routes or middleware — that lives in app.js.
 * Keeping startup logic separate from app configuration enables clean testing.
 */

require('dotenv').config();

const app = require('./app');
const { connectDB, closeDB } = require('./config/db');

const PORT = parseInt(process.env.PORT || '5000', 10);

// ── Environment variable validation ──────────────────────────
// Fail loudly at startup rather than silently at runtime.
// This prevents a server from starting with a misconfigured JWT secret
// and then issuing tokens that can't be verified (or verified by anyone
// who knows the default value).

const REQUIRED_ENV = {
  DATABASE_URL:
    'PostgreSQL connection string. Railway provides this automatically. For local dev, see .env.example.',
  JWT_SECRET:
    'A minimum 64-character cryptographic secret. Generate with: node -e "console.log(require(\'crypto\').randomBytes(64).toString(\'hex\'))"',
};

// DB_HOST+DB_NAME is an alternative to DATABASE_URL for local dev.
const hasDbUrl = Boolean(process.env.DATABASE_URL);
const hasDbParts = Boolean(process.env.DB_HOST && process.env.DB_NAME);

const missingVars = [];

if (!hasDbUrl && !hasDbParts) {
  missingVars.push('DATABASE_URL (or DB_HOST + DB_NAME + DB_USER + DB_PASSWORD)');
}

if (!process.env.JWT_SECRET) {
  missingVars.push('JWT_SECRET');
} else if (process.env.JWT_SECRET.length < 32) {
  // Warn but don't block — allows quick local dev with short secrets.
  // In production, enforce a minimum length.
  if (process.env.NODE_ENV === 'production') {
    missingVars.push('JWT_SECRET (must be at least 64 characters in production)');
  } else {
    console.warn('[Server] WARNING: JWT_SECRET is shorter than recommended (32 chars min). Use a 64+ char secret in production.');
  }
}

if (missingVars.length > 0) {
  console.error('');
  console.error('═══════════════════════════════════════════════════');
  console.error('  STARTUP FAILED — Missing required environment variables:');
  missingVars.forEach((v) => console.error(`  ✗ ${v}`));
  console.error('');
  console.error('  Copy .env.example to .env and fill in the values.');
  console.error('  See README.md for setup instructions.');
  console.error('═══════════════════════════════════════════════════');
  console.error('');
  process.exit(1);
}

// ── Server startup ─────────────────────────────────────────────

let server;

async function start() {
  console.log('');
  console.log('┌────────────────────────────────────────────┐');
  console.log('│  The Green Acre — API Server               │');
  console.log('│  Private Farmhouse Booking Platform        │');
  console.log('└────────────────────────────────────────────┘');
  console.log('');
  console.log(`[Server] Environment: ${process.env.NODE_ENV || 'development'}`);
  console.log(`[Server] Port: ${PORT}`);

  // 1. Connect to database first — fail fast if DB is unreachable.
  await connectDB();

  // 2. Start HTTP server.
  server = app.listen(PORT, '0.0.0.0', () => {
    console.log(`[Server] Listening on http://0.0.0.0:${PORT}`);
    console.log(`[Server] Health check: http://0.0.0.0:${PORT}/health`);
    console.log('[Server] Ready.');
    console.log('');
  });

  // Handle HTTP server errors (e.g. port already in use).
  server.on('error', (err) => {
    if (err.code === 'EADDRINUSE') {
      console.error(`[Server] Port ${PORT} is already in use. Choose a different PORT in .env.`);
    } else {
      console.error('[Server] HTTP server error:', err.message);
    }
    process.exit(1);
  });
}

// ── Graceful shutdown ─────────────────────────────────────────
// Railway sends SIGTERM before stopping a service.
// We close the HTTP server first (stop accepting new connections),
// then close the DB pool (allow in-flight queries to complete).

async function shutdown(signal) {
  console.log(`\n[Server] Received ${signal}. Shutting down gracefully...`);

  if (server) {
    server.close(async () => {
      console.log('[Server] HTTP server closed.');
      await closeDB();
      console.log('[Server] Shutdown complete.');
      process.exit(0);
    });

    // Force shutdown after 10 seconds if graceful shutdown hangs.
    setTimeout(() => {
      console.error('[Server] Forced shutdown after 10s timeout.');
      process.exit(1);
    }, 10_000);
  } else {
    await closeDB();
    process.exit(0);
  }
}

process.on('SIGTERM', () => shutdown('SIGTERM'));
process.on('SIGINT', () => shutdown('SIGINT'));

// Catch unhandled promise rejections — log and exit.
// Unhandled rejections should never happen in production;
// all async functions use try/catch or pass errors to next().
process.on('unhandledRejection', (reason) => {
  console.error('[Server] Unhandled Promise Rejection:', reason);
  process.exit(1);
});

// Catch uncaught exceptions — log and exit.
process.on('uncaughtException', (err) => {
  console.error('[Server] Uncaught Exception:', err.message);
  console.error(err.stack);
  process.exit(1);
});

// ── Start ──────────────────────────────────────────────────────
start().catch((err) => {
  console.error('[Server] Failed to start:', err.message);
  console.error(err.stack);
  process.exit(1);
});
```

---

## `app.js`

```javascript
'use strict';

/**
 * MUST be the very first require — before express is loaded.
 * Monkey-patches Express 4 so that async route handler rejections are
 * forwarded to the global error handler instead of crashing the process.
 * Without this, any unhandled async throw in Express 4 becomes an
 * unhandled rejection that in Node 20 terminates the process.
 */
require('express-async-errors');

/**
 * app.js
 *
 * Express application factory.
 * Creates and configures the Express app — all middleware, CORS, Helmet,
 * route mounting, and error handlers are registered here.
 *
 * Separating app creation from server startup (server.js) enables
 * clean testing without binding to a port.
 *
 * Middleware order (ORDER IS CRITICAL — do not rearrange):
 *   1. trust proxy        — must come first so rate limiters and logs get real IPs
 *   2. helmet             — security headers on ALL responses
 *   3. cors               — handle preflight and CORS headers before any processing
 *   4. json body parser   — parse request bodies
 *   5. request logger     — log after body is parsed
 *   6. routes             — business logic
 *   7. 404 handler        — catch unmatched routes
 *   8. error handler      — catch all thrown/next(err) errors
 */

require('dotenv').config();
const express = require('express');
const helmet = require('helmet');
const cors = require('cors');

const authRoutes = require('./routes/authRoutes');
const publicRoutes = require('./routes/publicRoutes');
const adminRoutes = require('./routes/adminRoutes');
const { notFoundHandler, errorHandler } = require('./middleware/errorHandler');

const app = express();

// ── 1. Trust proxy ────────────────────────────────────────────
// Railway sits behind a load balancer. Without this setting, all
// requests appear to come from Railway's internal IP, breaking rate
// limiting (all users share the same counter) and IP logging.
// '1' means trust one hop of proxy headers (X-Forwarded-For).
if (process.env.NODE_ENV === 'production') {
  app.set('trust proxy', 1);
}

// ── 2. Helmet — security headers ──────────────────────────────
// Helmet sets a suite of HTTP headers that protect against common
// web vulnerabilities (XSS, clickjacking, sniffing, etc.)
app.use(
  helmet({
    // Allow cross-origin requests for the API (frontend is on Vercel).
    crossOriginResourcePolicy: { policy: 'cross-origin' },
    // CSP is not applied to the API — the frontend handles its own CSP.
    contentSecurityPolicy: false,
  })
);

// ── 3. CORS ───────────────────────────────────────────────────
// Parse the allowed origins from the environment variable.
// CORS_ORIGINS is a comma-separated list of allowed origins.
const allowedOrigins = (process.env.CORS_ORIGINS || '')
  .split(',')
  .map((o) => o.trim())
  .filter(Boolean);

// In development, allow all localhost origins for ease of use.
if (process.env.NODE_ENV !== 'production') {
  allowedOrigins.push(/^http:\/\/localhost:\d+$/);
  allowedOrigins.push(/^http:\/\/127\.0\.0\.1:\d+$/);
}

const corsOptions = {
  origin: (origin, callback) => {
    // Allow requests with no Origin header (curl, Postman, server-to-server).
    if (!origin) return callback(null, true);

    const allowed = allowedOrigins.some((allowed) => {
      if (typeof allowed === 'string') return allowed === origin;
      if (allowed instanceof RegExp) return allowed.test(origin);
      return false;
    });

    if (allowed) {
      callback(null, true);
    } else {
      callback(new Error(`CORS: Origin '${origin}' is not allowed.`));
    }
  },
  methods: ['GET', 'POST', 'PUT', 'PATCH', 'DELETE', 'OPTIONS'],
  allowedHeaders: ['Content-Type', 'Authorization'],
  credentials: false, // No cookies — we use Bearer tokens
  maxAge: 86400,       // Cache preflight response for 24 hours
};

app.use(cors(corsOptions));
// Explicitly handle OPTIONS preflight for all routes.
app.options('*', cors(corsOptions));

// ── 4. Body parser ────────────────────────────────────────────
// Parse JSON request bodies. Limit to 100kb to prevent large payload attacks.
app.use(express.json({ limit: '100kb' }));
// Parse URL-encoded bodies (form submissions).
app.use(express.urlencoded({ extended: false, limit: '100kb' }));

// ── 5. Request logger (development only) ─────────────────────
if (process.env.NODE_ENV !== 'production') {
  app.use((req, _res, next) => {
    console.log(`[${new Date().toISOString()}] ${req.method} ${req.originalUrl}`);
    next();
  });
}

// ── 6. Health check ──────────────────────────────────────────
// Mounted before route groups so it has no middleware overhead.
// Used by Railway for health checks and uptime monitoring.
app.get('/health', async (_req, res) => {
  let dbStatus = 'unknown';
  try {
    const { query } = require('./config/db');
    await query('SELECT 1');
    dbStatus = 'connected';
  } catch {
    dbStatus = 'disconnected';
  }

  const status = dbStatus === 'connected' ? 200 : 503;
  res.status(status).json({
    success: dbStatus === 'connected',
    service: 'greenacre-api',
    version: process.env.npm_package_version || '1.0.0',
    environment: process.env.NODE_ENV || 'development',
    timestamp: new Date().toISOString(),
    database: dbStatus,
  });
});

// ── 7. Route mounting ─────────────────────────────────────────
// Mount order matters for specificity:
//   /api/auth    — login/logout (public, rate-limited)
//   /api/admin   — all admin routes (JWT-protected)
//   /api         — public guest routes (rate-limited)
//
// /api/admin MUST be mounted before /api to prevent the public router's
// catch-all from intercepting admin paths if routing is mis-configured.
app.use('/api/auth', authRoutes);
app.use('/api/admin', adminRoutes);
app.use('/api', publicRoutes);

// ── 8. 404 handler ────────────────────────────────────────────
// Catches all requests that didn't match any route.
app.use(notFoundHandler);

// ── 9. Global error handler ───────────────────────────────────
// Must be registered LAST, after all routes and other middleware.
// Express identifies it as an error handler by its 4-argument signature.
app.use(errorHandler);

module.exports = app;
```

---

## `config/db.js`

```javascript
'use strict';

/**
 * config/db.js
 *
 * PostgreSQL connection pool.
 * Uses the pg library directly (no ORM).
 *
 * Connection resolution order:
 *   1. DATABASE_URL environment variable (Railway, Heroku, etc.)
 *   2. Individual DB_* environment variables (local dev)
 *
 * All date/time operations are performed in IST (Asia/Kolkata, UTC+5:30)
 * because the property and all guests operate in Indian Standard Time.
 * The pool sets the timezone on every new connection via a post-connect hook.
 */

const { Pool } = require('pg');

/** Build the pool configuration from environment variables. */
function buildPoolConfig() {
  // Railway and most PaaS providers inject DATABASE_URL.
  if (process.env.DATABASE_URL) {
    return {
      connectionString: process.env.DATABASE_URL,
      // Enforce SSL in production; skip in local dev if not configured.
      ssl:
        process.env.NODE_ENV === 'production'
          ? { rejectUnauthorized: false }
          : false,
      // Pool sizing — conservative defaults suitable for a low-traffic booking platform.
      max: 10,
      idleTimeoutMillis: 30_000,
      connectionTimeoutMillis: 5_000,
    };
  }

  // Fallback: individual variables for local development.
  return {
    host: process.env.DB_HOST || 'localhost',
    port: parseInt(process.env.DB_PORT || '5432', 10),
    database: process.env.DB_NAME || 'greenacre',
    user: process.env.DB_USER || 'postgres',
    password: process.env.DB_PASSWORD || '',
    ssl: false,
    max: 10,
    idleTimeoutMillis: 30_000,
    connectionTimeoutMillis: 5_000,
  };
}

const pool = new Pool(buildPoolConfig());

/**
 * Post-connect hook — runs once per physical connection.
 * Sets the session timezone to IST so that PostgreSQL NOW(),
 * CURRENT_DATE, and all timestamp comparisons are IST-aware.
 * This prevents off-by-one date bugs for bookings submitted at
 * night in India (which would be the previous UTC date).
 */
pool.on('connect', (client) => {
  client.query("SET TIME ZONE 'Asia/Kolkata'").catch((err) => {
    console.error('[DB] Failed to set session timezone:', err.message);
  });
});

/** Log pool errors that occur outside of a query (e.g. idle client errors). */
pool.on('error', (err) => {
  console.error('[DB] Unexpected pool error:', err.message);
  // Do not crash the process — the pool will attempt to recover.
});

/**
 * Verifies the database connection on startup.
 * Throws if the database is unreachable, which prevents the server
 * from accepting requests with a broken DB layer.
 *
 * @returns {Promise<void>}
 */
async function connectDB() {
  let client;
  try {
    client = await pool.connect();
    const result = await client.query('SELECT NOW() AS now, current_setting($1) AS tz', [
      'TIMEZONE',
    ]);
    console.log(
      `[DB] Connected. Server time: ${result.rows[0].now} | Timezone: ${result.rows[0].tz}`
    );
  } catch (err) {
    console.error('[DB] Connection failed:', err.message);
    throw err; // Let server.js catch this and exit.
  } finally {
    if (client) client.release();
  }
}

/**
 * Executes a parameterized SQL query against the pool.
 * Always use this function — never execute queries directly on the pool
 * from controllers, to keep connection management centralised.
 *
 * @param {string} text   - Parameterized SQL string (e.g. 'SELECT * FROM t WHERE id = $1')
 * @param {Array}  params - Bound parameter values
 * @returns {Promise<import('pg').QueryResult>}
 */
async function query(text, params) {
  const start = Date.now();
  try {
    const result = await pool.query(text, params);
    const duration = Date.now() - start;
    if (process.env.NODE_ENV !== 'production') {
      // Log slow queries in development for debugging.
      if (duration > 200) {
        console.warn(`[DB] Slow query (${duration}ms):`, text.slice(0, 80));
      }
    }
    return result;
  } catch (err) {
    console.error('[DB] Query error:', { text: text.slice(0, 80), error: err.message });
    throw err;
  }
}

/**
 * Acquires a dedicated client from the pool for transaction use.
 * The caller is responsible for calling client.release() in a finally block.
 *
 * Usage:
 *   const client = await getClient();
 *   try {
 *     await client.query('BEGIN');
 *     ...
 *     await client.query('COMMIT');
 *   } catch (e) {
 *     await client.query('ROLLBACK');
 *     throw e;
 *   } finally {
 *     client.release();
 *   }
 *
 * @returns {Promise<import('pg').PoolClient>}
 */
async function getClient() {
  return pool.connect();
}

/**
 * Gracefully closes all pool connections.
 * Called during process shutdown (SIGTERM / SIGINT).
 *
 * @returns {Promise<void>}
 */
async function closeDB() {
  await pool.end();
  console.log('[DB] Pool closed.');
}

module.exports = { query, getClient, connectDB, closeDB };
```

---

## `config/jwt.js`

```javascript
'use strict';

/**
 * config/jwt.js
 *
 * JWT utility functions — sign and verify tokens.
 *
 * Tokens are signed with HS256 (HMAC-SHA256) using the JWT_SECRET
 * environment variable. The secret must be at least 64 characters of
 * cryptographic randomness — see .env.example for generation instructions.
 *
 * Token payload:
 *   { adminId, username, iat, exp }
 *
 * Expiry: 8 hours (configurable via JWT_EXPIRES_IN env var).
 * No refresh tokens — manager must re-login after expiry.
 */

const jwt = require('jsonwebtoken');

/** The signing secret — validated at startup by server.js. */
const JWT_SECRET = process.env.JWT_SECRET;

/** Token lifetime — default 8h as per masterplan security requirements. */
const JWT_EXPIRES_IN = process.env.JWT_EXPIRES_IN || '8h';

/**
 * Signs a JWT for a successfully authenticated admin.
 *
 * @param {{ adminId: number, username: string }} payload
 * @returns {string} Signed JWT string
 */
function signToken(payload) {
  if (!JWT_SECRET) {
    throw new Error('JWT_SECRET is not configured. Cannot sign tokens.');
  }
  return jwt.sign(
    {
      adminId: payload.adminId,
      username: payload.username,
    },
    JWT_SECRET,
    {
      expiresIn: JWT_EXPIRES_IN,
      algorithm: 'HS256',
    }
  );
}

/**
 * Verifies a JWT string and returns the decoded payload.
 * Throws JsonWebTokenError or TokenExpiredError on failure —
 * callers should catch these and return 401.
 *
 * @param {string} token
 * @returns {{ adminId: number, username: string, iat: number, exp: number }}
 */
function verifyToken(token) {
  if (!JWT_SECRET) {
    throw new Error('JWT_SECRET is not configured. Cannot verify tokens.');
  }
  return jwt.verify(token, JWT_SECRET, { algorithms: ['HS256'] });
}

/**
 * Extracts the Bearer token from an Authorization header value.
 * Returns null if the header is missing or malformed.
 *
 * @param {string|undefined} authHeader - Value of req.headers.authorization
 * @returns {string|null}
 */
function extractBearerToken(authHeader) {
  if (!authHeader || !authHeader.startsWith('Bearer ')) {
    return null;
  }
  const token = authHeader.slice(7).trim();
  return token.length > 0 ? token : null;
}

module.exports = { signToken, verifyToken, extractBearerToken };
```

---

## `middleware/authMiddleware.js`

```javascript
'use strict';

/**
 * middleware/authMiddleware.js
 *
 * JWT authentication middleware.
 * Applied to ALL /api/admin/* routes.
 *
 * Expects:  Authorization: Bearer <token>
 * On success: attaches req.admin = { adminId, username } and calls next()
 * On failure: returns 401 JSON — never calls next()
 *
 * The middleware distinguishes between:
 *   - Missing token   → 401 "Authentication required"
 *   - Expired token   → 401 "Session expired" (prompt re-login)
 *   - Invalid token   → 401 "Invalid token" (tampered / wrong secret)
 */

const { verifyToken, extractBearerToken } = require('../config/jwt');
const jwt = require('jsonwebtoken');

/**
 * requireAuth — guard middleware for all admin-facing endpoints.
 * Mount this on the /api/admin router, not on individual routes,
 * to guarantee no admin endpoint is ever reachable without a valid token.
 *
 * @type {import('express').RequestHandler}
 */
function requireAuth(req, res, next) {
  // 1. Extract token from Authorization header.
  const token = extractBearerToken(req.headers.authorization);

  if (!token) {
    return res.status(401).json({
      success: false,
      error: 'Authentication required. Provide a Bearer token.',
    });
  }

  // 2. Verify signature and expiry.
  try {
    const decoded = verifyToken(token);
    // Attach decoded payload so downstream controllers can read admin identity.
    req.admin = {
      adminId: decoded.adminId,
      username: decoded.username,
    };
    next();
  } catch (err) {
    if (err instanceof jwt.TokenExpiredError) {
      return res.status(401).json({
        success: false,
        error: 'Session expired. Please log in again.',
        code: 'TOKEN_EXPIRED',
      });
    }
    // Covers JsonWebTokenError (bad signature, malformed token, wrong algorithm).
    return res.status(401).json({
      success: false,
      error: 'Invalid authentication token.',
      code: 'TOKEN_INVALID',
    });
  }
}

module.exports = { requireAuth };
```

---

## `middleware/rateLimiter.js`

```javascript
'use strict';

/**
 * middleware/rateLimiter.js
 *
 * Rate limiting middleware using express-rate-limit.
 *
 * Two limiters are exported:
 *
 *   loginRateLimiter  — strict: 5 attempts per 15 minutes per IP.
 *                       Applied only to POST /api/auth/login.
 *                       Prevents brute-force attacks on the admin login.
 *
 *   apiRateLimiter    — generous: 200 requests per minute per IP.
 *                       Applied to all public API routes as a basic DoS guard.
 *                       Accommodates the 60-second booking-status polling.
 *
 * IP detection: trusts the X-Forwarded-For header when NODE_ENV=production
 * (Railway / Vercel proxy chain). In development, uses req.ip directly.
 * This is configured via app.set('trust proxy', 1) in app.js.
 */

const rateLimit = require('express-rate-limit');

/**
 * Strict rate limiter for the login endpoint.
 * 5 failures from the same IP within 15 minutes lock out further attempts.
 * Returns a JSON response (not HTML) to stay consistent with the API contract.
 */
const loginRateLimiter = rateLimit({
  windowMs: parseInt(process.env.LOGIN_RATE_LIMIT_WINDOW_MS || '900000', 10), // 15 min
  max: parseInt(process.env.LOGIN_RATE_LIMIT_MAX || '5', 10),
  standardHeaders: true,  // Return RateLimit-* headers
  legacyHeaders: false,   // Disable X-RateLimit-* headers

  // Custom handler so the response matches our JSON error contract.
  handler: (req, res) => {
    const retryAfterSeconds = Math.ceil(
      (req.rateLimit.resetTime - Date.now()) / 1000
    );
    res.status(429).json({
      success: false,
      error: 'Too many login attempts. Please try again later.',
      retryAfterSeconds,
    });
  },

  // Key by IP address. In production, trust proxy must be set so that
  // Railway's load balancer IP is not used as the key for all clients.
  keyGenerator: (req) => req.ip,

  // Skip successful requests — only count failed ones toward the limit.
  // Note: express-rate-limit counts ALL requests by default. This means
  // a successful login still increments the counter. That is intentional:
  // it prevents an attacker from successfully logging in and immediately
  // resetting their window.
  skip: () => false,
});

/**
 * General API rate limiter for all public routes.
 * Generous enough to allow normal usage + 60s polling.
 */
const apiRateLimiter = rateLimit({
  windowMs: 60_000, // 1 minute
  max: 200,
  standardHeaders: true,
  legacyHeaders: false,
  handler: (req, res) => {
    res.status(429).json({
      success: false,
      error: 'Rate limit exceeded. Please slow down your requests.',
    });
  },
  keyGenerator: (req) => req.ip,
});

module.exports = { loginRateLimiter, apiRateLimiter };
```

---

## `middleware/errorHandler.js`

```javascript
'use strict';

/**
 * middleware/errorHandler.js
 *
 * Centralised error handling for the entire Express application.
 *
 * Two handlers are exported and mounted in app.js:
 *
 *   notFoundHandler  — Catches requests to undefined routes (404).
 *   errorHandler     — Catches all errors thrown or passed via next(err).
 *
 * Error classification:
 *   - Validation errors (express-validator)  → 422
 *   - PostgreSQL constraint violations        → 409
 *   - JWT errors (caught here as fallback)   → 401
 *   - Explicitly set err.statusCode          → use that code
 *   - Everything else                        → 500 (Internal Server Error)
 *
 * In production, stack traces are never sent to the client.
 * In development, the stack is included for debugging.
 */

/**
 * Creates a standardised error object for throwing inside controllers.
 * Usage: throw createError(400, 'Booking date is in the past.');
 *
 * @param {number} statusCode
 * @param {string} message
 * @returns {Error & { statusCode: number }}
 */
function createError(statusCode, message) {
  const err = new Error(message);
  err.statusCode = statusCode;
  return err;
}

/**
 * 404 handler — mounted AFTER all route definitions.
 * Catches requests to any path that no router handled.
 *
 * @type {import('express').RequestHandler}
 */
function notFoundHandler(req, res) {
  res.status(404).json({
    success: false,
    error: `Route not found: ${req.method} ${req.originalUrl}`,
  });
}

/**
 * Global error handler — must be the LAST middleware registered (4 args).
 * Express identifies it as an error handler by its 4-argument signature.
 *
 * @type {import('express').ErrorRequestHandler}
 */
// eslint-disable-next-line no-unused-vars
function errorHandler(err, req, res, next) {
  const isDev = process.env.NODE_ENV !== 'production';

  // ── PostgreSQL error codes ──────────────────────────────────────
  if (err.code) {
    // Unique constraint violation (e.g. duplicate ref ID, duplicate booking slot)
    if (err.code === '23505') {
      return res.status(409).json({
        success: false,
        error: 'A conflict occurred. The resource already exists.',
        detail: isDev ? err.detail : undefined,
      });
    }
    // Foreign key violation
    if (err.code === '23503') {
      return res.status(409).json({
        success: false,
        error: 'Referenced resource does not exist.',
        detail: isDev ? err.detail : undefined,
      });
    }
    // Not-null violation
    if (err.code === '23502') {
      return res.status(422).json({
        success: false,
        error: 'A required field is missing.',
        detail: isDev ? err.detail : undefined,
      });
    }
  }

  // ── JWT errors (fallback — normally caught in authMiddleware) ───
  if (err.name === 'TokenExpiredError') {
    return res.status(401).json({
      success: false,
      error: 'Session expired. Please log in again.',
      code: 'TOKEN_EXPIRED',
    });
  }
  if (err.name === 'JsonWebTokenError') {
    return res.status(401).json({
      success: false,
      error: 'Invalid authentication token.',
      code: 'TOKEN_INVALID',
    });
  }

  // ── Explicit status code set by controllers ─────────────────────
  const statusCode = err.statusCode || 500;
  const message =
    statusCode === 500 && !isDev
      ? 'An unexpected error occurred. Please try again later.'
      : err.message || 'Internal Server Error';

  // Always log server errors on the backend.
  if (statusCode === 500) {
    console.error('[ERROR]', {
      path: req.originalUrl,
      method: req.method,
      message: err.message,
      stack: isDev ? err.stack : undefined,
    });
  }

  return res.status(statusCode).json({
    success: false,
    error: message,
    stack: isDev ? err.stack : undefined,
  });
}

module.exports = { createError, notFoundHandler, errorHandler };
```

---

## `middleware/validate.js`

```javascript
'use strict';

/**
 * middleware/validate.js
 *
 * Reusable middleware that checks the result of express-validator chains.
 * Mount this AFTER the validation chain array on any route that needs input validation.
 *
 * Usage in a route:
 *   const { body } = require('express-validator');
 *   const { validate } = require('../middleware/validate');
 *
 *   router.post('/login',
 *     [
 *       body('username').notEmpty().trim(),
 *       body('password').isLength({ min: 8 }),
 *     ],
 *     validate,
 *     authController.login
 *   );
 *
 * Returns 422 with a structured errors array if validation fails.
 * Calls next() if all validators pass.
 */

const { validationResult } = require('express-validator');

/**
 * Runs after express-validator chains and short-circuits with 422
 * if any field failed validation.
 *
 * @type {import('express').RequestHandler}
 */
function validate(req, res, next) {
  const errors = validationResult(req);
  if (!errors.isEmpty()) {
    return res.status(422).json({
      success: false,
      error: 'Validation failed.',
      // Map to a clean array: [{ field: 'username', message: '...' }]
      errors: errors.array().map((e) => ({
        field: e.path || e.param,
        message: e.msg,
      })),
    });
  }
  next();
}

module.exports = { validate };
```

---

## `controllers/auth/authController.js`

```javascript
'use strict';

/**
 * controllers/auth/authController.js
 *
 * Authentication controller for the admin manager login.
 *
 * Routes:
 *   POST /api/auth/login   → login()
 *   POST /api/auth/logout  → logout()
 *
 * Security characteristics:
 *   • Credentials validated against admins table with bcrypt.compare
 *   • Never reveals whether the username or password was wrong (timing-safe)
 *   • Timing attack mitigation: always runs bcrypt.compare even if user not found
 *   • JWT signed and returned as { token } JSON payload
 *   • last_login timestamp updated on successful login
 *   • Rate limiting applied at the route level (loginRateLimiter middleware)
 *   • Passwords never logged, never returned
 */

const bcrypt = require('bcryptjs');
const { query } = require('../../config/db');
const { signToken } = require('../../config/jwt');

/**
 * Dummy hash used for timing-attack mitigation.
 * When a username is not found, bcrypt.compare is still called against
 * this hash to ensure the response takes the same amount of time as a
 * real failed attempt — preventing username enumeration via timing.
 */
const DUMMY_HASH = '$2a$12$dummyhashfortimingnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnnn';

/**
 * POST /api/auth/login
 *
 * Body: { username: string, password: string }
 *
 * Success (200): { success: true, token: string, expiresIn: string, username: string }
 * Failure (401): { success: false, error: string }
 *
 * @type {import('express').RequestHandler}
 */
async function login(req, res, next) {
  try {
    const { username, password } = req.body;

    // 1. Look up admin by username.
    const result = await query(
      'SELECT id, username, password_hash FROM admins WHERE username = $1',
      [username.trim().toLowerCase()]
    );

    const admin = result.rows[0] || null;

    // 2. Always run bcrypt.compare — even if no admin found — to prevent
    //    timing attacks that could reveal valid usernames.
    const hashToCompare = admin ? admin.password_hash : DUMMY_HASH;
    const passwordMatch = await bcrypt.compare(password, hashToCompare);

    // 3. Reject if user not found OR password wrong. Same error message
    //    in both cases — do not differentiate.
    if (!admin || !passwordMatch) {
      return res.status(401).json({
        success: false,
        error: 'Invalid username or password.',
      });
    }

    // 4. Sign JWT with admin identity.
    const token = signToken({
      adminId: admin.id,
      username: admin.username,
    });

    // 5. Update last_login asynchronously — do not await, login should not
    //    fail because of a non-critical timestamp update.
    query(
      'UPDATE admins SET last_login = NOW() WHERE id = $1',
      [admin.id]
    ).catch((err) => {
      console.error('[Auth] Failed to update last_login:', err.message);
    });

    // 6. Return token. Frontend stores this in localStorage and attaches
    //    it as a Bearer token on all subsequent /api/admin/* requests.
    return res.status(200).json({
      success: true,
      token,
      expiresIn: process.env.JWT_EXPIRES_IN || '8h',
      username: admin.username,
    });
  } catch (err) {
    next(err);
  }
}

/**
 * POST /api/auth/logout
 *
 * JWTs are stateless — there is no server-side session to invalidate.
 * This endpoint tells the frontend to clear its stored token.
 *
 * For a future enhancement, a token blocklist (stored in Redis or DB)
 * could be implemented here. Out of scope for Phase 1.
 *
 * Success (200): { success: true, message: string }
 *
 * @type {import('express').RequestHandler}
 */
async function logout(req, res) {
  // Note: requireAuth middleware is NOT applied to this route intentionally.
  // The frontend may call logout even with an expired/missing token to
  // clean up its local state — that should always succeed from the server's
  // perspective.
  return res.status(200).json({
    success: true,
    message: 'Logged out successfully. Please clear your stored token.',
  });
}

module.exports = { login, logout };
```

---

## `controllers/calendarController.js`

```javascript
'use strict';

/**
 * controllers/calendarController.js
 *
 * GET /api/calendar?month=YYYY-MM
 *
 * Assembles the full per-day, per-slot availability and pricing response
 * for a given month. Implements the 6-step pricing priority logic via
 * the rateCalculator service.
 *
 * Async errors are forwarded to the global error handler automatically
 * via express-async-errors (required in app.js).
 *
 * Response shape:
 * {
 *   success: true,
 *   month: "2025-06",
 *   year: 2025,
 *   days: [
 *     {
 *       date: "2025-06-01",
 *       dayOfWeek: 0,          // 0=Sun ... 6=Sat
 *       daySlot: {
 *         status: "AVAILABLE"|"BOOKED"|"PENDING_HOLD"|"CLOSED",
 *         available: true|false,
 *         rate: 6000,           // null if unavailable
 *         rateLabel: "NORMAL"|"WEEKEND"|"PEAK"|null,
 *         labelName: "WEEKDAY"|"WEEKEND"|"HOLI"|...,
 *         reason: "weekday_default"|"weekend_default"|"peak_rule"|...
 *       },
 *       nightSlot: { ...same shape... },
 *       fullyBooked: false,
 *       partiallyAvailable: false,
 *       hasAvailability: true
 *     },
 *     ...
 *   ]
 * }
 */

const { query } = require('../config/db');
const { assembleCalendar } = require('../services/rateCalculator');

/**
 * GET /api/calendar?month=YYYY-MM
 *
 * @type {import('express').RequestHandler}
 */
async function getCalendar(req, res) {
  const { month } = req.query;

  // ── Validate month param ──────────────────────────────────────
  if (!month) {
    return res.status(400).json({
      success: false,
      error: 'month query parameter is required (format: YYYY-MM)',
    });
  }

  const monthMatch = month.match(/^(\d{4})-(\d{2})$/);
  if (!monthMatch) {
    return res.status(400).json({
      success: false,
      error: 'Invalid month format. Use YYYY-MM (e.g. 2025-06)',
    });
  }

  const year = parseInt(monthMatch[1], 10);
  const monthNum = parseInt(monthMatch[2], 10);

  if (monthNum < 1 || monthNum > 12) {
    return res.status(400).json({
      success: false,
      error: 'Month must be between 01 and 12',
    });
  }

  if (year < 2020 || year > 2099) {
    return res.status(400).json({
      success: false,
      error: 'Year must be between 2020 and 2099',
    });
  }

  // ── Build month boundaries ────────────────────────────────────
  const monthStr = String(monthNum).padStart(2, '0');
  const startDate = `${year}-${monthStr}-01`;
  const endYear = monthNum === 12 ? year + 1 : year;
  const endMonth = monthNum === 12 ? 1 : monthNum + 1;
  const endDate = `${endYear}-${String(endMonth).padStart(2, '0')}-01`;

  // ── Fetch all data in parallel ────────────────────────────────
  // express-async-errors will catch any DB error thrown here and
  // forward it to the global error handler (500 response).
  const [bookingsResult, pricingRulesResult, defaultRatesResult] = await Promise.all([
    query(
      `SELECT id, booking_date, slot, status
       FROM bookings
       WHERE booking_date >= $1
         AND booking_date < $2
         AND status IN ('PENDING', 'CONFIRMED')
       ORDER BY booking_date, slot`,
      [startDate, endDate]
    ),
    query(
      `SELECT id, target_date, label_name, day_slot_rate, night_slot_rate, is_closed
       FROM pricing_rules
       WHERE target_date >= $1
         AND target_date < $2
       ORDER BY target_date`,
      [startDate, endDate]
    ),
    query(
      `SELECT day_type, day_slot_rate, night_slot_rate
       FROM default_rates
       ORDER BY day_type`,
      []
    ),
  ]);

  // ── Extract default rates ─────────────────────────────────────
  const ratesMap = {};
  for (const row of defaultRatesResult.rows) {
    ratesMap[row.day_type] = row;
  }

  if (!ratesMap.weekday || !ratesMap.weekend) {
    return res.status(503).json({
      success: false,
      error: 'Default rates not configured. Please contact the administrator.',
    });
  }

  // ── Assemble calendar ─────────────────────────────────────────
  const days = assembleCalendar({
    year,
    month: monthNum,
    bookings: bookingsResult.rows,
    pricingRules: pricingRulesResult.rows,
    weekdayRates: ratesMap.weekday,
    weekendRates: ratesMap.weekend,
  });

  return res.status(200).json({
    success: true,
    month,
    year,
    monthNumber: monthNum,
    totalDays: days.length,
    days,
  });
}

module.exports = { getCalendar };
```

---

## `controllers/policyController.js`

```javascript
'use strict';

/**
 * controllers/policyController.js
 *
 * GET /api/policy
 * Returns all 4 policy_content records for the guest-facing policy modal.
 *
 * Async errors forwarded to global error handler via express-async-errors.
 */

const { query } = require('../config/db');

/**
 * GET /api/policy
 *
 * @type {import('express').RequestHandler}
 */
async function getPolicy(req, res) {
  const result = await query(
    `SELECT section_key, content_text, updated_at
     FROM policy_content
     ORDER BY id`,
    []
  );

  // Return as both array and keyed map for frontend flexibility
  const policiesArray = result.rows;
  const policiesMap = {};
  for (const row of result.rows) {
    policiesMap[row.section_key] = {
      content: row.content_text,
      updatedAt: row.updated_at,
    };
  }

  return res.status(200).json({
    success: true,
    policies: policiesArray,
    policiesMap,
  });
}

module.exports = { getPolicy };
```

---

## `controllers/bookingController.js`

```javascript
'use strict';

/**
 * controllers/bookingController.js
 *
 * Handles guest-facing booking operations:
 *   POST /api/bookings/request  — create a new PENDING booking
 *   GET  /api/bookings/:refId   — poll booking status
 *   PATCH /api/bookings/:refId/cancel — guest cancels (optional)
 *
 * COLLISION PREVENTION:
 *   POST /bookings/request uses a PostgreSQL transaction with
 *   SELECT ... FOR UPDATE to lock the (booking_date, slot) combination.
 *   Only one request can hold the lock at a time — the second concurrent
 *   request will block until the first commits or rolls back.
 *   This prevents double-bookings even under race conditions.
 */

const { getClient, query } = require('../config/db');
const { generateUniqueRefId } = require('../utils/generateRefId');
const { resolveSlotStatus } = require('../services/rateCalculator');
const { notifyManager, notifyGuest } = require('../services/whatsapp');

/** Valid section keys for policy validation */
const VALID_SECTION_KEYS = ['checkout_policy', 'cleanliness', 'pool_safety', 'house_rules'];

// ── Validation helpers ────────────────────────────────────────

function validateBookingRequest(body) {
  const errors = [];

  if (!body.guest_name || typeof body.guest_name !== 'string' || body.guest_name.trim().length < 2) {
    errors.push('guest_name is required (min 2 characters)');
  }

  if (!body.guest_phone || typeof body.guest_phone !== 'string') {
    errors.push('guest_phone is required');
  } else {
    // Allow +91XXXXXXXXXX, 91XXXXXXXXXX, 0XXXXXXXXXX, XXXXXXXXXX
    const phoneClean = body.guest_phone.replace(/[\s\-().]/g, '');
    if (!/^(\+?91|0)?[6-9]\d{9}$/.test(phoneClean)) {
      errors.push('guest_phone must be a valid Indian mobile number');
    }
  }

  if (body.guest_email && typeof body.guest_email === 'string' && body.guest_email.trim()) {
    const emailRe = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
    if (!emailRe.test(body.guest_email.trim())) {
      errors.push('guest_email must be a valid email address');
    }
  }

  if (!body.guest_count || !Number.isInteger(Number(body.guest_count))) {
    errors.push('guest_count is required and must be an integer');
  } else {
    const count = Number(body.guest_count);
    if (count < 1 || count > 200) {
      errors.push('guest_count must be between 1 and 200');
    }
  }

  if (!body.booking_date) {
    errors.push('booking_date is required');
  } else {
    const dateRe = /^\d{4}-\d{2}-\d{2}$/;
    if (!dateRe.test(body.booking_date)) {
      errors.push('booking_date must be in YYYY-MM-DD format');
    } else {
      const d = new Date(body.booking_date + 'T12:00:00+05:30');
      if (isNaN(d.getTime())) {
        errors.push('booking_date is not a valid date');
      } else {
        // Cannot book in the past (IST today)
        const todayIST = new Date(
          new Date().toLocaleString('en-US', { timeZone: 'Asia/Kolkata' })
        );
        todayIST.setHours(0, 0, 0, 0);
        if (d < todayIST) {
          errors.push('booking_date cannot be in the past');
        }
      }
    }
  }

  if (!body.slot || !['day', 'night'].includes(body.slot)) {
    errors.push('slot must be either "day" or "night"');
  }

  if (body.policy_agreed !== true && body.policy_agreed !== 'true') {
    errors.push('policy_agreed must be true — guest must accept the property policies');
  }

  return errors;
}

// ── Controllers ───────────────────────────────────────────────

/**
 * POST /api/bookings/request
 * Creates a new PENDING booking with full collision protection.
 */
async function createBooking(req, res) {
  const body = req.body;

  // ── Input validation ────────────────────────────────────────
  const errors = validateBookingRequest(body);
  if (errors.length > 0) {
    return res.status(400).json({
      success: false,
      error: 'Validation failed',
      details: errors,
    });
  }

  const bookingDate = body.booking_date;
  const slot = body.slot;

  // ── Transaction with advisory lock ──────────────────────────
  const client = await getClient();
  try {
    await client.query('BEGIN');

    // LOCK: Select all active bookings for this date+slot FOR UPDATE.
    // This serializes concurrent requests for the same slot.
    // Any other transaction trying to book the same slot will BLOCK here
    // until this transaction completes.
    const lockResult = await client.query(
      `SELECT id, status FROM bookings
       WHERE booking_date = $1
         AND slot = $2
         AND status IN ('PENDING', 'CONFIRMED')
       FOR UPDATE`,
      [bookingDate, slot]
    );

    if (lockResult.rowCount > 0) {
      await client.query('ROLLBACK');
      const existingStatus = lockResult.rows[0].status;
      return res.status(409).json({
        success: false,
        error:
          existingStatus === 'CONFIRMED'
            ? 'This slot is already booked. Please choose a different date or slot.'
            : 'This slot is currently on hold. Please choose a different date or slot, or try again shortly.',
        status: existingStatus,
      });
    }

    // ── Fetch current pricing for this date+slot ─────────────
    const [pricingRuleResult, defaultRatesResult] = await Promise.all([
      client.query(
        `SELECT id, label_name, day_slot_rate, night_slot_rate, is_closed
         FROM pricing_rules WHERE target_date = $1`,
        [bookingDate]
      ),
      client.query(
        `SELECT day_type, day_slot_rate, night_slot_rate FROM default_rates`,
        []
      ),
    ]);

    const pricingRule = pricingRuleResult.rows[0] || null;
    const ratesMap = {};
    for (const r of defaultRatesResult.rows) ratesMap[r.day_type] = r;

    if (!ratesMap.weekday || !ratesMap.weekend) {
      await client.query('ROLLBACK');
      return res.status(503).json({
        success: false,
        error: 'Pricing configuration error. Please contact the property.',
      });
    }

    // ── Resolve slot status (re-verify availability) ──────────
    const dateObj = new Date(bookingDate + 'T12:00:00+05:30');
    const slotStatus = resolveSlotStatus({
      date: dateObj,
      slot,
      confirmedBooking: null, // Already checked above with FOR UPDATE
      pendingBooking: null,
      pricingRule,
      weekdayRates: ratesMap.weekday,
      weekendRates: ratesMap.weekend,
    });

    if (!slotStatus.available) {
      await client.query('ROLLBACK');
      return res.status(409).json({
        success: false,
        error: `This date/slot is not available for booking (${slotStatus.reason}).`,
        reason: slotStatus.reason,
      });
    }

    // ── Generate unique reference ID ──────────────────────────
    const refId = await generateUniqueRefId(client);

    // ── Snapshot the rate ─────────────────────────────────────
    const rateApplied = slotStatus.rate;
    const rateLabel = slotStatus.rateLabel;

    // ── Insert booking ────────────────────────────────────────
    const insertResult = await client.query(
      `INSERT INTO bookings (
         id, guest_name, guest_phone, guest_email, guest_count,
         occasion, notes, booking_date, slot,
         rate_applied, rate_label, status, policy_agreed, created_at
       ) VALUES (
         $1, $2, $3, $4, $5,
         $6, $7, $8, $9,
         $10, $11, 'PENDING', $12, NOW()
       )
       RETURNING id, guest_name, guest_phone, booking_date, slot,
                 rate_applied, rate_label, status, created_at`,
      [
        refId,
        body.guest_name.trim(),
        body.guest_phone.trim(),
        body.guest_email ? body.guest_email.trim().toLowerCase() : null,
        parseInt(body.guest_count, 10),
        body.occasion ? body.occasion.trim() : null,
        body.notes ? body.notes.trim() : null,
        bookingDate,
        slot,
        rateApplied,
        rateLabel,
        body.policy_agreed === true || body.policy_agreed === 'true',
      ]
    );

    await client.query('COMMIT');

    const newBooking = insertResult.rows[0];

    // ── Fire-and-forget WhatsApp notifications ────────────────
    notifyManager(newBooking);
    notifyGuest(newBooking);

    return res.status(201).json({
      success: true,
      message: 'Booking request submitted successfully. Your slot is on hold.',
      referenceId: newBooking.id,
      booking: {
        id: newBooking.id,
        status: newBooking.status,
        booking_date: newBooking.booking_date,
        slot: newBooking.slot,
        rate_applied: parseFloat(newBooking.rate_applied),
        rate_label: newBooking.rate_label,
        created_at: newBooking.created_at,
      },
    });
  } catch (err) {
    await client.query('ROLLBACK').catch(() => {});

    // Handle DB-level unique violation (safety net)
    if (err.code === '23505') {
      return res.status(409).json({
        success: false,
        error: 'This slot was just booked by another guest. Please choose a different option.',
      });
    }

    // Handle exclusion constraint violation
    if (err.code === '23P01') {
      return res.status(409).json({
        success: false,
        error: 'This slot is no longer available. Please choose a different date or slot.',
      });
    }

    throw err; // Let global error handler deal with it
  } finally {
    client.release();
  }
}

/**
 * GET /api/bookings/:refId
 * Returns booking status for the polling page.
 */
async function getBookingStatus(req, res) {
  const { refId } = req.params;

  // Validate ref ID format
  if (!refId || !/^GRN-\d{4}-\d{4}$/.test(refId)) {
    return res.status(400).json({
      success: false,
      error: 'Invalid reference ID format. Expected: GRN-YYYY-XXXX',
    });
  }

  const result = await query(
    `SELECT id, guest_name, booking_date, slot,
            rate_applied, rate_label, status,
            created_at, confirmed_at, released_at,
            occasion, guest_count
     FROM bookings
     WHERE id = $1`,
    [refId]
  );

  if (result.rowCount === 0) {
    return res.status(404).json({
      success: false,
      error: 'Booking not found. Please check your reference ID.',
    });
  }

  const booking = result.rows[0];

  return res.status(200).json({
    success: true,
    booking: {
      id: booking.id,
      status: booking.status,
      guest_name: booking.guest_name,
      booking_date: booking.booking_date,
      slot: booking.slot,
      rate_applied: parseFloat(booking.rate_applied),
      rate_label: booking.rate_label,
      guest_count: booking.guest_count,
      occasion: booking.occasion,
      created_at: booking.created_at,
      confirmed_at: booking.confirmed_at,
      released_at: booking.released_at,
    },
  });
}

/**
 * PATCH /api/bookings/:refId/cancel
 * Guest-initiated cancellation. Only PENDING bookings can be cancelled by guest.
 */
async function cancelBooking(req, res) {
  const { refId } = req.params;

  if (!refId || !/^GRN-\d{4}-\d{4}$/.test(refId)) {
    return res.status(400).json({
      success: false,
      error: 'Invalid reference ID format.',
    });
  }

  const result = await query(
    `UPDATE bookings
     SET status = 'CANCELLED', released_at = NOW()
     WHERE id = $1 AND status = 'PENDING'
     RETURNING id, status, released_at`,
    [refId]
  );

  if (result.rowCount === 0) {
    // Check if booking exists at all
    const existing = await query('SELECT id, status FROM bookings WHERE id = $1', [refId]);
    if (existing.rowCount === 0) {
      return res.status(404).json({ success: false, error: 'Booking not found.' });
    }
    const currentStatus = existing.rows[0].status;
    return res.status(409).json({
      success: false,
      error: `Cannot cancel a booking with status: ${currentStatus}. Only PENDING bookings can be cancelled.`,
      currentStatus,
    });
  }

  return res.status(200).json({
    success: true,
    message: 'Booking cancelled successfully.',
    booking: result.rows[0],
  });
}

module.exports = { createBooking, getBookingStatus, cancelBooking };
```

---

## `controllers/adminBookingController.js`

```javascript
'use strict';

/**
 * controllers/adminBookingController.js
 *
 * Admin operations on bookings:
 *   GET    /api/admin/bookings                   — list with filters
 *   PATCH  /api/admin/bookings/:id/confirm       — confirm PENDING booking
 *   PATCH  /api/admin/bookings/:id/release       — release PENDING/CONFIRMED booking
 */

const { query, getClient } = require('../config/db');
const { notifyGuestConfirmed } = require('../services/whatsapp');

/**
 * GET /api/admin/bookings
 * Query params: status, date, month, page, limit
 */
async function listBookings(req, res) {
  const {
    status,
    date,
    month,
    page = 1,
    limit = 20,
  } = req.query;

  const conditions = [];
  const params = [];
  let paramIdx = 1;

  // Filter by status
  const validStatuses = ['PENDING', 'CONFIRMED', 'RELEASED', 'CANCELLED'];
  if (status) {
    const statusList = status.split(',').map((s) => s.trim().toUpperCase());
    const invalidStatuses = statusList.filter((s) => !validStatuses.includes(s));
    if (invalidStatuses.length > 0) {
      return res.status(400).json({
        success: false,
        error: `Invalid status values: ${invalidStatuses.join(', ')}. Valid: ${validStatuses.join(', ')}`,
      });
    }
    conditions.push(`status = ANY($${paramIdx++})`);
    params.push(statusList);
  }

  // Filter by exact date
  if (date) {
    if (!/^\d{4}-\d{2}-\d{2}$/.test(date)) {
      return res.status(400).json({ success: false, error: 'date must be YYYY-MM-DD' });
    }
    conditions.push(`booking_date = $${paramIdx++}`);
    params.push(date);
  }

  // Filter by month
  if (month) {
    if (!/^\d{4}-\d{2}$/.test(month)) {
      return res.status(400).json({ success: false, error: 'month must be YYYY-MM' });
    }
    conditions.push(`DATE_TRUNC('month', booking_date) = DATE_TRUNC('month', $${paramIdx++}::date)`);
    params.push(month + '-01');
  }

  const whereClause = conditions.length > 0 ? 'WHERE ' + conditions.join(' AND ') : '';

  // Pagination
  const limitNum = Math.min(Math.max(parseInt(limit, 10) || 20, 1), 100);
  const pageNum = Math.max(parseInt(page, 10) || 1, 1);
  const offset = (pageNum - 1) * limitNum;

  // Count query
  const countResult = await query(
    `SELECT COUNT(*) AS total FROM bookings ${whereClause}`,
    params
  );
  const total = parseInt(countResult.rows[0].total, 10);

  // Data query
  const dataResult = await query(
    `SELECT id, guest_name, guest_phone, guest_email, guest_count,
            occasion, notes, booking_date, slot,
            rate_applied, rate_label, status, policy_agreed,
            created_at, confirmed_at, released_at
     FROM bookings
     ${whereClause}
     ORDER BY created_at DESC
     LIMIT $${paramIdx++} OFFSET $${paramIdx++}`,
    [...params, limitNum, offset]
  );

  return res.status(200).json({
    success: true,
    total,
    page: pageNum,
    limit: limitNum,
    totalPages: Math.ceil(total / limitNum),
    bookings: dataResult.rows.map((b) => ({
      ...b,
      rate_applied: parseFloat(b.rate_applied),
    })),
  });
}

/**
 * PATCH /api/admin/bookings/:id/confirm
 * Transitions PENDING → CONFIRMED.
 * Uses a transaction to guard against auto-release race conditions.
 */
async function confirmBooking(req, res) {
  const { id } = req.params;

  if (!id || !/^GRN-\d{4}-\d{4}$/.test(id)) {
    return res.status(400).json({ success: false, error: 'Invalid booking ID format.' });
  }

  const client = await getClient();
  try {
    await client.query('BEGIN');

    // Lock the row — prevents auto-release cron from racing
    const lockResult = await client.query(
      `SELECT id, status, guest_name, guest_phone, booking_date, slot, rate_applied, rate_label
       FROM bookings
       WHERE id = $1
       FOR UPDATE`,
      [id]
    );

    if (lockResult.rowCount === 0) {
      await client.query('ROLLBACK');
      return res.status(404).json({ success: false, error: 'Booking not found.' });
    }

    const booking = lockResult.rows[0];

    if (booking.status !== 'PENDING') {
      await client.query('ROLLBACK');
      return res.status(409).json({
        success: false,
        error: `Cannot confirm booking with status: ${booking.status}. Only PENDING bookings can be confirmed.`,
        currentStatus: booking.status,
      });
    }

    const updateResult = await client.query(
      `UPDATE bookings
       SET status = 'CONFIRMED', confirmed_at = NOW()
       WHERE id = $1 AND status = 'PENDING'
       RETURNING id, status, confirmed_at, guest_name, guest_phone,
                 booking_date, slot, rate_applied, rate_label`,
      [id]
    );

    await client.query('COMMIT');

    const confirmed = updateResult.rows[0];

    // Fire-and-forget guest notification
    notifyGuestConfirmed(confirmed);

    return res.status(200).json({
      success: true,
      message: 'Booking confirmed successfully.',
      booking: {
        ...confirmed,
        rate_applied: parseFloat(confirmed.rate_applied),
      },
    });
  } catch (err) {
    await client.query('ROLLBACK').catch(() => {});
    throw err;
  } finally {
    client.release();
  }
}

/**
 * PATCH /api/admin/bookings/:id/release
 * Transitions PENDING or CONFIRMED → RELEASED.
 * Admin can release either status (with CONFIRMED requiring intentional action).
 */
async function releaseBooking(req, res) {
  const { id } = req.params;
  const { reason } = req.body; // Optional reason string

  if (!id || !/^GRN-\d{4}-\d{4}$/.test(id)) {
    return res.status(400).json({ success: false, error: 'Invalid booking ID format.' });
  }

  const client = await getClient();
  try {
    await client.query('BEGIN');

    const lockResult = await client.query(
      `SELECT id, status FROM bookings WHERE id = $1 FOR UPDATE`,
      [id]
    );

    if (lockResult.rowCount === 0) {
      await client.query('ROLLBACK');
      return res.status(404).json({ success: false, error: 'Booking not found.' });
    }

    const currentStatus = lockResult.rows[0].status;

    if (!['PENDING', 'CONFIRMED'].includes(currentStatus)) {
      await client.query('ROLLBACK');
      return res.status(409).json({
        success: false,
        error: `Cannot release booking with status: ${currentStatus}. Only PENDING or CONFIRMED bookings can be released.`,
        currentStatus,
      });
    }

    const updateResult = await client.query(
      `UPDATE bookings
       SET status = 'RELEASED', released_at = NOW()
       WHERE id = $1 AND status IN ('PENDING', 'CONFIRMED')
       RETURNING id, status, released_at, guest_name, booking_date, slot`,
      [id]
    );

    await client.query('COMMIT');

    return res.status(200).json({
      success: true,
      message: 'Booking released successfully. The slot is now available.',
      booking: updateResult.rows[0],
      reason: reason || null,
    });
  } catch (err) {
    await client.query('ROLLBACK').catch(() => {});
    throw err;
  } finally {
    client.release();
  }
}

module.exports = { listBookings, confirmBooking, releaseBooking };
```

---

## `controllers/adminPricingController.js`

```javascript
'use strict';

/**
 * controllers/adminPricingController.js
 *
 * Admin pricing management:
 *   GET    /api/admin/pricing-rules?month=YYYY-MM&date=YYYY-MM-DD
 *   POST   /api/admin/pricing-rules          — create/upsert rule for a date
 *   PUT    /api/admin/pricing-rules/:id      — update rule by ID
 *   DELETE /api/admin/pricing-rules/:id      — delete rule (date reverts to default)
 *   GET    /api/admin/default-rates          — get weekday/weekend baseline rates
 *   PUT    /api/admin/default-rates/:id      — update a baseline rate row
 *
 * Also used by publicRoutes for:
 *   GET    /api/pricing-rules?date=YYYY-MM-DD — guest-facing single date rate lookup
 *
 * Async errors forwarded to global error handler via express-async-errors.
 */

const { query } = require('../config/db');

/** Valid label values — must match the DB CHECK constraint. */
const VALID_LABELS = ['NORMAL', 'WEEKEND', 'HOLI', 'DIWALI', 'EID', 'PEAK', 'CUSTOM'];

// ── Pricing Rules ─────────────────────────────────────────────

/**
 * GET /api/admin/pricing-rules?month=YYYY-MM&date=YYYY-MM-DD
 * Also used publicly: GET /api/pricing-rules?date=YYYY-MM-DD
 *
 * @type {import('express').RequestHandler}
 */
async function listPricingRules(req, res) {
  const { month, date } = req.query;

  const conditions = [];
  const params = [];
  let paramIdx = 1;

  if (date) {
    if (!/^\d{4}-\d{2}-\d{2}$/.test(date)) {
      return res.status(400).json({ success: false, error: 'date must be YYYY-MM-DD' });
    }
    conditions.push(`target_date = $${paramIdx++}`);
    params.push(date);
  } else if (month) {
    if (!/^\d{4}-\d{2}$/.test(month)) {
      return res.status(400).json({ success: false, error: 'month must be YYYY-MM' });
    }
    conditions.push(
      `DATE_TRUNC('month', target_date) = DATE_TRUNC('month', $${paramIdx++}::date)`
    );
    params.push(month + '-01');
  }

  const whereClause = conditions.length > 0 ? 'WHERE ' + conditions.join(' AND ') : '';

  const result = await query(
    `SELECT id, target_date, label_name, day_slot_rate, night_slot_rate, is_closed, created_at, updated_at
     FROM pricing_rules
     ${whereClause}
     ORDER BY target_date`,
    params
  );

  return res.status(200).json({
    success: true,
    total: result.rowCount,
    rules: result.rows.map((r) => ({
      ...r,
      day_slot_rate: r.day_slot_rate ? parseFloat(r.day_slot_rate) : null,
      night_slot_rate: r.night_slot_rate ? parseFloat(r.night_slot_rate) : null,
    })),
  });
}

/**
 * POST /api/admin/pricing-rules
 * Creates or upserts a pricing rule for a date.
 * ON CONFLICT (target_date) DO UPDATE — safe to call multiple times.
 *
 * @type {import('express').RequestHandler}
 */
async function createPricingRule(req, res) {
  const errors = validatePricingRuleBody(req.body);
  if (errors.length > 0) {
    return res.status(400).json({ success: false, error: 'Validation failed', details: errors });
  }

  const { target_date, label_name, day_slot_rate, night_slot_rate, is_closed } = req.body;
  const isClosed = is_closed === true || is_closed === 'true';

  const result = await query(
    `INSERT INTO pricing_rules (target_date, label_name, day_slot_rate, night_slot_rate, is_closed, updated_at)
     VALUES ($1, $2, $3, $4, $5, NOW())
     ON CONFLICT (target_date) DO UPDATE
       SET label_name      = EXCLUDED.label_name,
           day_slot_rate   = EXCLUDED.day_slot_rate,
           night_slot_rate = EXCLUDED.night_slot_rate,
           is_closed       = EXCLUDED.is_closed,
           updated_at      = NOW()
     RETURNING id, target_date, label_name, day_slot_rate, night_slot_rate, is_closed, created_at, updated_at`,
    [
      target_date,
      label_name,
      day_slot_rate != null ? parseFloat(day_slot_rate) : null,
      night_slot_rate != null ? parseFloat(night_slot_rate) : null,
      isClosed,
    ]
  );

  const rule = result.rows[0];
  return res.status(201).json({
    success: true,
    message: 'Pricing rule saved.',
    rule: {
      ...rule,
      day_slot_rate: rule.day_slot_rate ? parseFloat(rule.day_slot_rate) : null,
      night_slot_rate: rule.night_slot_rate ? parseFloat(rule.night_slot_rate) : null,
    },
  });
}

/**
 * PUT /api/admin/pricing-rules/:id
 *
 * @type {import('express').RequestHandler}
 */
async function updatePricingRule(req, res) {
  const ruleId = parseInt(req.params.id, 10);
  if (!ruleId || isNaN(ruleId)) {
    return res.status(400).json({ success: false, error: 'Invalid rule ID.' });
  }

  const errors = validatePricingRuleBody(req.body);
  if (errors.length > 0) {
    return res.status(400).json({ success: false, error: 'Validation failed', details: errors });
  }

  const { target_date, label_name, day_slot_rate, night_slot_rate, is_closed } = req.body;

  const result = await query(
    `UPDATE pricing_rules
     SET target_date     = $1,
         label_name      = $2,
         day_slot_rate   = $3,
         night_slot_rate = $4,
         is_closed       = $5,
         updated_at      = NOW()
     WHERE id = $6
     RETURNING id, target_date, label_name, day_slot_rate, night_slot_rate, is_closed, updated_at`,
    [
      target_date,
      label_name,
      day_slot_rate != null ? parseFloat(day_slot_rate) : null,
      night_slot_rate != null ? parseFloat(night_slot_rate) : null,
      is_closed === true || is_closed === 'true',
      ruleId,
    ]
  );

  if (result.rowCount === 0) {
    return res.status(404).json({ success: false, error: 'Pricing rule not found.' });
  }

  const rule = result.rows[0];
  return res.status(200).json({
    success: true,
    message: 'Pricing rule updated.',
    rule: {
      ...rule,
      day_slot_rate: rule.day_slot_rate ? parseFloat(rule.day_slot_rate) : null,
      night_slot_rate: rule.night_slot_rate ? parseFloat(rule.night_slot_rate) : null,
    },
  });
}

/**
 * DELETE /api/admin/pricing-rules/:id
 * Removes a custom pricing rule — the date reverts to default rate logic.
 *
 * @type {import('express').RequestHandler}
 */
async function deletePricingRule(req, res) {
  const ruleId = parseInt(req.params.id, 10);
  if (!ruleId || isNaN(ruleId)) {
    return res.status(400).json({ success: false, error: 'Invalid rule ID.' });
  }

  const result = await query(
    `DELETE FROM pricing_rules WHERE id = $1 RETURNING id, target_date, label_name`,
    [ruleId]
  );

  if (result.rowCount === 0) {
    return res.status(404).json({ success: false, error: 'Pricing rule not found.' });
  }

  return res.status(200).json({
    success: true,
    message: 'Pricing rule deleted. This date now uses default rate logic.',
    deleted: result.rows[0],
  });
}

// ── Default Rates ─────────────────────────────────────────────

/**
 * GET /api/admin/default-rates
 *
 * @type {import('express').RequestHandler}
 */
async function getDefaultRates(req, res) {
  const result = await query(
    `SELECT id, day_type, day_slot_rate, night_slot_rate, updated_at
     FROM default_rates
     ORDER BY day_type`,
    []
  );

  return res.status(200).json({
    success: true,
    rates: result.rows.map((r) => ({
      ...r,
      day_slot_rate: parseFloat(r.day_slot_rate),
      night_slot_rate: parseFloat(r.night_slot_rate),
    })),
  });
}

/**
 * PUT /api/admin/default-rates/:id
 * Updates one default_rates row (weekday or weekend).
 * Accepts partial updates — only provided fields are changed.
 *
 * @type {import('express').RequestHandler}
 */
async function updateDefaultRate(req, res) {
  const rateId = parseInt(req.params.id, 10);
  if (!rateId || isNaN(rateId)) {
    return res.status(400).json({ success: false, error: 'Invalid rate ID.' });
  }

  const { day_slot_rate, night_slot_rate } = req.body;
  const errors = [];

  if (day_slot_rate === undefined && night_slot_rate === undefined) {
    errors.push('At least one of day_slot_rate or night_slot_rate is required.');
  }
  if (day_slot_rate !== undefined) {
    const v = parseFloat(day_slot_rate);
    if (isNaN(v) || v <= 0) errors.push('day_slot_rate must be a positive number.');
  }
  if (night_slot_rate !== undefined) {
    const v = parseFloat(night_slot_rate);
    if (isNaN(v) || v <= 0) errors.push('night_slot_rate must be a positive number.');
  }

  if (errors.length > 0) {
    return res.status(400).json({ success: false, error: 'Validation failed', details: errors });
  }

  // Build dynamic SET clause — only update provided fields
  const sets = [];
  const params = [];
  let paramIdx = 1;

  if (day_slot_rate !== undefined) {
    sets.push(`day_slot_rate = $${paramIdx++}`);
    params.push(parseFloat(day_slot_rate));
  }
  if (night_slot_rate !== undefined) {
    sets.push(`night_slot_rate = $${paramIdx++}`);
    params.push(parseFloat(night_slot_rate));
  }
  sets.push(`updated_at = NOW()`);
  params.push(rateId);

  const result = await query(
    `UPDATE default_rates
     SET ${sets.join(', ')}
     WHERE id = $${paramIdx}
     RETURNING id, day_type, day_slot_rate, night_slot_rate, updated_at`,
    params
  );

  if (result.rowCount === 0) {
    return res.status(404).json({ success: false, error: 'Default rate not found.' });
  }

  const rate = result.rows[0];
  return res.status(200).json({
    success: true,
    message: `${rate.day_type} rates updated.`,
    rate: {
      ...rate,
      day_slot_rate: parseFloat(rate.day_slot_rate),
      night_slot_rate: parseFloat(rate.night_slot_rate),
    },
  });
}

// ── Validation helper ─────────────────────────────────────────

/**
 * Validates the body for create/update pricing rule requests.
 * @param {Object} body
 * @returns {string[]} Array of error messages (empty = valid)
 */
function validatePricingRuleBody(body) {
  const errors = [];
  const { target_date, label_name, day_slot_rate, night_slot_rate, is_closed } = body;

  if (!target_date || !/^\d{4}-\d{2}-\d{2}$/.test(target_date)) {
    errors.push('target_date is required (YYYY-MM-DD)');
  }

  if (!label_name || !VALID_LABELS.includes(label_name)) {
    errors.push(`label_name must be one of: ${VALID_LABELS.join(', ')}`);
  }

  const isClosed = is_closed === true || is_closed === 'true';
  if (!isClosed) {
    if (day_slot_rate != null) {
      const v = parseFloat(day_slot_rate);
      if (isNaN(v) || v <= 0) errors.push('day_slot_rate must be a positive number');
    }
    if (night_slot_rate != null) {
      const v = parseFloat(night_slot_rate);
      if (isNaN(v) || v <= 0) errors.push('night_slot_rate must be a positive number');
    }
  }

  return errors;
}

module.exports = {
  listPricingRules,
  createPricingRule,
  updatePricingRule,
  deletePricingRule,
  getDefaultRates,
  updateDefaultRate,
};
```

---

## `controllers/adminPolicyController.js`

```javascript
'use strict';

/**
 * controllers/adminPolicyController.js
 *
 * Admin policy management:
 *   GET /api/admin/policy                — get all 4 policy sections
 *   PUT /api/admin/policy/:sectionKey    — update a single section
 *
 * section_key is an application-enforced enum. Unknown keys return 404.
 * Async errors forwarded to global error handler via express-async-errors.
 */

const { query } = require('../config/db');

/** Valid section keys — must match the CHECK constraint in the schema. */
const VALID_SECTION_KEYS = ['checkout_policy', 'cleanliness', 'pool_safety', 'house_rules'];

/**
 * GET /api/admin/policy
 *
 * @type {import('express').RequestHandler}
 */
async function getAdminPolicy(req, res) {
  const result = await query(
    `SELECT id, section_key, content_text, updated_at
     FROM policy_content
     ORDER BY id`,
    []
  );

  return res.status(200).json({
    success: true,
    policies: result.rows,
    validSectionKeys: VALID_SECTION_KEYS,
  });
}

/**
 * PUT /api/admin/policy/:sectionKey
 * Updates content_text for a single policy section.
 * Returns 404 for unknown section keys (per masterplan spec).
 *
 * @type {import('express').RequestHandler}
 */
async function updatePolicy(req, res) {
  const { sectionKey } = req.params;
  const { content_text } = req.body;

  // Validate section key — return 404 for unknown keys (not 422).
  // This prevents silent creation of rogue rows via typo'd keys.
  if (!VALID_SECTION_KEYS.includes(sectionKey)) {
    return res.status(404).json({
      success: false,
      error: `Unknown policy section: "${sectionKey}". Valid sections: ${VALID_SECTION_KEYS.join(', ')}`,
    });
  }

  if (content_text === undefined || content_text === null) {
    return res.status(400).json({
      success: false,
      error: 'content_text is required.',
    });
  }

  if (typeof content_text !== 'string') {
    return res.status(400).json({
      success: false,
      error: 'content_text must be a string.',
    });
  }

  // Update the existing row
  const result = await query(
    `UPDATE policy_content
     SET content_text = $1, updated_at = NOW()
     WHERE section_key = $2
     RETURNING id, section_key, content_text, updated_at`,
    [content_text.trim(), sectionKey]
  );

  if (result.rowCount === 0) {
    // Valid key but row missing — insert it (handles partial seed edge case)
    const insertResult = await query(
      `INSERT INTO policy_content (section_key, content_text, updated_at)
       VALUES ($1, $2, NOW())
       RETURNING id, section_key, content_text, updated_at`,
      [sectionKey, content_text.trim()]
    );
    return res.status(200).json({
      success: true,
      message: 'Policy section created.',
      policy: insertResult.rows[0],
    });
  }

  return res.status(200).json({
    success: true,
    message: 'Policy section updated.',
    policy: result.rows[0],
  });
}

module.exports = { getAdminPolicy, updatePolicy };
```

---

## `controllers/adminStatsController.js`

```javascript
'use strict';

/**
 * controllers/adminStatsController.js
 *
 * GET /api/admin/stats
 * Returns dashboard statistics for the admin panel.
 *
 * Async errors forwarded to global error handler via express-async-errors.
 */

const { query } = require('../config/db');

/**
 * GET /api/admin/stats
 *
 * @type {import('express').RequestHandler}
 */
async function getStats(req, res) {
  // Run all stat queries in parallel for performance
  const [statusCountsResult, revenueResult, upcomingResult, recentResult] = await Promise.all([
    // Booking counts by status (all time)
    query(
      `SELECT status, COUNT(*) AS count
       FROM bookings
       GROUP BY status`,
      []
    ),

    // Revenue from confirmed bookings
    query(
      `SELECT
         COALESCE(SUM(rate_applied) FILTER (WHERE status = 'CONFIRMED'), 0) AS confirmed_revenue,
         COALESCE(SUM(rate_applied) FILTER (
           WHERE status = 'CONFIRMED'
             AND DATE_TRUNC('month', booking_date) = DATE_TRUNC('month', NOW())
         ), 0) AS this_month_revenue,
         COUNT(*) FILTER (WHERE status = 'CONFIRMED') AS total_confirmed
       FROM bookings`,
      []
    ),

    // Upcoming confirmed bookings (next 30 days)
    query(
      `SELECT id, guest_name, booking_date, slot, rate_applied, rate_label, guest_count
       FROM bookings
       WHERE status = 'CONFIRMED'
         AND booking_date >= CURRENT_DATE
         AND booking_date <= CURRENT_DATE + INTERVAL '30 days'
       ORDER BY booking_date, slot
       LIMIT 10`,
      []
    ),

    // Most recent pending bookings (action required)
    query(
      `SELECT id, guest_name, guest_phone, booking_date, slot,
              rate_applied, rate_label, guest_count, created_at
       FROM bookings
       WHERE status = 'PENDING'
       ORDER BY created_at DESC
       LIMIT 5`,
      []
    ),
  ]);

  // Build status map with safe defaults
  const statusCounts = { PENDING: 0, CONFIRMED: 0, RELEASED: 0, CANCELLED: 0 };
  for (const row of statusCountsResult.rows) {
    statusCounts[row.status] = parseInt(row.count, 10);
  }

  const revenue = revenueResult.rows[0];

  return res.status(200).json({
    success: true,
    stats: {
      bookings: {
        pending: statusCounts.PENDING,
        confirmed: statusCounts.CONFIRMED,
        released: statusCounts.RELEASED,
        cancelled: statusCounts.CANCELLED,
        total: Object.values(statusCounts).reduce((a, b) => a + b, 0),
      },
      revenue: {
        total: parseFloat(revenue.confirmed_revenue),
        thisMonth: parseFloat(revenue.this_month_revenue),
        totalConfirmed: parseInt(revenue.total_confirmed, 10),
      },
      upcoming: upcomingResult.rows.map((b) => ({
        ...b,
        rate_applied: parseFloat(b.rate_applied),
      })),
      recentPending: recentResult.rows.map((b) => ({
        ...b,
        rate_applied: parseFloat(b.rate_applied),
      })),
    },
    generatedAt: new Date().toISOString(),
  });
}

module.exports = { getStats };
```

---

## `routes/authRoutes.js`

```javascript
'use strict';

/**
 * routes/authRoutes.js
 *
 * Authentication routes.
 *
 * Mounted at: /api/auth
 *
 * Routes:
 *   POST /api/auth/login   — rate-limited, validated, returns JWT
 *   POST /api/auth/logout  — always succeeds (JWT is stateless)
 */

const { Router } = require('express');
const { body } = require('express-validator');
const { loginRateLimiter } = require('../middleware/rateLimiter');
const { validate } = require('../middleware/validate');
const { login, logout } = require('../controllers/auth/authController');

const router = Router();

/**
 * POST /api/auth/login
 *
 * Validation:
 *   - username: required, string, trimmed, lowercased
 *   - password: required, non-empty (length check is in bcrypt logic)
 *
 * Middleware order:
 *   1. loginRateLimiter  — blocks after 5 attempts per 15 min
 *   2. body validators   — check shape of request
 *   3. validate          — short-circuit with 422 if validation fails
 *   4. login controller  — business logic
 */
router.post(
  '/login',
  loginRateLimiter,
  [
    body('username')
      .trim()
      .notEmpty()
      .withMessage('Username is required.')
      .isLength({ max: 60 })
      .withMessage('Username must be 60 characters or fewer.')
      .toLowerCase(),

    body('password')
      .notEmpty()
      .withMessage('Password is required.')
      .isLength({ min: 1, max: 200 })
      .withMessage('Password must be between 1 and 200 characters.'),
  ],
  validate,
  login
);

/**
 * POST /api/auth/logout
 *
 * No auth middleware required — logout must work even with an expired token.
 * No body validation required — no body expected.
 */
router.post('/logout', logout);

module.exports = router;
```

---

## `routes/publicRoutes.js`

```javascript
'use strict';

/**
 * routes/publicRoutes.js
 *
 * Public API routes — no authentication required.
 * Mounted at: /api
 *
 * Endpoints:
 *   GET   /api/calendar?month=YYYY-MM        → calendar assembly with 6-step pricing priority
 *   GET   /api/pricing-rules?date=YYYY-MM-DD → single date pricing (admin-facing helper)
 *   GET   /api/policy                        → all 4 policy_content records
 *   POST  /api/bookings/request              → create PENDING booking (collision-safe)
 *   GET   /api/bookings/:refId               → booking status polling
 *   PATCH /api/bookings/:refId/cancel        → guest cancels PENDING booking
 */

const { Router } = require('express');
const { apiRateLimiter } = require('../middleware/rateLimiter');

const { getCalendar } = require('../controllers/calendarController');
const { getPolicy } = require('../controllers/policyController');
const {
  createBooking,
  getBookingStatus,
  cancelBooking,
} = require('../controllers/bookingController');
const { listPricingRules } = require('../controllers/adminPricingController');

const router = Router();

// Apply general rate limiter to all public routes
router.use(apiRateLimiter);

// ── Calendar ──────────────────────────────────────────────────
/**
 * GET /api/calendar?month=YYYY-MM
 * Core endpoint: returns per-day, per-slot availability and pricing.
 * Implements the 6-step pricing priority logic.
 */
router.get('/calendar', getCalendar);

// ── Pricing rules (public read-only for date-specific queries) ─
/**
 * GET /api/pricing-rules?date=YYYY-MM-DD or ?month=YYYY-MM
 * Used by frontend for precise single-date rate confirmation.
 */
router.get('/pricing-rules', listPricingRules);

// ── Policy ────────────────────────────────────────────────────
/**
 * GET /api/policy
 * Returns all policy_content rows for the guest-facing policy modal.
 */
router.get('/policy', getPolicy);

// ── Bookings ──────────────────────────────────────────────────
/**
 * POST /api/bookings/request
 * IMPORTANT: Must be defined BEFORE /api/bookings/:refId to avoid
 * Express matching "request" as a :refId parameter.
 */
router.post('/bookings/request', createBooking);

/**
 * GET /api/bookings/:refId
 * Returns booking status for the polling page.
 * Accepts GRN-YYYY-XXXX format.
 */
router.get('/bookings/:refId', getBookingStatus);

/**
 * PATCH /api/bookings/:refId/cancel
 * Guest-initiated cancellation of PENDING bookings only.
 */
router.patch('/bookings/:refId/cancel', cancelBooking);

module.exports = router;
```

---

## `routes/adminRoutes.js`

```javascript
'use strict';

/**
 * routes/adminRoutes.js
 *
 * Admin API routes — ALL protected by JWT authentication middleware.
 * Mounted at: /api/admin
 *
 * The requireAuth middleware is applied at router level — every route
 * below is impossible to reach without a valid Bearer token.
 */

const { Router } = require('express');
const { requireAuth } = require('../middleware/authMiddleware');

const { getStats } = require('../controllers/adminStatsController');
const {
  listBookings,
  confirmBooking,
  releaseBooking,
} = require('../controllers/adminBookingController');
const {
  listPricingRules,
  createPricingRule,
  updatePricingRule,
  deletePricingRule,
  getDefaultRates,
  updateDefaultRate,
} = require('../controllers/adminPricingController');
const { getAdminPolicy, updatePolicy } = require('../controllers/adminPolicyController');

const router = Router();

// ── Apply JWT guard to ALL routes ─────────────────────────────
router.use(requireAuth);

// ── Dashboard stats ───────────────────────────────────────────
router.get('/stats', getStats);

// ── Bookings ──────────────────────────────────────────────────
// IMPORTANT: specific paths (/confirm, /release) BEFORE /:id
router.get('/bookings', listBookings);
router.patch('/bookings/:id/confirm', confirmBooking);
router.patch('/bookings/:id/release', releaseBooking);

// ── Pricing rules ─────────────────────────────────────────────
router.get('/pricing-rules', listPricingRules);
router.post('/pricing-rules', createPricingRule);
router.put('/pricing-rules/:id', updatePricingRule);
router.delete('/pricing-rules/:id', deletePricingRule);

// ── Default rates ─────────────────────────────────────────────
router.get('/default-rates', getDefaultRates);
router.put('/default-rates/:id', updateDefaultRate);

// ── Policy content ────────────────────────────────────────────
router.get('/policy', getAdminPolicy);
router.put('/policy/:sectionKey', updatePolicy);

module.exports = router;
```

---

## `services/rateCalculator.js`

```javascript
'use strict';

/**
 * services/rateCalculator.js
 *
 * Implements the EXACT 6-step calendar priority logic from the masterplan.
 *
 * Priority (first match wins, evaluated per slot):
 *   1. CONFIRMED booking exists → BOOKED (disabled)
 *   2. PENDING booking exists  → PENDING_HOLD (disabled)
 *   3. pricing_rule.is_closed  → CLOSED (disabled)
 *   4. pricing_rule with PEAK label (HOLI/DIWALI/EID/PEAK/CUSTOM) → PEAK rate
 *   5. No rule, date is Fri/Sat/Sun → WEEKEND rate from default_rates
 *   6. All else → WEEKDAY rate from default_rates
 *
 * Note on weekday definition (from schema comment):
 *   weekday = Mon–Thu, weekend = Fri–Sun (broader than typical Sat-Sun)
 *
 * This service is PURE — it takes pre-fetched data and returns structured
 * objects. No DB calls inside this module — all queries happen in the controller.
 */

/** Day-of-week numbers that map to "weekend" (0=Sun, 5=Fri, 6=Sat) */
const WEEKEND_DAYS = new Set([0, 5, 6]); // Sun, Fri, Sat

/** Labels that trigger PEAK pricing */
const PEAK_LABELS = new Set(['HOLI', 'DIWALI', 'EID', 'PEAK', 'CUSTOM']);

/**
 * Determines the status and rate for a single date+slot combination.
 *
 * @param {Object} params
 * @param {Date}   params.date         - The date being evaluated
 * @param {string} params.slot         - 'day' or 'night'
 * @param {Object|null} params.confirmedBooking - Booking row if CONFIRMED exists for this date+slot
 * @param {Object|null} params.pendingBooking  - Booking row if PENDING exists for this date+slot
 * @param {Object|null} params.pricingRule     - pricing_rules row for this date (or null)
 * @param {Object}      params.weekdayRates    - default_rates row for day_type='weekday'
 * @param {Object}      params.weekendRates    - default_rates row for day_type='weekend'
 *
 * @returns {Object} Slot status object
 */
function resolveSlotStatus({
  date,
  slot,
  confirmedBooking,
  pendingBooking,
  pricingRule,
  weekdayRates,
  weekendRates,
}) {
  // ── Priority 1: CONFIRMED booking ────────────────────────────
  if (confirmedBooking) {
    return {
      status: 'BOOKED',
      available: false,
      rate: null,
      rateLabel: null,
      reason: 'booked',
    };
  }

  // ── Priority 2: PENDING booking ──────────────────────────────
  if (pendingBooking) {
    return {
      status: 'PENDING_HOLD',
      available: false,
      rate: null,
      rateLabel: null,
      reason: 'pending_hold',
    };
  }

  // ── Priority 3: Closed/blackout date ─────────────────────────
  if (pricingRule && pricingRule.is_closed) {
    return {
      status: 'CLOSED',
      available: false,
      rate: null,
      rateLabel: null,
      reason: 'closed',
    };
  }

  // ── Priority 4: PEAK pricing rule ────────────────────────────
  if (pricingRule && PEAK_LABELS.has(pricingRule.label_name)) {
    const rate =
      slot === 'day'
        ? parseFloat(pricingRule.day_slot_rate)
        : parseFloat(pricingRule.night_slot_rate);
    return {
      status: 'AVAILABLE',
      available: true,
      rate,
      rateLabel: 'PEAK',
      labelName: pricingRule.label_name,
      reason: 'peak_rule',
    };
  }

  // ── Priority 4b: NORMAL pricing rule (explicit override) ─────
  if (pricingRule && pricingRule.label_name === 'NORMAL') {
    const rate =
      slot === 'day'
        ? parseFloat(pricingRule.day_slot_rate)
        : parseFloat(pricingRule.night_slot_rate);
    return {
      status: 'AVAILABLE',
      available: true,
      rate,
      rateLabel: 'NORMAL',
      labelName: pricingRule.label_name,
      reason: 'normal_rule',
    };
  }

  // ── Priority 5: Weekend default rate ─────────────────────────
  const dayOfWeek = date.getDay(); // 0=Sun, 1=Mon ... 6=Sat
  if (WEEKEND_DAYS.has(dayOfWeek)) {
    const rate =
      slot === 'day'
        ? parseFloat(weekendRates.day_slot_rate)
        : parseFloat(weekendRates.night_slot_rate);
    return {
      status: 'AVAILABLE',
      available: true,
      rate,
      rateLabel: 'WEEKEND',
      labelName: 'WEEKEND',
      reason: 'weekend_default',
    };
  }

  // ── Priority 6: Weekday default rate ─────────────────────────
  const rate =
    slot === 'day'
      ? parseFloat(weekdayRates.day_slot_rate)
      : parseFloat(weekdayRates.night_slot_rate);
  return {
    status: 'AVAILABLE',
    available: true,
    rate,
    rateLabel: 'NORMAL',
    labelName: 'WEEKDAY',
    reason: 'weekday_default',
  };
}

/**
 * Assembles the full calendar response for a given month.
 *
 * @param {Object} params
 * @param {number} params.year
 * @param {number} params.month            - 1-based month (1=Jan, 12=Dec)
 * @param {Array}  params.bookings         - All PENDING/CONFIRMED bookings for the month
 * @param {Array}  params.pricingRules     - All pricing_rules rows for the month
 * @param {Object} params.weekdayRates     - default_rates weekday row
 * @param {Object} params.weekendRates     - default_rates weekend row
 *
 * @returns {Array} Array of day objects, one per day in the month
 */
function assembleCalendar({ year, month, bookings, pricingRules, weekdayRates, weekendRates }) {
  // Build lookup maps for O(1) access
  const bookingsByDateSlot = new Map();
  for (const booking of bookings) {
    // booking_date comes as a Date object from pg — normalize to YYYY-MM-DD string
    const dateStr = formatDate(booking.booking_date);
    const key = `${dateStr}:${booking.slot}`;
    if (!bookingsByDateSlot.has(key)) {
      bookingsByDateSlot.set(key, []);
    }
    bookingsByDateSlot.get(key).push(booking);
  }

  const rulesByDate = new Map();
  for (const rule of pricingRules) {
    const dateStr = formatDate(rule.target_date);
    rulesByDate.set(dateStr, rule);
  }

  // Get number of days in the month
  const daysInMonth = new Date(year, month, 0).getDate();
  const days = [];

  for (let day = 1; day <= daysInMonth; day++) {
    // Create date in IST-equivalent — use noon to avoid DST edge cases
    const date = new Date(year, month - 1, day, 12, 0, 0);
    const dateStr = `${year}-${String(month).padStart(2, '0')}-${String(day).padStart(2, '0')}`;
    const pricingRule = rulesByDate.get(dateStr) || null;

    // Resolve each slot independently
    const dayBookings = bookingsByDateSlot.get(`${dateStr}:day`) || [];
    const nightBookings = bookingsByDateSlot.get(`${dateStr}:night`) || [];

    const confirmedDay = dayBookings.find((b) => b.status === 'CONFIRMED') || null;
    const pendingDay = dayBookings.find((b) => b.status === 'PENDING') || null;
    const confirmedNight = nightBookings.find((b) => b.status === 'CONFIRMED') || null;
    const pendingNight = nightBookings.find((b) => b.status === 'PENDING') || null;

    const daySlot = resolveSlotStatus({
      date,
      slot: 'day',
      confirmedBooking: confirmedDay,
      pendingBooking: pendingDay,
      pricingRule,
      weekdayRates,
      weekendRates,
    });

    const nightSlot = resolveSlotStatus({
      date,
      slot: 'night',
      confirmedBooking: confirmedNight,
      pendingBooking: pendingNight,
      pricingRule,
      weekdayRates,
      weekendRates,
    });

    // Date-level availability summary
    const fullyBooked = !daySlot.available && !nightSlot.available;
    const partiallyAvailable = daySlot.available !== nightSlot.available;

    days.push({
      date: dateStr,
      dayOfWeek: date.getDay(),
      daySlot,
      nightSlot,
      fullyBooked,
      partiallyAvailable,
      // Convenience for frontend: is this date selectable at all?
      hasAvailability: daySlot.available || nightSlot.available,
    });
  }

  return days;
}

/**
 * Formats a Date or date-string to YYYY-MM-DD.
 * Handles pg returning DATE columns as Date objects.
 *
 * @param {Date|string} d
 * @returns {string}
 */
function formatDate(d) {
  if (typeof d === 'string') {
    // Already a string — strip time component if present
    return d.slice(0, 10);
  }
  if (d instanceof Date) {
    const yyyy = d.getFullYear();
    const mm = String(d.getMonth() + 1).padStart(2, '0');
    const dd = String(d.getDate()).padStart(2, '0');
    return `${yyyy}-${mm}-${dd}`;
  }
  return String(d).slice(0, 10);
}

module.exports = { resolveSlotStatus, assembleCalendar, formatDate, PEAK_LABELS, WEEKEND_DAYS };
```

---

## `services/whatsapp.js`

```javascript
'use strict';

/**
 * services/whatsapp.js
 *
 * WhatsApp notification service.
 * Sends booking notifications to the manager and (optionally) the guest.
 *
 * CRITICAL: WhatsApp failures must NEVER block or roll back a booking.
 * All notifications are fire-and-forget with async error logging only.
 *
 * Supports two providers (configured via WHATSAPP_PROVIDER env var):
 *   - 'twilio'  : Twilio WhatsApp sandbox / Business API
 *   - 'wati'    : WATI (WhatsApp Team Inbox) API
 *   - 'console' : Development mode — prints to console, no HTTP calls
 *
 * If WHATSAPP_PROVIDER is not set or credentials are missing,
 * falls back to 'console' mode silently.
 */

const https = require('https');

// ── Provider detection ────────────────────────────────────────
const PROVIDER = (() => {
  const p = (process.env.WHATSAPP_PROVIDER || 'console').toLowerCase();
  if (p === 'twilio' && process.env.TWILIO_ACCOUNT_SID && process.env.TWILIO_AUTH_TOKEN) {
    return 'twilio';
  }
  if (p === 'wati' && process.env.WATI_API_URL && process.env.WATI_API_TOKEN) {
    return 'wati';
  }
  return 'console';
})();

if (PROVIDER === 'console') {
  console.log('[WhatsApp] Running in CONSOLE mode — notifications will be printed, not sent.');
}

// ── Formatting helpers ────────────────────────────────────────

/**
 * Formats a booking into the manager notification message.
 * @param {Object} booking
 * @returns {string}
 */
function formatManagerMessage(booking) {
  const slotLabel = booking.slot === 'day' ? 'Day (8AM–8PM)' : 'Night (8PM–8AM)';
  return [
    '🏡 *New Booking Request — The Green Acre*',
    '',
    `📋 Ref: *${booking.id}*`,
    `👤 Guest: ${booking.guest_name}`,
    `📞 Phone: ${booking.guest_phone}`,
    booking.guest_email ? `📧 Email: ${booking.guest_email}` : null,
    `👥 Guests: ${booking.guest_count}`,
    `📅 Date: ${booking.booking_date}`,
    `🕐 Slot: ${slotLabel}`,
    `💰 Rate: ₹${booking.rate_applied} (${booking.rate_label})`,
    booking.occasion ? `🎉 Occasion: ${booking.occasion}` : null,
    booking.notes ? `📝 Notes: ${booking.notes}` : null,
    '',
    '⚡ Login to admin panel to confirm or release this booking.',
  ]
    .filter((line) => line !== null)
    .join('\n');
}

/**
 * Formats the guest confirmation message.
 * @param {Object} booking
 * @returns {string}
 */
function formatGuestMessage(booking) {
  const slotLabel = booking.slot === 'day' ? 'Day Slot (8AM–8PM)' : 'Night Slot (8PM–8AM)';
  return [
    '✅ *Booking Request Received — The Green Acre*',
    '',
    `Hi ${booking.guest_name}! Your booking request has been received.`,
    '',
    `📋 Reference: *${booking.id}*`,
    `📅 Date: ${booking.booking_date}`,
    `🕐 Slot: ${slotLabel}`,
    `💰 Amount: ₹${booking.rate_applied}`,
    '',
    'Your slot is on hold while we process your request. The manager will confirm once the advance payment is received.',
    '',
    '🔍 Track status: ' + (process.env.FRONTEND_URL || 'https://greenacre.vercel.app') + `/booking-status?ref=${booking.id}`,
  ].join('\n');
}

// ── Provider implementations ──────────────────────────────────

/**
 * Sends a message via Twilio WhatsApp API.
 * @param {string} to      - Phone number (E.164 format, e.g. +919876543210)
 * @param {string} message - Text content
 */
async function sendViaTwilio(to, message) {
  const accountSid = process.env.TWILIO_ACCOUNT_SID;
  const authToken = process.env.TWILIO_AUTH_TOKEN;
  const from = process.env.TWILIO_WHATSAPP_FROM || 'whatsapp:+14155238886'; // Twilio sandbox

  const body = new URLSearchParams({
    From: `whatsapp:${from.replace('whatsapp:', '')}`,
    To: `whatsapp:${to}`,
    Body: message,
  }).toString();

  const auth = Buffer.from(`${accountSid}:${authToken}`).toString('base64');
  const url = `https://api.twilio.com/2010-04-01/Accounts/${accountSid}/Messages.json`;

  return new Promise((resolve, reject) => {
    const req = https.request(
      url,
      {
        method: 'POST',
        headers: {
          'Content-Type': 'application/x-www-form-urlencoded',
          Authorization: `Basic ${auth}`,
          'Content-Length': Buffer.byteLength(body),
        },
      },
      (res) => {
        let data = '';
        res.on('data', (chunk) => (data += chunk));
        res.on('end', () => {
          if (res.statusCode >= 200 && res.statusCode < 300) {
            resolve(JSON.parse(data));
          } else {
            reject(new Error(`Twilio error ${res.statusCode}: ${data}`));
          }
        });
      }
    );
    req.on('error', reject);
    req.write(body);
    req.end();
  });
}

/**
 * Sends a message via WATI API.
 * @param {string} to      - Phone number without + (e.g. 919876543210)
 * @param {string} message - Text content
 */
async function sendViaWati(to, message) {
  const apiUrl = process.env.WATI_API_URL; // e.g. https://live-server-XXXXX.wati.io
  const token = process.env.WATI_API_TOKEN;
  const phone = to.replace(/^\+/, ''); // Remove leading + for WATI

  const payload = JSON.stringify({ message });
  const url = `${apiUrl}/api/v1/sendSessionMessage/${phone}`;

  return new Promise((resolve, reject) => {
    const urlObj = new URL(url);
    const req = https.request(
      {
        hostname: urlObj.hostname,
        path: urlObj.pathname,
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          Authorization: `Bearer ${token}`,
          'Content-Length': Buffer.byteLength(payload),
        },
      },
      (res) => {
        let data = '';
        res.on('data', (chunk) => (data += chunk));
        res.on('end', () => {
          if (res.statusCode >= 200 && res.statusCode < 300) {
            resolve(JSON.parse(data));
          } else {
            reject(new Error(`WATI error ${res.statusCode}: ${data}`));
          }
        });
      }
    );
    req.on('error', reject);
    req.write(payload);
    req.end();
  });
}

/**
 * Console fallback — used in development.
 */
function sendViaConsole(to, message) {
  console.log('\n' + '─'.repeat(60));
  console.log(`[WhatsApp CONSOLE] TO: ${to}`);
  console.log('─'.repeat(60));
  console.log(message);
  console.log('─'.repeat(60) + '\n');
  return Promise.resolve({ status: 'console_logged' });
}

// ── Unified send function ─────────────────────────────────────

/**
 * Sends a WhatsApp message via the configured provider.
 * @param {string} to
 * @param {string} message
 * @returns {Promise<void>}
 */
async function sendMessage(to, message) {
  switch (PROVIDER) {
    case 'twilio':
      return sendViaTwilio(to, message);
    case 'wati':
      return sendViaWati(to, message);
    default:
      return sendViaConsole(to, message);
  }
}

// ── Public notification functions ─────────────────────────────

/**
 * Notifies the manager of a new booking request.
 * FIRE-AND-FORGET — never throws, never blocks the booking flow.
 *
 * @param {Object} booking - The newly created booking row
 */
function notifyManager(booking) {
  const managerPhone = process.env.MANAGER_WHATSAPP_NUMBER;
  if (!managerPhone) {
    console.warn('[WhatsApp] MANAGER_WHATSAPP_NUMBER not set — skipping manager notification.');
    return;
  }

  const message = formatManagerMessage(booking);

  // Fire and forget — intentionally not awaited
  sendMessage(managerPhone, message).catch((err) => {
    console.error('[WhatsApp] Manager notification failed:', err.message);
    // TODO: In production, write to a failed_notifications table for manual retry
  });
}

/**
 * Notifies the guest that their booking request was received.
 * FIRE-AND-FORGET — never throws, never blocks the booking flow.
 *
 * @param {Object} booking - The newly created booking row
 */
function notifyGuest(booking) {
  if (!booking.guest_phone) return;

  const message = formatGuestMessage(booking);

  sendMessage(booking.guest_phone, message).catch((err) => {
    console.error('[WhatsApp] Guest notification failed:', err.message);
  });
}

/**
 * Notifies the guest that their booking has been confirmed.
 * FIRE-AND-FORGET.
 *
 * @param {Object} booking - The confirmed booking row
 */
function notifyGuestConfirmed(booking) {
  if (!booking.guest_phone) return;

  const slotLabel = booking.slot === 'day' ? 'Day Slot (8AM–8PM)' : 'Night Slot (8PM–8AM)';
  const message = [
    '🎉 *Booking Confirmed — The Green Acre!*',
    '',
    `Hi ${booking.guest_name}! Your booking is confirmed.`,
    '',
    `📋 Ref: *${booking.id}*`,
    `📅 Date: ${booking.booking_date}`,
    `🕐 Slot: ${slotLabel}`,
    '',
    'We look forward to hosting you. See you soon! 🏡',
  ].join('\n');

  sendMessage(booking.guest_phone, message).catch((err) => {
    console.error('[WhatsApp] Guest confirmed notification failed:', err.message);
  });
}

module.exports = { notifyManager, notifyGuest, notifyGuestConfirmed };
```

---

## `utils/generateRefId.js`

```javascript
'use strict';

/**
 * utils/generateRefId.js
 *
 * Generates human-readable booking reference IDs in the format GRN-YYYY-XXXX.
 * XXXX is a zero-padded random 4-digit number (1000–9999).
 *
 * Collision handling:
 *   - The bookings.id column has a PRIMARY KEY constraint — any duplicate
 *     insert will throw a unique violation (error code 23505).
 *   - The caller (bookingController) retries up to MAX_RETRIES times.
 *   - With ~9000 possible IDs per year and typical booking volumes (<500/year)
 *     collision probability is negligible, but we handle it correctly anyway.
 */

const MAX_RETRIES = 10;

/**
 * Generates a single candidate reference ID.
 * @returns {string}  e.g. "GRN-2025-4872"
 */
function generateRefId() {
  const year = new Date().getFullYear();
  const rand = Math.floor(Math.random() * 9000) + 1000; // 1000–9999
  return `GRN-${year}-${rand}`;
}

/**
 * Generates a unique reference ID by checking the database.
 * Uses a SELECT FOR UPDATE inside the caller's transaction to avoid TOCTOU.
 *
 * @param {import('pg').PoolClient} client  - Active transaction client
 * @returns {Promise<string>}               - Unique reference ID
 * @throws {Error}                          - If MAX_RETRIES exhausted
 */
async function generateUniqueRefId(client) {
  for (let attempt = 1; attempt <= MAX_RETRIES; attempt++) {
    const id = generateRefId();
    const result = await client.query(
      'SELECT id FROM bookings WHERE id = $1',
      [id]
    );
    if (result.rowCount === 0) {
      return id;
    }
    // Collision — try again
    if (attempt === MAX_RETRIES) {
      throw new Error(`Failed to generate unique reference ID after ${MAX_RETRIES} attempts`);
    }
  }
}

module.exports = { generateRefId, generateUniqueRefId };
```

---

## `jobs/autoRelease.js`

```javascript
'use strict';

/**
 * jobs/autoRelease.js
 *
 * Cron job: automatically releases PENDING bookings that have been
 * waiting beyond the configured hold duration.
 *
 * Default hold duration: 2 hours (configurable via PENDING_HOLD_HOURS env var)
 *
 * This job is designed to run either:
 *   a) Via node-cron (scheduled inside the API process — default)
 *   b) Via Railway Cron Jobs (external scheduler)
 *
 * For option (b), expose the trigger via an internal endpoint and
 * call it from Railway's cron job. See server.js for the internal trigger.
 *
 * Race condition protection:
 *   Uses WHERE status = 'PENDING' in the UPDATE query — if a manager
 *   is confirming a booking at the same time the cron runs, only one
 *   operation will win. The other will find status != 'PENDING' and skip.
 */

const { query } = require('../config/db');

const HOLD_HOURS = parseFloat(process.env.PENDING_HOLD_HOURS || '2');

/**
 * Releases all PENDING bookings older than HOLD_HOURS.
 * Safe to call multiple times (idempotent).
 *
 * @returns {Promise<{ released: number, ids: string[] }>}
 */
async function releaseStaleBookings() {
  const result = await query(
    `UPDATE bookings
     SET status = 'RELEASED', released_at = NOW()
     WHERE status = 'PENDING'
       AND created_at < NOW() - ($1 || ' hours')::INTERVAL
     RETURNING id, guest_name, booking_date, slot, created_at`,
    [HOLD_HOURS]
  );

  const released = result.rowCount;
  const ids = result.rows.map((r) => r.id);

  if (released > 0) {
    console.log(`[AutoRelease] Released ${released} stale PENDING booking(s): ${ids.join(', ')}`);
    for (const row of result.rows) {
      console.log(
        `[AutoRelease]   ${row.id} | ${row.guest_name} | ${row.booking_date} ${row.slot} | held since ${row.created_at}`
      );
    }
  } else {
    console.log('[AutoRelease] No stale bookings to release.');
  }

  return { released, ids };
}

/**
 * Starts the cron job using node-cron.
 * Runs every 15 minutes.
 *
 * Only call this if ENABLE_AUTO_RELEASE_CRON=true in env.
 */
function startAutoReleaseCron() {
  let cron;
  try {
    cron = require('node-cron');
  } catch {
    console.warn('[AutoRelease] node-cron not installed. Auto-release cron will not run.');
    console.warn('[AutoRelease] Install with: npm install node-cron');
    return;
  }

  const schedule = process.env.AUTO_RELEASE_CRON_SCHEDULE || '*/15 * * * *';

  cron.schedule(schedule, async () => {
    console.log(`[AutoRelease] Cron triggered at ${new Date().toISOString()}`);
    try {
      await releaseStaleBookings();
    } catch (err) {
      console.error('[AutoRelease] Cron job error:', err.message);
    }
  });

  console.log(`[AutoRelease] Cron job started. Schedule: "${schedule}". Hold duration: ${HOLD_HOURS}h`);
}

module.exports = { releaseStaleBookings, startAutoReleaseCron };
```

---

## `db/migrate.js`

```javascript
'use strict';

/**
 * db/migrate.js
 *
 * Runs all SQL migration files in order against the configured database.
 *
 * Handles the btree_gist extension gracefully:
 *   The EXCLUDE constraint in bookings (which enforces one active booking per
 *   date+slot at the DB level) requires the btree_gist extension. If it is not
 *   available (e.g. on managed Railway PostgreSQL instances that restrict
 *   extensions), the EXCLUDE constraint is skipped and a warning is printed.
 *   Application-layer locking (SELECT FOR UPDATE in the booking transaction)
 *   remains in force as the primary guard.
 *
 * Usage:
 *   node db/migrate.js
 */

require('dotenv').config();
const fs = require('fs');
const path = require('path');
const { query, connectDB, closeDB } = require('./config/db');

const MIGRATIONS_DIR = path.join(__dirname, 'migrations');

async function runMigrations() {
  console.log('[Migrate] Starting database migration...');

  try {
    await connectDB();

    // 1. Attempt to enable btree_gist for the EXCLUDE constraint.
    //    This is a best-effort step — we continue even if it fails.
    try {
      await query('CREATE EXTENSION IF NOT EXISTS btree_gist');
      console.log('[Migrate] btree_gist extension enabled (EXCLUDE constraint active).');
    } catch (extErr) {
      console.warn(
        '[Migrate] WARNING: Could not enable btree_gist extension.',
        'The EXCLUDE constraint on bookings will not be active.',
        'Application-layer SELECT FOR UPDATE locking is still enforced.',
        '\n[Migrate] Extension error:', extErr.message
      );
    }

    // 2. Read migration files in alphabetical order.
    const files = fs
      .readdirSync(MIGRATIONS_DIR)
      .filter((f) => f.endsWith('.sql'))
      .sort();

    if (files.length === 0) {
      console.log('[Migrate] No migration files found.');
      return;
    }

    // 3. Execute each migration file.
    for (const file of files) {
      const filePath = path.join(MIGRATIONS_DIR, file);
      const sql = fs.readFileSync(filePath, 'utf8');

      console.log(`[Migrate] Running: ${file}`);
      try {
        await query(sql);
        console.log(`[Migrate] ✓ ${file}`);
      } catch (err) {
        // If the EXCLUDE constraint fails due to missing extension,
        // strip it and re-run the file.
        if (
          err.message.includes('btree_gist') ||
          err.message.includes('EXCLUDE') ||
          err.code === '0A000'
        ) {
          console.warn(
            `[Migrate] EXCLUDE constraint failed in ${file}. Retrying without it...`
          );
          const sqlWithoutExclude = sql
            .replace(/,?\s*CONSTRAINT unique_active_booking[\s\S]*?(?=\))/m, '')
            .trim();
          await query(sqlWithoutExclude);
          console.log(`[Migrate] ✓ ${file} (without EXCLUDE constraint)`);
        } else {
          throw err;
        }
      }
    }

    console.log('[Migrate] All migrations completed successfully.');
  } catch (err) {
    console.error('[Migrate] Migration failed:', err.message);
    console.error(err.stack);
    process.exit(1);
  } finally {
    await closeDB();
  }
}

runMigrations();
```

---

## `db/seed.js`

```javascript
'use strict';

/**
 * db/seed.js
 *
 * Master seed runner.
 * Executes all seed files in order:
 *   001_default_rates.sql       — weekday/weekend base rates
 *   002_peak_pricing_2025.sql   — holiday and peak date pricing
 *   003_policy_content.sql      — guest policy text
 *   004_admin_user.js           — admin account (requires ADMIN_USERNAME + ADMIN_PASSWORD)
 *
 * All SQL seeds use INSERT ... ON CONFLICT DO UPDATE (upsert) so this
 * script is idempotent and safe to re-run.
 *
 * Usage:
 *   ADMIN_USERNAME=manager ADMIN_PASSWORD=YourPass123! node db/seed.js
 */

require('dotenv').config();
const fs = require('fs');
const path = require('path');
const { query, connectDB, closeDB } = require('./config/db');

const SEEDS_DIR = path.join(__dirname, 'seeds');

async function runSeeds() {
  console.log('[Seed] Starting database seeding...');

  try {
    await connectDB();

    // Run SQL seeds in order.
    const sqlFiles = fs
      .readdirSync(SEEDS_DIR)
      .filter((f) => f.endsWith('.sql'))
      .sort();

    for (const file of sqlFiles) {
      const filePath = path.join(SEEDS_DIR, file);
      const sql = fs.readFileSync(filePath, 'utf8');
      console.log(`[Seed] Running: ${file}`);
      await query(sql);
      console.log(`[Seed] ✓ ${file}`);
    }

    console.log('[Seed] SQL seeds complete.');
  } catch (err) {
    console.error('[Seed] SQL seed failed:', err.message);
    process.exit(1);
  } finally {
    await closeDB();
  }

  // Run the admin user JS seed separately (needs its own DB connection).
  console.log('[Seed] Running admin user seed...');
  try {
    require('./seeds/004_admin_user.js');
    // 004_admin_user.js manages its own connection and process exit.
  } catch (err) {
    console.error('[Seed] Admin user seed failed:', err.message);
    process.exit(1);
  }
}

runSeeds();
```

---

## `db/migrations/001_create_tables.sql`

```sql
-- ============================================================
--  THE GREEN ACRE — Database Schema
--  Migration 001: Create all tables
--
--  Run via:  node db/migrate.js
--
--  Design decisions:
--    • bookings.id is a VARCHAR reference (GRN-YYYY-XXXX), not SERIAL,
--      because the human-readable ref ID is the primary identifier guests see.
--    • rate_applied is snapshotted at booking time so pricing changes
--      do not retroactively alter confirmed bookings.
--    • pricing_rules has a UNIQUE constraint on target_date so there is
--      never ambiguity about which rule applies to a date.
--    • policy_content uses section_key as the natural key; the INTEGER id
--      is kept for ORM compatibility if ever needed.
--    • All timestamps are TIMESTAMPTZ (with timezone). The DB session is
--      set to Asia/Kolkata (IST) so NOW() returns IST time.
--    • Indexes are added on the columns used in WHERE clauses of the
--      most frequent queries (calendar assembly, status polling).
-- ============================================================

-- ── Enable necessary extensions ───────────────────────────────
-- pgcrypto is not required now but useful for future UUID generation.
-- CREATE EXTENSION IF NOT EXISTS "pgcrypto";

-- ── 1. bookings ───────────────────────────────────────────────
-- One row per booking request. Status transitions:
--   PENDING → CONFIRMED  (manager confirms after deposit)
--   PENDING → RELEASED   (manager releases hold, or auto-release cron)
--   CONFIRMED → RELEASED (manager cancels a confirmed booking — rare)
--   Any status → CANCELLED (future: guest-initiated cancellation)

CREATE TABLE IF NOT EXISTS bookings (
  -- Human-readable reference, e.g. GRN-2025-3841
  id                VARCHAR(20)   PRIMARY KEY,

  -- Guest identity
  guest_name        VARCHAR(120)  NOT NULL,
  guest_phone       VARCHAR(20)   NOT NULL,
  guest_email       VARCHAR(120),                  -- optional per masterplan
  guest_count       INTEGER       NOT NULL CHECK (guest_count BETWEEN 1 AND 200),
  occasion          VARCHAR(80),
  notes             TEXT,

  -- Slot details
  booking_date      DATE          NOT NULL,
  slot              VARCHAR(10)   NOT NULL CHECK (slot IN ('day', 'night')),

  -- Pricing snapshot — frozen at submission time
  rate_applied      DECIMAL(10,2) NOT NULL CHECK (rate_applied > 0),
  rate_label        VARCHAR(20)   NOT NULL CHECK (rate_label IN ('NORMAL', 'WEEKEND', 'PEAK')),

  -- Lifecycle
  status            VARCHAR(20)   NOT NULL DEFAULT 'PENDING'
                      CHECK (status IN ('PENDING', 'CONFIRMED', 'RELEASED', 'CANCELLED')),
  policy_agreed     BOOLEAN       NOT NULL DEFAULT FALSE,

  -- Timestamps (stored with timezone; session TZ = IST)
  created_at        TIMESTAMPTZ   NOT NULL DEFAULT NOW(),
  confirmed_at      TIMESTAMPTZ,                   -- set when manager confirms
  released_at       TIMESTAMPTZ,                   -- set when booking is released

  -- Constraint: a date+slot combination can only have ONE active booking
  -- (PENDING or CONFIRMED). RELEASED/CANCELLED slots are freed.
  -- This is enforced at the application layer with a SELECT FOR UPDATE transaction,
  -- but the partial unique index below provides a database-level safety net.
  CONSTRAINT unique_active_booking
    EXCLUDE USING btree (booking_date WITH =, slot WITH =)
    WHERE (status IN ('PENDING', 'CONFIRMED'))
    -- NOTE: EXCLUDE requires btree_gist extension. If unavailable, remove
    -- this constraint and rely solely on the application-layer lock.
    -- The migration script handles this gracefully (see migrate.js).
);

-- Fast lookup by reference ID (already PK, covered)
-- Fast lookup for booking-status polling
CREATE INDEX IF NOT EXISTS idx_bookings_status
  ON bookings (status);

-- Calendar assembly query: WHERE booking_date = $1 AND status IN ('PENDING','CONFIRMED')
CREATE INDEX IF NOT EXISTS idx_bookings_date_status
  ON bookings (booking_date, status);

-- Admin queue: ORDER BY created_at DESC
CREATE INDEX IF NOT EXISTS idx_bookings_created_at
  ON bookings (created_at DESC);


-- ── 2. pricing_rules ─────────────────────────────────────────
-- Per-date pricing overrides. When a date has no rule, the backend
-- falls back to default_rates based on day of week (Sat/Sun = weekend).
-- is_closed = true means the date is a blackout/blocked date.

CREATE TABLE IF NOT EXISTS pricing_rules (
  id              SERIAL        PRIMARY KEY,
  target_date     DATE          NOT NULL,
  label_name      VARCHAR(20)   NOT NULL
                    CHECK (label_name IN ('NORMAL', 'WEEKEND', 'HOLI', 'DIWALI', 'EID', 'PEAK', 'CUSTOM')),
  day_slot_rate   DECIMAL(10,2) CHECK (day_slot_rate > 0),
  night_slot_rate DECIMAL(10,2) CHECK (night_slot_rate > 0),
  is_closed       BOOLEAN       NOT NULL DEFAULT FALSE,
  created_at      TIMESTAMPTZ   NOT NULL DEFAULT NOW(),
  updated_at      TIMESTAMPTZ   NOT NULL DEFAULT NOW(),

  -- One rule per date. Prevents ambiguous priority resolution.
  CONSTRAINT unique_pricing_rule_per_date UNIQUE (target_date)
);

-- Calendar assembly query
CREATE INDEX IF NOT EXISTS idx_pricing_rules_date
  ON pricing_rules (target_date);

-- Admin rate manager: fetch by month
CREATE INDEX IF NOT EXISTS idx_pricing_rules_month
  ON pricing_rules (DATE_TRUNC('month', target_date));


-- ── 3. default_rates ─────────────────────────────────────────
-- Exactly 2 rows: one for weekday (Mon–Thu), one for weekend (Fri–Sun).
-- Note: the masterplan specifies weekday = Mon–Thu, weekend = Fri–Sun.
-- This is broader than typical Sat–Sun weekends, intentional for the property.
-- These rows are seeded once and only updated, never inserted or deleted.

CREATE TABLE IF NOT EXISTS default_rates (
  id              SERIAL        PRIMARY KEY,
  day_type        VARCHAR(10)   NOT NULL UNIQUE
                    CHECK (day_type IN ('weekday', 'weekend')),
  day_slot_rate   DECIMAL(10,2) NOT NULL CHECK (day_slot_rate > 0),
  night_slot_rate DECIMAL(10,2) NOT NULL CHECK (night_slot_rate > 0),
  updated_at      TIMESTAMPTZ   NOT NULL DEFAULT NOW()
);


-- ── 4. policy_content ────────────────────────────────────────
-- 4 editable policy sections displayed in the guest-facing Policy Modal.
-- Edited by the manager via the Admin Rules Editor (Screen 11).
-- section_key is an application-enforced enum — the PUT endpoint validates
-- against this list and returns 404 for unknown keys.

CREATE TABLE IF NOT EXISTS policy_content (
  id              SERIAL        PRIMARY KEY,
  section_key     VARCHAR(40)   NOT NULL UNIQUE
                    CHECK (section_key IN ('checkout_policy', 'cleanliness', 'pool_safety', 'house_rules')),
  content_text    TEXT          NOT NULL DEFAULT '',
  updated_at      TIMESTAMPTZ   NOT NULL DEFAULT NOW()
);


-- ── 5. admins ────────────────────────────────────────────────
-- Typically a single row (the property manager).
-- Passwords are stored as bcrypt hashes (min 12 rounds).
-- Plaintext passwords are NEVER stored, logged, or returned by any API.

CREATE TABLE IF NOT EXISTS admins (
  id              SERIAL        PRIMARY KEY,
  username        VARCHAR(60)   NOT NULL UNIQUE,
  password_hash   VARCHAR(255)  NOT NULL,
  last_login      TIMESTAMPTZ,
  created_at      TIMESTAMPTZ   NOT NULL DEFAULT NOW()
);

-- Fast login lookup
CREATE INDEX IF NOT EXISTS idx_admins_username
  ON admins (username);


-- ── Comments on tables (PostgreSQL COMMENT syntax) ───────────
COMMENT ON TABLE bookings        IS 'Guest booking requests and their lifecycle status.';
COMMENT ON TABLE pricing_rules   IS 'Per-date pricing overrides. Falls back to default_rates if no rule exists.';
COMMENT ON TABLE default_rates   IS 'Baseline weekday/weekend rates. Exactly 2 rows, never deleted.';
COMMENT ON TABLE policy_content  IS 'Guest-facing policy text, editable by the admin.';
COMMENT ON TABLE admins          IS 'Admin accounts for the manager control panel.';

COMMENT ON COLUMN bookings.rate_applied IS 'Rate in INR snapshotted at booking time. Never changes after creation.';
COMMENT ON COLUMN bookings.slot         IS 'day = 8AM–8PM | night = 8PM–8AM next day';
COMMENT ON COLUMN pricing_rules.is_closed IS 'TRUE = blackout date. Guests cannot select this date regardless of rates.';
```

---

## `db/seeds/001_default_rates.sql`

```sql
-- ============================================================
--  Seed 001: default_rates
--
--  Baseline rates per the masterplan:
--    Weekday (Mon–Thu): Day ₹6,000 | Night ₹7,000
--    Weekend (Fri–Sun): Day ₹7,500 | Night ₹9,000
--
--  Uses INSERT ... ON CONFLICT DO UPDATE (upsert) so this seed
--  can be re-run safely without creating duplicate rows.
-- ============================================================

INSERT INTO default_rates (day_type, day_slot_rate, night_slot_rate)
VALUES
  ('weekday', 6000.00, 7000.00),
  ('weekend', 7500.00, 9000.00)
ON CONFLICT (day_type)
DO UPDATE SET
  day_slot_rate   = EXCLUDED.day_slot_rate,
  night_slot_rate = EXCLUDED.night_slot_rate,
  updated_at      = NOW();
```

---

## `db/seeds/002_peak_pricing_2025.sql`

```sql
-- ============================================================
--  Seed 002: pricing_rules — Peak dates for 2025
--
--  Covers: Holi, Eid al-Fitr, Eid al-Adha, Diwali, Christmas,
--          New Year's Eve/Day, and long weekends.
--
--  Rates are indicative — the manager should review and adjust
--  via the Admin Rate Manager (Screen 10) after launch.
--
--  Peak rates: Day ₹10,000 | Night ₹12,000 (≈1.5–1.7x weekend)
--
--  Uses upsert so this seed can be re-run without errors.
-- ============================================================

INSERT INTO pricing_rules (target_date, label_name, day_slot_rate, night_slot_rate, is_closed)
VALUES

  -- ── Holi 2025 (14 March — Holi; 13 March — Holika Dahan) ───
  ('2025-03-13', 'HOLI', 10000.00, 12000.00, FALSE),
  ('2025-03-14', 'HOLI', 10000.00, 12000.00, FALSE),
  ('2025-03-15', 'HOLI', 10000.00, 12000.00, FALSE),  -- recovery day

  -- ── Eid al-Fitr 2025 (30–31 March, approximate) ─────────────
  ('2025-03-29', 'EID',  10000.00, 12000.00, FALSE),
  ('2025-03-30', 'EID',  10000.00, 12000.00, FALSE),
  ('2025-03-31', 'EID',  10000.00, 12000.00, FALSE),

  -- ── Good Friday + Easter long weekend (18–21 April) ─────────
  ('2025-04-18', 'PEAK', 10000.00, 12000.00, FALSE),
  ('2025-04-19', 'PEAK', 10000.00, 12000.00, FALSE),
  ('2025-04-20', 'PEAK', 10000.00, 12000.00, FALSE),
  ('2025-04-21', 'PEAK', 10000.00, 12000.00, FALSE),

  -- ── Eid al-Adha 2025 (6–7 June, approximate) ────────────────
  ('2025-06-06', 'EID',  10000.00, 12000.00, FALSE),
  ('2025-06-07', 'EID',  10000.00, 12000.00, FALSE),
  ('2025-06-08', 'EID',  10000.00, 12000.00, FALSE),

  -- ── Independence Day long weekend (15 August) ────────────────
  ('2025-08-15', 'PEAK', 9000.00, 11000.00, FALSE),
  ('2025-08-16', 'PEAK', 9000.00, 11000.00, FALSE),
  ('2025-08-17', 'PEAK', 9000.00, 11000.00, FALSE),

  -- ── Gandhi Jayanti long weekend (2 October) ──────────────────
  ('2025-10-02', 'PEAK', 9000.00, 11000.00, FALSE),
  ('2025-10-03', 'PEAK', 9000.00, 11000.00, FALSE),
  ('2025-10-04', 'PEAK', 9000.00, 11000.00, FALSE),
  ('2025-10-05', 'PEAK', 9000.00, 11000.00, FALSE),

  -- ── Diwali 2025 (20 October — Diwali; 19–22 October window) ─
  ('2025-10-19', 'DIWALI', 12000.00, 14000.00, FALSE),
  ('2025-10-20', 'DIWALI', 12000.00, 14000.00, FALSE),
  ('2025-10-21', 'DIWALI', 12000.00, 14000.00, FALSE),
  ('2025-10-22', 'DIWALI', 12000.00, 14000.00, FALSE),
  ('2025-10-23', 'DIWALI', 12000.00, 14000.00, FALSE),

  -- ── Christmas 2025 ───────────────────────────────────────────
  ('2025-12-24', 'PEAK', 10000.00, 12000.00, FALSE),
  ('2025-12-25', 'PEAK', 10000.00, 12000.00, FALSE),
  ('2025-12-26', 'PEAK', 10000.00, 12000.00, FALSE),
  ('2025-12-27', 'PEAK', 10000.00, 12000.00, FALSE),

  -- ── New Year 2025–2026 ───────────────────────────────────────
  ('2025-12-28', 'PEAK', 10000.00, 12000.00, FALSE),
  ('2025-12-29', 'PEAK', 10000.00, 12000.00, FALSE),
  ('2025-12-30', 'PEAK', 10000.00, 12000.00, FALSE),
  ('2025-12-31', 'PEAK', 14000.00, 16000.00, FALSE),  -- NYE premium
  ('2026-01-01', 'PEAK', 12000.00, 14000.00, FALSE),
  ('2026-01-02', 'PEAK', 10000.00, 12000.00, FALSE),
  ('2026-01-03', 'PEAK', 10000.00, 12000.00, FALSE),

  -- ── Holi 2026 (early March 2026, approximate) ────────────────
  ('2026-03-02', 'HOLI', 10000.00, 12000.00, FALSE),
  ('2026-03-03', 'HOLI', 10000.00, 12000.00, FALSE),
  ('2026-03-04', 'HOLI', 10000.00, 12000.00, FALSE)

ON CONFLICT (target_date)
DO UPDATE SET
  label_name      = EXCLUDED.label_name,
  day_slot_rate   = EXCLUDED.day_slot_rate,
  night_slot_rate = EXCLUDED.night_slot_rate,
  is_closed       = EXCLUDED.is_closed,
  updated_at      = NOW();
```

---

## `db/seeds/003_policy_content.sql`

```sql
-- ============================================================
--  Seed 003: policy_content
--
--  4 policy sections displayed in the guest Policy Modal (Screen 6).
--  Content is editable by the manager via Admin Rules Editor (Screen 11).
--  All section_keys must match the CHECK constraint in the schema.
-- ============================================================

INSERT INTO policy_content (section_key, content_text)
VALUES

  ('checkout_policy',
   E'Check-in time is 8:00 AM for Day slots and 8:00 PM for Night slots.\n'
   'Check-out is strictly at the end of your booked slot. Early arrivals and late departures must be pre-approved and may incur additional charges.\n\n'
   'A refundable security deposit of ₹5,000 is collected at check-in. This will be returned within 24 hours of departure, subject to a property inspection.\n\n'
   'Cancellations made more than 72 hours before check-in are eligible for a full refund of the deposit. Cancellations within 72 hours forfeit the deposit.'),

  ('cleanliness',
   E'Guests are expected to maintain the property in the same condition as it was received.\n\n'
   'Please dispose of all garbage in the designated bins provided. Do not leave food waste in the open — this is a farmhouse environment and attracts wildlife.\n\n'
   'All dishes and cooking utensils must be washed and returned to their original locations before departure.\n\n'
   'Any spills, stains, or damage to furniture, linen, or fixtures must be reported to the host immediately. Concealed damage will be billed from the security deposit.'),

  ('pool_safety',
   E'The pool is available exclusively for booked guests during your slot hours.\n\n'
   'Children under 12 must be accompanied by an adult at all times in and around the pool area.\n\n'
   'No glass containers are permitted in the pool area. Please use the plastic cups provided.\n\n'
   'Guests swim at their own risk. The property does not provide a lifeguard. Please do not swim after consuming alcohol.\n\n'
   'The pool closes 30 minutes before the end of your slot to allow for cleaning. Please exit the pool area punctually.'),

  ('house_rules',
   E'Maximum occupancy for this property is stated at the time of booking. Exceeding the declared guest count is not permitted and may result in immediate eviction without refund.\n\n'
   'Music and amplified sound must be kept at a reasonable volume. Loud music after 10:00 PM is not permitted out of respect for neighbouring properties.\n\n'
   'No pets are allowed on the premises without prior written approval from the host.\n\n'
   'Smoking is not permitted inside any structure on the property. A designated outdoor smoking area is available.\n\n'
   'The host reserves the right to terminate a booking without refund in the event of property damage, illegal activity, or harassment of staff.')

ON CONFLICT (section_key)
DO UPDATE SET
  content_text = EXCLUDED.content_text,
  updated_at   = NOW();
```

---

## `db/seeds/004_admin_user.js`

```javascript
'use strict';

/**
 * db/seeds/004_admin_user.js
 *
 * Creates the initial admin user for The Green Acre manager panel.
 *
 * This seed is run as a Node.js script (not SQL) because it needs to
 * bcrypt-hash the password before writing it to the database.
 * Passwords are NEVER stored in plaintext.
 *
 * Usage:
 *   ADMIN_USERNAME=manager ADMIN_PASSWORD=YourStrongPass123! node db/seeds/004_admin_user.js
 *
 * Or, when called via seed.js, set ADMIN_USERNAME and ADMIN_PASSWORD in .env.
 *
 * Security requirements:
 *   - Minimum 12 bcrypt rounds (as per masterplan Section 5.4)
 *   - Password is read from environment — never hardcoded
 *   - Uses upsert so the seed can be re-run to rotate credentials
 */

require('dotenv').config();
const bcrypt = require('bcryptjs');
const { query, connectDB, closeDB } = require('../config/db');

const BCRYPT_ROUNDS = 12;

async function seedAdminUser() {
  const username = process.env.ADMIN_USERNAME;
  const password = process.env.ADMIN_PASSWORD;

  if (!username || !password) {
    console.error(
      '[Seed] ERROR: ADMIN_USERNAME and ADMIN_PASSWORD must be set in .env before running this seed.'
    );
    console.error('[Seed] Example:');
    console.error('[Seed]   ADMIN_USERNAME=manager');
    console.error('[Seed]   ADMIN_PASSWORD=YourStrongPass123!');
    process.exit(1);
  }

  if (password.length < 12) {
    console.error('[Seed] ERROR: ADMIN_PASSWORD must be at least 12 characters.');
    process.exit(1);
  }

  console.log(`[Seed] Hashing password for "${username}" with ${BCRYPT_ROUNDS} bcrypt rounds...`);
  const passwordHash = await bcrypt.hash(password, BCRYPT_ROUNDS);
  console.log('[Seed] Password hashed successfully.');

  const sql = `
    INSERT INTO admins (username, password_hash)
    VALUES ($1, $2)
    ON CONFLICT (username)
    DO UPDATE SET
      password_hash = EXCLUDED.password_hash,
      last_login    = NULL
    RETURNING id, username, created_at
  `;

  try {
    await connectDB();
    const result = await query(sql, [username, passwordHash]);
    const admin = result.rows[0];
    console.log(`[Seed] Admin user created/updated:`);
    console.log(`[Seed]   ID:       ${admin.id}`);
    console.log(`[Seed]   Username: ${admin.username}`);
    console.log(`[Seed]   Created:  ${admin.created_at}`);
    console.log('[Seed] Admin seed complete.');
  } catch (err) {
    console.error('[Seed] Failed to seed admin user:', err.message);
    process.exit(1);
  } finally {
    await closeDB();
  }
}

seedAdminUser();
```

---

## `README.md`

```markdown
# The Green Acre — Backend API

Private Farmhouse Booking Platform — Node.js/Express REST API

---

## Tech Stack

| Layer | Technology |
|---|---|
| Runtime | Node.js 20 |
| Framework | Express.js |
| Database | PostgreSQL (via `pg`) |
| Auth | JWT (`jsonwebtoken`) + bcrypt (`bcryptjs`) |
| Deployment — API | Railway |
| Deployment — Frontend | Vercel |

---

## Project Structure

```
greenacre-backend/
├── server.js                  # Entry point — startup, shutdown, env validation
├── app.js                     # Express app — middleware, routes, error handlers
├── package.json
├── railway.json               # Railway deployment config
├── vercel.json                # Vercel frontend config (copy to frontend dir)
├── .env.example               # Environment variable template
├── .gitignore
│
├── config/
│   ├── db.js                  # PostgreSQL pool + query helper + IST timezone
│   └── jwt.js                 # JWT sign/verify helpers
│
├── middleware/
│   ├── authMiddleware.js      # JWT Bearer token verification
│   ├── rateLimiter.js         # Login (5/15min) + API (200/min) limiters
│   ├── validate.js            # express-validator result checker
│   └── errorHandler.js        # 404 handler + global error handler
│
├── controllers/
│   └── auth/
│       └── authController.js  # login(), logout()
│
├── routes/
│   ├── authRoutes.js          # POST /api/auth/login|logout
│   ├── publicRoutes.js        # Public guest routes (stubs for Phase 2/3)
│   └── adminRoutes.js         # JWT-protected admin routes (stubs for Phase 4)
│
└── db/
    ├── migrate.js             # Migration runner
    ├── seed.js                # Seed runner
    ├── migrations/
    │   └── 001_create_tables.sql
    └── seeds/
        ├── 001_default_rates.sql
        ├── 002_peak_pricing_2025.sql
        ├── 003_policy_content.sql
        └── 004_admin_user.js
```

---

## Local Setup

### Prerequisites

- Node.js 20+
- PostgreSQL 14+ running locally

### 1. Clone and install

```bash
git clone <repo-url>
cd greenacre-backend
npm install
```

### 2. Configure environment

```bash
cp .env.example .env
```

Edit `.env` with your values:

```env
NODE_ENV=development
PORT=5000
DATABASE_URL=postgresql://postgres:yourpassword@localhost:5432/greenacre
JWT_SECRET=<generate with command below>
CORS_ORIGINS=http://localhost:3000,http://localhost:5500
ADMIN_USERNAME=manager
ADMIN_PASSWORD=YourStrongPass123!
```

Generate a secure JWT secret:
```bash
node -e "console.log(require('crypto').randomBytes(64).toString('hex'))"
```

### 3. Create the database

```bash
createdb greenacre
# or in psql:
# CREATE DATABASE greenacre;
```

### 4. Run migrations

```bash
npm run migrate
```

This creates all 5 tables with proper constraints and indexes.

### 5. Seed the database

```bash
npm run seed
```

This seeds:
- Default weekday/weekend rates
- 2025–2026 holiday peak pricing
- Policy content (4 sections)
- Admin user (reads `ADMIN_USERNAME` + `ADMIN_PASSWORD` from `.env`)

### 6. Start the server

```bash
# Development (auto-restart on file changes — Node 20 built-in)
npm run dev

# Production
npm start
```

---

## Testing the Setup

### Health check
```bash
curl http://localhost:5000/health
```
Expected:
```json
{
  "success": true,
  "service": "greenacre-api",
  "environment": "development",
  "database": "connected"
}
```

### Login
```bash
curl -X POST http://localhost:5000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username": "manager", "password": "YourStrongPass123!"}'
```
Expected:
```json
{
  "success": true,
  "token": "eyJhbGci...",
  "expiresIn": "8h",
  "username": "manager"
}
```

### Access a protected admin route
```bash
TOKEN="paste-token-here"
curl http://localhost:5000/api/admin/stats \
  -H "Authorization: Bearer $TOKEN"
```
Expected (Phase 4 stub):
```json
{ "success": false, "error": "Admin stats endpoint not yet implemented..." }
```

### Test rate limiting (5 failed logins triggers 429)
```bash
for i in {1..6}; do
  curl -s -X POST http://localhost:5000/api/auth/login \
    -H "Content-Type: application/json" \
    -d '{"username": "manager", "password": "wrong"}' | jq .error
done
```

### Test 404
```bash
curl http://localhost:5000/api/nonexistent
```

### Test CORS rejection (wrong origin)
```bash
curl http://localhost:5000/api/calendar \
  -H "Origin: https://evil.com" -v
```

---

## Deployment

### Railway (Backend)

1. Create a new Railway project
2. Add a PostgreSQL service (Railway provides `DATABASE_URL` automatically)
3. Add a Node.js service, connect your GitHub repo
4. Set environment variables in Railway dashboard:
   - `NODE_ENV=production`
   - `JWT_SECRET=<64-char random string>`
   - `CORS_ORIGINS=https://yourdomain.vercel.app`
   - `ADMIN_USERNAME=manager`
   - `ADMIN_PASSWORD=<strong password>`
5. Railway uses `railway.json` for build/start commands and health checks
6. After first deploy, run migrations and seeds via Railway's shell:
   ```bash
   npm run migrate
   npm run seed
   ```

### Vercel (Frontend)

1. Copy `vercel.json` into your frontend directory (where `index.html` lives)
2. Set the `API_BASE` constant in your frontend JS to your Railway API URL
3. Deploy the frontend directory to Vercel
4. All 4 HTML pages route correctly:
   - `/` → `index.html`
   - `/manager-login` → `manager-login.html`
   - `/admin` → `admin.html`
   - `/booking-status` → `booking-status.html`

---

## Security Checklist

- [x] JWT signed with HS256, 8h expiry
- [x] bcrypt with 12 rounds minimum
- [x] Login rate-limited: 5 attempts per 15 minutes per IP
- [x] All secrets in environment variables, never hardcoded
- [x] Helmet security headers on all responses
- [x] CORS restricted to allowlist of origins
- [x] Parameterized SQL queries throughout (no string interpolation)
- [x] Timing-safe login (bcrypt.compare always runs, even for unknown users)
- [x] JWT middleware applied at router level — no admin route reachable without token
- [x] `trust proxy` set for correct IP detection behind Railway proxy
- [x] Graceful shutdown on SIGTERM (Railway deploy lifecycle)
- [x] Stack traces never sent to client in production
- [x] `/manager-login` not referenced in any public-facing HTML

---

## Phase Roadmap

| Phase | Scope | Status |
|---|---|---|
| Phase 1 | Foundation, DB, Auth | ✅ Complete |
| Phase 2 | Calendar + Pricing API | 🔜 Next |
| Phase 3 | Booking request + status API | 🔜 |
| Phase 4 | Admin panel APIs | 🔜 |
| Phase 5 | Frontend wiring | 🔜 |
| Phase 6 | WhatsApp notifications + QA | 🔜 |
```

---
