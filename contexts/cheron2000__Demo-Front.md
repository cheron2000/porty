# Repository Context: cheron2000/Demo-Front

> Reviewed on 2026-11-24. GitHub repository metadata and the complete recursive tree (234 entries, not truncated) were inspected. Review covered the pnpm workspace, Express API, OpenAPI/code-generation packages, Drizzle database package, Space Mission Control frontend, mockup preview server, Replit deployment manifests, and project prompt notes. This was a static review: install, typecheck, build, runtime, browser, database, and deployment commands were not executed.

## 1. Executive summary

**Demo-Front is a Replit-oriented TypeScript monorepo containing two related but distinct deliverables:**

1. **Space Mission Control** — a polished, dark sci-fi themed React/Vite frontend demo with Home, Missions, Telemetry, and About pages. It uses Framer Motion, Tailwind CSS, Recharts, CSS/SVG orbital visuals, and a custom canvas/WebGL-like starfield implementation.
2. **A full-stack starter scaffold** — an Express 5 API server, PostgreSQL/Drizzle package, OpenAPI specification, generated React Query hooks, and generated API schema/types.

The frontend is the most complete user-visible artifact. It is a design/interaction demo, not an actual satellite/mission operations system:
- Mission records and statuses are hard-coded.
- Telemetry values, charts, latency, and command responses are generated locally with timers and Math.random().
- The API server only exposes a health-check endpoint.
- The frontend does not appear to call the API client or API server.
- The database schema is empty and no migrations are present.

The backend is currently a scaffold rather than an implemented product API. The README is stale and still describes a conventional server/client directory layout, npm commands, database migrations, and an .env.example that do not match the actual workspace.

**Highest-priority issues:**
- The API uses unrestricted CORS defaults. The only route is currently a non-sensitive health check, but this must be narrowed before adding authenticated or data-bearing endpoints.
- The Vite dev and preview servers bind to 0.0.0.0 and set allowedHosts: true. Keep this confined to trusted Replit/dev environments; do not expose an untrusted preview server.
- The post-merge script runs `pnpm --filter db push`, but the actual workspace package is named `@workspace/db`; the command appears to target a nonexistent package and would likely fail.
- Root README setup commands do not match the pnpm workspace scripts or actual package layout.
- There are no test files, CI workflows, environment examples, or database migrations in the tree.

## 2. Repository metadata

| Field | Observed value |
|---|---|
| Repository | `cheron2000/Demo-Front` |
| Visibility | Public |
| Default branch | `main` |
| GitHub size | 340 KB |
| Archived | No |
| Recursive tree | 234 entries; complete |
| Runtime/tooling | Node.js 24, pnpm workspaces, TypeScript 5.9 |
| Frontend | React 19 + Vite 7 |
| Backend | Express 5 |
| Database package | PostgreSQL + Drizzle ORM |
| API contract | OpenAPI 3.1 |
| Client generation | Orval 8.5.3 generated React Query hooks and Zod/types |
| Deployment | Replit application router / autoscale |
| Tests | No test files detected in repository tree |
| CI | No `.github/workflows/` files detected |

No `.env.example`, migration directory, or committed test files were found in the recursive tree. This does not prove no external Replit environment variables or hosted database exist; it only describes the checked-in repository.

## 3. Workspace layout

```text
Demo-Front/
├── artifacts/
│   ├── api-server/             # Express 5 API service
│   ├── mission-control/        # Space Mission Control React/Vite site
│   └── mockup-sandbox/         # Replit component preview/design service
├── lib/
│   ├── api-client-react/       # Generated React Query client + custom fetch
│   ├── api-spec/               # OpenAPI source + Orval configuration
│   ├── api-zod/                # Generated API types/schemas
│   └── db/                     # Drizzle/PostgreSQL connection and schema
├── scripts/                    # Workspace scripts and post-merge hook
├── attached_assets/             # Original design/product prompt notes
├── package.json                 # Root workspace scripts
├── pnpm-workspace.yaml          # Workspace packages/catalog/overrides
├── pnpm-lock.yaml
├── tsconfig.base.json
├── tsconfig.json
├── .replit
├── readme.md
└── replit.md
```

This is not the `server/` + `client/` structure described in README. The actual deployable app and services live under `artifacts/`, while reusable workspace libraries live under `lib/`.

## 4. Frontend: Space Mission Control

### Product intent
The attached project notes describe a cinematic space mission-control experience emphasizing smooth graphics, transitions, and premium visual design over a large backend. The intended navigation is:
- `/` — Home
- `/missions` — Mission cards, timeline, and detail panel
- `/telemetry` — Orbital visualization, metrics, chart, and simulated command center
- `/about` — Project and technology overview

