# Repository Context: cheron2000/IEEE-Website

> Reviewed on 2026-11-24 from the public GitHub repository's complete recursive tree (180 entries, not truncated). Static inspection covered documentation, backend models/routes/middleware/configuration/tests, frontend App Router pages/components/API/auth code/tests, and deployment manifests. No dependency installation, test execution, build, live API request, database connection, or deployment verification was performed.

## 1. Executive summary

**MMIT IEEE Official Website** is a full-stack website and content-management system for the IEEE Student Branch at Marathwada Mitra Mandal's Institute of Technology (MMIT), Pune.

It has three main parts:

1. **Public website:** Next.js App Router application with cinematic scroll-driven design, events, members, achievements, updates, and event detail pages.
2. **Admin CMS:** Protected dashboard for managing members, events, achievements, updates, and site settings, including Cloudinary image uploads.
3. **Backend API:** Express 4 + MongoDB/Mongoose, JWT access tokens, rotating refresh-token cookies, role-aware middleware, Zod request validation, generic content CRUD routes, and Cloudinary uploads.

This is substantially more complete than a static portfolio/demo: the backend has persistent models, admin authentication, content CRUD, and a public/private content distinction. The generic content router is a meaningful maintainability improvement over duplicated CRUD implementations.

**Most important findings:**
- The checked-in backend environment example does not match the environment variable names consumed by the code. Following it literally can prevent database connection and break JWT/CORS configuration.
- The upload-specific rate limiter is placed before Multer and skips requests when `req.file` is absent. At that point Multer has not populated `req.file`, so the intended 10 uploads / 15 minutes limit is effectively bypassed.
- Request validation middleware logs complete request and validated bodies. This can leak admin-entered personal data or other sensitive content into application logs.
- The generic content router validates multipart form data before normalizing array fields. Event gallery form values represented as JSON/comma-separated strings can fail validation before the normalizer runs.
- Public settings endpoints return every setting value. This is acceptable only if the collection is strictly limited to public configuration.
- Deployment instructions are stale in places and include an unsafe recommendation to whitelist MongoDB Atlas from `0.0.0.0/0`.
- There are some backend/frontend unit and integration tests plus a Playwright smoke suite, but no CI workflow was found and tests were not executed.

## 2. Repository metadata

| Field | Observed value |
|---|---|
| Repository | `cheron2000/IEEE-Website` |
| Visibility | Public |
| Default branch | `main` |
| GitHub size | 270 KB |
| Archived | No |
| Recursive tree | 180 entries; complete |
| Frontend | Next.js 16, React 18, TypeScript, Tailwind CSS |
| Backend | Node.js, Express 4, CommonJS |
| Database | MongoDB + Mongoose 8 |
| Authentication | JWT access + refresh cookie |
| Media | Cloudinary + Multer |
| Validation | Zod + express-validator |
| Backend tests | Vitest, Supertest, fast-check |
| Frontend tests | Vitest, Testing Library, Playwright |
| CI workflows | No `.github/workflows/` files found |

## 3. Architecture and data flow

```text
Public visitor
  └── Next.js App Router
       ├── Home
       ├── Events ──> Express API ──> MongoDB
       ├── Members ─> Express API ──> MongoDB
       ├── Achievements ────────────> MongoDB
       └── Updates ────────────────> MongoDB

Admin browser
  └── Next.js /admin
       ├── AdminAuthContext
       ├── access token held in JS module memory
       └── httpOnly refresh cookie
             └── Express /api/auth
                    ├── Admin model
                    ├── JWT verification
                    └── refresh-token hashes in MongoDB

Admin content manager
  └── /api/{members,events,achievements,updates}
       ├── requireAuth
       ├── Multer upload -> Cloudinary
       ├── Zod validation
       ├── generic contentRouterFactory
       └── Mongoose model -> MongoDB
```

