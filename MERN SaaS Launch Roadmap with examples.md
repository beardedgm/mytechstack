# MERN SaaS Launch Roadmap

> **A repeatable blueprint for shipping subscription-based web apps as a solo developer.**

---

## Technology Assignments

### Core Stack

| Layer | Choice | Fit | Alternative Suggestion |
| ----- | ------ | --- | ---------------------- |
| Runtime | Node.js + Express.js | **Best** — lightweight API server built for I/O-heavy workloads (auth, CRUD, Stripe webhooks, Resend calls). Single language across frontend and backend eliminates context-switching for a solo dev. Express is unopinionated enough to port your existing session-based auth pattern directly without fighting framework conventions. | — |
| Database | MongoDB Atlas | **Best** — SaaS user data is document-shaped (user profiles, subscription records, token collections). Schema flexibility lets you iterate fast without migrations. Atlas handles backups, scaling, monitoring, and TTL indexes for auto-cleanup of sessions, login attempts, and email tokens — all critical to your auth pattern. | — |
| ODM | Mongoose | **Best** — schema validation at the application layer catches bad data before it hits the database. Middleware hooks (pre-save password hashing, pre-remove cleanup) centralize logic. Model definitions become reusable across every app — User, Subscription, EmailToken, LoginAttempt are always the same shape. | — |
| Frontend | React + Vite | **Best** — React handles the SPA architecture you need for dashboard-style subscription apps (auth state, protected routes, modals, forms). Vite gives instant dev server startup, sub-second hot reload, and optimized production builds via Rollup. | — |
| Styling | Custom CSS (Tailwind when needed) | **Best** — no framework overhead by default. You write what you need, nothing more. Keeps bundle size minimal and avoids locking every project into a utility-class workflow. Bring in Tailwind only when a project benefits from its structure (rapid prototyping, design-system-heavy UIs). | Tailwind CSS — **Strong** — useful when you need to move fast on UI-heavy apps. Add it per-project, not as a default. |
| Client Routing | React Router | **Best** — industry standard for React SPAs. Supports the nested route pattern you need: public routes (landing, login, pricing), auth-protected routes (dashboard, settings), and subscription-gated routes (premium features). Outlet-based layouts keep the component tree clean. | — |
| Server State | TanStack Query (React Query) | **Best** — eliminates the repetitive useEffect + useState fetch boilerplate that otherwise appears in every component. Built-in caching, automatic background refetching, loading/error states, and mutation invalidation. Handles the auth check on app load and subscription status polling cleanly. | — |
| Client State | Zustand | **Best** — covers the thin layer of global state that TanStack Query doesn't: UI state (sidebar, modals, toasts) and cached auth flags for instant access. 1KB, no boilerplate, works with React without providers or context wrappers. | — |
| Input Validation | Zod | **Best** — lightweight, TypeScript-native schema validation. Define the shape of request body, params, and query strings once, reuse across routes. Key advantage: validation schemas can be shared between Express backend and React frontend, so you define your rules once. Strips unknown fields by default (critical for NoSQL injection prevention). | — |
| Logging | pino | **Best** — structured JSON logging built for Node.js. Fastest logger in the ecosystem (low overhead in production). Structured output means logs are searchable and filterable. pino-pretty for readable dev output, raw JSON for production. | — |

### Auth & Security

| Layer | Choice | Fit | Alternative Suggestion |
| ----- | ------ | --- | ---------------------- |
| Authentication | Session-based (express-session + connect-mongo) | **Best** — server-side sessions with HTTP-only cookies are inherently more secure than JWT for browser-based SPAs. No token stored in localStorage (XSS-proof), no client-side expiration logic, and session revocation is instant (delete from MongoDB). Your entire auth pattern depends on this: password change invalidates all sessions, login lockouts persist across restarts, session regeneration prevents fixation attacks. JWT would require workarounds for every one of those features. connect-mongo keeps sessions in your existing Atlas cluster with no additional dependencies. Running a single Express service means you don't need stateless auth across multiple servers. | — |
| Password Hashing | Node.js crypto.scrypt | **Best** — built into Node, zero external dependencies. Memory-hard hashing function recommended by OWASP. Stronger brute-force resistance than bcrypt because it's tunable on both CPU and memory cost. No native compilation issues since it's part of the Node runtime. | — |
| CSRF Protection | X-header + constant-time comparison | **Best** — double-submit pattern via custom header avoids the complexity of CSRF token middleware libraries while providing equivalent protection. Constant-time comparison prevents timing attacks against the token. | — |
| Rate Limiting | MongoDB sliding window | **Best** — single rate limiting system for everything: general API throttling, login attempt tracking, and account lockouts. Uses your existing Atlas infrastructure with no additional dependencies. Sliding window is more accurate than fixed window. TTL indexes auto-clean expired records. Persistent storage means counters survive server restarts — critical for lockout security. Thresholds are tuned per route: 10 login attempts per 15 min, 5 registrations per hour, 3 password resets per hour, 100 general API calls per minute. | — |
| Bot Protection | Cloudflare Turnstile | **Best** — invisible CAPTCHA alternative that doesn't degrade UX. Free tier is generous. Server-side verification is a single POST to Cloudflare. Applied to all public-facing forms: registration, login, password reset, and any app-specific submission forms. | — |
| Security Headers | Helmet.js | **Best** — single middleware call sets all recommended HTTP security headers (X-Frame-Options, X-Content-Type-Options, Strict-Transport-Security, CSP, etc.). Zero configuration for the defaults, customizable when needed. | — |
| CORS | cors (Express middleware) | **Best** — even with a single-service deployment (Express serves the SPA), CORS is a defense-in-depth layer that restricts which origins can call your API. Simple configuration, keeps your security posture tight if you ever split services later. | — |
| Admin Role | Database role column | **Best** — `role` field on the User model with values `user` (default) and `owner`. Industry standard RBAC pattern used by every major framework. No redeploy to transfer ownership — update one document. Queryable, auditable, and scales naturally if you ever add a third role. The owner authenticates through the normal login flow like every other user. | — |

### Payments & Email

| Layer | Choice | Fit | Alternative Suggestion |
| ----- | ------ | --- | ---------------------- |
| Billing | Stripe Checkout (hosted) + Webhooks | **Best** — hosted checkout pushes PCI compliance entirely to Stripe. You never touch card numbers. Webhook-driven subscription lifecycle is Stripe's recommended pattern and keeps your billing state in sync without polling. Customer Portal gives users self-service subscription management for free. | — |
| Transactional Email | Resend + HTML template literals | **Best** — Resend has the cleanest developer experience of any email API (single function call to send). Plain HTML template literals keep the stack lean with zero extra build tooling. Sufficient for transactional emails (verify, reset, receipt). Covers verification, password reset, payment receipts, and onboarding drip sequences. | React Email — **Strong** — add per-project if you need complex branded email templates with component reuse. Adds build dependencies. |

### Infrastructure & Monitoring

| Layer | Choice | Fit | Alternative Suggestion |
| ----- | ------ | --- | ---------------------- |
| Hosting | Render (single service) | **Best** — Express serves the built React SPA and the API from one web service. One URL, one deployment, one bill. Blueprint-as-code (render.yaml) makes deployment reproducible across apps. Auto-deploy from GitHub on push to main. Free SSL via Let's Encrypt. No infrastructure management overhead for a solo dev. Split into two services only if you need independent scaling or a CDN for static assets. | — |
| Env Management | dotenv | **Best** — standard pattern for loading environment variables from .env files in development. Production uses Render's built-in env var management. The .env.example file in your template repo documents every required variable for each new app. | — |
| Error Tracking | Sentry | **Best** — captures runtime errors in both frontend and backend with full stack traces, breadcrumbs, and user context. Alerts you to issues before users report them. The free tier covers a solo dev's volume easily. Performance monitoring at 10% sample rate gives you baseline metrics without cost overhead. | — |
| Analytics | PostHog | **Best** — product analytics that goes far beyond traffic tracking. Feature usage, retention cohorts, session replays, churn funnels, and custom events. Free tier (1M events/month) is generous for a solo dev. Tells you *how* users behave inside your app, not just *that* they visited. Self-hosted option available if you want full data ownership. | — |
| CI/CD | GitHub Actions | **Best** — runs tests on every PR, verifies the production build succeeds, and gates merges to main. Combined with Render's auto-deploy on push, this gives you a complete pipeline with zero infrastructure to manage. Free tier covers the volume of a solo dev easily. | — |
| Version Control | GitHub (template repo) | **Best** — the template repo feature is purpose-built for your workflow. "Use this template" creates a fresh repo with full commit history isolation. Every new app starts from the same proven boilerplate without inheriting git history from previous projects. | — |

### Development Tools

| Layer | Choice | Fit | Alternative Suggestion |
| ----- | ------ | --- | ---------------------- |
| IDE | Visual Studio Code | **Best** — dominant editor for web development. Extensions for ESLint, Prettier, Tailwind IntelliSense (when needed), MongoDB, and GitLens. Integrated terminal for running dev servers. Free. | — |
| AI Assistant | Claude Code | **Best** — runs in your terminal alongside VS Code. Understands your full codebase context, generates and edits files directly, and can run tests and lint. Purpose-built for the agentic coding workflow a solo dev needs — scaffold a new feature, write tests, debug, and commit without leaving the terminal. | — |

### Edge Case Tools (per-app as needed)

| Layer | Choice | Fit | Alternative Suggestion |
| ----- | ------ | --- | ---------------------- |
| Canvas / Drawing | Raw Canvas 2D API | **Best** — zero dependencies, full pixel-level control, no abstraction overhead. The `useCanvas` hook in your template handles DPI scaling and cleanup. Reach for this whenever an app needs drawing, annotations, image manipulation, or visual editing. | — |
| Large File Storage | MongoDB GridFS | **Strong** — stores files larger than MongoDB's 16MB document limit directly in your existing Atlas cluster. No second service to manage, no separate billing, no CORS configuration. Combined with multer for uploads. | AWS S3 or Cloudflare R2 — **Best** — purpose-built for object storage. Better performance for high-volume file serving and files over 100MB. Switch when file storage becomes a core feature rather than a supporting one. |
| Markdown Processing | marked | **Best** — Node-native Markdown parser that stays in your JS-only stack. Fast, extensible, and well-maintained. Always paired with DOMPurify to sanitize rendered HTML. | — |
| Audio Encoding | ffmpeg (system binary) | **Best** — the industry standard for audio/video processing. Pre-installed on Render's build images, so no installation step. Wrapped in a promise-based service layer for clean async usage in Express. For files over 50MB, combine with a background job queue (BullMQ) instead of processing inline. | — |
| Extended Bot Protection | Cloudflare Turnstile | **Best** — same tool as the core stack, extended to additional surfaces. Reusable `TurnstileWidget.jsx` component drops into any form that faces the public internet: contact forms, waitlist signups, user-generated content submissions, file upload endpoints. | — |

