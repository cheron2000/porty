# Repository Context: cheron2000/Ocenoid

## 1. Executive summary

**Repository:** `cheron2000/Ocenoid`  
**Visibility / branch:** Public / `main`  
**Repository size reported by GitHub:** 41,443 KB  
**Product:** Four-player LAN ocean-survival sandbox prototype  
**Stack:** React 19, TypeScript, Three.js, React Three Fiber, Drei, Vite 7, Node.js, TypeScript, WebSocket (`ws`), shared TypeScript contracts; Python AI/simulation dependencies are reserved but not implemented.

Ocenoid has a real codebase and a clear staged game direction. The current committed implementation is primarily a networked 3D character/boat/ocean prototype:
- React Three Fiber scene with a procedural boat, a shader-based ocean surface, an island placeholder, and four player appearance presets.
- A WebSocket server intended to host one four-player room and broadcast player snapshots at 20 Hz.
- A shared package defining player, input, and server-message contracts.
- Movement unit tests and a multiplayer integration test.
- A development plan and Kiro bug-fix specification.

**Critical blocker:** `server/src/index.ts` imports `./physics.js` and calls `PhysicsSimulation.create()`, `addPlayer()`, `removePlayer()`, and `step()`. The complete recursive repository tree contains no `server/src/physics.ts`, `server/src/physics.js`, or corresponding module. The committed server entry point therefore has an unresolved runtime import and cannot start as checked in. The integration test launches this entry point, so it is also expected to fail before exercising multiplayer behavior. This is a source-tree finding; no test command was executed.

There are also two likely scene correctness defects:
1. `OceanSurface.tsx` displaces the plane's local Y coordinate even though the plane is rotated onto the XZ world plane. The displacement therefore moves vertices along world Z rather than vertically, and the shader's wave equations read local Z values that are zero for the undeformed plane. The advertised vertical animated waves are not implemented correctly by this shader as written.
2. `App.tsx:snapToFloor()` skips every raycast hit whose object name is empty. The active procedural boat, island meshes, and ocean mesh are created without explicit object names, so their default names are empty. The function will skip those surfaces and fall back to Y=0, rather than snapping the local player onto the deck.

The repository's own development plan says Phase 1 is complete and Phase 2 is nearly complete, but those milestone claims need revalidation against the current source because of the missing server physics module and the scene issues above.

## 2. Repository map

| Path | Responsibility |
|---|---|
| `client/src/App.tsx` | Main React/R3F scene, WebSocket client, input collection, player interpolation, camera, interaction prompts |
| `client/src/game/OceanSurface.tsx` | Custom GLSL ocean surface shader |
| `client/src/game/BoatModel.tsx` | Procedural boat geometry and canvas-generated wood/sail textures |
| `client/src/game/OcenoidCharacter.tsx` | Procedural block-style humanoid, four appearance presets, procedural walk/idle animation |
| `client/src/game/SailingShipModel.tsx` | Drei GLTF loader for `/sailingship.glb`; not imported by the active App |
| `client/public/boat.glb` | GLB asset (about 1.0 MB); not used by active App |
| `client/public/sailingship.glb` | GLB asset (about 58.5 MB); referenced by unused SailingShip component |
| `server/src/index.ts` | WebSocket room, connection limit, player slots, input parsing, 20 Hz simulation/broadcast loop |
| `server/src/movement.ts` | Pure camera-relative movement helper with diagonal normalization and finite-number guards |
| `server/src/movement.test.ts` | Two unit tests for movement helper |
| `server/src/multiplayer.integration.test.ts` | Spawns the server and tests four-player capacity/disconnect/replacement |
| `server/src/physics.*` | **Expected by server entry point but absent from tracked tree** |
| `shared/src/network.ts` | PlayerState, ClientInput, ClientMessage, ServerMessage types |
| `docs/DEVELOPMENT_PLAN.md` | Product vision, architecture, milestone checklist |
| `.kiro/specs/ocenoid-bugfixes/` | Requirements, design and tasks for TypeScript compile-time fixes |
| `requirements.txt` | Placeholder Python FastAPI/Uvicorn/Pydantic/NumPy dependencies |

The recursive tree contained 140 entries and was not truncated. No `.github/workflows/` CI workflow, persistence layer, database schema, game-content pipeline, or Python service source was found.

## 3. Product intent and gameplay scope

The README and development plan describe a cooperative survival sandbox:
- Four players start together on a boat in an ocean.
- Players explore the ocean, underwater areas, floating debris, and islands.
- Players gather resources and use them for tools, crafting, vehicles, and structures.
- Islands eventually contain NPC settlements, trade, resource production, and a dynamic economy.
- Future survival systems include health, hunger, thirst, weather, storms, and ocean hazards.

