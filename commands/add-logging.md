---
description: Add dev-only structured logging to any project. Scans tech stack, creates a branch, instruments frontend/backend/DB layers — logs visible in dev, disabled in production.
argument-hint: [scope — e.g. "full project", "frontend only", "backend only", "API layer"]
allowed-tools: Bash, Read, Write, Glob, Grep, Edit, Task
---

# Add Dev-Only Logging for QA Testing

You are instrumenting a project with structured, dev-only logging that helps QA catch and reproduce issues from logs. Logs are visible during development and **completely disabled in production**.

## Context

- Project: !`basename $(pwd)`
- Branch: !`git branch --show-current 2>/dev/null`
- Package manager: !`[ -f pnpm-lock.yaml ] && echo "pnpm" || ([ -f yarn.lock ] && echo "yarn" || ([ -f bun.lockb ] && echo "bun" || echo "npm"))`
- Node version: !`node -v 2>/dev/null || echo "not detected"`
- Git user: !`git config user.name 2>/dev/null || echo "unknown"`

## User Request

$ARGUMENTS

## Phase 0: Identify Git User

Before creating a branch, determine the username for branch prefix:

1. Check `git config user.name` — if available, convert to lowercase kebab-case (e.g. "Deepak Tiwari" → "deepak-tiwari")
2. Check `git config user.email` — extract username before @ (e.g. "deepak@company.com" → "deepak")
3. If neither found or unclear, **ask the user**: "What username should I prefix the branch with? (e.g. `yourname/logs-for-testing`)"

Branch name: `<username>/logs-for-testing`

## Phase 1: Project Analysis

Before writing any code, understand what you're working with. Run these scans:

### 1A: Detect Tech Stack

```
Frontend framework:
  - Check package.json for: react, next, vue, nuxt, angular, svelte, solid
  - Check for: vite.config, next.config, angular.json, svelte.config

Backend framework:
  - Check for: express, fastify, nestjs, koa, hono (Node.js)
  - Check for: go.mod (Go), requirements.txt/pyproject.toml (Python), Cargo.toml (Rust)
  - Check for: main.go, app.py, main.rs

Database:
  - Check for ORM: prisma, drizzle, typeorm, sequelize, mongoose, gorm
  - Check for DB clients: pg, mysql2, mongodb, redis
  - Check .env for DATABASE_URL, MONGO_URI, REDIS_URL

State management:
  - Check for: zustand, redux, pinia, jotai, recoil
```

### 1B: Check Existing Logging

```
Scan for existing logging patterns:
  - console.log / console.debug / console.error usage
  - Logger imports (winston, pino, bunyan, morgan, zap, logrus, structlog)
  - Custom logger utils (src/lib/logger, src/utils/logger)
  - Log level env variables (LOG_LEVEL, DEBUG, VERBOSE)
```

### 1C: Map Critical Paths

Identify where logging is most valuable:
```
Frontend:
  - API call layer (axios/fetch interceptors or service files)
  - Auth state changes (login, logout, token refresh)
  - Route transitions
  - Error boundaries
  - Form submissions on critical flows
  - State store mutations (if zustand/redux)

Backend:
  - Request/response middleware (incoming HTTP)
  - Database query layer (ORM hooks or query wrappers)
  - Business logic decision points (if/else that changes outcomes)
  - External API calls (third-party services)
  - Auth middleware (token validation, role checks)
  - Error handlers (catch blocks, error middleware)

Database:
  - Slow query threshold config
  - Migration logging
  - Connection pool events
```

### 1D: Present Analysis to User

Before creating the branch, show:

```
-----------------------------------------
Project Analysis for Logging
-----------------------------------------

Tech Stack Detected:
- Frontend: [framework] + [bundler]
- Backend: [framework/language]
- Database: [type] via [ORM]
- State: [state manager]

Existing Logging:
- [N] console.log statements found
- [N] console.error statements found
- Logger library: [none / name]
- Log level env: [none / variable name]

Logging Plan:
1. [Layer] — [what will be instrumented] — [N files]
2. [Layer] — [what will be instrumented] — [N files]
3. [Layer] — [what will be instrumented] — [N files]

Branch: <username>/logs-for-testing
Total files to touch: [N]
New files to create: [N]
-----------------------------------------
```