### Technology
- React 19, Vite 7, TypeScript
- Tailwind CSS 4 and CSS custom-property theme
- Framer Motion for entrance, card, and panel transitions
- Recharts for telemetry chart
- Lucide and React Icons
- Three.js and React Three Fiber are declared dependencies, but the inspected primary pages use CSS/SVG for the orbital map; the starfield is a custom canvas implementation.
- Wouter client-side routing
- shadcn/Radix-style UI component collection is present, although the primary screens mostly use custom components.

### Page behavior

**Home**
- Large "SYSTEMS ONLINE" hero, animated entrance, navigation to Missions.
- Counter animation increments to six satellites and four missions.
- CSS orbital rings and decorative motion.
- Feature cards with fixed values such as telemetry rate and uptime.
- These figures are presentation copy, not observed backend metrics.

**Missions**
- Six hard-coded mission objects: Lunar Survey, Mars Relay, Deep Space Probe, Orbital Station, Solar Observer, Explorer X.
- Cards show status and progress; hover reveals crew/risk/duration.
- Clicking opens a slide-in detail panel with an animated circular progress ring.
- No API fetch or persistent mission data exists.
- The timeline currently marks 2023–2026, while the original prompt sketch described 2024–2027. The final code and original design notes are not perfectly synchronized.
- Mission cards are clickable `motion.div` elements rather than semantic buttons. Keyboard focus/activation and accessible names should be added. Hover-only detail content should also be reachable on touch and keyboard.

**Telemetry**
- A 1.5-second simulated connection sequence.
- Fuel, power, signal, temperature, and chart values fluctuate using Math.random() every 1.5 seconds.
- OrbitalMap draws six static satellite nodes with animated rotation, connection lines, and hover tooltips.
- Command buttons ("PING SATELLITE", "REFRESH SYSTEMS", "EMERGENCY SCAN") return locally generated messages after a timeout.
- No real telemetry stream, satellite API, command execution, or remote system connection is implemented. UI labels such as "LIVE" and "STREAM ACTIVE" are simulated states and should be clearly identified as demo data.

**About**
- Explains the project as a demonstration of modern interactive web experiences.
- Lists React, Three.js, GSAP, Framer Motion, Tailwind, and Recharts.
- Includes an architecture illustration and development timeline.
- GSAP and Three.js are described in the page's technology stack, but the main inspected page implementations use Framer Motion, CSS, and SVG; validate actual usage before presenting them as core runtime features.

### Starfield subsystem
`src/components/starfield.tsx` is a substantial custom visual component (about 34 KB):
- Quaternion math for orientation interpolation and device orientation handling.
- Thousands of stars, nebulae, comets, and motion/warp effects.
- Synthesized ambient audio using Web Audio oscillators.
- This is a custom rendering/interaction subsystem and is likely one of the most complex frontend files.
- Review its animation-frame lifecycle, event listener cleanup, device-orientation permission behavior, AudioContext lifecycle, and reduced-motion handling on mobile devices.
- Audio should remain user-controlled and respect browser autoplay restrictions; test cleanup when navigating/unmounting.

## 5. API server

### Current implementation
Package: `@workspace/api-server`

- Express 5 application.
- Pino + pino-http structured logging.
- JSON and URL-encoded body parsing.
- API mounted at `/api`.
- `GET /api/healthz` returns `{ "status": "ok" }`, validated through the generated HealthCheckResponse schema.
- Port is mandatory via `PORT`; invalid or missing values throw during startup.
- Build uses esbuild to bundle TypeScript to ESM output under `artifacts/api-server/dist/`.

The server currently has no authentication, user model, business routes, database-backed route, rate limiting, or API-level test coverage. The database package is imported in TypeScript project references but not used by the health endpoint.

### Security and operations
- `app.ts` calls `cors()` with default permissive behavior. For a public health-only endpoint this is low impact, but define an explicit allowed-origin policy before introducing private data, cookies, or state-changing routes.
- `express.json()` uses default body size limits. Set an explicit limit and validation strategy when write endpoints are added.
- No rate limiting, request schema validation middleware, authentication, or authorization currently exists. Add these when endpoints require them rather than treating the current health endpoint as a complete API.
- Logger redacts authorization and cookie request headers and set-cookie response headers, which is a good baseline. Confirm future application-level logs do not emit tokens or personal data.
- Startup validates `PORT` with Number.isNaN and positive check; also require an integer within the valid TCP port range (1–65535).
- The error callback passed to Express `app.listen` is not the usual Node server error-event pattern. Prefer capturing the returned Server and handling its `error` event explicitly.
- Production API artifact manifest runs the built ESM bundle and exposes startup health at `/api/healthz`.

