# Spider — repository context and engineering handoff

**Repository:** `cheron2000/spider-`  
**Default branch:** `main`  
**Inspected:** 2026-11-22  
**Repository tree:** `eddab826d437e8c30f60af90aa595af61ce2e34f`  
**Purpose:** Source-aware context for future coding agents. Treat the current repository as the source of truth; use this document as an orientation and verify code before changing behavior.

---

## 1. Executive summary

Spider is a Node.js multi-agent coordination framework. It reserves six provider-backed agent slots—Antigravity, Codex, Copilot, Kiro, Manus, and Replit—and coordinates work through a central dependency-aware scheduler. The repository contains:

- An Express API and Socket.IO event layer.
- SQLite persistence for agent identities, per-agent credential hashes, messages, coordination plans/steps, assignment leases, reports, reviews, execution journal entries, and GitHub issue mirror state.
- A provider-neutral adapter contract and runtime manager, with four local CLI adapters, a Manus API adapter, and a Replit HTTP bridge adapter.
- A React/Vite dashboard that visualizes the provider topology and dependency graph, displays report/review queues, and exposes coordinator actions.
- Node's built-in test runner tests and a GitHub Actions workflow that runs tests and builds the dashboard.

This is an evolving framework, not a production-ready multi-tenant agent platform. Source code and project documentation explicitly distinguish source-level safeguards from verified live provider execution.

## 2. Product intent and system boundaries

The central product idea is a coordinator that decomposes an objective into dependent steps, assigns eligible work to isolated provider identities, tracks execution and evidence, and requires a human/coordinator review before downstream steps are unlocked.

Important distinctions:

1. **Provider slots are not live agents by default.** All provider adapters start offline. A provider must be enabled, configured with an existing dedicated workspace or service credentials, and pass runtime checks before scheduling.
2. **A successful provider invocation is not approval.** Provider output is submitted as a report in `review`; a coordinator decision is the final acceptance gate.
3. **Peer review is advisory.** Lateral agent feedback does not replace coordinator approval.
4. **GitHub issue mirroring is separate from the durable provider DAG.** Syncing issues does not launch provider work. The legacy issue endpoints and the coordinator DAG should not be conflated.
5. **The Replit adapter is a bridge contract.** Spider does not deploy the bridge or prove that a Replit execution environment exists.
6. **Local provider CLIs run on the same host/container as the API.** A configured workspace and path-collision check are not OS-level sandboxing.

The README's description of a “four-neighbor peer topology” is a UI/topology concept; it should not be mistaken for an independent decentralized scheduler. The coordinator retains final authority.

## 3. Repository map

### Root and configuration
- `package.json`: ESM Node project, npm workspace for the dashboard, scripts for API/dashboard development, tests, build, and start.
- `package-lock.json`: dependency lockfile.
- `.env.example`: configuration reference; never copy real credentials into source control.
- `.github/workflows/ci.yml`: Node 20 CI; installs dependencies, runs `npm test`, then `npm run build`.
- `schemas/task.schema.json`: generic task JSON Schema. It is not the complete schema for the durable coordination-plan data model.
- `hello_world.py`, `agents/python_worker.py`, `agents/requirements.txt`: small Python examples/worker utility; the main coordinator and provider runtime are JavaScript.

### API and persistence
- `services/api/src/index.js`: Express/Socket.IO application, HTTP routes, coordinator/worker authentication, provider wiring, scheduling and orchestration. This file is very large and currently combines many responsibilities.
- `services/api/src/agentRegistry.js`: SQLite-backed stores for agent registry, messages, coordination state, and GitHub issue mirror.
- `services/api/src/providers/index.js`: constructs provider adapters.
- `services/api/src/providers/base.js`: common local CLI execution, environment allowlisting, output parsing/redaction, workspace inspection, timeout handling, and adapter base.
- `services/api/src/providers/runtime.js`: provider health polling, readiness snapshots, concurrency accounting, workspace conflict checks, dispatch, and provenance normalization.
- `services/api/src/providers/{antigravity,codex,copilot,kiro,manus,replit}.js`: vendor-specific adapters.
- `services/api/test/`: coordinator, GitHub issue store/webhook, lease recovery, provider invocation/runtime/base and Kiro tests.

