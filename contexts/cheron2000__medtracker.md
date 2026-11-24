# Repository Context: cheron2000/medtracker

> Reviewed on 2026-11-24. The recursive Git tree contained 45 entries and was not truncated. Review covered the top-level README and V2 artifact, backend README/package/configuration, Express entry point, database pool and migration, auth/validation middleware, auth/medicine/history/fired routes, frontend Vite package and React app, and API client. This is a source review; no server, database, browser, push-notification flow, build, or test suite was executed.

## 1. Executive summary

`cheron2000/medtracker` is a medication schedule and dose-adherence tracker. It has evolved from a single-file React prototype (V2) into a split React/Vite frontend and Node.js/Express/PostgreSQL backend (V3 / Layer 1).

Current implemented capabilities:
- User registration/login with bcrypt password hashes and JWT bearer tokens.
- Per-user medicine CRUD, including soft deletion.
- Dose history with taken/missed statuses and pagination.
- Server-side fired-reminder keys for basic cross-session deduplication.
- Browser-side clock polling, audible alerts, notification permission, pre-scheduled browser notification timers, and snoozing.
- PostgreSQL persistence with indexes and timestamp triggers.

**Main architectural gap:** the top-level README describes reliable background scheduling with BullMQ and web-push notifications that work while the app is closed. The inspected backend does not implement BullMQ, a worker, a scheduler, a push-subscription API, or web-push delivery. Actual reminders are driven by the React app while it is running. This means the central "remind me even if the app is closed" promise is not currently delivered.

The foundation is useful, and the backend has a reasonable separation of routes, middleware, and database access. However, scheduling, time-zone correctness, data validation, and reliability need work before users should depend on it for medication adherence.

## 2. Verified repository metadata

| Field | Observed value |
|---|---|
| Repository | `cheron2000/medtracker` |
| Visibility | Public |
| Default branch | `main` |
| GitHub size | 122 KB |
| Archived | No |
| Recursive tree | 45 entries; complete |
| Frontend | React 19 + Vite 8 |
| Backend | Node.js 18+ / Express 4 |
| Database | PostgreSQL via `pg` |
| Authentication | bcryptjs + JWT |
| Security middleware | Helmet, CORS, express-rate-limit |
| Main backend port | 3001 by default |
| Reminder implementation | Browser-side React timers and polling |
| Server-side scheduler | Not present in inspected source |
| Push delivery | Not present in inspected source |

Repository structure is unusual: the backend source is nested under `medtracker-backend/medtracker-backend/`, and the frontend under `medtracker-frontend/`. A backend ZIP archive is also tracked at the root. The repository includes a separate top-level `MedTrackerV2.jsx` prototype.

## 3. Product evolution and source-of-truth map

### V2 prototype
File: `MedTrackerV2.jsx`.

- Single-file React implementation with embedded CSS and charting.
- Local persistence through `window.storage`, described as an Artifact persistence API.
- Sample medicine data is embedded in the component.
- Browser clock polling checks medicine times.
- Browser notifications and snooze timers operate within the page/session.
- This file is a prototype, not a normal standalone browser app unless the environment supplies the custom `window.storage` API.

### V3 frontend
Files: `medtracker-frontend/src/App.jsx`, `package.json`.

- React/Vite application.
- Login/register screens.
- JWT saved in browser `localStorage`.
- API requests use `VITE_API_URL` or default to `http://localhost:3001`.
- Medicines, history, and fired keys load from the backend.
- Reminder detection and notifications still run in the browser.

### Backend Layer 1
Directory: `medtracker-backend/medtracker-backend/`.

- Express REST API.
- PostgreSQL schema and migration script.
- JWT auth and per-user data isolation.
- Medicines, history, and fired-key routes.
- No scheduler or push-delivery worker in the inspected implementation.

Use V3 frontend + Layer 1 backend as the active implementation baseline. Treat the V2 artifact and root ZIP as legacy/packaging artifacts until their contents are reconciled.

## 4. Architecture