This is a roadmap, not the current feature set. The committed playable scope is much narrower: a 3D scene, player representation, camera, keyboard movement input, WebSocket snapshots, and basic interaction prompt scaffolding.

### Current implementation vs planned scope

| System | Evidence in current source | Assessment |
|---|---|---|
| Four-player room | Server limit and occupied spawn slots | Implemented in source, but server startup is blocked by missing physics module |
| LAN networking | WebSocket server bound to `0.0.0.0`; client defaults to hostname port 8787 | Intended and wired, not revalidated |
| Player synchronization | 20 Hz snapshot broadcast and client interpolation | Wired to missing physics simulation |
| Character appearance | Four procedural presets | Implemented |
| Character animation | Frame-driven arm/leg swing and idle bob | Basic procedural animation |
| Boat | Procedural geometry in active App | Implemented as prototype geometry |
| GLB ship assets | Loader component exists but is not used by active App | Assets are currently unused in active scene |
| Ocean | Custom shader component | Present, but vertex displacement coordinate-space defect undermines intended wave motion |
| Deck collision | Downward raycast in local PlayerMesh | Likely broken by empty-name filtering; returns fallback Y=0 |
| Camera collision | Raycast from player pivot | Present; should be validated against self-hits and unnamed geometry |
| Steering/crafting/storage | Static `[E]` prompts | Prompt-only; no E-key action or gameplay system |
| Resource gathering/inventory/crafting | Roadmap only | Not implemented |
| Island world generation | One cylinder placeholder | Not implemented |
| NPCs/economy | Roadmap only | Not implemented |
| Save/load/persistence | No storage layer found | Not implemented |
| Python AI services | Requirements placeholder only | Not implemented |

## 4. Architecture and runtime flows

### 4.1 Client

The client uses React 19, Vite 7, TypeScript, Three.js 0.179, React Three Fiber 9, Drei 10, and GSAP (declared dependency; no GSAP use was found in the reviewed active files).

The active App:
1. Creates a WebSocket connection to `VITE_SERVER_URL`, defaulting to `ws://<current-host>:8787`.
2. Receives a `welcome` message and records the local player ID and server tick rate.
3. Receives snapshots and replaces the local player map.
4. Tracks WASD/arrow keys globally.
5. Sends movement input at the server-provided tick rate, default 20 Hz.
6. Interpolates player mesh positions in the render loop.
7. Uses a right-mouse drag camera and scroll-wheel zoom.
8. Displays player labels and proximity-based static interaction prompts.

The client currently duplicates its own `Player` and `ServerMessage` types instead of importing the shared contracts. This creates a future protocol-drift risk.

### 4.2 Server

The server intends to:
- Bind a WebSocket server to `0.0.0.0:<PORT>`, default port 8787.
- Admit at most four concurrent players.
- Assign a free one of four deck spawn slots.
- Assign fixed names (`Coral`, `Kelp`, `Abyss`, `Frost`).
- Parse client input and clamp forward/right axes to [-1, 1].
- Run a fixed 20 Hz simulation and broadcast snapshots.

However, the entry point imports the missing `./physics.js`. The existing `movement.ts` helper is not imported by `index.ts`; the entry point instead calls a `PhysicsSimulation` abstraction that is absent from the tracked tree. Therefore the actual server-side movement implementation cannot be established from this repository snapshot.

### 4.3 Movement helper

`server/src/movement.ts:applyInput()`:
- Replaces non-finite input values with safe fallbacks.
- Normalizes diagonal movement to avoid a √2 speed advantage.
- Converts camera-relative forward/right values into world X/Z movement.
- Updates facing yaw only while moving.

The two unit tests cover diagonal normalization and non-finite input. These tests validate the helper in isolation; they do not prove that the production server calls it. In the current source, the production entry point delegates to the missing PhysicsSimulation module.

### 4.4 Shared protocol

`shared/src/network.ts` declares:
- `PlayerState`: ID, position, yaw, display name.
- `ClientInput`: forward, right, yaw, sequence.
- `ClientMessage`: join or input.
- `ServerMessage`: welcome or snapshot.

The `join` message is declared but the server does not use it; names are assigned from fixed slots. The `sequence` field is transmitted but the server does not enforce monotonicity, reject stale input, or use acknowledgements.

## 5. Critical and high-priority findings

### P0 — Server entry point imports a missing physics module

**Evidence:** `server/src/index.ts` imports `PhysicsSimulation` from `./physics.js`. The complete `main` tree contains no `server/src/physics.ts` or `server/src/physics.js`.