### Dashboard
- `apps/dashboard/src/App.jsx`: main React dashboard; large single-file view/state/action surface.
- `apps/dashboard/src/SpiderWeb.jsx`: provider topology visualization.
- `apps/dashboard/src/dashboardState.js`: small pure state/status helpers.
- `apps/dashboard/src/styles.css`: dashboard styling.
- `apps/dashboard/src/main.jsx`: React entry point.
- `apps/dashboard/test/dashboardState.test.js`: dashboard state helper tests.
- `apps/dashboard/vite.config.js`, `apps/dashboard/package.json`: Vite and React Flow configuration/dependencies.

### Documentation
- `docs/CONTEXT_WINDOW.md`: extensive historical architecture and handoff context.
- `docs/EXECUTION_AUDIT.md`: source-level security/execution audit and known limitations.
- `docs/PROVIDER_ADAPTERS.md`: provider configuration and lifecycle guide.
- `docs/SPIDER_WEB_TOPOLOGY.md`: topology model and dashboard semantics.

## 4. Runtime architecture and execution flow

### Startup
1. The API loads environment configuration from the repository-root `.env` path and dotenv defaults.
2. Express and Socket.IO are created. CORS is configured from `DASHBOARD_ORIGIN` (default localhost dashboard).
3. SQLite stores are instantiated. The agent registry is restored, with agents marked offline until provider runtime checks.
4. Six canonical provider identities are ensured and associated with the six provider slots.
5. The provider runtime starts health polling. Disabled, unconfigured, or unavailable providers remain offline and are excluded from assignment.
6. The HTTP API listens on the configured port (documented default 4000); the Vite dashboard is expected on port 5173.

### Durable DAG flow
1. Coordinator creates a draft plan using `POST /api/coordinator/plans/draft`, or submits a custom plan.
2. Plan and step dependency graph are validated and persisted in SQLite.
3. Scheduler identifies steps whose dependencies are complete and assigns eligible agents/providers.
4. Assignment includes a unique token, attempt number, and lease timestamps. The token fences stale workers or provider results.
5. Provider-backed agents dispatch through `ProviderRuntimeManager`; external workers poll their authenticated work queue and start/submit with identity plus assignment token.
6. Provider/worker submits summary and evidence. The report and state transition are persisted atomically; the step enters `review`.
7. Peer reviews can be attached as advisory information.
8. Coordinator approves or rejects. Approval marks the step done and may unblock dependants; rejection returns work to ready within policy or fails it.
9. Execution lifecycle events are appended to a durable journal and exposed through coordinator-authenticated, paginated endpoints.

### Lease and recovery semantics
- Default assignment lease is 120 seconds, clamped to 30 seconds–1 hour by `SPIDER_ASSIGNMENT_LEASE_MS`.
- Lease renewal/start/submit are fenced by the current assignment token and expiry.
- Startup reconciliation revokes stale in-flight tokens and journals the recovery decision.
- Expired work that never started may be requeued if attempt budget remains.
- Expired work that entered `in_progress` is failed closed rather than automatically retried: the provider may already have changed files or external systems.
- A provider invocation still active in the current process is excluded from lease sweeping until it exits.
- Adapter exceptions after execution begins also require reconciliation; an exception does not prove no side effect occurred.
- Explicit retry requires coordinator action, a reconciliation reason, and completed dependencies. Token fencing prevents stale submissions but cannot undo external effects.

Do not weaken these guarantees or introduce automatic retries for non-idempotent work without tests covering stale tokens, leases, restarts, and external side effects.

## 5. Provider adapter architecture

The shared adapter contract centers on health/readiness and task execution. The runtime manager:
- polls adapter health on a configurable interval (documented default 30 seconds);
- rechecks readiness before dispatch;
- tracks in-flight work and adapter capacity;
- refuses dispatch when two enabled local providers resolve to the same canonical workspace;
- bounds and redacts summaries/evidence;
- normalizes provider provenance;
- reports provider state changes to the coordinator.

### Provider matrix

| Slot | Execution mechanism | Key configuration | Important boundary |
|---|---|---|---|
| Antigravity | Local CLI `agy` | `SPIDER_ANTIGRAVITY_*` | Requires installed/authenticated CLI and dedicated existing workspace |
| Codex | Local CLI `codex` | `SPIDER_CODEX_*` | Sandbox setting is read-only or workspace-write; invalid values fall back to read-only |
| Copilot | Local CLI `copilot` | `SPIDER_COPILOT_*` | Requires native CLI login and workspace |
| Kiro | Local CLI `kiro-cli` | `SPIDER_KIRO_*` | Requires native CLI login and workspace |
| Manus | Manus API v2 | `MANUS_API_KEY`, Manus adapter enablement | Creates/polls private tasks; waiting/error states are not successful completion |
| Replit | Remote authenticated HTTP bridge | `SPIDER_REPLIT_BRIDGE_URL`, `SPIDER_REPLIT_BRIDGE_TOKEN` | Bridge must be separately implemented, deployed, and secured |