Ask the user: "Proceed with this plan? Anything to skip or add?"

## Phase 2: Create Branch

```bash
git checkout -b <username>/logs-for-testing
```

If the branch already exists, ask the user whether to reuse or create a fresh one.

## Phase 3: Implement Logging (By Layer)

Work step-by-step. **Commit after each layer** as a checkpoint.

---

### 3A: Create Logger Utility (Always First)

Create a dev-only logger that wraps console methods and is silenced in production.

#### For React / Vite Projects

Create `src/lib/logger.ts`:

```typescript
/**
 * Dev-only structured logger — silenced in production builds.
 * Usage: logger.api('GET /patients', { status: 200, ms: 42 })
 */

const isDev = import.meta.env.DEV;

type LogData = Record<string, unknown>;

const noop = (..._args: unknown[]) => {};

const createLogger = (namespace: string, color: string) => {
  if (!isDev) return noop;
  return (message: string, data?: LogData) => {
    console.debug(
      `%c[${namespace}]%c ${message}`,
      `color: ${color}; font-weight: bold`,
      'color: inherit',
      data ?? ''
    );
  };
};

export const logger = {
  /** API calls — request/response/error */
  api: createLogger('API', '#3b82f6'),
  /** Auth state changes — login/logout/refresh */
  auth: createLogger('AUTH', '#f59e0b'),
  /** Router — navigation/redirects */
  router: createLogger('ROUTER', '#8b5cf6'),
  /** Store — state mutations */
  store: createLogger('STORE', '#10b981'),
  /** UI — user actions on critical flows */
  ui: createLogger('UI', '#ec4899'),
  /** Error — caught errors */
  error: createLogger('ERROR', '#ef4444'),
};
```

#### For Next.js Projects

Same as above but use `process.env.NODE_ENV !== 'production'` check. Also create a server-side logger:

```typescript
// src/lib/server-logger.ts (server components / API routes)
const isDev = process.env.NODE_ENV !== 'production';

export const serverLogger = {
  request: (method: string, path: string, data?: Record<string, unknown>) => {
    if (!isDev) return;
    console.log(`[REQ] ${method} ${path}`, data ?? '');
  },
  db: (query: string, ms: number, data?: Record<string, unknown>) => {
    if (!isDev) return;
    const slow = ms > 100 ? ' [SLOW]' : '';
    console.log(`[DB${slow}] ${query} (${ms}ms)`, data ?? '');
  },
  error: (context: string, error: unknown) => {
    if (!isDev) return;
    console.error(`[ERR] ${context}`, error);
  },
};
```

#### For Go (Backend)

Use `log/slog` (stdlib, Go 1.21+):

```go
// internal/logger/logger.go
package logger

import (
    "log/slog"
    "os"
)

var Log *slog.Logger

func Init() {
    env := os.Getenv("APP_ENV")
    if env == "production" {
        Log = slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{Level: slog.LevelWarn}))
    } else {
        Log = slog.New(slog.NewTextHandler(os.Stdout, &slog.HandlerOptions{Level: slog.LevelDebug}))
    }
}
```

#### For Node.js Backend (Express/Fastify/NestJS)

Use `pino` (fast, JSON structured):

```typescript
// src/lib/logger.ts
import pino from 'pino';

export const logger = pino({
  level: process.env.LOG_LEVEL || (process.env.NODE_ENV === 'production' ? 'warn' : 'debug'),
  transport: process.env.NODE_ENV !== 'production'
    ? { target: 'pino-pretty', options: { colorize: true } }
    : undefined,
});
```

#### For Python Backend (FastAPI/Django/Flask)

Use `structlog`:

```python
# app/logger.py
import os, structlog

structlog.configure(
    processors=[
        structlog.stdlib.add_log_level,
        structlog.dev.ConsoleRenderer() if os.getenv("APP_ENV") != "production"
        else structlog.processors.JSONRenderer(),
    ],
    wrapper_class=structlog.stdlib.BoundLogger,
)

logger = structlog.get_logger()
```