The public Next.js pages fetch content server-side through `frontend/src/lib/api.ts`. The admin client uses a separate request helper in `adminApi.ts`, adding the in-memory bearer access token and browser credentials for the refresh cookie.

## 4. Backend architecture

### Entry point and middleware
- `backend/src/server.js` loads dotenv, configures Express, connects MongoDB, and mounts routes.
- Helmet security headers.
- CORS with credentials and configured origin list.
- General API rate limit: 120 requests per minute.
- JSON body limit: 1 MB.
- Cookie parser and Morgan request logging.
- Health endpoint: `GET /api/health`.
- Central not-found and error handlers.
- Exports the Express app and starts listening only when invoked directly, allowing Supertest imports.

### Models
- **Admin:** name, email, password hash, role (`superadmin` / `editor`), hashed refresh-token sessions, last login. Password hashing uses bcryptjs with 12 rounds.
- **Member:** name, designation, academic year, photo URL/public ID, LinkedIn URL, order, current/counsellor flags.
- **Event:** title, generated slug, date, summary/description, cover image, gallery, category, attendee count, published flag.
- **Achievement:** title, description, date, image, recipient names, published flag.
- **Update:** title/body/date, external link, image, pinned and published flags.
- **Settings:** unique key, mixed value, optional description.

Indexes exist for common member/event/achievement/update list ordering and published queries.

### Generic content router
`contentRouterFactory.js` centralizes the standard CRUD behavior for four collections:
- Public list returns published records, unless a valid optional admin bearer token is supplied.
- Public detail lookup supports Mongo ObjectId or slug where applicable.
- Authenticated count endpoint.
- Authenticated create/update/delete with optional Cloudinary image upload.
- Optional Zod create/update schemas.
- Optimistic concurrency check using Mongoose `__v`.
- Public projections prevent selected internal fields from being returned.

This is a good abstraction and reduces copy/paste. It should remain carefully tested because a single factory change affects all four resources.

## 5. Authentication and session design

### Implemented flow
- Login validates email/password with express-validator and uses a dedicated 10-attempt / 15-minute rate limit.
- Password comparison uses bcrypt.
- Access token is returned in JSON and kept in frontend module memory rather than localStorage.
- Refresh token is placed in an httpOnly cookie with `sameSite: strict`, `secure` in production, and path `/api/auth`.
- Only SHA-256 hashes of refresh tokens are stored in the Admin document.
- Refresh validates JWT, checks its hash against the stored allowlist, invalidates the old token, and issues a replacement.
- Admin sessions are capped at five refresh tokens.
- Logout removes the presented refresh-token hash.
- `requireAuth`, `optionalAuth`, and `requireRole` middleware are available.

### Risks and recommended hardening
1. Validate that both JWT secrets are present and sufficiently strong at server startup. The current environment example does not define the names consumed by the code.
2. Add CSRF/origin protections for cookie-authenticated refresh/logout operations. SameSite strict helps, but should not be the only control.
3. Consider refresh-token rotation reuse detection and concurrent refresh handling. Two simultaneous refreshes using the same cookie can race; one may invalidate the token the other is trying to use.
4. Add explicit role authorization to operations intended only for superadmins. `requireRole` exists, but the inspected content/settings routes use `requireAuth` only. If editors are meant to have full CMS rights, document that policy; otherwise enforce the intended role matrix.
5. Ensure frontend logout handles network failure without leaving the server-side refresh session valid indefinitely; consider a clear recovery path.
6. Add tests for cookie flags, rotation, replay, logout, role restrictions, and CSRF/origin behavior.

The browser-side admin layout guard is a usability/navigation guard, not the security boundary. The backend `requireAuth` checks are the actual authorization boundary, which is the correct general design.

## 6. High-priority defects and security observations

### P1 — environment variable mismatch (startup/configuration blocker)
The code reads:
- `MONGO_URI` in `config/db.js`, `seed.js`, and `seedSettings.js`.
- `JWT_ACCESS_SECRET`, `JWT_REFRESH_SECRET`, and optional expiry variables in token/auth code.
- `CORS_ORIGINS` in `server.js`.