All providers are offline by default. A CLI executable and configured workspace can produce a ready/configured state with `authState: not-probed`; this is not proof that the user's account is authenticated. The first actual task call is the functional authentication check.

### Local process safeguards and limitations
- CLI launch uses `cross-spawn` with `shell:false`, bounded timeout, output limits, and process termination.
- Child processes receive a conservative environment allowlist. Extra environment variables must be explicitly named in the provider passthrough configuration.
- Output parsing handles structured JSON/JSONL and common nested failure statuses; common secret formats are redacted.
- Each local CLI adapter is limited to one concurrent task per workspace.
- Canonical workspace collision checks prevent two enabled providers from sharing one configured directory.
- These controls are not a filesystem sandbox. A provider process may still access paths permitted by the host and its native tool permissions.
- Provider adapters are instructed not to commit, push, merge, deploy, or release.
- Provider-reported success, zero exit code, and provenance metadata are not independent proof that files changed as claimed.

### Replit bridge
The documented contract includes authenticated `GET /health` and `POST /execute`, with task and prompt input and a non-empty summary/output response. Spider's repository contains a bridge adapter and local API bridge routes, but this is not equivalent to a separately deployed, isolated Replit execution service. Verify the current route implementation before relying on it; do not call it a production bridge without independent deployment/security evidence.

## 6. API surfaces and security model

### Authentication
- Coordinator endpoints use `SPIDER_COORDINATOR_API_KEY`, requiring a non-placeholder value of at least 32 characters and the `x-spider-coordinator-key` header.
- Worker endpoints require `x-agent-id` plus `x-spider-agent-key`. Per-agent credentials are generated/rotated, stored as SHA-256 hashes in SQLite, and checked with constant-time comparison; ownership checks follow authentication.
- Agent Socket.IO connections are bound to the registered identity and credential. Coordinator sockets receive the coordinator room snapshot/events.
- GitHub webhook requests are checked using HMAC SHA-256 against a configured non-placeholder `GITHUB_WEBHOOK_SECRET`.
- `GET /api/health` is intentionally minimal/public; detailed coordinator health is protected.
- CORS is not an authorization boundary.

### Dashboard credential handling
The dashboard stores the coordinator key in browser local storage and sends it in API requests and the Socket.IO handshake. Only use it from a trusted browser profile. Shared browsers, untrusted extensions, and exposed dashboard hosts create credential risk. Consider a safer session/proxy model before public deployment.

### Important endpoint groups
- Coordinator health/providers: `GET /api/coordinator/health`, `GET /api/providers`, provider refresh/configure routes.
- Plans and scheduler: `POST /api/coordinator/plans/draft`, custom plan creation, plan listing/detail, advance.
- Reviews and retry: `POST /api/coordinator/steps/:id/review`, peer-review endpoints, `POST /api/coordinator/steps/:id/retry`.
- Journal: coordinator execution-journal and step-history endpoints.
- Workers: registration, credential rotation, heartbeat, work retrieval, start, submit, lease renewal.
- Peer messaging: message send, recipient inbox, acknowledgement.
- GitHub mirror: sync, task claim/progress/complete, webhook; distinct from DAG execution.
- CEO issue planning: `POST /api/ceo/plan` creates a deterministic starter epic/issues. It is not equivalent to provider DAG planning.

Consult `services/api/src/index.js` for current route definitions and middleware; the list above is an orientation, not an exhaustive API contract.

## 7. Persistence and data ownership

SQLite uses `better-sqlite3`, WAL mode, busy timeout and foreign keys. Default DB path is documented as `services/api/data/spider.db`; `SPIDER_DB_PATH` can override it.

Persisted data includes:
- Agent identities and stable slot allocation.
- Per-agent credential hashes.
- Messages, delivery/acknowledgement state.
- Coordination plans/steps, assignment leases/tokens, attempts, reports and review feedback.
- Peer review records.
- Append-only execution journal (database triggers reject update/delete).
- GitHub issue mirror metadata, claims, progress and completion reports.