**Commit checkpoint:** `feat: add dev-only logger utility`

---

### 3B: Instrument API Layer (Frontend)

Find the API service layer and add request/response logging.

#### Axios Interceptor Pattern

If project uses axios, add interceptors:

```typescript
// In the axios instance file (e.g., src/lib/axios.ts or src/api/client.ts)
import { logger } from '@/lib/logger';

api.interceptors.request.use((config) => {
  (config as any).__startTime = Date.now();
  logger.api(`→ ${config.method?.toUpperCase()} ${config.url}`, {
    params: config.params,
  });
  return config;
});

api.interceptors.response.use(
  (response) => {
    const ms = Date.now() - ((response.config as any).__startTime || 0);
    logger.api(`← ${response.status} ${response.config.url} (${ms}ms)`, {
      dataKeys: response.data ? Object.keys(response.data) : [],
    });
    return response;
  },
  (error) => {
    const ms = Date.now() - ((error.config as any)?.__startTime || 0);
    logger.error(`← ${error.response?.status || 'NETWORK'} ${error.config?.url} (${ms}ms)`, {
      message: error.message,
      data: error.response?.data,
    });
    return Promise.reject(error);
  }
);
```

#### Fetch Wrapper Pattern

If project uses fetch, wrap it:

```typescript
// src/lib/api.ts
import { logger } from '@/lib/logger';

export async function apiFetch<T>(url: string, options?: RequestInit): Promise<T> {
  const start = Date.now();
  logger.api(`→ ${options?.method || 'GET'} ${url}`);

  const response = await fetch(url, options);
  const ms = Date.now() - start;

  if (!response.ok) {
    logger.error(`← ${response.status} ${url} (${ms}ms)`);
    throw new Error(`API error: ${response.status}`);
  }

  logger.api(`← ${response.status} ${url} (${ms}ms)`);
  return response.json();
}
```

**Commit checkpoint:** `feat: add API request/response logging`

---

### 3C: Instrument Auth Layer

Find auth hooks/stores and log state changes:

```typescript
// In auth store or hook (e.g., useAuth, authStore)
import { logger } from '@/lib/logger';

// On login success:
logger.auth('Login successful', { userId: user.id, role: user.role });

// On logout:
logger.auth('User logged out');

// On token refresh:
logger.auth('Token refreshed', { expiresIn: decoded.exp });

// On auth error:
logger.error('Auth failed', { status: error.status, message: error.message });
```

**Commit checkpoint:** `feat: add auth state change logging`

---

### 3D: Instrument State Store (If Applicable)

#### Zustand

Use Zustand's middleware pattern:

```typescript
import { logger } from '@/lib/logger';

// Add to any store that has business-critical state
const logMiddleware = (config) => (set, get, api) =>
  config(
    (...args) => {
      const prev = get();
      set(...args);
      const next = get();
      logger.store('State updated', {
        changed: Object.keys(next).filter((k) => prev[k] !== next[k]),
      });
    },
    get,
    api
  );
```

#### Redux

Use Redux middleware:

```typescript
const loggerMiddleware = (store) => (next) => (action) => {
  logger.store(`Action: ${action.type}`, { payload: action.payload });
  return next(action);
};
```

**Commit checkpoint:** `feat: add state store logging`

---

### 3E: Instrument Backend Request Pipeline

#### Express Middleware

```typescript
import { logger } from './lib/logger';

app.use((req, res, next) => {
  const start = Date.now();
  res.on('finish', () => {
    const ms = Date.now() - start;
    logger.info({
      method: req.method,
      path: req.path,
      status: res.statusCode,
      ms,
      ...(ms > 500 && { slow: true }),
    });
  });
  next();
});
```

#### Go (net/http or Gin)

```go
func LoggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        next.ServeHTTP(w, r)
        logger.Log.Info("request",
            slog.String("method", r.Method),
            slog.String("path", r.URL.Path),
            slog.Duration("duration", time.Since(start)),
        )
    })
}
```

**Commit checkpoint:** `feat: add backend request logging middleware`

---

### 3F: Instrument Database Layer