---

## Phase 0: GitHub Template Repo

Before building any app, you maintain a single **template repository** that contains the entire boilerplate. Every new project starts with "Use this template" on GitHub.

### Repo Structure

```
saas-boilerplate/
├── client/                     # React + Vite frontend
│   ├── public/
│   │   └── robots.txt
│   ├── src/
│   │   ├── api/                # TanStack Query hooks & Axios instance
│   │   │   ├── axiosInstance.js
│   │   │   ├── useAuth.js
│   │   │   └── useSubscription.js
│   │   ├── components/
│   │   │   ├── auth/           # Login, Register, ForgotPassword, ResetPassword
│   │   │   ├── layout/         # Navbar, Footer, Sidebar, ProtectedRoute, OwnerRoute
│   │   │   ├── payments/       # PricingTable, SubscriptionStatus
│   │   │   └── ui/             # Shared buttons, modals, inputs
│   │   ├── pages/
│   │   │   ├── Landing.jsx
│   │   │   ├── Login.jsx
│   │   │   ├── Register.jsx
│   │   │   ├── Dashboard.jsx
│   │   │   ├── Settings.jsx
│   │   │   ├── Pricing.jsx
│   │   │   ├── NotFound.jsx        # 404 page
│   │   │   └── admin/          # AdminDashboard, AdminUsers (owner-only)
│   │   ├── store/              # Zustand stores
│   │   │   └── useAuthStore.js
│   │   ├── utils/
│   │   │   └── turnstile.js
│   │   ├── hooks/
│   │   │   └── useCanvas.js    # Raw Canvas 2D hook (delete if not needed)
│   │   ├── App.jsx
│   │   ├── main.jsx
│   │   └── index.css           # Custom CSS (add Tailwind config if needed per-project)
│   ├── index.html
│   ├── vite.config.js
│   └── package.json
│
├── server/                     # Express.js backend
│   ├── config/
│   │   ├── db.js               # Mongoose connection
│   │   ├── session.js          # express-session config
│   │   ├── logger.js           # Pino logger setup
│   │   ├── stripe.js           # Stripe client init
│   │   ├── resend.js           # Resend client init
│   │   ├── turnstile.js        # Turnstile verification helper
│   │   └── gridfs.js           # GridFS bucket init (delete if not needed)
│   ├── models/
│   │   ├── User.js
│   │   ├── LoginAttempt.js     # TTL-indexed lockout tracking
│   │   ├── EmailToken.js       # TTL-indexed verification/reset tokens
│   │   ├── ProcessedEvent.js   # TTL-indexed Stripe webhook deduplication
│   │   └── Session.js          # connect-mongo session store
│   ├── validators/
│   │   ├── auth.js             # Zod schemas: register, login, reset
│   │   ├── user.js             # Zod schemas: profile update, delete account
│   │   └── billing.js          # Zod schemas: checkout session creation
│   ├── middleware/
│   │   ├── validate.js         # Zod validation middleware
│   │   ├── errorHandler.js     # Centralized error handler
│   │   ├── requestLogger.js    # Pino HTTP request logging
│   │   ├── requireAuth.js      # Session-based auth check
│   │   ├── requireOwner.js     # Owner role check (user.role === 'owner')
│   │   ├── requireSubscription.js  # Stripe subscription gate
│   │   ├── csrfProtection.js   # X-header CSRF with constant-time comparison
│   │   ├── rateLimiter.js      # MongoDB sliding window rate limiter
│   │   └── validateTurnstile.js    # Turnstile server-side verification
│   ├── routes/
│   │   ├── auth.js             # Register, login, logout, verify, reset
│   │   ├── user.js             # Profile, settings, password change, export, delete
│   │   ├── admin.js            # Owner-only admin endpoints
│   │   ├── billing.js          # Stripe checkout session, portal, webhooks
│   │   ├── email-webhooks.js   # Resend bounce/complaint webhook handler
│   │   ├── files.js            # GridFS upload/download (delete if not needed)
│   │   ├── sitemap.js          # Dynamic sitemap.xml generation
│   │   └── health.js           # Health check endpoint for Render
│   ├── services/
│   │   ├── emailService.js     # Resend send helpers (checks bounce/suppress flags)
│   │   ├── stripeService.js    # Stripe business logic
│   │   └── audioService.js     # ffmpeg wrappers (delete if not needed)
│   ├── emails/                 # HTML email template functions
│   │   ├── verifyEmail.js
│   │   ├── resetPassword.js
│   │   ├── paymentReceipt.js
│   │   └── welcomeOnboard.js
│   ├── utils/
│   │   ├── asyncHandler.js     # Async route error wrapper
│   │   ├── timingSafeCompare.js
│   │   ├── generateToken.js
│   │   └── markdown.js         # marked + DOMPurify (delete if not needed)
│   ├── __tests__/              # Jest test suites
│   │   ├── auth.test.js
│   │   ├── validation/
│   │   │   └── auth.test.js    # Input validation edge cases
│   │   ├── webhooks.test.js    # Stripe webhook idempotency tests
│   │   └── middleware.test.js
│   ├── app.js                  # Express app setup + graceful shutdown
│   └── package.json
│
├── scripts/
│   └── promote-owner.js        # One-time CLI script to assign owner role
├── .github/
│   ├── workflows/
│   │   └── ci.yml              # CI pipeline (hard deploy gate)
│   └── dependabot.yml          # Automated dependency updates
├── .env.example                # Template environment variables
├── .gitignore
├── render.yaml                 # Render Blueprint (IaC)
└── README.md
```

### Template Repo Workflow

1. **Click "Use this template"** on GitHub to create a new repo.
2. **Clone it locally**, run `npm install` in both `/client` and `/server`.
3. **Copy `.env.example` to `.env`** and fill in your Atlas, Stripe, Resend, and Turnstile keys.
4. **Rename the app** — update `package.json` name fields, Render service names, and Stripe product references.
5. **Strip the placeholder pages** you don't need. Keep all the auth, billing, and security scaffolding.
6. **Start building your app-specific features.**

### README.md Template

Every app gets a README. The template repo includes a pre-structured one — update the placeholders per project.

```markdown
# [App Name]

[One-line description of what this app does.]

## Tech Stack

- **Frontend:** React + Vite
- **Backend:** Express.js + Node.js
- **Database:** MongoDB Atlas + Mongoose
- **Auth:** Session-based (express-session + connect-mongo)
- **Payments:** Stripe Checkout + Customer Portal
- **Email:** Resend + HTML template literals
- **Bot Protection:** Cloudflare Turnstile
- **Hosting:** Render (single service)

## Getting Started

### Prerequisites

- Node.js 20+
- MongoDB Atlas account
- Stripe account
- Resend account
- Cloudflare account (Turnstile)

### Setup

1. Clone the repo:
   ```bash
   git clone https://github.com/[your-username]/[repo-name].git
   cd [repo-name]
   ```

2. Install dependencies:
   ```bash
   npm install
   cd client && npm install && cd ..
   ```

3. Copy environment variables:
   ```bash
   cp .env.example .env
   ```

4. Fill in `.env` with your keys (see `.env.example` for all required values).

5. Start the dev server:
   ```bash
   npm run dev
   ```

## Scripts

| Command | Description |
| ------- | ----------- |
| `npm run dev` | Start Express + Vite dev servers |
| `npm test` | Run backend tests (Jest) |
| `cd client && npm test` | Run frontend tests (Vitest) |
| `cd client && npm run build` | Build React for production |
| `npm start` | Start production server |

## Project Structure

```
client/          # React + Vite frontend
server/          # Express.js backend
  config/        # DB, session, Stripe, Resend, logger configs
  models/        # Mongoose schemas
  validators/    # Zod validation schemas
  middleware/    # Auth, CSRF, rate limiting, error handling
  routes/        # API endpoints
  services/      # Business logic (email, Stripe)
  emails/        # HTML email template functions
  __tests__/     # Test suites
```

## Deployment

Deployed on Render via `render.yaml` Blueprint. Push to `main` triggers auto-deploy.

## License

[Choose your license]
```

### .gitignore

```gitignore
# Dependencies
node_modules/
client/node_modules/

# Environment
.env
.env.local
.env.production

# Build output
client/dist/

# OS files
.DS_Store
Thumbs.db

# IDE
.vscode/
!.vscode/settings.json
!.vscode/extensions.json
*.swp
*.swo

# Logs
logs/
*.log
npm-debug.log*

# Testing
coverage/
client/coverage/

# MongoDB backups (local only — never commit)
backup-*/

# Misc
.cache/
```

### .env.example

This file is committed to the repo. It documents every required variable without exposing secrets.

```env
# Server
NODE_ENV=development
PORT=5000
SESSION_SECRET=
LOG_LEVEL=debug
APP_URL=http://localhost:5000
APP_NAME=

# MongoDB
MONGODB_URI=

# Stripe
STRIPE_SECRET_KEY=
STRIPE_WEBHOOK_SECRET=
STRIPE_PRICE_ID_MONTHLY=
STRIPE_PRICE_ID_YEARLY=

# Resend
RESEND_API_KEY=
EMAIL_FROM=

# Cloudflare Turnstile
TURNSTILE_SECRET_KEY=

# Sentry
SENTRY_DSN=

# PostHog
VITE_POSTHOG_KEY=
VITE_POSTHOG_HOST=

# Admin (optional — one-time seed, remove after first deploy)
SEED_OWNER_EMAIL=
```

---

## Phase 1: Project Setup & Infrastructure

### 1.1 — Create the Repo

- Create a new repo from your GitHub template.
- Clone locally, install dependencies.
- Copy `.env.example` → `.env` and fill in placeholder values.

### 1.2 — MongoDB Atlas Setup

- Create a new Atlas project for this app.
- Create an M0 (free) cluster to start — upgrade to M10+ when you approach launch.
- Set up a database user with least-privilege access.
- Whitelist Render's outbound IPs (or use `0.0.0.0/0` for dev, restrict for prod).
- Create the following collections with TTL indexes:

```javascript
// email_tokens — auto-delete after 24 hours
db.emailtokens.createIndex({ "createdAt": 1 }, { expireAfterSeconds: 86400 })

// login_attempts — auto-delete after 15 minutes
db.loginattempts.createIndex({ "createdAt": 1 }, { expireAfterSeconds: 900 })

// sessions — auto-delete after 24 hours (matches cookie expiry)
db.sessions.createIndex({ "expires": 1 }, { expireAfterSeconds: 0 })
```