**Impact:** Node/tsx cannot resolve the runtime module. The server will fail during module loading before the WebSocket listener starts. The integration test spawns this same entry point and therefore cannot validate room capacity or movement.

**Required fix:**
1. Restore or implement `server/src/physics.ts` (or change the entry point to a tracked implementation).
2. Define the expected API explicitly: `create`, `addPlayer`, `removePlayer`, and deterministic `step`.
3. Make the simulation authoritative for player position and validate all input values at its boundary.
4. Add an integration test that proves the process starts and that movement snapshots change as expected.
5. Add a build/start smoke check to CI.

Do not mark Phase 1 networking/movement complete until the tracked source can start and the integration test passes.

### P1 — Ocean shader displaces the wrong coordinate

**Evidence:** `OceanSurface.tsx` uses a `planeGeometry` in local XY, rotates the mesh by `[-Math.PI / 2, 0, 0]` to lay it on the world XZ plane, but the vertex shader modifies `pos.y`. It also computes its wave functions from `pos.x` and `pos.z`; for the undeformed plane, local `pos.z` is zero.

**Impact:** Local Y maps to world Z after the mesh rotation, so the displacement is horizontal rather than vertical. The intended wave field is not evaluated across the world XZ plane. The shader also constructs a normal in local coordinates but uses it as though it were world-space for camera/sun shading.

**Required fix:**
- Define the wave function in the coordinate space actually used by the shader.
- For a rotated XY plane, use local X/Y as the two surface axes and displace local Z (the plane normal); alternatively use an unrotated XZ geometry and displace local Y.
- Transform normals with an appropriate normal matrix/model-space transform.
- Validate wave amplitude and normals visually at multiple camera angles and test shader compilation on target GPUs.

### P1 — Deck snap skips unnamed meshes and falls back to water level

**Evidence:** `App.tsx:snapToFloor()` ignores any hit where `hit.object.name === ""`. The active `BoatModel`, `OceanSurface`, and island placeholder meshes are created without explicit names; Three.js objects therefore retain the empty default name unless assigned elsewhere. The function skips these hits and returns `0` when no named floor is found.

**Impact:** The local player is likely pulled toward Y=0 instead of the deck. The documented deck-snap feature is not reliable with the active procedural scene.

**Required fix:**
- Do not use non-empty object names as the sole collision filter.
- Use explicit collision layers, userData tags, dedicated floor colliders, or a curated list of collidable objects.
- Exclude the water mesh by object reference/layer rather than a name-prefix heuristic.
- Add a deterministic test or debug scene asserting the player snaps to the expected deck height and does not snap to the ocean.

### P1 — Multiplayer server trust boundary is too permissive for internet exposure

**Evidence:** Server binds to `0.0.0.0`; no authentication, room token, Origin allowlist, rate limit, or per-message schema library is present. Any reachable client can attempt to occupy one of the four slots. Message parsing only checks JSON syntax, message type, and finite forward/right/yaw values.

**Impact:** On an exposed network, unauthenticated clients can consume all player slots, spoof arbitrary movement inputs, send excessive messages, or attempt oversized payloads. The current design is acceptable only as a controlled prototype on a trusted LAN.

**Required fix before public hosting:**
- Explicitly enforce a trusted Origin policy (and understand that Origin is not authentication).
- Add short-lived room/session tokens or authenticated admission.
- Set a conservative `maxPayload`, message rate limits, and per-connection backpressure policy.
- Validate exact message shape, numeric ranges, sequence freshness, and allowed message types.
- Add heartbeat/ping-pong handling and stale-client cleanup.
- Use TLS/WSS behind a correctly configured reverse proxy for internet deployment.
- Separate room identity from the single global in-memory room.

## 6. Additional correctness and performance risks

### Client and scene