But `backend/.env.example` defines:
- `MONGODB_URI`
- `JWT_SECRET`
- `CORS_ORIGIN`

The README's environment section uses the newer JWT access/refresh and plural CORS names, but uses `MONGODB_URI`, still inconsistent with the code's `MONGO_URI`. The deployment guide also uses older `JWT_SECRET` and singular `CORS_ORIGIN`.

**Impact:** Copying the checked-in example and starting the backend leaves `process.env.MONGO_URI` undefined; the connection attempt fails. JWT signing/verification also cannot use the intended secrets, and CORS origin configuration may be empty.

**Fix:** Choose one canonical naming convention, update the code/example/README/deployment guide together, validate required variables at startup, and add a config test that fails clearly when a required value is absent.

### P1 — upload rate limit bypass due middleware order
In the generic content router, middleware is ordered as:
1. `requireAuth`
2. `uploadRateLimiter`
3. set upload folder
4. `upload.single("image")`

The upload limiter's `skip` callback returns true when `req.file` is absent. Since Multer populates `req.file` only after the limiter has run, the limiter skips the request before every upload. The stated 10 uploads / 15 minutes protection is therefore not operating as intended.

**Fix:** Put the upload limiter on the upload route without relying on `req.file`, or determine whether the request is an upload using route/method/content-type. Use a stable authenticated admin ID or trusted IP key, and add a test proving the 11th upload is rejected.

### P1 — sensitive request bodies written to logs
`validateRequestBody()` logs both the raw request body and validated body. The content router also logs create payloads. These values may include personal data, member details, event content, external links, and future fields that should not be logged.

**Fix:** Remove body logging by default. Log request IDs, route, validation outcome, and safe field names only. If debugging requires payload inspection, use explicit redaction and a short-lived opt-in debug mode.

### P1 — array normalization happens after schema validation
The content router calls the Zod validator before `normalizeArrayFields()`. The Event route config declares `gallery` as an array field that may arrive from the CMS as a JSON or comma-separated string. The event schema expects an array of gallery objects. The normalizer that converts the string to an array runs only after validation, so valid form input can be rejected before conversion.

**Fix:** Normalize multipart fields before Zod parsing, or define Zod preprocessors that accept the exact form encoding. Add an integration test for event create/update with gallery JSON and empty gallery values.

### P1/P2 — public settings endpoint exposes every setting
`GET /api/settings` and `GET /api/settings/:key` are public and return stored values without an allowlist. This is safe only while the collection contains public display settings. A future admin setting containing secrets, private integration config, or internal metadata would be exposed automatically.

**Fix:** Separate public settings from private configuration, or maintain an explicit allowlist of public keys and projections. Never store credentials in this collection.

### P2 — validation error compatibility
The validation middleware formats errors from `error.errors`. Zod 4 error structure/API should be verified against the pinned package version; use the supported Zod 4 issue collection consistently and test malformed requests. A mismatch could turn validation failures into generic server errors or empty detail arrays.

### P2 — upload validation and lifecycle
- Multer checks the declared MIME type and Cloudinary allowed extensions; client-provided MIME values are not authoritative. Validate file signatures where feasible.
- The old Cloudinary image is deleted before the MongoDB update is saved. If the database save then fails, the existing record can point to a deleted image.
- On create/update failure after a new Cloudinary upload, the newly uploaded asset may be orphaned.
- Delete operations call Cloudinary destroy asynchronously without awaiting or recording failures.
- Add compensation/cleanup and operational logging, and make asset deletion retryable.
- The 5 MB limit is enforced by Multer; confirm Cloudinary transformation behavior and allowed formats remain aligned.