### 1.3 — Environment Variables

Copy `.env.example` (documented in Phase 0) to `.env` and fill in real values. Here's what they look like populated:

```env
# Server
NODE_ENV=development
PORT=5000
SESSION_SECRET=<generate-a-64-char-random-string>
LOG_LEVEL=debug
APP_URL=http://localhost:5000
APP_NAME=MyApp

# MongoDB
MONGODB_URI=mongodb+srv://<user>:<pass>@<cluster>.mongodb.net/<dbname>

# Stripe
STRIPE_SECRET_KEY=sk_test_...
STRIPE_WEBHOOK_SECRET=whsec_...
STRIPE_PRICE_ID_MONTHLY=price_...
STRIPE_PRICE_ID_YEARLY=price_...

# Resend
RESEND_API_KEY=re_...
EMAIL_FROM=noreply@yourdomain.com

# Cloudflare Turnstile
TURNSTILE_SECRET_KEY=0x...

# Sentry
SENTRY_DSN=https://...@sentry.io/...

# PostHog
VITE_POSTHOG_KEY=phc_...
VITE_POSTHOG_HOST=https://us.i.posthog.com

# Admin (optional — one-time seed, remove after first deploy)
SEED_OWNER_EMAIL=you@yourdomain.com
```

---

## Phase 2: Authentication System

This is the core of every app. Port your battle-tested Flask auth pattern directly to Express.

### 2.1 — Session Configuration

```javascript
// server/config/session.js
import session from 'express-session';
import MongoStore from 'connect-mongo';

export default session({
  secret: process.env.SESSION_SECRET,
  resave: false,
  saveUninitialized: false,
  store: MongoStore.create({
    mongoUrl: process.env.MONGODB_URI,
    collectionName: 'sessions',
    ttl: 60 * 60 * 24, // 24 hours
  }),
  cookie: {
    httpOnly: true,
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'lax',
    maxAge: 1000 * 60 * 60 * 24, // 24 hours
  },
});
```

### 2.2 — Password Hashing (crypto.scrypt)

Zero external dependencies. Uses Node's built-in `crypto` module.

```javascript
// server/utils/password.js
import { scrypt, randomBytes, timingSafeEqual } from 'crypto';
import { promisify } from 'util';

const scryptAsync = promisify(scrypt);
const KEY_LENGTH = 64;
const SALT_LENGTH = 16;

export async function hashPassword(password) {
  const salt = randomBytes(SALT_LENGTH).toString('hex');
  const hash = await scryptAsync(password, salt, KEY_LENGTH);
  return `${salt}:${hash.toString('hex')}`;
}

export async function verifyPassword(password, stored) {
  const [salt, hash] = stored.split(':');
  const hashBuffer = Buffer.from(hash, 'hex');
  const derivedKey = await scryptAsync(password, salt, KEY_LENGTH);
  return timingSafeEqual(hashBuffer, derivedKey);
}
```

`timingSafeEqual` prevents timing attacks on password comparison. The salt is stored alongside the hash, separated by `:`.

### 2.3 — Auth Flow Summary

```
Registration
├── Validate input
├── Verify Cloudflare Turnstile token (server-side)
├── Check if email already exists (timing-safe)
├── Hash password with crypto.scrypt (Node native)
├── Create user in MongoDB (emailVerified: false)
├── Generate verification token → store in email_tokens (TTL: 24h)
├── Send verification email via Resend
└── Return success (do NOT auto-login)

Login
├── Validate input
├── Verify Cloudflare Turnstile token
├── Check login_attempts for lockout status
│   ├── ≥10 attempts → hard lockout (return generic error)
│   └── ≥5 attempts → cooldown period (return generic error)
├── Find user by email
├── Compare password with crypto.scrypt (timing-safe)
├── Check emailVerified === true
├── Regenerate session (prevent fixation)
├── Reset login_attempts for this email
└── Return user data + set session cookie

Logout
├── Destroy server-side session
└── Clear session cookie

Password Reset
├── User submits email
├── Generate reset token → store in email_tokens (TTL: 1h)
├── Send reset email via Resend
├── User clicks link → validate token
├── Hash new password, update user
├── Delete all sessions for this user (force re-login everywhere)
└── Delete the used token

Password Change (authenticated)
├── Verify current password
├── Hash new password, update user
├── Delete all OTHER sessions for this user
└── Regenerate current session
```

### 2.4 — CSRF Protection

```javascript
// Middleware: validate X-CSRF-Token header using constant-time comparison
// Frontend: send a CSRF token with every state-changing request
// Pattern: double-submit cookie — set token in cookie, send in header, compare server-side
```

**CRITICAL: CSRF + CORS must work together.** The double-submit cookie pattern relies on the browser's same-origin policy to prevent cross-site requests from reading the cookie. However, if your CORS configuration is too permissive (e.g., `origin: '*'`), an attacker on another domain can make requests with credentials and bypass the custom header check. Your CORS must strictly whitelist only your own frontend origin:

```javascript
// server/app.js
app.use(cors({
  origin: process.env.APP_URL, // e.g., 'https://yourapp.com' — NEVER '*'
  credentials: true,            // Allow session cookies
}));
```

Never use `origin: '*'` or `origin: true` in production. A single misconfiguration here undermines your entire CSRF defense.

### 2.5 — Rate Limiting (MongoDB Sliding Window)

```javascript
// server/middleware/rateLimiter.js
// Uses a MongoDB collection to track request counts per IP per time window.
// Schema: { ip, endpoint, timestamps[] }
// On each request: push current timestamp, filter out expired ones, check count.
// Thresholds:
//   - Login: 10 requests per 15 minutes
//   - Registration: 5 per hour
//   - Password reset: 3 per hour
//   - General API: 100 per minute
```

---

## Phase 3: Cloudflare Turnstile Integration

### 3.1 — Frontend (React Component)

Place the Turnstile widget on login, registration, and password reset forms.

```
- Load Turnstile script from Cloudflare CDN
- Render invisible or managed widget
- On success, capture the token
- Send token alongside form data to your API
```

### 3.2 — Backend Verification

```javascript
// server/config/turnstile.js
// POST to https://challenges.cloudflare.com/turnstile/v0/siteverify
// Body: { secret: TURNSTILE_SECRET_KEY, response: tokenFromClient }
// If success === false → reject the request
// Apply to: registration, login, password reset, contact forms
```

---

## Phase 4: Stripe Checkout Integration

### 4.1 — Stripe Setup

- Create your Stripe account and enable test mode.
- Create a Product with Monthly and Yearly Price objects.
- Set up the Customer Portal for self-service subscription management.

### 4.2 — Checkout Flow

```
User clicks "Subscribe" on Pricing page
├── Frontend calls POST /api/billing/create-checkout-session
├── Backend creates a Stripe Checkout Session:
│   ├── mode: 'subscription'
│   ├── customer_email: user's email (or existing Stripe customer ID)
│   ├── price: STRIPE_PRICE_ID_MONTHLY or _YEARLY
│   ├── success_url: /dashboard?session_id={CHECKOUT_SESSION_ID}
│   └── cancel_url: /pricing
├── Backend returns the Checkout Session URL
├── Frontend redirects user to Stripe-hosted checkout
└── Stripe handles payment, then redirects back to success_url
```

### 4.3 — Webhook Handler (Idempotent)

```
POST /api/billing/webhook (raw body, no JSON parsing)
├── Verify Stripe signature using STRIPE_WEBHOOK_SECRET
├── Check idempotency:
│   ├── Look up event.id in a processed_events collection
│   ├── If found → return 200 immediately (already handled)
│   └── If not found → process the event, then store event.id with TTL (30 days)
├── Handle events:
│   ├── checkout.session.completed
│   │   └── Link Stripe customer ID to your user, set subscription active
│   ├── invoice.paid
│   │   └── Update subscription period, send receipt via Resend
│   ├── invoice.payment_failed
│   │   └── Flag user, send payment failure email
│   ├── customer.subscription.updated
│   │   └── Handle plan changes, update user record
│   └── customer.subscription.deleted
│       └── Revoke access, update user record
└── Return 200 immediately (process async if needed)
```

**Why idempotency matters:** Stripe can (and will) send the same webhook multiple times — on network timeouts, retries, or their own infrastructure hiccups. Without deduplication, you'll double-credit subscriptions, send duplicate receipts, or corrupt user records.

```javascript
// server/models/ProcessedEvent.js
const processedEventSchema = new mongoose.Schema({
  eventId: { type: String, required: true, unique: true },
  processedAt: { type: Date, default: Date.now },
});

// Auto-delete after 30 days — Stripe doesn't retry beyond this
processedEventSchema.index({ processedAt: 1 }, { expireAfterSeconds: 2592000 });
```

### 4.4 — Subscription Gating Middleware

```javascript
// server/middleware/requireSubscription.js
// Check user.subscriptionStatus === 'active' or 'trialing'
// If not → return 403 with { error: 'subscription_required' }
// Frontend catches this and redirects to Pricing page
```

### 4.5 — Customer Portal

```
User clicks "Manage Subscription" in Settings
├── Backend calls stripe.billingPortal.sessions.create()
├── Returns portal URL
└── Frontend redirects — user can cancel, update payment, switch plans
```

---

## Phase 5: Resend Email System

### 5.1 — Email Types

| Email                    | Trigger                                   | Template              |
| ------------------------ | ----------------------------------------- | --------------------- |
| Email Verification       | User registers                            | verifyEmail.js        |
| Password Reset           | User requests reset                       | resetPassword.js      |
| Welcome / Onboarding     | User verifies email                       | welcomeOnboard.js     |
| Payment Receipt          | Stripe `invoice.paid` webhook             | paymentReceipt.js     |
| Payment Failed           | Stripe `invoice.payment_failed` webhook   | paymentFailed.js      |
| Subscription Cancelled   | Stripe `customer.subscription.deleted`    | subCancelled.js       |

### 5.2 — HTML Email Templates

Each email template is a function that returns an HTML string. No build step, no extra dependencies.

```javascript
// server/emails/verifyEmail.js
export function verifyEmailTemplate({ token, appName }) {
  const verifyUrl = `${process.env.APP_URL}/verify-email/${token}`;

  return `
    <!DOCTYPE html>
    <html>
    <head>
      <meta charset="utf-8">
      <meta name="viewport" content="width=device-width, initial-scale=1.0">
      <title>Verify your email</title>
    </head>
    <body style="font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif; max-width: 600px; margin: 0 auto; padding: 40px 20px; color: #1a1a1a;">
      <h1 style="font-size: 24px; margin-bottom: 16px;">Verify your email</h1>
      <p style="font-size: 16px; line-height: 1.5; color: #4a4a4a;">
        Thanks for signing up for ${appName}. Click the button below to verify your email address.
      </p>
      <a href="${verifyUrl}" style="display: inline-block; background: #000; color: #fff; padding: 12px 24px; border-radius: 6px; text-decoration: none; margin: 24px 0;">
        Verify Email
      </a>
      <p style="font-size: 14px; color: #888; margin-top: 32px;">
        This link expires in 24 hours. If you didn't create an account, ignore this email.
      </p>
    </body>
    </html>
  `;
}
```