```text
React 19 + Vite frontend
  |
  | fetch / JSON / Authorization: Bearer JWT
  v
Express API (port 3001)
  |
  +-- Helmet / CORS / rate limiting / JSON parser
  |
  +-- /auth
  |     +-- register / login
  |     +-- GET/PATCH current user
  |
  +-- /medicines
  |     +-- list / create / read / update / soft-delete
  |
  +-- /history
  |     +-- paginated history / log dose / delete
  |
  +-- /fired
        +-- get / insert / purge reminder keys
  |
  v
PostgreSQL
  +-- users
  +-- medicines
  +-- dose_logs
  +-- fired_keys

Reminder execution currently remains in the browser:
  React clock tick -> compare current HH:MM -> create alert
      -> optional browser Notification
      -> POST fired key / POST dose history
```

## 5. Backend components

### 5.1 Express application

File: `src/index.js`.

- Uses Helmet security headers.
- CORS allowlist comes from comma-separated `CORS_ORIGINS`.
- Allows requests without an Origin header (useful for curl/Postman).
- JSON body limit is 100 KB.
- Global rate limit: 300 requests per 15 minutes.
- Auth routes receive a tighter 20 requests per 15 minutes limiter.
- Exposes `GET /health`.
- Registers auth, medicines, history, and fired routes.
- Starts the HTTP listener and schedules a 24-hour fired-key purge interval.

The entry point calls `app.listen()` at module load even though it exports `app` for testing. This makes importing it from a test process start a real listener and complicates isolated integration tests. Split app construction from the server bootstrap.

### 5.2 Authentication

Files: `src/routes/auth.js`, `src/middleware/auth.js`.

- Registration lowercases email and hashes passwords with bcryptjs cost 12.
- Login compares submitted password with stored hash.
- JWT includes `sub`, email, and timezone; default expiry is seven days.
- Protected routes validate bearer tokens and attach `req.user`.
- User response omits password hash.

Security and correctness improvements:
- Enforce a strong, non-placeholder `JWT_SECRET` at startup; the example file contains `replace_me_with_64_byte_hex_string`. The source currently passes the environment value directly to `jwt.sign/verify` and does not show a startup guard against missing or weak secrets.
- Normalize email consistently before querying (registration lowercases, login lowercases but does not trim).
- Validate email/password types before calling string methods; malformed JSON values can currently lead to 500 responses.
- Validate and canonicalize IANA time zones rather than accepting arbitrary strings.
- Consider refresh/revocation strategy and short-lived access tokens for a production service.
- Add account deletion and password reset flows before real users rely on it.

### 5.3 Database

Files: `src/db/pool.js`, `src/db/migrate.js`.

Tables:
- `users`: UUID, unique email, password hash, timezone, display name, timestamps.
- `medicines`: user ownership, name, dosage, JSONB times, color, notes, active flag.
- `dose_logs`: user and medicine IDs, denormalized medicine name/dosage, scheduled time, status, snoozed flag, logged timestamp.
- `fired_keys`: user ID, unique reminder key, date, created timestamp.

Good choices:
- Parameterized SQL is used for user-supplied values.
- Most user-owned queries constrain by `user_id`.
- Medicine deletion is soft-delete, preserving history.
- Useful indexes exist on user IDs, active medicines, history date, and fired-key date.
- Foreign keys cascade on account deletion.
- Updated-at triggers are installed for users and medicines.

Concerns:
- The migration is a single large SQL string, not a versioned migration system. Re-running it is mostly idempotent but future schema evolution needs tracked migration versions.
- `migrate:down` drops all application tables and the timestamp function. It is destructive and must never be run against production without explicit safeguards/backups.
- PostgreSQL SSL uses `rejectUnauthorized: false` in production or when the URL contains `render.com`. This disables certificate verification and weakens transport identity validation. Configure the correct CA and keep certificate verification enabled.
- No connection-pool sizing, statement timeout, idle timeout, or graceful shutdown handling is visible.
- The fired-key cleanup interval runs in every backend process; multi-instance deployment creates duplicate cleanup jobs (mostly harmless but unnecessary).

### 5.4 Medicine routes

File: `src/routes/medicines.js`.

- Router applies `requireAuth` to all routes.
- Lists only active medicines owned by the user.
- Creates medicines with unique, sorted time values.
- Reads and updates by both medicine ID and authenticated user ID.
- Soft-deletes by setting `active = FALSE`.