### P2 — MongoDB connection and operational readiness
- The DB connection failure exits the process, which is reasonable for startup failure.
- The code logs database host and errors to console; use structured logging and avoid exposing connection strings or credentials.
- Add graceful shutdown for HTTP server and Mongoose connection.
- Add readiness checks that distinguish process health from database readiness.
- The deployment guide recommends Atlas IP allowlist `0.0.0.0/0`; prefer provider egress IP allowlisting or private networking when available.

## 7. Frontend architecture

### Public site
Next.js App Router pages include:
- Home
- Events list and event detail
- Members
- Achievements
- Updates
- Welcome page

The homepage is a large client component with scroll-linked word reveals, count-up statistics, animated particles, motion/tilt cards, and cinematic sections. Supporting components include Navbar, Footer, Reveal, SignalTrace, SmoothScroll, BracketButton, EventCard, MemberCard, AchievementCard, and UpdateCard.

Public list pages fetch data server-side and pass results to client components. Event detail pages generate page metadata from fetched event content. The frontend API helper caches content for 60 seconds and falls back to empty/null data if the API is unavailable, avoiding hard failures during initial setup.

**Trade-off:** graceful empty fallbacks improve resilience but can conceal a misconfigured API or backend outage. Add visible observability (server logs/monitoring) and a non-production diagnostic mode so failures are not mistaken for genuinely empty content.

### Admin CMS
- Login page and AdminAuthContext manage sign-in/session restoration.
- Dashboard loads counts for members/events/achievements/updates.
- Generic ResourceManager handles CRUD forms, optional image upload, list reload, and deletion confirmation.
- Dedicated pages exist for members, events, achievements, updates, and settings.
- Admin access token is held in module memory; refresh token is an httpOnly cookie.
- Generic admin API helper retries a request once after a 401 by refreshing the access token.

The frontend's route guard is client-side only; API routes must continue enforcing auth independently. Avoid adding sensitive data to client state or logs.

### Frontend issues to review
- The `frontend/package.json` lint script is `next lint`. Check compatibility with the installed Next.js 16 CLI; use ESLint CLI directly if that command is no longer supported.
- Next.js is declared at 16.x while React and React DOM are declared as `^18`. Verify the exact Next.js peer dependency range and lockfile resolution before upgrading or deploying.
- `@studio-freight/lenis` and `lenis` are both declared; determine whether both are used or remove the obsolete duplicate.
- The event detail page uses `next/image`; remote patterns include Cloudinary and legacy/test hosts. Remove unused test/placeholder hosts from production config.
- Add explicit accessible labels/keyboard/focus tests for motion-driven navigation, scroll interactions, and admin modal/form flows.
- Respect reduced-motion preferences consistently. The custom hook is used in the homepage but not necessarily across every animated component.

## 8. Testing and quality gates

### Backend
- Vitest config includes V8 coverage thresholds of 80% for lines, functions, branches, and statements.
- Unit tests exist for auth middleware and generic content router.
- Integration health test uses Supertest and a MongoDB test database.
- Integration setup connects to `MONGODB_URI` or defaults to local `mmit-ieee-test`, then deletes all collections after each test.
- fast-check is used in property-based content router tests.

**Important:** Integration test setup is destructive to every collection in the configured test database. Ensure `MONGODB_URI` cannot point at production or a developer's normal database. Add an explicit database-name safety assertion before connecting and before collection cleanup.

### Frontend
- Vitest and Testing Library tests include admin API behavior and dashboard counts.
- Playwright homepage smoke tests cover title, navigation, route change, mobile viewport, and console errors.
- Playwright config targets Chromium, Firefox, and WebKit; it starts `npm run dev` and assumes port 3000.
- Tests/builds were not run. Browser binaries and test server requirements were not verified.

### CI
No GitHub Actions workflows were found. The configured tests and coverage thresholds therefore are not shown to run automatically in GitHub.

**Recommended CI:** backend and frontend frozen installs, backend unit tests, integration tests against an isolated MongoDB service, frontend Vitest, Playwright Chromium smoke test, frontend production build, and lint/type checks.