Follow this same pattern for all templates. Keep the styling inline (email clients ignore external CSS). Keep the structure simple — a heading, a short message, a call-to-action button, and a footer note.

### 5.3 — Resend Service

```javascript
// server/services/emailService.js
import { Resend } from 'resend';
import { verifyEmailTemplate } from '../emails/verifyEmail.js';

const resend = new Resend(process.env.RESEND_API_KEY);

export async function sendVerificationEmail(to, token) {
  const html = verifyEmailTemplate({ token, appName: process.env.APP_NAME });
  await resend.emails.send({
    from: process.env.EMAIL_FROM,
    to,
    subject: 'Verify your email',
    html,
  });
}
```

### 5.4 — Domain Configuration

- Add your sending domain in Resend dashboard.
- Add the DNS records (DKIM, SPF, DMARC) to your domain registrar.
- Verify the domain before going to production.

### 5.5 — Bounce & Complaint Handling

If you ignore bounces and complaints, your sender reputation degrades and emails start hitting spam for all users.

```
Resend Webhook Events to Handle
├── email.bounced
│   ├── Hard bounce → mark email as undeliverable in your User model
│   ├── Stop sending to this address immediately
│   └── Flag user in dashboard (prompt to update email)
├── email.complained (spam report)
│   ├── Immediately suppress all future sends to this address
│   └── Log for monitoring — high complaint rates trigger Resend account review
└── email.delivered
    └── Optional: update send status for audit trail
```

```javascript
// server/routes/email-webhooks.js
router.post('/api/webhooks/resend', express.json(), async (req, res) => {
  const { type, data } = req.body;

  if (type === 'email.bounced') {
    await User.findOneAndUpdate(
      { email: data.to },
      { emailBounced: true, emailBouncedAt: new Date() }
    );
  }

  if (type === 'email.complained') {
    await User.findOneAndUpdate(
      { email: data.to },
      { emailSuppressed: true, emailSuppressedAt: new Date() }
    );
  }

  res.sendStatus(200);
});
```

Check `emailBounced` and `emailSuppressed` flags before sending any email. Never send to a suppressed address.

---

## Phase 6: Frontend Architecture

### 6.1 — Routing Structure

```javascript
// src/App.jsx
<Routes>
  {/* Public */}
  <Route path="/" element={<Landing />} />
  <Route path="/pricing" element={<Pricing />} />
  <Route path="/login" element={<Login />} />
  <Route path="/register" element={<Register />} />
  <Route path="/forgot-password" element={<ForgotPassword />} />
  <Route path="/reset-password/:token" element={<ResetPassword />} />
  <Route path="/verify-email/:token" element={<VerifyEmail />} />

  {/* Protected — requires auth */}
  <Route element={<ProtectedRoute />}>
    <Route path="/dashboard" element={<Dashboard />} />
    <Route path="/settings" element={<Settings />} />

    {/* Protected — requires active subscription */}
    <Route element={<SubscriptionGate />}>
      <Route path="/app/*" element={<AppFeatures />} />
    </Route>

    {/* Protected — requires owner role */}
    <Route element={<OwnerRoute />}>
      <Route path="/admin" element={<AdminDashboard />} />
      <Route path="/admin/users" element={<AdminUsers />} />
    </Route>
  </Route>

  {/* 404 — must be last */}
  <Route path="*" element={<NotFound />} />
</Routes>
```

### 6.2 — API Layer (TanStack Query + Axios)

```javascript
// src/api/axiosInstance.js
// - baseURL pointed at your Express API
// - withCredentials: true (sends session cookie)
// - Interceptor: on 401 → redirect to /login
// - Interceptor: on 403 (subscription_required) → redirect to /pricing

// src/api/useAuth.js
// - useQuery('currentUser') — checks session on app load
// - useMutation for login, register, logout
// - onSuccess: invalidate 'currentUser' query

// src/api/useSubscription.js
// - useQuery('subscription') — fetch current subscription status
// - useMutation for createCheckoutSession, createPortalSession
```

### 6.3 — Zustand Auth Store

```javascript
// Minimal global state — only what TanStack Query doesn't cover
// - isAuthenticated (derived from currentUser query, but cached here for instant access)
// - UI state: sidebar open/closed, modal state, toast queue
```

### 6.4 — Landing Page Strategy

Every app needs a landing page that converts visitors to signups. Standard structure:

```
Landing Page
├── Hero section — headline, subheadline, CTA button
├── Problem/solution — what pain does this solve?
├── Feature highlights — 3-4 key features with icons
├── Social proof — testimonials, user count, logos (when available)
├── Pricing section — embedded PricingTable component
├── FAQ — 4-6 common questions
└── Footer — links, legal pages, social links
```

Build this once in the template. Swap copy and images per app.

### 6.5 — 404 Page

```javascript
// src/pages/NotFound.jsx
export default function NotFound() {
  return (
    <div>
      <h1>404</h1>
      <p>The page you're looking for doesn't exist.</p>
      <a href="/">Go home</a>
    </div>
  );
}
```

Keep it simple. Include a link back to the landing page. Match the app's visual style. The `<Route path="*">` catch-all in `App.jsx` handles any URL that doesn't match a defined route.

### 6.6 — robots.txt

```
// client/public/robots.txt
User-agent: *
Allow: /
Disallow: /dashboard
Disallow: /settings
Disallow: /admin
Disallow: /app

Sitemap: https://yourdomain.com/sitemap.xml
```

Lives in `client/public/` so Vite includes it in the build output. Tells search engines to index your public pages (landing, pricing, legal) but not authenticated pages (dashboard, settings, admin, app features). Update the domain per project.

### 6.7 — sitemap.xml

```javascript
// server/routes/sitemap.js
import { Router } from 'express';

const router = Router();

router.get('/sitemap.xml', (req, res) => {
  const baseUrl = process.env.APP_URL;

  const routes = [
    { path: '/', priority: '1.0', changefreq: 'weekly' },
    { path: '/pricing', priority: '0.8', changefreq: 'monthly' },
    { path: '/login', priority: '0.3', changefreq: 'yearly' },
    { path: '/register', priority: '0.3', changefreq: 'yearly' },
    { path: '/terms', priority: '0.2', changefreq: 'yearly' },
    { path: '/privacy', priority: '0.2', changefreq: 'yearly' },
  ];

  const xml = `<?xml version="1.0" encoding="UTF-8"?>
<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
${routes.map(r => `  <url>
    <loc>${baseUrl}${r.path}</loc>
    <changefreq>${r.changefreq}</changefreq>
    <priority>${r.priority}</priority>
  </url>`).join('\n')}
</urlset>`;

  res.set('Content-Type', 'application/xml');
  res.send(xml);
});

export default router;
```

Generated dynamically by Express so it always reflects your actual public routes. Only include pages you want search engines to index — never authenticated routes. Mount this route **before** the SPA catch-all in `app.js`.

---

## Phase 7: Deployment on Render

### 7.1 — Render Blueprint (`render.yaml`)

```yaml
services:
  - type: web
    name: myapp
    runtime: node
    region: oregon
    plan: starter
    buildCommand: npm install && cd client && npm install && npm run build
    startCommand: node server/app.js
    healthCheckPath: /api/health
    envVars:
      - key: NODE_ENV
        value: production
      - key: MONGODB_URI
        sync: false
      - key: SESSION_SECRET
        sync: false
      - key: STRIPE_SECRET_KEY
        sync: false
      - key: STRIPE_WEBHOOK_SECRET
        sync: false
      - key: RESEND_API_KEY
        sync: false
      - key: TURNSTILE_SECRET_KEY
        sync: false
      - key: SENTRY_DSN
        sync: false
      - key: SEED_OWNER_EMAIL
        sync: false  # Optional — remove after first deploy
```

### 7.2 — Express Serves the SPA

```javascript
// server/app.js — after all API routes
import path from 'path';

// Serve React build
app.use(express.static(path.join(__dirname, '../client/dist')));

// Catch-all: send index.html for client-side routing
app.get('*', (req, res) => {
  res.sendFile(path.join(__dirname, '../client/dist/index.html'));
});
```

### 7.3 — Deployment Checklist

- Push to `main` branch → Render auto-deploys the single service.
- Set all environment variables in Render dashboard (values marked `sync: false`).
- Configure custom domain.
- Enable Render's auto-SSL (free Let's Encrypt certificate).
- Set up Stripe webhook endpoint pointing to `https://yourdomain.com/api/billing/webhook`.
- Verify health check is responding at `/api/health`.

---

## Phase 8: Testing Strategy (Hard Deploy Gate)

**Tests are not optional. No test suite, no deploy.** GitHub Actions enforces this — if tests fail, the PR cannot merge and Render never sees the code.

### 8.1 — Backend (Express)

**Tools:** Jest + Supertest

```
Priority test coverage:
├── Auth flows — register, login, logout, verify, reset (highest priority)
├── Input validation — malformed payloads, missing fields, boundary values, injection attempts
├── Stripe webhook handler — mock webhook events, verify DB updates, test idempotency (duplicate events)
├── Middleware — requireAuth, requireSubscription, rateLimiter, validate
├── Rate limiting — verify lockout thresholds, verify counter persistence
├── CSRF — verify rejection of requests without valid token
├── Error handling — verify no stack traces leak to client, verify error response format
└── Account deletion — verify all user data is removed, verify Stripe cancellation
```

### 8.2 — Input Validation Tests

This is its own category because it's the most common source of security bugs.

```javascript
// server/__tests__/validation/auth.test.js

describe('Registration validation', () => {
  // Missing fields
  test('rejects empty body', ...);
  test('rejects missing email', ...);
  test('rejects missing password', ...);

  // Email validation
  test('rejects invalid email format', ...);
  test('normalizes email to lowercase', ...);
  test('trims whitespace from email', ...);

  // Password boundaries
  test('rejects password under 8 characters', ...);
  test('rejects password over 128 characters', ...);
  test('accepts password at exactly 8 characters', ...);

  // Injection / abuse
  test('rejects HTML in name field', ...);
  test('strips unknown fields from body', ...);
  test('rejects missing Turnstile token', ...);
});
```

Write equivalent tests for every route that accepts input: login, password reset, profile update, checkout session creation.