Validation weaknesses:
- `validateMedicine` checks non-empty name/dosage, at least one valid HH:MM time, and optional hex color.
- It does not impose maximum string lengths or maximum number of reminder times.
- PATCH validates times only when supplied; color is not validated on PATCH.
- PATCH silently ignores unknown properties and uses truthiness for name/dosage, making explicit empty values behave like omitted fields.
- Notes type/length and `active` are not part of the update schema.
- Time arrays are not bounded, and duplicate times are deduplicated only after validation.

Implement a schema-based allowlist for POST/PATCH and validate all supplied fields consistently.

### 5.5 Dose history

File: `src/routes/history.js`.

- Supports date filtering, limit/offset pagination, and total count.
- Verifies that the medicine belongs to the authenticated user before inserting a dose log.
- Stores a snapshot of medicine name/dosage for historical display.
- Supports deleting a single log or clearing all of the current user's history.

Issues:
- `validateDoseLog` does not require `scheduled_time`, but the database column is `NOT NULL`. Missing time therefore passes application validation and fails at the database, returning 500 instead of 400.
- `snoozed` is not checked to be boolean.
- Date query parameters are not validated before SQL date casting; malformed dates return a generic 500.
- Limit/offset accept negative values; limit is capped above but not clamped below.
- The history date filter explicitly uses UTC, while frontend schedule times are local wall-clock times. This creates confusing day boundaries for users outside UTC.
- No idempotency key protects dose-log creation, so client retries can create duplicate dose records.
- A history entry can be logged with a valid owned medicine even if the medicine is inactive; decide whether that is intended.

### 5.6 Fired-key routes

File: `src/routes/fired.js`.

- Per-user reminder keys stored with unique `(user_id, key)`.
- Duplicate inserts are ignored with `ON CONFLICT DO NOTHING`.
- Supports date lookup and purging old keys.
- Nightly backend interval also deletes keys older than two days.

Issues:
- `POST /fired` does not verify that `keys` is an array. A string is iterable and will be processed character-by-character.
- No maximum batch size or key length is enforced.
- Date validation checks only a YYYY-MM-DD shape, not that the date is a real calendar date.
- The route reports keys as inserted even when the database ignored them as duplicates.
- This is not an atomic reminder-claim operation: two browsers can both fetch the same empty set, both decide to alert, and then both submit the same key. The unique constraint prevents duplicate rows, not duplicate notifications.
- Purge is user-scoped for the route; nightly global purge runs in the server process.

For true cross-device deduplication, add an atomic claim endpoint that performs INSERT ... ON CONFLICT and returns whether the caller won the claim. The client must only display the alert if it won.

## 6. Frontend and reminder behavior

File: `medtracker-frontend/src/App.jsx`.

### 6.1 Implemented frontend behavior
- Auth token stored as `mt_token` in localStorage.
- On mount, calls `GET /auth/me` to validate an existing token.
- On user authentication, loads medicines, up to 200 history records, and today's fired keys.
- A one-second interval updates the UI clock.
- A minute-level effect compares local HH:MM with each medicine's schedule.
- When a match is found, it creates an alert, plays a buzzer, best-effort posts a fired key, and optionally sends a browser notification.
- User can acknowledge taken/missed, snooze, add/delete medicines, clear history, and log out.

### 6.2 Critical reminder reliability gap

The browser is the scheduler. The reminder check runs only while the React app is active. The pre-scheduled notification timers are also created in the browser process and are cleared when the component unmounts or the page is closed. Browser timers are throttled in background tabs and are not a reliable alarm service.

The backend has no scheduled jobs and no push subscription/delivery endpoints. The top-level README's promise that BullMQ and web-push deliver reminders when the tab is closed is therefore not implemented in the inspected source.

**Consequence:** if the user closes the tab/browser, loses the browser process, or the device suspends the page, reminders may not fire. Do not market this version as reliable closed-app reminders.

### 6.3 Time-zone / date bug

Frontend `dateKey(d)` uses `d.toISOString().slice(0, 10)`, which is the UTC calendar date, while schedule matching uses local `getHours()` and `getMinutes()`. The backend stores an IANA timezone on the user but the history route filters UTC dates.