- **Per-frame React state update:** `PlayerMesh` invokes `onPosUpdate` from `useFrame`; `World.handlePosUpdate()` calls `setLocalPos()` on every frame. This can cause React reconciliation at render frequency just to update an interaction prompt. Store position in a ref and update prompt state only when the nearest interaction target changes (or at a bounded interval).
- **Camera effect churn:** `setCameraYaw` is recreated on each App render, while `ThirdPersonCamera` registers pointer listeners in an effect depending on `onYaw`. Since snapshots update the App at 20 Hz, listeners can be repeatedly removed/re-added. Stabilize the callback with `useCallback` or store it in a ref.
- **No reconnect state machine:** the client creates one WebSocket and closes it on unmount. It does not retry after a server restart or transient network failure. The HUD can still display “Connected” if `localId` remains set after a later close event.
- **No close-state handling beyond code 1008:** other close reasons are not surfaced, and the client does not clear local connection state.
- **HTTPS mismatch:** the default URL always uses `ws://`; an HTTPS-hosted client needs `wss://` or it will encounter mixed-content restrictions.
- **Input sequence unused:** client increments sequence, but server does not reject stale/out-of-order input or acknowledge processed input.
- **Static interaction prompts:** `[E]` prompts are rendered when the player is near fixed coordinates, but there is no E-key handler or action implementation. The prompt currently implies functionality that does not exist.
- **Appearance assignment:** client derives the outfit preset from a hash of the UUID, while server names and spawn slots are assigned separately. A player's displayed name may not correspond to the documented slot/preset pairing.
- **Camera collision needs validation:** raycasts traverse scene children and rely on object names to skip water; the active water object is unnamed. The ray may also hit player/scene geometry not intended as an occluder.
- **Duplicate protocol types:** client-local `Player` and `ServerMessage` can drift from `shared/src/network.ts`.
- **Unused assets/components:** active App imports the procedural `BoatModel`; it does not load `boat.glb` or `SailingShipModel`. The 58.5 MB `sailingship.glb` is referenced only by the unused loader component. Confirm whether these assets are intended for the current scene; compress/optimize or use Git LFS/asset hosting if needed.
- **Unused/declared dependencies:** GSAP is declared but no use was found in the reviewed active source. Avoid carrying dependencies without a planned use.

### Server and protocol

- The current implementation stores all players and inputs in process memory. A server restart resets the room; there is no persistence or multi-room model.
- The server trusts client-provided camera yaw after only a finite-number check. Clamp or normalize yaw and validate sequence numbers.
- There is no explicit server message-size limit, rate limiter, or input coalescing policy.
- No movement speed, acceleration, collision, water boundary, boat collision, or terrain constraints can be confirmed because the imported physics module is missing.
- `movement.ts` is currently a helper disconnected from the production entry point. Either integrate it into the simulation or remove/replace it with the actual authoritative implementation.
- The shared `join` message type is unused; player names are fixed server-side.
- No explicit room lifecycle, room code, private invite, or matchmaking mechanism exists.

## 7. Build, package, and test configuration

### Workspace setup

Root `package.json` declares npm workspaces for `client`, `server`, and `shared`. A `pnpm-workspace.yaml` is also present, but the tree contains a root `package-lock.json` and no `pnpm-lock.yaml`. Choose one package manager and document it to avoid lockfile drift.

Root scripts:
- `dev`: runs workspace `dev` scripts through npm workspaces.
- `build`: runs workspace build scripts.
- `typecheck`: runs workspace typecheck scripts.

Potential orchestration issue: the root `dev` script does not request parallel workspace execution. npm workspace scripts run sequentially by default; the Vite dev server is long-running, so the server workspace may never start. Use a process supervisor/concurrently command or npm's supported parallel mode and verify both processes start.

### Shared package declaration output

`shared/package.json` exports `./network.js` with a `types` target at `./dist/network.d.ts` and a default target at `./src/network.ts`. The shared package's `tsconfig.json` is configured to emit declaration-only output to `dist`, but its package scripts define only `typecheck` (with `--noEmit`) and no `build` script. No `dist` output is committed.

This creates a packaging/type-resolution mismatch: the declared type target is not generated by the available workspace scripts. Add a shared build script and ensure the root build runs it before consumers, or revise exports to a resolution strategy that matches the workspace's actual build process.

### Tests

- Two unit tests exist in `server/src/movement.test.ts`.
- One integration test exists in `server/src/multiplayer.integration.test.ts`.
- Server package defines `node --import tsx --test src/*.test.ts`.
- No client test suite, browser automation, physics tests, shader tests, load tests, or CI workflow was found.
- Tests/build/typecheck were **not executed** during this repository review.
- Because the integration test launches `server/src/index.ts`, the missing physics module is a direct blocker to its intended execution.

### Kiro bug-fix documentation drift

The bugfix spec documents three compile-time bugs, while the design and task documents refer to a fourth missing `sequence` fixture field. The task checklist still shows all tasks unchecked, even though current source contains several described changes:
- `shared/package.json` now has an `exports` field.
- `client/tsconfig.json` includes `vite/client`.
- `ThirdPersonCamera` uses `PointerEvent` and `pointerdown`.
- Both movement test fixtures include `sequence: 0`.

The spec should be reconciled with the actual final state, test evidence, and any remaining blockers. Do not interpret the checked-in design summary as proof that the full repository builds.

## 8. Security and deployment posture

**Current appropriate deployment scope:** trusted LAN prototype only, after restoring the missing physics module and validating the server.