#### Prisma (Query Logging)

```typescript
// In prisma client setup
const prisma = new PrismaClient({
  log: process.env.NODE_ENV !== 'production'
    ? [
        { emit: 'event', level: 'query' },
        { emit: 'stdout', level: 'warn' },
        { emit: 'stdout', level: 'error' },
      ]
    : ['warn', 'error'],
});

if (process.env.NODE_ENV !== 'production') {
  prisma.$on('query', (e) => {
    const slow = e.duration > 100 ? ' [SLOW]' : '';
    console.debug(`[DB${slow}] ${e.query} (${e.duration}ms)`);
  });
}
```

#### Drizzle (Query Logging)

```typescript
import { drizzle } from 'drizzle-orm/...';

const db = drizzle(client, {
  logger: process.env.NODE_ENV !== 'production',
});
```

#### GORM (Go)

```go
if os.Getenv("APP_ENV") != "production" {
    db = db.Debug() // Logs all SQL queries
}
```

#### Mongoose (MongoDB)

```typescript
if (process.env.NODE_ENV !== 'production') {
  mongoose.set('debug', (collectionName, method, query, doc) => {
    console.debug(`[DB] ${collectionName}.${method}`, JSON.stringify(query));
  });
}
```

**Commit checkpoint:** `feat: add database query logging`

---

### 3G: Instrument Error Boundaries (Frontend)

If React, add logging to error boundary:

```typescript
import { logger } from '@/lib/logger';

// In error boundary componentDidCatch or ErrorBoundary component:
logger.error('React error boundary caught', {
  error: error.message,
  component: errorInfo.componentStack?.split('\n')[1]?.trim(),
});
```

**Commit checkpoint:** `feat: add error boundary logging`

---

### 3H: Add Environment Variable

Add `LOG_LEVEL` to `.env.example` / `.env.development`:

```
# Logging — only active in development
LOG_LEVEL=debug
# Options: debug | info | warn | error | silent
```

**Commit checkpoint:** `feat: add LOG_LEVEL env variable`

---

## Phase 4: Verification

After all instrumentation:

1. **Build check** — run the project's build command, must pass with zero errors
2. **Lint check** — run linter, must pass
3. **Dev server test** — start dev server, trigger a few flows, verify logs appear in console
4. **Production build test** — build for production, verify NO logs appear (tree-shaken or silenced)

Show results:

```
-----------------------------------------
Logging Implementation Complete
-----------------------------------------

Logger utility: src/lib/logger.ts
Branch: <username>/logs-for-testing

Instrumented:
- [x] Logger utility created
- [x] API layer — [N] interceptors/wrappers
- [x] Auth — [N] state change points
- [x] State store — [store name]
- [x] Backend middleware — request pipeline
- [x] Database — query logging
- [x] Error boundary — React errors
- [x] ENV variable — LOG_LEVEL

Verification:
- Build: PASS
- Lint: PASS
- Dev logs visible: YES
- Prod logs silenced: YES

Commits: [N] checkpoint commits
-----------------------------------------
```

## Phase 5: Offer Next Steps

After implementation, ask:

1. "Create a PR for this branch?"
2. "Want me to run `/qa` to test the logging output?"
3. "Should I add any custom log points for specific business flows?"

## Rules

1. **Dev-only** — every log MUST be gated by env check. Zero logs in production.
2. **No secrets** — never log tokens, passwords, API keys, PII (emails, phone numbers)
3. **Structured** — use namespaced loggers, not raw `console.log`
4. **Non-breaking** — logging must not change any existing behavior or return values
5. **Step-by-step commits** — commit after each layer for easy rollback
6. **Detect, don't assume** — always scan the project first, adapt to actual tech stack
7. **Skip what exists** — if a layer already has good logging, note it and move on
8. **Keep it light** — don't over-log. Focus on: API calls, auth, DB queries, errors, critical business logic
9. **Color-coded** — frontend browser logs should use colors for easy scanning
10. **Performance** — use `console.debug` not `console.log` (filterable in DevTools), use lazy evaluation for expensive data
11. **Generic branch prefix** — use the git user's name or ask, never hardcode a specific username