In time zones such as Asia/Kolkata, local midnight does not match UTC midnight. As a result:
- Today's dashboard/history grouping can switch dates at the wrong local time.
- Fired-key date prefixes can refer to the previous or next local day.
- A reminder around local midnight can be grouped under the wrong date.
- Backend user timezone is not consistently used for date filtering.

Use the authenticated user's IANA timezone consistently. Prefer server-generated local date keys or explicit UTC timestamps plus timezone-aware conversion, and document the rule for scheduled wall-clock times.

### 6.4 Snooze and acknowledgement persistence
- Snooze uses in-memory `setTimeout`; it is lost on page reload/close.
- Snoozed state is persisted only if the user later acknowledges the dose.
- Optimistic history entry replacement matches by `opt_` prefix and medicine ID, not by a unique request correlation ID. If two pending acknowledgements for the same medicine overlap, one response may replace the wrong optimistic row.
- History POST is best-effort; failure only shows a toast, with no durable retry queue.
- The frontend can optimistically remove a medicine and reload on failure; other mutation paths have less complete rollback handling.

### 6.5 Authentication and token storage
- JWT is stored in localStorage, which is exposed to any successful same-origin XSS.
- Use a strict Content Security Policy, escape/render user-provided text safely, and consider an HttpOnly Secure SameSite cookie architecture if the deployment model supports it.
- Logout clears local state and token but does not revoke an already issued JWT.
- Auth screen and main app implement API calls inline; a separate `src/api.js` duplicates the API client and is not the implementation used by `App.jsx`.

## 7. Documentation and configuration mismatches

1. **BullMQ/web-push mismatch:** root README describes BullMQ, web-push, background recurring jobs, and closed-tab reminders. Backend `package.json` has neither BullMQ nor web-push, and no worker/subscription route exists.
2. **Roadmap mismatch:** root README still marks multi-user auth as incomplete, while Layer 1 backend already implements registration/login and per-user data.
3. **Frontend environment variable mismatch:** backend README suggests `REACT_APP_API_URL`; the actual Vite frontend reads `VITE_API_URL`. The separate unused `src/api.js` also uses `process.env.REACT_APP_API_URL`, which is not the Vite convention.
4. **Frontend storage claim mismatch:** V3 comments say `window.storage` remains as a local optimistic cache, but the inspected V3 app uses API responses and localStorage only for JWT; no `window.storage` use was found in that app.
5. **API client duplication:** `src/api.js` is a separate older API client with different environment configuration and no usage in the active app.
6. **README separation:** frontend README is still the generic Vite starter guide, not MedTracker setup/integration documentation.
7. **Backend nesting:** backend README quick-start commands assume the working directory is the nested backend package, but root README setup commands describe `cd medtracker` and `npm install` at repository root. There is no root package.json in the inspected tree.

## 8. Security and privacy assessment

Positive controls:
- Password hashing with bcryptjs.
- Parameterized SQL.
- JWT verification on data routes.
- Per-user filters on medicine/history/fired data.
- Helmet, CORS allowlist, JSON body limit, and rate limiting.
- Soft-delete of medicines retains history.

Gaps and recommendations:
- Reject missing/placeholder JWT secret during startup.
- Keep TLS certificate verification enabled for PostgreSQL.
- Add robust validation for all request bodies, query parameters, UUIDs, dates, and batch sizes.
- Use a real atomic reminder-claim endpoint.
- Add account deletion and data export controls.
- Add audit logs for authentication and sensitive account operations without logging secrets.
- Add explicit data retention and backup/restore policy; medication schedules and adherence history are sensitive personal data.
- Set production CORS origins explicitly; avoid permissive wildcard configuration.
- Configure reverse-proxy trust and rate-limit keys correctly when deployed behind a proxy.
- Add a Content Security Policy and assess XSS exposure because the JWT is in localStorage.
- Ensure secrets, database URLs, and real user data are never committed. The checked-in `.env.example` contains placeholders, not a real secret.

This review did not identify a direct cross-user SQL access path in the inspected medicine/history/fired routes: queries consistently include the authenticated user ID. Still, add integration tests that attempt cross-account ID access and verify 404/denial.

## 9. Testing and CI

Observed:
- Frontend package defines `dev`, `build`, `lint`, and `preview` scripts.
- Backend package defines `start`, `dev`, `migrate`, and destructive `migrate:down`.
- No test script or test directory appeared in the reviewed tree.
- No GitHub Actions workflow appeared in the recursive tree.
- No BullMQ worker, web-push service, service worker, or push-subscription schema/route appeared.