### 8.3 — Frontend (React)

**Tools:** Vitest + React Testing Library

```
Priority test coverage:
├── Auth forms — validation messages display correctly, submit disabled until valid
├── Protected routes — redirect behavior when not authenticated
├── Subscription gate — redirect when subscription inactive
├── API error handling — 401/403 interceptor behavior, error toast display
└── Account settings — export and delete flows render confirmation modals
```

### 8.4 — Testing Rules for a Solo Dev

1. **Auth, payments, and input validation** — these are the three areas where bugs cost you users, money, or security. Test them thoroughly.
2. **Webhook handler** — test idempotency specifically. Fire the same event twice, verify the database state doesn't double-apply.
3. **Happy path E2E** — one end-to-end test: register → verify → subscribe → access feature → cancel.
4. **Validation edge cases** — test every boundary (min/max length, empty strings, null values, wrong types). These are the bugs that slip through manual testing.
5. **Never skip tests to ship faster.** The CI pipeline won't let you, and that's by design.

---

## Phase 9: CI/CD Pipeline (Tests Block Deploys)

### 9.1 — GitHub Actions Workflow

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test-server:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
      - run: cd server && npm ci
      - run: cd server && npm audit --audit-level=high
      - run: cd server && npm test

  test-client:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
      - run: cd client && npm ci
      - run: cd client && npm audit --audit-level=high
      - run: cd client && npm test
      - run: cd client && npm run build  # Verify production build succeeds
```

### 9.2 — Branch Protection Rules

Configure on GitHub → Settings → Branches → `main`:

- Require pull request before merging.
- Require status checks to pass: `test-server` and `test-client`.
- No direct pushes to main.

This makes the test gate **absolute** — there is no way to deploy untested code.

### 9.3 — Pipeline Flow

```
Push to feature branch → GitHub Actions runs tests + audit
├── Tests pass + no high/critical vulnerabilities → PR is mergeable
├── Merge to main → Render auto-deploys
├── Tests fail → PR blocked, fix before merging
└── npm audit fails → PR blocked, update vulnerable packages
```

---

## Phase 10: Monitoring & Error Tracking

### 10.1 — Sentry Setup

**Backend:**

```javascript
// Top of server/app.js
import * as Sentry from '@sentry/node';

Sentry.init({
  dsn: process.env.SENTRY_DSN,
  environment: process.env.NODE_ENV,
  tracesSampleRate: 0.1, // 10% of transactions for performance monitoring
});

// Add Sentry error handler AFTER all routes
app.use(Sentry.Handlers.errorHandler());
```

**Frontend:**

```javascript
// Top of src/main.jsx
import * as Sentry from '@sentry/react';

Sentry.init({
  dsn: process.env.VITE_SENTRY_DSN,
  environment: import.meta.env.MODE,
  tracesSampleRate: 0.1,
});
```

### 10.2 — PostHog Analytics

**Frontend setup:**

```javascript
// src/main.jsx
import posthog from 'posthog-js';

posthog.init(import.meta.env.VITE_POSTHOG_KEY, {
  api_host: import.meta.env.VITE_POSTHOG_HOST || 'https://us.i.posthog.com',
  person_profiles: 'identified_only',
});
```

**Identify users after login:**

```javascript
// After successful login, link analytics to the user
posthog.identify(user.id, {
  email: user.email,
  name: user.name,
  subscriptionStatus: user.subscriptionStatus,
});
```

**Track key conversion events:**

```javascript
posthog.capture('signup_completed');
posthog.capture('email_verified');
posthog.capture('subscription_started', { plan: 'monthly' });
posthog.capture('subscription_cancelled');
```

**Reset on logout:**

```javascript
posthog.reset();
```

**What PostHog gives you that GA doesn't:** session replays (watch exactly where users get stuck), feature flags (roll out features to a percentage of users), retention cohorts (which users come back and why), and funnels built from your custom events.

### 10.3 — Health Monitoring

```javascript
// server/routes/health.js
router.get('/api/health', async (req, res) => {
  try {
    await mongoose.connection.db.admin().ping();
    res.json({ status: 'ok', db: 'connected' });
  } catch (err) {
    res.status(503).json({ status: 'error', db: 'disconnected' });
  }
});
```

Set Render's health check to hit this endpoint. You'll get alerts if your server goes down.

---

## Phase 11: Legal Pages

### 11.1 — Required Pages

Every SaaS app needs:

- **Terms of Service** — governs use of your application.
- **Privacy Policy** — required by law in most jurisdictions (GDPR, CCPA). Explains what data you collect (email, payment info via Stripe, cookies, analytics) and how you use it.
- **Cookie Policy** — if using PostHog and session cookies, you need this.

### 11.2 — Approach

- Use a generator (Termly, Iubenda, or similar) to create a baseline.
- Customize for your specific data practices.
- Host as static pages in your React app (`/terms`, `/privacy`, `/cookies`).
- Link from the footer on every page and from the registration form.
- **Have a lawyer review before launch** if your app handles sensitive data.

---

## Phase 12: Edge Case Tooling

These are the specialized tools you reach for when a specific app requires capabilities beyond the standard stack. They're not in every app, but when you need them, the decision is already made.

### 12.1 — Canvas Rendering: Raw Canvas 2D API

**When:** Any app that needs drawing, image manipulation, charts from scratch, annotations, or visual editors.

**Why raw Canvas 2D over a library:** Zero dependencies, full pixel-level control, no abstraction overhead. Libraries like Fabric.js or Konva add convenience but also add bundle size and opinions you don't need for most use cases.

**Standard pattern:**

```javascript
// src/hooks/useCanvas.js
// Custom hook that:
// - Takes a ref to a <canvas> element
// - Returns the 2D context
// - Handles resize/DPI scaling for retina displays
// - Cleans up on unmount

import { useRef, useEffect } from 'react';

export function useCanvas(draw) {
  const canvasRef = useRef(null);

  useEffect(() => {
    const canvas = canvasRef.current;
    const ctx = canvas.getContext('2d');

    // Handle retina/HiDPI displays
    const dpr = window.devicePixelRatio || 1;
    const rect = canvas.getBoundingClientRect();
    canvas.width = rect.width * dpr;
    canvas.height = rect.height * dpr;
    ctx.scale(dpr, dpr);

    draw(ctx, rect.width, rect.height);

    return () => ctx.clearRect(0, 0, canvas.width, canvas.height);
  }, [draw]);

  return canvasRef;
}
```

**Template repo location:** Include `useCanvas.js` in `client/src/hooks/` — delete if not needed per app.

### 12.2 — Large File Storage: MongoDB GridFS

**When:** Any app that needs to store files larger than 16MB (MongoDB's document size limit). Common use cases: user-uploaded images, audio files, PDFs, exports.

**Why GridFS over S3:** Keeps everything in your existing MongoDB Atlas infrastructure. No second service to manage, no separate billing, no CORS configuration. For a solo dev, fewer moving parts wins. Move to S3 only if you hit GridFS performance limits at scale.

**Standard pattern:**

```javascript
// server/config/gridfs.js
import mongoose from 'mongoose';
import { GridFSBucket } from 'mongodb';

let bucket;

export function getGridFSBucket() {
  if (!bucket) {
    const db = mongoose.connection.db;
    bucket = new GridFSBucket(db, { bucketName: 'uploads' });
  }
  return bucket;
}

// Upload a file
export function uploadToGridFS(filename, readStream, metadata = {}) {
  const bucket = getGridFSBucket();
  const uploadStream = bucket.openUploadStream(filename, { metadata });
  readStream.pipe(uploadStream);
  return new Promise((resolve, reject) => {
    uploadStream.on('finish', () => resolve(uploadStream.id));
    uploadStream.on('error', reject);
  });
}

// Download a file
export function downloadFromGridFS(fileId) {
  const bucket = getGridFSBucket();
  return bucket.openDownloadStream(new mongoose.Types.ObjectId(fileId));
}

// Delete a file
export function deleteFromGridFS(fileId) {
  const bucket = getGridFSBucket();
  return bucket.delete(new mongoose.Types.ObjectId(fileId));
}
```

**Express route pattern:**

```javascript
// server/routes/files.js
import multer from 'multer';
import { uploadToGridFS, downloadFromGridFS } from '../config/gridfs.js';

const upload = multer({ storage: multer.memoryStorage() });

// Upload
router.post('/upload', requireAuth, upload.single('file'), async (req, res) => {
  const { Readable } = await import('stream');
  const readStream = Readable.from(req.file.buffer);
  const fileId = await uploadToGridFS(req.file.originalname, readStream, {
    userId: req.session.userId,
    contentType: req.file.mimetype,
  });
  res.json({ fileId });
});

// Download
router.get('/download/:id', requireAuth, async (req, res) => {
  const stream = downloadFromGridFS(req.params.id);
  stream.pipe(res);
});
```

**Additional dependency:** `multer` for handling multipart file uploads.

### 12.3 — Markdown Processing: marked

**When:** Any app where users write or consume Markdown — blog platforms, documentation tools, note-taking apps, content editors with preview.

**Standard pattern:**

```javascript
// server/utils/markdown.js
import { marked } from 'marked';
import DOMPurify from 'isomorphic-dompurify';

// Always sanitize rendered HTML to prevent XSS
export function renderMarkdown(raw) {
  const html = marked.parse(raw);
  return DOMPurify.sanitize(html);
}
```

**Why marked:** Node-native, fast, extensible, well-maintained, and stays in your JS-only stack. Supports GFM (GitHub Flavored Markdown) out of the box including tables, strikethrough, and task lists.

**Security note:** Always sanitize rendered Markdown HTML before sending to the frontend. User-generated Markdown is an XSS vector. `isomorphic-dompurify` works on both server and client.

### 12.4 — MP3 Encoding: ffmpeg

**When:** Any app that processes audio — podcast tools, voice note apps, music utilities, transcription preprocessors.

**Render deployment note:** ffmpeg is available on Render's native environments. You do not need to install it separately — it's included in Render's build images.

**Standard pattern:**

```javascript
// server/services/audioService.js
import { exec } from 'child_process';
import { promisify } from 'util';
import path from 'path';
import fs from 'fs';

const execAsync = promisify(exec);

// Convert any audio file to MP3
export async function convertToMp3(inputPath, outputPath, options = {}) {
  const {
    bitrate = '128k',
    sampleRate = 44100,
    channels = 2,
  } = options;

  const cmd = [
    'ffmpeg -i', `"${inputPath}"`,
    `-b:a ${bitrate}`,
    `-ar ${sampleRate}`,
    `-ac ${channels}`,
    '-y',  // Overwrite output
    `"${outputPath}"`,
  ].join(' ');

  await execAsync(cmd);
  return outputPath;
}