## 6. OpenAPI, generated client, and type contracts

### OpenAPI source
`lib/api-spec/openapi.yaml` is the source contract. It currently describes only:
- `GET /healthz`
- Response object `HealthStatus` with required string property `status`

The backend route validates its response using `HealthCheckResponse.parse()`. Orval generates:
- `lib/api-client-react/src/generated/api.ts` — React Query health hook and query helpers.
- `lib/api-zod/src/generated/types/healthStatus.ts` — generated TypeScript interface.
- Generated Zod schemas under `lib/api-zod/src/generated/`.

### Client fetch layer
`lib/api-client-react/src/custom-fetch.ts` is a substantial shared fetch adapter:
- Supports JSON/text/blob/auto response parsing.
- Handles empty-body status codes and BOM.
- Builds structured `ApiError` and `ResponseParseError` objects.
- Supports optional base URL and bearer token getter for native/non-cookie clients.
- Merges request headers and normalizes error messages.

The client library is a good foundation, but the current Mission Control UI does not appear to import or invoke `useHealthCheck` or any other generated API hook. The frontend and API are currently independent services rather than a data-connected full-stack application.

Keep OpenAPI as the source of truth: update the spec first, regenerate with `pnpm --filter @workspace/api-spec run codegen`, then typecheck dependent packages. Do not hand-edit generated files.

## 7. Database package

Package: `@workspace/db`

- PostgreSQL connection via `pg.Pool`.
- Drizzle ORM initialized with imported schema.
- Requires `DATABASE_URL` at module import time; missing configuration throws.
- Drizzle config points at `lib/db/src/schema/index.ts`.
- Schema file is currently an empty export with commented example code.
- No actual tables, insert schemas, migrations, or DB-backed API handlers were found.

The README's instruction to run `npm run db:migrate` does not match the current package scripts. The database package exposes `push` and `push-force`, and root `replit.md` documents `pnpm --filter @workspace/db run push` for development. `push-force` should be treated as destructive and not used casually. A production migration workflow should generate and review migrations rather than relying on schema push.

Add a graceful pool shutdown hook when the API server begins using the database, and add schema/migration integration tests.

## 8. Mockup preview service

Package: `@workspace/mockup-sandbox`

- Vite-based component preview server intended for Replit Canvas under `/__mockup`.
- `mockupPreviewPlugin.ts` scans `src/components/mockups/**/*.tsx` with fast-glob.
- Generates a module map in `src/.generated/mockup-components.ts`.
- Watches for add/remove changes with chokidar and refreshes the generated map.
- App parses `/preview/<componentPath>`, lazy-loads the matching component, resolves a component export, and renders it.
- Root gallery explains the preview URL.

No actual `src/components/mockups/` content was present in the tree, so the service currently provides the preview infrastructure but no committed preview examples.

The plugin generates dynamic imports from discovered local TSX files. This is intended developer tooling; keep it confined to trusted workspace code and do not evolve it into arbitrary user-supplied module loading. Consider writing generated files atomically and testing watcher shutdown/rebuild behavior.

## 9. Workspace, build, and deployment

### Workspace
- pnpm workspaces include `artifacts/*`, `lib/*`, `lib/integrations/*`, and `scripts`.
- Node.js 24 configured in Replit.
- TypeScript 5.9.
- Shared dependency catalog and lockfile.
- `minimumReleaseAge: 1440` requires packages to be at least 24 hours old before install, with a small explicit exclusion list. This is a useful supply-chain delay control, though not a substitute for lockfile review and dependency scanning.
- Overrides exclude many optional platform-specific binaries and pin esbuild.

### Root scripts
- `pnpm run typecheck:libs` — TypeScript build references.
- `pnpm run typecheck` — libraries plus artifact/scripts typechecks.
- `pnpm run build` — typecheck then recursive package builds.

No root `dev` script exists. Start services with their workspace package scripts as described in `replit.md`.

### Replit services
- API server: port 8080, path `/api`, production build and Node process.
- Mission Control: port 26054, path `/`, static production build and SPA fallback rewrite.
- Mockup sandbox: port 8081, path `/__mockup`, development preview service.
- Replit deployment target is autoscale, with application router.

The Vite config binds dev/preview to `0.0.0.0` and sets `allowedHosts: true`. This is permissive host validation. Use a narrow host allowlist where feasible and ensure preview/dev services are not exposed as public production surfaces. The production Mission Control artifact is served statically and does not need the Vite dev server.

### Post-merge hook — likely broken filter
`scripts/post-merge.sh` runs:
1. `pnpm install --frozen-lockfile`
2. `pnpm --filter db push`