No npm install, frontend build, lint, backend startup, migration, API integration test, or browser notification test was run.

Recommended minimum test suite:
- Auth: registration validation, duplicate email, login, JWT expiry/invalid secret.
- Ownership: user A cannot read/update/delete user B's medicines or logs.
- Validation: malformed times, invalid date, missing scheduled time, invalid snoozed type, huge arrays, negative pagination.
- Fired keys: concurrent claim requests result in exactly one winner.
- Time zones: UTC, Asia/Kolkata, DST zones, midnight rollover.
- Reminder lifecycle: reload, tab close, browser suspension, offline/online, duplicate tabs.
- Migration: fresh database, rerun, rollback safeguards.
- Frontend: API failure rollback, duplicate optimistic updates, token expiry/logout.

## 10. Prioritized roadmap

### P0 — implement reliable reminders
1. Decide the delivery target: browser-open-only reminders versus closed-tab/device reminders.
2. For closed-tab delivery, add a service worker, push subscription persistence, VAPID key management, and a backend scheduler/worker (BullMQ + Redis or a suitable managed job system).
3. Persist scheduled reminder jobs server-side with timezone-aware recurrence.
4. Implement retry, snooze, missed-dose, and idempotency semantics.
5. Add an atomic fired/claim operation so multiple tabs/devices cannot all trigger the same alert.

### P0 — correct local-date and timezone behavior
1. Use the user's stored IANA timezone for schedule and date calculations.
2. Stop using UTC ISO date slices as local calendar dates.
3. Make history date filters timezone-aware and consistent with frontend grouping.
4. Add tests for Asia/Kolkata and midnight transitions.

### P1 — validation and backend reliability
1. Add schema validation for every route and query parameter.
2. Require scheduled_time and validate snoozed boolean.
3. Clamp pagination and validate date/UUID formats.
4. Bound medicine name/dosage/notes, time-array length, fired-key batch size, and key length.
5. Split app construction from server startup.
6. Add graceful shutdown, pool tuning, statement timeouts, and versioned migrations.
7. Keep PostgreSQL TLS certificate verification enabled.

### P1 — security and account lifecycle
1. Enforce a strong JWT secret at startup.
2. Add password reset, account deletion, and data export.
3. Harden browser XSS defenses and consider HttpOnly cookie auth.
4. Add rate-limit configuration for production proxies and test account isolation.
5. Document data privacy, retention, and backup behavior.

### P2 — consolidate project and docs
1. Remove the duplicate unused `src/api.js` or make it the single shared client and use Vite env conventions.
2. Reconcile the root README with actual Layer 1 capabilities and mark BullMQ/web-push as future work until implemented.
3. Add proper frontend setup docs and a root-level development guide.
4. Reconcile or remove the legacy V2 file and tracked backend ZIP after confirming which artifact is canonical.
5. Add CI for lint/build, backend tests, migration checks, and dependency/security scanning.

## 11. Coding-agent handoff

Treat `medtracker-frontend/src/App.jsx` plus `medtracker-backend/medtracker-backend/src/` as the current V3/Layer 1 implementation. The V2 artifact and the top-level feature claims are not reliable indicators of backend capability.

The first major engineering task is not adding more dashboard UI: it is implementing a real server-side reminder delivery system and making date/time semantics correct. At present, the browser is the scheduler and reminders are not guaranteed when the app is closed. In parallel, fix request validation and require a strong JWT secret. Add concurrency and timezone tests before calling the reminder system reliable.

## 12. Validation status and limitations

- Repository metadata and complete recursive tree: inspected (45 entries).
- Top-level README, V2 prototype, backend README/package/config: inspected.
- Express app, database pool/migration, auth/validation middleware, all four route modules: inspected.
- V3 frontend app, Vite package, frontend README, duplicate API client: inspected.
- Runtime, database migrations, builds, lint, tests, notifications, and push delivery: not executed.
- ZIP archive contents: not extracted/reviewed.
- Full UI and every component/helper: not line-by-line reviewed.
- Deployment environment and production database configuration: not inspected.