Some transient state remains in memory, including the bounded activity event buffer and provider in-flight tracking. The SQLite journal is the durable execution history; do not treat the in-memory activity stream as the complete audit record.

The generic `schemas/task.schema.json` is not a complete representation of all persisted plan/step fields. Prefer the actual validators and store methods in `index.js` and `agentRegistry.js` when implementing DAG changes.

## 8. Dashboard behavior

The dashboard uses React 18, Vite, React Flow, Socket.IO client, and Lucide icons. The main `App.jsx` currently combines authentication, API calls, periodic refresh, socket event handling, plan management, graph rendering, reviews, execution history, provider status and GitHub issue mirror UI.

Documented primary workflow:
- Authenticated coordinator health and plan refresh, with periodic polling (15 seconds) as a fallback to live events.
- Draft durable plan creation.
- DAG nodes/edges and step state visualization.
- Review queue with evidence/report summaries.
- Approve/reject and guarded retry with reconciliation notes and explicit confirmation.
- Execution journal/history.
- Provider topology/spider-web visualization.
- GitHub issue mirror presented separately; issue sync does not start provider work.

The dashboard's large single-file implementation is a maintainability risk. Split into feature components/hooks/API client modules while preserving the current auth boundary and action semantics.

## 9. Tests and validation evidence

Root scripts:
- `npm run dev`: API + dashboard concurrently.
- `npm run dev:api`: Node watch API.
- `npm run dev:web`: Vite dashboard.
- `npm test`: Node built-in test runner.
- `npm run build`: dashboard production build.
- `npm start`: API process.

CI workflow uses Node 20, runs `npm install`, `npm test`, and `npm run build`.

README and `docs/CONTEXT_WINDOW.md` report a CI run with 8 Node test cases passing and dashboard build succeeding on commit `db8cfd99dba5ef1e32f5427b6cdd6d7aa3c63a82`. This is historical repository documentation; this audit did not independently rerun the workflow or tests on the current tree.

Coverage described by the repository includes coordinator authorization, DAG validation/dependency gating, worker identity/credentials, lease token fencing, coordinator review/retry, journal/restart recovery, peer-message spoof/self-send rejection, recipient-only inbox/acknowledgement, webhook fail-closed behavior, provider output parsing, nested failure detection, timeout bounds, secret redaction, transient reconnect notices, and child-process environment allowlisting.

Important unverified areas:
- Live authenticated execution for each CLI/provider.
- Actual Replit bridge deployment and security.
- Real CLI-version compatibility and output variations.
- Full timeout/termination behavior across Windows/Linux and provider process trees.
- Positive and negative live webhook signature validation.
- Deployment security, TLS/ingress, resource limits and OS-level provider isolation.
- Full lease sweeper edge cases and provider-specific failure recovery.

No tests or builds were run as part of this Porty inspection. Do not present historical CI evidence as a fresh test run.

## 10. Engineering risks and recommended priorities

### P0 — Runtime isolation and public exposure
Before exposing Spider to untrusted users or networks:
- Put the API behind authenticated TLS ingress or a trusted gateway.
- Add request rate limits, bounded request/worker/provider concurrency and resource quotas.
- Run local provider processes in dedicated OS/container sandboxes, not merely separate directories.
- Use per-provider worktrees/workspaces with least privilege.
- Ensure the SQLite database is on durable storage in deployment.
- Revisit dashboard localStorage key handling and session lifecycle.
- Validate CORS and Socket.IO origins, while retaining real authentication.

### P1 — Decompose oversized modules
- Split `services/api/src/index.js` into route modules, middleware/auth, coordinator service, scheduler, GitHub integration, socket layer and configuration.
- Split `apps/dashboard/src/App.jsx` into page/feature components, hooks, typed API client and state boundaries.
- Keep orchestration semantics in services, not UI components or route handlers.
- Add module-level contracts and integration tests before moving behavior.

### P1 — Provider compatibility and execution evidence
- Pin/document tested versions for each CLI.
- Create adapter contract tests with representative real provider outputs.
- Run one authenticated smoke test per provider in a disposable workspace.
- Test process-tree termination, timeout, auth failure, malformed output, output truncation and workspace collision.
- Attach verifiable artifact/diff/commit/tree references after execution; current provenance is largely adapter/coordinator-reported and `changedFileRefs` can be empty.
- Keep coordinator approval as final gate.