The workspace package is named `@workspace/db`, not `db`. Unless pnpm resolves a matching alias or package name not visible in the manifest, this filter will not select the intended package. Change it to `pnpm --filter @workspace/db run push` if automatic development schema push is actually intended. More importantly, avoid automatic schema mutation in a post-merge hook for production; use a reviewed migration pipeline.

## 10. Documentation and repository hygiene

The root `readme.md` is a starter template that does not describe the current repository:
- Says Express 5 / React / Drizzle / Orval, broadly accurate at stack level.
- Describes `server/` and `client/`, but actual code lives under `artifacts/` and `lib/`.
- Recommends npm install, while preinstall rejects non-pnpm package managers.
- References `.env.example`, but none is committed.
- References `npm run db:migrate`, which is not defined.
- Recommends `npm run orval`, while codegen is a package script.
- Recommends `npm run dev`, but root has no dev script.
- Roadmap still says add CI, while there is no CI workflow in the checked-in tree.

`replit.md` is more current about workspace commands and Node/pnpm stack, but still has template placeholders for product description, package map, architecture decisions, and gotchas.

The attached design prompt gives useful product intent and page-level UX direction; it should be retained as design context, but current implementation should be the source of truth for behavior.

## 11. Main defects and technical debt

### P0/P1 before treating it as a reusable full-stack starter
1. Fix README commands and directory structure; document exact pnpm service commands and environment requirements.
2. Correct the post-merge package filter and decide whether DB push belongs in a hook at all.
3. Replace permissive API CORS with explicit origins before adding authenticated or stateful routes.
4. Restrict Vite dev/preview host exposure; separate trusted development services from production.
5. Add a minimal CI workflow: frozen install, typecheck, build, lint, and tests.
6. Add tests for API health contract, generated client parsing, database connection/schema, and mockup plugin behavior.

### P1 — connect frontend and backend
1. Decide whether Mission Control is intentionally a standalone visual demo or is supposed to consume the API.
2. If full-stack is intended, replace hard-coded mission and telemetry data with API-backed data models.
3. Add mission/telemetry OpenAPI endpoints, generated schemas/hooks, backend validation, and UI loading/error/empty states.
4. Clearly label synthetic/demo data and disable fake operational commands in any production presentation.

### P2 — frontend quality
1. Make mission cards semantic keyboard-operable buttons and ensure detail panels support focus management, Escape-to-close, and screen readers.
2. Provide reduced-motion support for Framer Motion, CSS animation, starfield, and telemetry timers.
3. Review starfield canvas/audio/device-orientation lifecycle and mobile performance; add explicit user controls for audio and motion intensity.
4. Remove unused dependencies or document why Three.js/React Three Fiber/GSAP are retained if they are not currently used in the inspected primary views.
5. Add responsive and accessibility tests for the four routes and interactive overlays.

### P2 — database and API readiness
1. Add real schema modules, migration generation, migration review, and seed data if persistence is in scope.
2. Add DB pool lifecycle/shutdown handling and health checks that verify dependency status only if the endpoint is meant to report readiness.
3. Add auth, authorization, request validation, rate limiting, and security headers when non-health endpoints are introduced.
4. Define API error schema and status enum in OpenAPI rather than leaving health status as an arbitrary string.
5. Add environment validation and safe configuration examples without real credentials.

## 12. Coding-agent handoff

Recommended sequence:
1. Repair repository documentation and the post-merge DB filter.
2. Add CI with frozen pnpm install, typecheck, and builds for all three artifacts and libraries.
3. Add API tests and decide CORS/host policy.
4. Confirm whether the backend scaffold is intentionally dormant or should back Mission Control.
5. If backend integration is desired, define Mission and Telemetry schemas in Drizzle and OpenAPI, regenerate Zod/client code, implement API routes, and connect React Query hooks to the pages.
6. Make all demo telemetry clearly synthetic and ensure no button claims to control real infrastructure.
7. Add accessibility/reduced-motion coverage and profile the starfield on mobile.

## 13. Validation status and limitations

- Repository metadata and full recursive tree: inspected (234 entries).
- Workspace manifests, lockfile presence, TypeScript configs, Replit manifests: inspected.
- API server, OpenAPI spec, generated client, database package: inspected.
- Mission Control routes/components/theme/starfield: inspected.
- Mockup sandbox and Vite plugin: inspected.
- Test files: none found in tree.
- GitHub Actions workflows: none found in tree.
- Environment example files: none found in tree.
- Database migrations: none found in tree.
- Install, typecheck, lint, build, browser tests, server startup, API requests, database connection, and deployment: not executed.
- No claim is made that the application is currently deployed or that its visual metrics represent real mission telemetry.