// Get audio metadata (duration, format, bitrate)
export async function getAudioMetadata(filePath) {
  const cmd = `ffprobe -v quiet -print_format json -show_format -show_streams "${filePath}"`;
  const { stdout } = await execAsync(cmd);
  return JSON.parse(stdout);
}
```

**Integration with GridFS:** For apps that both store and process audio, combine ffmpeg with GridFS:

```
User uploads audio file
├── multer receives file → temp directory
├── ffmpeg converts to MP3 → temp directory
├── Upload MP3 to GridFS → get fileId
├── Delete temp files
└── Return fileId to client
```

**Processing considerations:**

- Always process audio asynchronously — never block the Express request/response cycle for long conversions.
- For files over 50MB, consider a background job queue (BullMQ + Redis) instead of processing inline.
- Set reasonable file size limits in multer to prevent abuse.

### 12.5 — Bot Protection: Cloudflare Turnstile (Extended)

Turnstile is already in the standard stack for auth forms, but some apps need it on additional surfaces.

**Extended use cases:**

```
Standard (always on)
├── Registration form
├── Login form
└── Password reset form

App-specific (add as needed)
├── Contact / support forms
├── Public-facing submission forms (e.g., user-generated content)
├── API endpoints exposed without auth (e.g., waitlist signup)
├── Rate-limited actions (combine Turnstile + rate limiter for extra protection)
└── File upload endpoints (prevent bot-driven storage abuse)
```

**Reusable Turnstile React component:**

```jsx
// src/components/ui/TurnstileWidget.jsx
// - Wraps the Cloudflare Turnstile script
// - Accepts onVerify callback that returns the token
// - Handles loading state and expiration/retry
// - Drop into any form that needs protection
```

### Edge Case Decision Matrix

| Need                          | Tool                  | npm Package / Binary  | Notes                                    |
| ----------------------------- | --------------------- | --------------------- | ---------------------------------------- |
| 2D drawing / image editing    | Canvas 2D API         | None (browser-native) | Use `useCanvas` hook                     |
| File storage > 16MB           | MongoDB GridFS        | `mongodb` (built-in)  | Add `multer` for uploads                 |
| Markdown → HTML               | marked                | `marked`                | Always sanitize with `isomorphic-dompurify` |
| Audio conversion to MP3       | ffmpeg                | System binary         | Pre-installed on Render                  |
| Bot protection (extended)     | Cloudflare Turnstile  | None (script tag)     | Reuse `TurnstileWidget.jsx` component    |

---

## Phase 13: Input Validation & Sanitization

Every Express route that accepts user input must validate it before processing. No exceptions.

### 13.1 — Zod Validation Middleware

```javascript
// server/middleware/validate.js
import { ZodError } from 'zod';

export function validate(schema) {
  return (req, res, next) => {
    try {
      // parse() validates AND strips unknown fields by default
      req.body = schema.parse(req.body);
      next();
    } catch (err) {
      if (err instanceof ZodError) {
        const errors = err.errors.map((e) => e.message);
        return res.status(400).json({ error: 'Validation failed', details: errors });
      }
      next(err);
    }
  };
}
```

### 13.2 — Schema Definitions

```javascript
// server/validators/auth.js
import { z } from 'zod';

export const registerSchema = z.object({
  email: z.string().email().toLowerCase().trim(),
  password: z.string().min(8).max(128),
  name: z.string().min(1).max(100).trim(),
  turnstileToken: z.string(),
});

export const loginSchema = z.object({
  email: z.string().email().toLowerCase().trim(),
  password: z.string(),
  turnstileToken: z.string(),
});

export const resetPasswordSchema = z.object({
  password: z.string().min(8).max(128),
  token: z.string(),
});
```

**Shared schemas:** These same Zod schemas can be imported on the frontend to validate form input before it's sent to the API. Define once, enforce everywhere.

### 13.3 — Usage in Routes

```javascript
// server/routes/auth.js
import { validate } from '../middleware/validate.js';
import { registerSchema, loginSchema } from '../validators/auth.js';

router.post('/register', validate(registerSchema), async (req, res) => {
  // req.body is already validated, sanitized, and stripped of unknowns
});
```

### 13.4 — Validation Rules by Route Category

```
Auth Routes
├── Registration: email (valid, lowercase, trimmed), password (8-128 chars), name (1-100 chars, trimmed), turnstileToken
├── Login: email, password, turnstileToken
├── Password reset request: email
├── Password reset confirm: token, new password (8-128 chars)
└── Password change: currentPassword, newPassword (8-128 chars)

Billing Routes
├── Create checkout session: priceId (must match known price IDs)
└── Webhook: raw body (no Zod — Stripe signature verification handles this)

User Routes
├── Update profile: name (1-100 chars, trimmed), email (valid, lowercase)
└── Delete account: password (confirmation)

File Upload Routes (if using GridFS)
├── Upload: multer handles file validation (mimetype, size limit)
└── Download: fileId (valid MongoDB ObjectId format)
```

### 13.5 — CRITICAL: NoSQL Injection Prevention

MongoDB is immune to SQL injection but vulnerable to NoSQL injection. An attacker can send `{"email": {"$gt": ""}}` instead of a string, and MongoDB will happily query with it — potentially bypassing authentication.

**Your defense is Zod's `z.string()`.** When you define `email: z.string()`, Zod will reject any value that isn't a string. An object like `{"$gt": ""}` fails validation before it ever reaches Mongoose.

**This only works if every field in every schema has an explicit type.** If you accidentally use `z.any()` or skip validation on a field, you've opened a NoSQL injection vector.

```javascript
// SAFE — Zod enforces string type, rejects objects
email: z.string().email()

// DANGEROUS — never do this
email: z.any()  // Accepts {"$gt": ""} — NoSQL injection
```

**Defense in depth:** Even with Zod, add Mongoose's `sanitizeFilter` option as a second layer:

```javascript
// server/config/db.js — add to your Mongoose connection
mongoose.set('sanitizeFilter', true);
```

This tells Mongoose to strip `$`-prefixed operators from query filters, catching anything that slips past validation.

---

## Phase 14: Centralized Error Handling

### 14.1 — Express Error Handler

```javascript
// server/middleware/errorHandler.js
import logger from '../config/logger.js';

export function errorHandler(err, req, res, next) {
  // Log the full error server-side
  logger.error({
    message: err.message,
    stack: err.stack,
    method: req.method,
    url: req.originalUrl,
    userId: req.session?.userId || null,
  });

  // Mongoose validation errors
  if (err.name === 'ValidationError') {
    return res.status(400).json({
      error: 'Validation failed',
      details: Object.values(err.errors).map((e) => e.message),
    });
  }

  // Mongoose duplicate key (e.g., duplicate email)
  if (err.code === 11000) {
    return res.status(409).json({ error: 'Resource already exists' });
  }

  // Zod validation errors (if not caught by middleware)
  if (err.name === 'ZodError') {
    return res.status(400).json({
      error: 'Validation failed',
      details: err.errors.map((e) => e.message),
    });
  }

  // Default: never leak stack traces to the client
  const statusCode = err.statusCode || 500;
  const message = statusCode === 500 ? 'Internal server error' : err.message;

  res.status(statusCode).json({ error: message });
}
```

### 14.2 — Async Route Wrapper

Express doesn't catch async errors by default. Wrap every async route handler:

```javascript
// server/utils/asyncHandler.js
export function asyncHandler(fn) {
  return (req, res, next) => {
    Promise.resolve(fn(req, res, next)).catch(next);
  };
}

// Usage in routes:
router.post('/register', validate(registerSchema), asyncHandler(async (req, res) => {
  // If this throws, it's caught and forwarded to errorHandler
}));
```

### 14.3 — Registration Order in app.js

```javascript
// 1. Sentry request handler (first)
// 2. Middleware (session, helmet, cors, json parser)
// 3. Stripe webhook route (needs raw body — BEFORE express.json())
// 4. express.json() for all other routes
// 5. API routes (auth, user, billing)
// 6. Admin routes (/api/admin/* — requireAuth + requireOwner baked into router)
// 7. Sitemap route (/sitemap.xml — before SPA catch-all)
// 8. SPA catch-all (serves React build)
// 9. Sentry error handler
// 10. Custom errorHandler (last)
```

---

## Phase 15: Graceful Shutdown

Render sends SIGTERM on every deploy. Without handling it, in-flight requests get dropped and database connections hang.

```javascript
// server/app.js — at the bottom
import mongoose from 'mongoose';
import logger from './config/logger.js';

const server = app.listen(process.env.PORT || 5000, () => {
  logger.info(`Server running on port ${process.env.PORT || 5000}`);
});

function gracefulShutdown(signal) {
  logger.info(`${signal} received. Starting graceful shutdown...`);

  // Stop accepting new connections
  server.close(async () => {
    logger.info('HTTP server closed');

    try {
      // Close database connection
      await mongoose.connection.close();
      logger.info('MongoDB connection closed');
      process.exit(0);
    } catch (err) {
      logger.error('Error during shutdown', err);
      process.exit(1);
    }
  });

  // Force shutdown after 10 seconds if graceful shutdown stalls
  setTimeout(() => {
    logger.error('Forced shutdown — graceful shutdown timed out');
    process.exit(1);
  }, 10000);
}

process.on('SIGTERM', () => gracefulShutdown('SIGTERM'));
process.on('SIGINT', () => gracefulShutdown('SIGINT'));
```

---

## Phase 16: Structured Logging

### 16.1 — Pino Setup

```javascript
// server/config/logger.js
import pino from 'pino';

const logger = pino({
  level: process.env.LOG_LEVEL || 'info',
  transport:
    process.env.NODE_ENV !== 'production'
      ? { target: 'pino-pretty', options: { colorize: true } }
      : undefined,
});

export default logger;
```

### 16.2 — Request Logging Middleware

```javascript
// server/middleware/requestLogger.js
import pinoHttp from 'pino-http';
import logger from '../config/logger.js';

export const requestLogger = pinoHttp({
  logger,
  customLogLevel: (req, res, err) => {
    if (res.statusCode >= 500 || err) return 'error';
    if (res.statusCode >= 400) return 'warn';
    return 'info';
  },
  // Don't log health check spam
  autoLogging: {
    ignore: (req) => req.url === '/api/health',
  },
});
```

### 16.3 — What to Log

```
Always log:
├── Errors with full stack traces and request context
├── Auth events: login success/failure, registration, password reset, session invalidation
├── Payment events: checkout created, webhook received, subscription status changes
├── Rate limit triggers: IP, endpoint, current count
└── Startup and shutdown events