### P1 — Clarify the dual task systems
- Decide whether GitHub issues are merely a planning input/mirror for durable DAG steps or whether a supported mapping should exist.
- Document the distinction in API/UI and remove stale docs that imply `/api/ceo/plan` is the dashboard's primary DAG path.
- Avoid two competing sources of truth for assignment, status and completion.

### P2 — Reliability, observability and developer experience
- Expand tests for peer-review authorization matrix, lease sweeper edge cases, real process termination and workspace collision.
- Add structured logs/correlation IDs and operational metrics for provider auth, stuck plans, lease expiry, recovery and retry.
- Add API schema/OpenAPI and generated client types; current generic task schema is incomplete for DAG.
- Improve dashboard component boundaries, accessibility, empty/error states and loading behavior.
- Add a reproducible local development/test guide, including Windows-specific CLI notes.

## 11. Coding-agent rules for future work

1. Read `docs/EXECUTION_AUDIT.md`, `docs/PROVIDER_ADAPTERS.md`, and the relevant implementation before modifying behavior.
2. Treat source as authoritative where documentation conflicts; update docs in the same change.
3. Do not enable providers by default, create credentials, or invent workspace paths.
4. Never log or commit secrets. Do not put credentials in prompts, task descriptions, evidence, or dashboard source.
5. Do not assume `authState: not-probed` means authenticated.
6. Do not equate provider success/zero exit with coordinator acceptance.
7. Do not auto-retry work after it starts unless side effects have been reconciled; preserve lease-token fencing and explicit retry.
8. Do not weaken coordinator or per-agent authentication. CORS alone is not security.
9. Do not treat a separate workspace directory as a security sandbox.
10. Preserve the append-only journal and atomic report/state transitions.
11. Keep GitHub issue mirroring distinct from DAG scheduling unless an explicit mapping is designed and tested.
12. For each change, add regression tests and run `npm test` plus `npm run build`; report exact commands and outcomes.
13. For provider changes, test with a fake adapter first, then a real provider in a disposable isolated workspace. Do not claim live compatibility without evidence.
14. Avoid large unrelated refactors. Decompose one subsystem at a time and preserve public route contracts unless intentionally versioned.

## 12. Recommended implementation sequence

1. Review current branch/PR state and confirm the latest CI baseline.
2. Add missing scheduler/recovery/provider process edge-case tests before changing semantics.
3. Harden deployment isolation, ingress, rate limits and credential/session handling.
4. Refactor API and dashboard into modules with tests protecting existing behavior.
5. Establish a versioned provider contract and real per-provider smoke-test evidence.
6. Decide the GitHub issue-to-DAG relationship and remove stale dual-workflow documentation.
7. Improve artifact provenance and observability.
8. Update README, provider docs, audit, and this context file after meaningful architecture changes.

## 13. Source references inspected

- Repository: https://github.com/cheron2000/spider-
- README: https://github.com/cheron2000/spider-/blob/main/README.md
- API/coordinator: https://github.com/cheron2000/spider-/blob/main/services/api/src/index.js
- SQLite stores: https://github.com/cheron2000/spider-/blob/main/services/api/src/agentRegistry.js
- Provider base: https://github.com/cheron2000/spider-/blob/main/services/api/src/providers/base.js
- Provider runtime: https://github.com/cheron2000/spider-/blob/main/services/api/src/providers/runtime.js
- Provider adapters: https://github.com/cheron2000/spider-/tree/main/services/api/src/providers
- Dashboard: https://github.com/cheron2000/spider-/blob/main/apps/dashboard/src/App.jsx
- Topology component: https://github.com/cheron2000/spider-/blob/main/apps/dashboard/src/SpiderWeb.jsx
- Execution audit: https://github.com/cheron2000/spider-/blob/main/docs/EXECUTION_AUDIT.md
- Provider guide: https://github.com/cheron2000/spider-/blob/main/docs/PROVIDER_ADAPTERS.md
- Context window: https://github.com/cheron2000/spider-/blob/main/docs/CONTEXT_WINDOW.md
- CI workflow: https://github.com/cheron2000/spider-/blob/main/.github/workflows/ci.yml

---

## Continuation handoff

Continue from the current `main` branch of `cheron2000/spider-`. The durable DAG, coordinator review gate, lease fencing, SQLite journal, provider adapter boundaries, and coordinator/worker credential split are core invariants. Inspect the latest source and CI state before implementation. Prioritize isolation/security and missing reliability tests. Never claim production readiness or live provider compatibility from source inspection or historical CI alone.