## 9. Documentation and deployment gaps

### README
The high-level project structure and stack are broadly accurate, but setup instructions and environment names are not fully synchronized with the backend. The README says Node 18+ and npm, which is supported by package manifests, but confirm both apps' lockfiles remain current.

### DEPENDENCIES.md
Provides useful package rationale, but version claims appear stale/inconsistent with the actual manifests. Examples include backend Zod and frontend Next/React version references. Treat package manifests and lockfiles as the source of truth.

### DEPLOYMENT.md
- Documents Vercel frontend, Railway backend, MongoDB Atlas, and Cloudinary.
- Contains older variable names and a sample admin-creation flow.
- Suggests Atlas allowlist `0.0.0.0/0`, which is broader than necessary.
- Mentions a `/api/health` route, which does exist.
- Gives an example `next.config.js` although the repo uses `next.config.mjs`.
- Includes a manually inserted admin document example and a script example; the repo's actual seed script is different and uses environment variables.
- The documented optional GitHub Actions pipeline is not present in the repo.

Update docs after config names and deployment conventions are standardized. Do not paste real credentials into tracked files or seed scripts.

## 10. Recommended remediation order

### Immediate
1. Standardize `MONGO_URI`, JWT access/refresh secret names, and `CORS_ORIGINS` across code, examples, and docs.
2. Add startup environment validation and tests.
3. Fix upload rate limiter ordering/skip logic and add a regression test.
4. Remove raw request-body logging from validation/content routes.
5. Move array normalization before Zod validation or make schemas accept the actual multipart representation.
6. Add public-settings allowlisting.

### Next
7. Add explicit role policy and enforce superadmin-only operations where intended.
8. Add safe test DB name checks before integration-test collection cleanup.
9. Improve Cloudinary/MongoDB asset lifecycle compensation and deletion retry handling.
10. Add graceful shutdown, structured logs, and DB readiness behavior.
11. Review refresh rotation race/reuse handling and CSRF/origin protection.
12. Remove unused remote image hosts and resolve frontend dependency/lint compatibility.

### Engineering maturity
13. Add GitHub Actions CI for backend, frontend, typecheck/build, and browser smoke tests.
14. Add tests for all content route operations, Zod edge cases, file upload limits, publication visibility, settings exposure, auth refresh/logout, and role authorization.
15. Add pagination/search to public content endpoints if content volume grows; current generic list routes fetch all matching records.
16. Add admin audit trail for content changes, publication state, image replacement, and settings edits.

## 11. Coding-agent handoff

When making changes, preserve these architectural decisions:
- Keep public and admin API behavior separate: public routes return published-only projections; authenticated CMS routes can manage drafts.
- Keep generic content CRUD centralized in `contentRouterFactory.js`.
- Keep validation schemas close to their models/routes and align them with multipart form serialization.
- Keep raw refresh tokens out of MongoDB; store only token hashes.
- Keep access tokens out of localStorage; module-memory storage is intentional.
- Keep Cloudinary assets referenced by public ID so they can be replaced/deleted.
- Keep Next.js public pages server-rendered where possible; use client components for interactive presentation.

Before any production deployment, resolve the environment-variable mismatch and upload rate limiter defect, then run the full tests/build against isolated services.

## 12. Validation status and limitations

- Repository metadata and complete tree: inspected (180 entries).
- Backend server, configuration, middleware, models, schemas, routes, seed scripts, and tests: inspected.
- Frontend manifests, public pages, admin auth/API, resource manager, tests, and Playwright config: inspected.
- Deployment and dependency documentation: inspected.
- Test files exist; test suite not executed.
- CI workflow: none found.
- Install, lint, typecheck, frontend build, backend startup, database tests, Cloudinary upload, browser E2E, and deployment: not executed.
- Findings are static code-review conclusions; runtime exploitability and production configuration were not independently verified.