Never log:
├── Passwords (even hashed)
├── Session tokens or secrets
├── Full credit card numbers (Stripe handles this, but be cautious)
└── Health check requests (noise)
```

---

## Phase 17: Database Indexing Strategy

### 17.1 — Required Indexes

```javascript
// TTL Indexes (already in Phase 1)
db.emailtokens.createIndex({ "createdAt": 1 }, { expireAfterSeconds: 86400 });
db.loginattempts.createIndex({ "createdAt": 1 }, { expireAfterSeconds: 900 });
db.sessions.createIndex({ "expires": 1 }, { expireAfterSeconds: 0 });
db.processedevents.createIndex({ "processedAt": 1 }, { expireAfterSeconds: 2592000 });

// User lookups
db.users.createIndex({ "email": 1 }, { unique: true });
db.users.createIndex({ "stripeCustomerId": 1 }, { sparse: true });

// Email token lookups
db.emailtokens.createIndex({ "token": 1 });
db.emailtokens.createIndex({ "userId": 1 });

// Login attempt lookups (rate limiting)
db.loginattempts.createIndex({ "ip": 1, "endpoint": 1 });

// Rate limiting general
db.ratelimits.createIndex({ "ip": 1, "endpoint": 1 });

// Stripe event deduplication
db.processedevents.createIndex({ "eventId": 1 }, { unique: true });
```

### 17.2 — Mongoose Model Indexes

Define indexes in your Mongoose schemas so they're created automatically:

```javascript
// server/models/User.js
const userSchema = new mongoose.Schema({
  email: { type: String, required: true, unique: true, lowercase: true, trim: true },
  stripeCustomerId: { type: String, sparse: true, index: true },
  // ...
});
```

### 17.3 — Monitoring Index Performance

- Use Atlas Performance Advisor (free on M10+) to identify slow queries and missing indexes.
- On M0 (free tier), check `db.collection.getIndexes()` and review query patterns manually.
- Rule of thumb: if a query runs more than once per request and filters on a field, that field needs an index.

---

## Phase 18: Backup & Disaster Recovery

### 18.1 — Atlas Backup Configuration

| Atlas Tier | Backup Type | Frequency | Retention |
| ---------- | ----------- | --------- | --------- |
| M0 (free)  | None — Atlas does not backup free clusters | — | — |
| M2/M5      | Daily snapshots | Every 24 hours | Varies by tier |
| M10+       | Continuous backup + point-in-time recovery | Continuous | Configurable |

**Recommendation:** Start on M0 for development. Upgrade to M10 before launch. The cost is worth it for continuous backups and point-in-time recovery.

### 18.2 — Manual Backup Strategy (M0)

If you're on the free tier during development:

```bash
# Export all collections (run before any risky migration)
mongodump --uri="mongodb+srv://..." --out=./backup-$(date +%Y%m%d)

# Restore from backup
mongorestore --uri="mongodb+srv://..." ./backup-20260101
```

### 18.3 — Disaster Recovery Checklist

```
If your database is corrupted or data is lost:
├── M10+: Restore from Atlas point-in-time recovery (pick exact timestamp)
├── M0: Restore from most recent mongodump
├── Stripe data: Stripe is your source of truth for billing — resync from Stripe API
├── Sessions: Users will need to log in again (acceptable)
└── Email tokens: Expired tokens are regenerated on request (acceptable)
```

### 18.4 — What You Cannot Lose

```
Critical data (must be backed up):
├── User accounts (email, hashed password, profile)
├── Subscription records (stripeCustomerId, subscription status)
└── App-specific user data (whatever your SaaS product stores)

Regenerable data (loss is inconvenient but not catastrophic):
├── Sessions → users log in again
├── Email tokens → users request new verification/reset
├── Login attempts → rate limits reset (brief window of exposure)
└── Processed webhook events → small risk of duplicate processing
```

---

## Phase 19: Dependency Security

### 19.1 — GitHub Dependabot

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "npm"
    directory: "/server"
    schedule:
      interval: "weekly"
    open-pull-requests-limit: 5

  - package-ecosystem: "npm"
    directory: "/client"
    schedule:
      interval: "weekly"
    open-pull-requests-limit: 5
```

### 19.2 — npm Audit in CI

Add to your GitHub Actions workflow:

```yaml
- run: cd server && npm audit --audit-level=high
- run: cd client && npm audit --audit-level=high
```

This fails the build if any high or critical vulnerabilities are found.

### 19.3 — Dependency Review Workflow

```
Weekly Dependabot PRs arrive
├── Review the changelog for breaking changes
├── CI runs tests automatically
├── If tests pass → merge
├── If tests fail → investigate before merging
└── npm audit --audit-level=high catches newly disclosed vulnerabilities
```

---

## Phase 20: Data Export & Account Deletion (GDPR)

Required by GDPR (EU), CCPA (California), and an increasing number of privacy laws worldwide. Even if your initial audience is US-only, build this in from the start — retrofitting it is painful.

### 20.1 — Data Export

```javascript
// server/routes/user.js
router.get('/api/user/export', requireAuth, asyncHandler(async (req, res) => {
  const userId = req.session.userId;

  const user = await User.findById(userId).select('-password').lean();
  const sessions = await Session.find({ 'session.userId': userId }).lean();

  // Include app-specific data
  // const appData = await AppModel.find({ userId }).lean();

  const exportData = {
    exportedAt: new Date().toISOString(),
    account: user,
    activeSessions: sessions.length,
    // appData,
  };

  res.setHeader('Content-Disposition', 'attachment; filename=my-data-export.json');
  res.json(exportData);
}));
```

### 20.2 — Account Deletion

```javascript
// server/routes/user.js
router.delete('/api/user/account', requireAuth, asyncHandler(async (req, res) => {
  const { password } = req.body;
  const userId = req.session.userId;
  const user = await User.findById(userId);

  // Require password confirmation
  const isMatch = await verifyPassword(password, user.password);
  if (!isMatch) return res.status(401).json({ error: 'Incorrect password' });

  // Cancel Stripe subscription if active
  if (user.stripeSubscriptionId) {
    await stripe.subscriptions.cancel(user.stripeSubscriptionId);
  }

  // Delete all user data
  await Promise.all([
    User.findByIdAndDelete(userId),
    Session.deleteMany({ 'session.userId': userId }),
    EmailToken.deleteMany({ userId }),
    LoginAttempt.deleteMany({ userId }),
    // Delete app-specific data
    // AppModel.deleteMany({ userId }),
    // Delete GridFS files if applicable
  ]);

  // Destroy the current session
  req.session.destroy();

  res.json({ message: 'Account deleted' });
}));
```

### 20.3 — Frontend

- Add "Export My Data" button in Settings → downloads JSON file.
- Add "Delete My Account" button in Settings → confirmation modal → requires password → calls API → redirects to landing page.

---

## Phase 21: Owner Role (Admin Access)

Two roles: `user` and `owner`. The role is stored as a field on the User document in MongoDB. This is the industry-standard RBAC pattern — the same approach used by Django, Rails, Laravel, and every major framework.

### 21.1 — How It Works

```
User signs up → normal registration flow → role defaults to "user"
You promote a user to owner via a one-time CLI script or Atlas query
On every admin request:
├── requireAuth checks session (is this person logged in?)
├── requireOwner checks role === 'owner' on the user document
├── Both pass → admin access granted
└── Either fails → 403 Forbidden

Transfer ownership:
├── Set new user's role to 'owner'
├── Set old user's role back to 'user'
└── Takes effect immediately — no redeploy needed
```

### 21.2 — User Model

```javascript
// server/models/User.js
const userSchema = new mongoose.Schema({
  email: { type: String, required: true, unique: true, lowercase: true, trim: true },
  password: { type: String, required: true },
  name: { type: String, required: true, trim: true },
  role: { type: String, enum: ['user', 'owner'], default: 'user' },
  emailVerified: { type: Boolean, default: false },
  stripeCustomerId: { type: String, sparse: true, index: true },
  subscriptionStatus: { type: String, default: 'inactive' },
  emailBounced: { type: Boolean, default: false },
  emailSuppressed: { type: Boolean, default: false },
  // ... app-specific fields
}, { timestamps: true });
```

### 21.3 — Owner Middleware

```javascript
// server/middleware/requireOwner.js
export function requireOwner(req, res, next) {
  if (req.user.role !== 'owner') {
    return res.status(403).json({ error: 'Forbidden' });
  }
  next();
}
```

Stack it after `requireAuth` on every admin route. The owner must be fully authenticated through the normal login flow — Turnstile, rate limiting, session, everything. The role only determines *what* an authenticated user can do.

### 21.4 — Promoting the First Owner

You need a way to make the first user an owner. Three options, all valid:

**Option A — One-time CLI script (recommended):**

```javascript
// scripts/promote-owner.js
import mongoose from 'mongoose';
import User from '../server/models/User.js';
import 'dotenv/config';

const email = process.argv[2];
if (!email) {
  console.error('Usage: node scripts/promote-owner.js user@example.com');
  process.exit(1);
}

await mongoose.connect(process.env.MONGODB_URI);
const user = await User.findOneAndUpdate(
  { email },
  { role: 'owner' },
  { new: true }
);

if (!user) {
  console.error(`No user found with email: ${email}`);
} else {
  console.log(`Promoted ${user.email} to owner`);
}

await mongoose.connection.close();
```

Run it once after deploying: `node scripts/promote-owner.js you@yourdomain.com`

**Option B — Atlas UI:** Open your Atlas cluster, find the user document, set `role: "owner"`. Quick and simple.

**Option C — Seed on startup (for the template repo):**

```javascript
// server/config/seed.js
import User from '../models/User.js';
import logger from './logger.js';

export async function seedOwner() {
  const ownerEmail = process.env.SEED_OWNER_EMAIL;
  if (!ownerEmail) return;

  const updated = await User.findOneAndUpdate(
    { email: ownerEmail },
    { role: 'owner' },
    { new: true }
  );

  if (updated) {
    logger.info(`Owner role assigned to ${ownerEmail}`);
  }

  // Clear the env var purpose — it's a one-time seed, not a permanent config
}
```

This uses an optional `SEED_OWNER_EMAIL` env var to promote a user on the first deploy. Remove the env var after the seed runs. The database is the source of truth from that point forward.

### 21.5 — Admin Routes