Before any public or untrusted-network deployment:
1. Authenticate room admission and prevent arbitrary slot consumption.
2. Add Origin policy, message schema validation, size limits, rate limits, heartbeat, and abuse controls.
3. Deploy behind TLS/WSS and a reverse proxy with explicit WebSocket upgrade support.
4. Add server-side authoritative collision and movement constraints.
5. Avoid logging raw user-controlled messages or sensitive session tokens.
6. Define room ownership, room isolation, and restart behavior.
7. Add monitoring for disconnect rates, tick overruns, snapshot size, and malformed input.

No secrets or database credentials were observed in the reviewed tracked files. The `.gitignore` excludes `.env` and `.env.*`.

## 9. Recommended remediation roadmap

### Phase A — restore a reproducible server baseline
- Restore/implement `server/src/physics.ts`.
- Ensure `server/src/index.ts` imports a tracked module.
- Resolve shared package declaration generation and workspace build order.
- Make root development command launch client and server concurrently.
- Run server typecheck, build, unit tests, and integration test.
- Add CI for client build/typecheck and server/shared typecheck/tests.

**Acceptance:** clean clone installs from one documented lockfile; server starts; two clients connect; four-player cap and disconnect/replacement integration test pass.

### Phase B — repair scene correctness
- Correct OceanSurface coordinate-space displacement and normal transformation.
- Replace empty-name collision filtering with explicit collidable layers/tags.
- Verify local player snaps to deck height and camera collision ignores water/self geometry.
- Fix per-frame React state updates and stabilize camera event callbacks.
- Add reconnect state handling and WSS-aware URL construction.

**Acceptance:** stable vertical waves, local player remains on deck, camera behaves correctly, and the client recovers from server restart.

### Phase C — harden multiplayer protocol
- Add room/session tokens and Origin policy.
- Add payload limits, input rate limits, sequence validation, heartbeat, and schema validation.
- Add movement/collision bounds and authoritative simulation tests.
- Add multi-room support only if required by product scope.

**Acceptance:** untrusted clients cannot trivially consume all slots or send unbounded input; simulation remains deterministic under malformed and high-frequency input.

### Phase D — build actual game systems
- Implement interaction actions for steering, storage, and crafting.
- Add resource nodes, gathering, inventory, and recipe-driven crafting.
- Add island terrain, underwater environment, resource spawning, and world boundaries.
- Add save/load and world/session persistence.
- Add NPC needs, navigation, trading, and economy only after deterministic gameplay foundations exist.

### Phase E — content and performance
- Decide whether procedural boat geometry or GLB assets are authoritative.
- Optimize the 58.5 MB ship asset and use proper asset versioning.
- Add rigged character model and animation pipeline.
- Profile GPU draw calls, shader cost, network snapshot bandwidth, and client frame time.
- Add accessibility, audio, UI/HUD, and packaging.

## 10. Coding-agent handoff

**First task: restore the server's missing physics dependency. Do not begin NPCs, crafting, or new world features before the multiplayer baseline is reproducible.**

Recommended execution order:
1. Inspect the current tree and verify whether `server/src/physics.ts` is intentionally absent.
2. Implement a minimal deterministic `PhysicsSimulation` contract or restore the intended file.
3. Make the existing `movement.ts` helper part of the production simulation, or consolidate the movement logic.
4. Add tests for player creation/removal, valid movement, invalid input, boundaries, and stable ticks.
5. Fix shared declaration build output and root process orchestration.
6. Run and report exact commands and results.
7. Repair the ocean shader and deck collision using explicit coordinate/collider semantics.
8. Only then update the development plan's milestone checkboxes based on actual passing evidence.

Coding-agent rules:
- Treat `docs/DEVELOPMENT_PLAN.md` as intended scope, not proof of completed features.
- Preserve server-authoritative movement; never trust client position.
- Keep network contracts in `shared/src/network.ts` and import them in both client and server.
- Keep external network payloads untrusted and bounded.
- Avoid adding heavyweight AI services before the core multiplayer loop is stable.
- Record exact files changed, commands run, and test results. Do not claim LAN, build, or multiplayer verification without reproducing it.

## 11. Inspection limitations

- Repository tree and key source/config/spec files were inspected through GitHub.
- No local clone, runtime, browser, GPU, or LAN multiplayer session was available in this review.
- Build, typecheck, unit tests, integration tests, shader compilation, and performance profiling were not executed.
- The missing `physics.js` import is established from the complete tracked tree; the exact runtime error was not reproduced.
- The visual effects of the ocean and deck raycast findings are derived from the inspected coordinate transforms and object creation code and should be confirmed after running the app.