```javascript
// server/routes/admin.js
import { Router } from 'express';
import { requireAuth } from '../middleware/requireAuth.js';
import { requireOwner } from '../middleware/requireOwner.js';
import { asyncHandler } from '../utils/asyncHandler.js';
import logger from '../config/logger.js';
import User from '../models/User.js';

const router = Router();

// All admin routes require auth + owner role
router.use(requireAuth);
router.use(requireOwner);

// Log every admin action
router.use((req, res, next) => {
  logger.info({
    adminAction: true,
    owner: req.user.email,
    method: req.method,
    url: req.originalUrl,
  });
  next();
});

// List all users
router.get('/users', asyncHandler(async (req, res) => {
  const users = await User.find()
    .select('-password')
    .sort({ createdAt: -1 })
    .lean();
  res.json(users);
}));

// View a specific user's details
router.get('/users/:id', asyncHandler(async (req, res) => {
  const user = await User.findById(req.params.id).select('-password').lean();
  if (!user) return res.status(404).json({ error: 'User not found' });
  res.json(user);
}));

// Get app-wide stats (total users, active subscriptions)
router.get('/stats', asyncHandler(async (req, res) => {
  const [totalUsers, activeSubscriptions] = await Promise.all([
    User.countDocuments(),
    User.countDocuments({ subscriptionStatus: 'active' }),
  ]);
  res.json({ totalUsers, activeSubscriptions });
}));

// Transfer ownership to another user
router.post('/transfer-ownership/:id', asyncHandler(async (req, res) => {
  const newOwner = await User.findById(req.params.id);
  if (!newOwner) return res.status(404).json({ error: 'User not found' });

  // Demote current owner, promote new owner
  await User.findByIdAndUpdate(req.user._id, { role: 'user' });
  await User.findByIdAndUpdate(newOwner._id, { role: 'owner' });

  logger.info({
    adminAction: true,
    type: 'ownership_transfer',
    from: req.user.email,
    to: newOwner.email,
  });

  // Destroy current session — former owner must re-login as regular user
  req.session.destroy();
  res.json({ message: `Ownership transferred to ${newOwner.email}` });
}));

export default router;
```

### 21.6 — Expose Role to the Frontend

```javascript
// server/routes/user.js — in the "get current user" endpoint
router.get('/api/user/me', requireAuth, asyncHandler(async (req, res) => {
  const user = await User.findById(req.session.userId).select('-password').lean();
  res.json({
    ...user,
    isOwner: user.role === 'owner',
  });
}));
```

### 21.7 — Frontend Admin Gate

```javascript
// src/components/layout/OwnerRoute.jsx
import { Navigate, Outlet } from 'react-router-dom';
import { useAuth } from '../../api/useAuth';

export default function OwnerRoute() {
  const { data: user, isLoading } = useAuth();

  if (isLoading) return null;
  if (!user?.isOwner) return <Navigate to="/dashboard" />;

  return <Outlet />;
}
```

```javascript
// src/App.jsx — admin routes inside ProtectedRoute
<Route element={<OwnerRoute />}>
  <Route path="/admin" element={<AdminDashboard />} />
  <Route path="/admin/users" element={<AdminUsers />} />
</Route>
```

### 21.8 — What the Owner Can Do

```
Owner-only capabilities:
├── View all registered users and their subscription status
├── View app-wide stats (total users, active subscriptions)
├── View a specific user's account details (for support)
├── Transfer ownership to another registered user
├── Access admin dashboard at /admin
└── Any app-specific admin features you add per project

What the owner CANNOT bypass:
├── Normal login flow (Turnstile, rate limiting, lockout)
├── Session-based auth (no special session, no backdoor)
├── Password requirements (same rules as every user)
└── CSRF protection (same for all requests)
```

### 21.9 — Security Rules

1. **Never bypass auth.** The owner logs in like everyone else. The `role` field only controls authorization, never authentication.
2. **Log everything.** Every admin action goes through pino with `adminAction: true` so you can filter and audit.
3. **Admin routes are a separate router.** Mounted at `/api/admin/*`, isolated from user routes. The `requireOwner` middleware runs on every request in the router, not per-route (so you can't accidentally forget it).
4. **The database is the single source of truth.** The role lives in the User document. No env vars, no config files, no second source.
5. **Ownership transfer is logged and destroys the former owner's session.** This prevents the former owner from continuing to make admin requests on a stale session.

---

## Launch Checklist

This is the final gate before every app goes live.

### Testing (Hard Gate)

- [ ] All backend tests passing (Jest + Supertest)
- [ ] All frontend tests passing (Vitest + React Testing Library)
- [ ] Input validation tests cover every route: missing fields, boundary values, injection attempts
- [ ] Stripe webhook idempotency tested (duplicate event produces no duplicate side effects)
- [ ] npm audit shows no high or critical vulnerabilities (both client and server)
- [ ] CI pipeline green — GitHub Actions enforces all of the above

### Security

- [ ] All environment variables set in production (no `.env` file deployed)
- [ ] `NODE_ENV=production`
- [ ] Session cookie: `httpOnly`, `secure`, `sameSite: lax`
- [ ] Helmet.js enabled with all default headers
- [ ] CORS configured
- [ ] CSRF protection active on all state-changing routes
- [ ] Rate limiting active on all endpoints (auth + general API)
- [ ] Turnstile active on registration, login, and password reset forms
- [ ] Input validation (Zod) on every route that accepts user input
- [ ] Every Zod schema uses explicit types (`z.string()`, never `z.any()`) — NoSQL injection prevention
- [ ] Mongoose `sanitizeFilter: true` enabled as defense in depth
- [ ] CORS origin is strictly set to your frontend URL (never `*` or `true`)
- [ ] Centralized error handler registered — no stack traces leak to client
- [ ] MongoDB Atlas IP whitelist configured for Render IPs
- [ ] All secrets are strong, random, and unique per app

### Auth

- [ ] Registration → verification email → login flow works end-to-end
- [ ] Login lockout triggers at correct thresholds (5/10 attempts)
- [ ] Password reset flow works end-to-end
- [ ] Password change invalidates other sessions
- [ ] Unverified emails cannot log in
- [ ] Account deletion removes all user data and cancels Stripe subscription
- [ ] Data export downloads complete JSON of user data
- [ ] Owner user promoted via CLI script or Atlas (role: 'owner' in database)
- [ ] Owner can access `/admin` routes after normal login
- [ ] Non-owner users get 403 on `/api/admin/*` endpoints
- [ ] `isOwner` flag returned from `/api/user/me` only for users with role 'owner'
- [ ] Admin actions logged with `adminAction: true` in pino
- [ ] Ownership transfer endpoint works and destroys former owner's session

### Payments

- [ ] Stripe keys swapped from test to live
- [ ] Webhook endpoint configured and verified in Stripe dashboard
- [ ] Webhook handler is idempotent (ProcessedEvent collection active)
- [ ] Checkout flow completes successfully (test with Stripe test cards)
- [ ] Customer portal accessible from settings
- [ ] Subscription status correctly gates premium features
- [ ] Payment receipt emails send on successful charge
- [ ] Payment failure emails send on failed charge
- [ ] Cancellation flow works and revokes access at period end

### Email

- [ ] Resend domain verified with DNS records
- [ ] All transactional emails rendering correctly
- [ ] From address matches verified domain
- [ ] Unsubscribe link present in marketing/onboarding emails
- [ ] Resend bounce/complaint webhook configured
- [ ] Bounced and suppressed email flags checked before sending

### Infrastructure

- [ ] Render service deployed and healthy
- [ ] Custom domain configured with SSL
- [ ] Health check endpoint responding at `/api/health`
- [ ] `render.yaml` blueprint committed
- [ ] `README.md` updated with app-specific details
- [ ] `.gitignore` in place — no `.env`, `node_modules`, or `dist/` committed
- [ ] Graceful shutdown handles SIGTERM (in-flight requests complete before exit)
- [ ] Structured logging active (pino in production, pino-pretty in dev)
- [ ] Request logging middleware active (excluding health checks)
- [ ] 404 page renders for unknown routes
- [ ] `robots.txt` accessible and blocks authenticated routes
- [ ] `sitemap.xml` accessible and lists only public pages
- [ ] `sitemap.xml` domain matches production URL

### Database

- [ ] All required indexes created (user email, stripeCustomerId, rate limit lookups)
- [ ] All TTL indexes active (sessions, email tokens, login attempts, processed events)
- [ ] Atlas tier upgraded to M10+ for production (continuous backups enabled)
- [ ] Backup strategy verified — can restore from point-in-time or snapshot

### Monitoring

- [ ] Sentry capturing errors in both frontend and backend
- [ ] PostHog tracking key conversion events (signup, verify, subscribe, cancel)
- [ ] Render health check alerts enabled

### Dependency Security

- [ ] Dependabot enabled for both client and server directories
- [ ] npm audit passing with no high/critical vulnerabilities
- [ ] CI pipeline includes audit step

### Legal & Privacy

- [ ] Terms of Service page live
- [ ] Privacy Policy page live (includes data export and deletion rights)
- [ ] Cookie consent banner (if required for your audience)
- [ ] Links in footer and registration form
- [ ] Data export endpoint functional
- [ ] Account deletion endpoint functional

### Final Smoke Test

- [ ] Register a new account with real email
- [ ] Verify email
- [ ] Log in
- [ ] Subscribe via Stripe Checkout
- [ ] Access premium features
- [ ] Manage subscription via Customer Portal
- [ ] Cancel subscription
- [ ] Verify access revoked at period end
- [ ] Reset password
- [ ] Log in with new password
- [ ] Export account data
- [ ] Promote your account to owner — verify admin dashboard accessible
- [ ] Verify non-owner account cannot access /admin
- [ ] Transfer ownership via admin endpoint — verify new owner has access, old owner does not
- [ ] Visit a nonexistent URL — verify 404 page renders
- [ ] Visit /robots.txt — verify it's accessible
- [ ] Visit /sitemap.xml — verify it lists public pages with correct domain
- [ ] Delete account — verify all data removed

---

## Quick Reference: New App Workflow

```
1.  GitHub      → "Use this template" to create new repo
2.  Local       → Clone, npm install, copy .env.example → .env
3.  Atlas       → Create new project + cluster (M10+ for production)
4.  Stripe      → Create new product + price objects
5.  Resend      → Add sending domain + verify DNS + configure bounce webhook
6.  Turnstile   → Create new site in Cloudflare dashboard
7.  Build       → Strip placeholder pages, build your app-specific features
8.  README      → Update README.md with app name, description, and specifics
9.  Test        → Write tests for new features, run full test suite (hard gate)
10. Render      → Deploy via Blueprint, set env vars, configure domain
11. Owner       → Register your account, promote to owner via CLI script or Atlas
12. Stripe      → Point webhook to production API URL, switch to live keys
13. Sentry      → Create new project, add DSN to env vars
14. PostHog     → Create new project, add API key to env vars
15. Dependabot  → Verify enabled on new repo
16. Legal       → Generate/customize ToS + Privacy Policy
17. Launch      → Run the launch checklist above
```
