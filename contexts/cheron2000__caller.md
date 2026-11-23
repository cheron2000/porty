# Callvanta — source-aware repository context

**Repository:** `cheron2000/caller`  
**Product name:** Callvanta  
**Default branch:** `main`  
**Visibility:** Public  
**Inspected:** 2026-11-23  
**Tree snapshot:** `ff37ab372316e7fb6e7a15e194cecbe8a67660ea`  
**Repository size:** 198 KB  
**Repository type:** Android application + Python/FastAPI telephony gateway scaffold.

---

## 1. Executive summary

Callvanta is an Android application for local management of incoming SIM/cellular calls, with deterministic caller screening, locally stored screening rules and a limited history of screening events. A separate Python/FastAPI backend is a protocol scaffold for a future cloud AI receptionist reached through conditional forwarding from the user's existing Airtel number.

The repository has real Android and backend source code, unit tests and separate GitHub Actions workflows. The Android app implements the first local screening foundation. The voice backend currently authenticates an Exotel-style WebSocket, parses events and validates PCM frames, but discards inbound audio. It does not contain speech recognition, LLM orchestration, text-to-speech, a greeting or a working AI receptionist.

**Do not conflate the two paths:**
- Android `CallScreeningService` makes a fast local allow/silence/reject decision. It is not a call-answering interface and does not provide unrestricted cellular audio.
- Conditional forwarding routes unanswered/busy/unreachable calls through the carrier to a cloud telephony provider and then to a future voice backend. This is separate from Android's screening callback.

## 2. Product scope and approved direction

### Implemented Android milestone
- Incoming ordinary SIM/cellular call screening.
- User-controlled Android call-screening role.
- Local exact-number and prefix rules.
- Allow, silence and reject actions.
- Saved-contact calls can be delivered to the screening service after the user grants Contacts permission; exact/prefix rules are then evaluated against the supplied telephone number.
- Local history of Callvanta-observed screening events only.
- Per-event deletion, delete-all and automatic 30-day retention.
- Android cloud backup disabled.

The owner confirmed saved-contact screening rules work on the Vivo Y39 after granting Contacts permission (recorded in the project work queue). This is project-reported device validation, not a test performed during this repository review.

### Future AI receptionist direction
The owner selected conditional forwarding from the existing Airtel India mobile number. The phone should ring normally first, then forward unanswered, busy or unreachable calls where supported. Unconditional forwarding is not the default.

Exotel Voicebot Applet is the first provider candidate, not a confirmed production provider. Account eligibility, number compatibility, feature enablement, pricing and forwarding behavior remain validation gates. The project explicitly says not to activate forwarding to the current backend scaffold or the owner's personal number before the full pipeline is proven on a provider test number and the owner approves.

## 3. Repository map

### Android application
- `app/build.gradle.kts` — Android app module, Kotlin Compose plugin, compile/target SDK 36, min SDK 29, Java/Kotlin 17, version 0.4.0.
- `app/src/main/AndroidManifest.xml` — READ_CONTACTS permission, application, launcher activity and protected CallScreeningService declaration.
- `app/src/main/java/com/callvanta/ai/MainActivity.kt` — Compose UI, role and contacts permission state, rule CRUD, number validation, history display/deletion.
- `CallvantaApplication.kt` — loads rules and events at process startup, publishes in-memory rule snapshot, serializes event writes on a single-thread executor, schedules daily retention cleanup.
- `domain/screening/` — caller identity, matchers, action/rule/decision models, deterministic policy, phone-number normalization.
- `telecom/screening/` — Android Telecom service adapter and caller identity mapper.
- `data/rules/` — local JSON SharedPreferences rule store.
- `data/history/` — local JSON SharedPreferences event store and WorkManager cleanup worker.
- `app/src/test/` — JVM tests for policy, number normalization, caller identity mapping and event retention.

### Backend
- `backend/app/main.py` — FastAPI health/readiness, provider status and token-protected WebSocket endpoint.
- `backend/app/exotel.py` — Exotel event parser, base64/PCM validation and outbound media-frame encoder.
- `backend/tests/test_exotel.py` — parser and media-frame tests.
- `backend/pyproject.toml` — Python 3.11+, FastAPI, Uvicorn and pytest/httpx dev dependencies.
- `backend/.env.example` — environment configuration template.

### Project governance and documentation
- `AGENTS.md` — single-agent workflow, safety rules and current milestone boundaries.
- `docs/PROJECT_PLAN.md`, `docs/ARCHITECTURE.md`, `docs/ANDROID_PLATFORM_CONSTRAINTS.md`, `docs/ROADMAP.md`.
- `docs/coordination/DECISIONS.md`, `WORK_QUEUE.md`, implementation/review/handoff documents.
- `.agents/skills/` and mirrored `.claude/skills/` — repository-specific instructions for coordination, Android, telecom feasibility, backend, security and testing.
- `.github/workflows/android-ci.yml` and `backend-ci.yml` — separate build/test workflows.

## 4. Android implementation details

### 4.1 Application and role lifecycle

The manifest registers `CallvantaCallScreeningService` with:
- `android.permission.BIND_SCREENING_SERVICE`
- `android.telecom.CallScreeningService` intent action
- exported service declaration as required for the system binding.

The app requests `RoleManager.ROLE_CALL_SCREENING` only through a user-triggered action. UI state distinguishes unavailable, not enabled, granted, denied and revoked. Contacts permission is separately requested at user action and rechecked on resume.

The app does not request default-dialer role, call-log access, microphone or audio-recording permissions in the inspected manifest.

### 4.2 Screening callback

`CallvantaCallScreeningService.onScreenCall`:
1. Ignores outgoing callbacks.
2. Reads the provided call handle.
3. Maps only a `tel` scheme into a normalized telephone identity; missing/non-telephone metadata maps to `CallerIdentity.Missing`.
4. Evaluates the in-memory immutable rule snapshot with the pure `ScreeningPolicy`.
5. Maps policy results to Android `CallResponse.Builder` behavior.
6. Uses allow as the exception/fail-open fallback.
7. Calls `respondToCall` before queueing event persistence.

No network, LLM, audio capture, UI launch or synchronous storage is performed in the callback. This is consistent with the short Telecom response deadline and the project's local deterministic policy.

### 4.3 Policy semantics

`ScreeningPolicy` supports:
- `ALLOW`
- `SILENCE`
- `REJECT`

Caller identities distinguish normalized number, withheld, unknown and missing. The matcher model includes exact number, number prefix and withheld/unknown category. Matching order is:
1. Higher explicit priority.
2. More-specific matcher.
3. Earlier creation order.

Disabled rules are ignored. Missing caller metadata and unavailable policy data fail open to allow. No match defaults to allow. Saved contacts default to allow when no explicit rule matches; an explicit exact/prefix rule can still apply.

**Platform nuance:** Android does not provide third-party screening callbacks for all restricted/unknown/unavailable/payphone caller-ID presentations. The UI labels the legacy withheld/unknown rule as unsupported and prevents editing/toggling it. It should not be described as a reliable way to screen hidden caller IDs.

### 4.4 Phone-number normalization

`NumberNormalizer`:
- Removes common formatting characters.
- Normalizes 10-digit Indian numbers.
- Handles leading national trunk zero and +91 country code.
- Preserves explicit international numbers in +E.164 form.
- Supports prefix matching with a minimum of six significant digits and rejects overly broad prefixes.
- Rejects malformed characters and invalid lengths.

Unit tests cover common Indian formats, international E.164, invalid exact numbers, valid specific prefixes and broad-prefix rejection.

### 4.5 Persistence and retention

Rules are serialized as JSON in a dedicated SharedPreferences file. The screening service never reads preferences directly; `CallvantaApplication` loads them at startup and publishes an immutable snapshot. UI rule updates save to preferences and replace the snapshot.

Screening events contain only:
- generated event ID,
- occurrence timestamp,
- normalized caller number when available,
- screening action.

Events are stored as JSON in a separate SharedPreferences file. A single-thread executor queues writes/deletes after the Telecom response. WorkManager schedules daily cleanup; retention is 30 days. UI exposes per-record delete and delete-all. Backup is disabled in the manifest.

**Privacy limitation:** the inspected storage is app-private SharedPreferences JSON, not encrypted Room storage. The app minimizes data and disables backup, but the report should not claim at-rest encryption. Consider encrypted storage only after threat-model and dependency review; keep data minimization and retention as the first protections.

## 5. Backend implementation details

### 5.1 HTTP endpoints
- `GET /health` — process health.
- `GET /ready` — reports whether `CALLVANTA_EXOTEL_WS_TOKEN` is configured and explicitly reports `voice_pipeline: not_implemented`.
- `GET /v1/telephony/exotel/status` — reports a WebSocket protocol scaffold, not a working voice integration.

### 5.2 WebSocket endpoint
`WS /v1/telephony/exotel/media?token=...`:
- Reads configured token from server environment.
- Uses constant-time `hmac.compare_digest` to compare supplied token.
- Rejects unauthenticated clients before accepting the socket.
- Parses incoming text JSON events.
- Tracks the start stream ID locally.
- Currently discards validated media; clear/DTMF events are placeholders; stop/disconnect exits the loop.
- Does not persist or log raw audio/transcripts/provider payloads.

The token is supplied in the query string. Reverse-proxy and application access logs must redact it. A future implementation should consider provider-native signature/authentication and per-call credentials rather than relying on a long-lived shared query token alone.

### 5.3 Exotel protocol parser
`parse_event` validates JSON object shape and supports:
- `start` with stream ID.
- `connected`, `stop`, `clear`.
- `media` with base64 payload, stream ID and bounded audio size.
- `dtmf`.
- `mark`.

PCM constraints in code: 8 kHz mono linear16; minimum 3,200 bytes, maximum 100,000 bytes, size multiple of 320 bytes. `make_media_frame` encodes validated outbound media frames, but no live pipeline currently calls it.

**Potential protocol consistency issue:** `main.py` contains a `disconnect` branch, but `parse_event` does not accept `disconnect` as a supported event type. If the actual provider emits this event, it will be rejected by the parser before the branch is reached. Confirm the provider's exact event contract and either remove the dead branch or explicitly support the documented event with tests.

### 5.4 Missing voice pipeline

Not implemented:
- streaming speech-to-text,
- conversation/session orchestrator,
- LLM client and tool policy,
- streaming text-to-speech,
- outbound audio playback loop,
- AI identity/privacy disclosure prompt,
- interruption/barge-in,
- caller-requested exit,
- human/voicemail escalation and provider fallback,
- durable call-session lifecycle and operational metrics.

The backend README explicitly warns not to forward live Airtel calls to this scaffold.

## 6. Tests and CI evidence

### Tests present
Android JVM tests cover:
- policy fail-open/default allow,
- exact/prefix rule matching and precedence,
- disabled rules,
- withheld/unknown/missing identity semantics,
- saved-contact default allow,
- Indian/international number normalization,
- caller identity mapping,
- 30-day retention and event ordering.

Backend tests cover:
- start event parsing,
- valid PCM decode,
- outbound media-frame encoding,
- invalid base64,
- invalid audio frame sizes,
- missing stream ID,
- unknown event and malformed start frame.

### GitHub Actions
- Android CI uses JDK 17, Android SDK platform 36 and Gradle 9.6.0; runs `gradle :app:assembleDebug :app:testDebugUnitTest` and uploads a debug APK artifact for seven days.
- Backend CI uses Python 3.12, installs `.[dev]` and runs `pytest` on backend changes, pull requests and manual dispatch.

**Verification limitation:** source and workflow files were inspected, but no build, test suite, workflow run or physical-device test was executed as part of this review. CI configuration is not proof of a passing run.

## 7. Findings, inconsistencies and risks

### High priority
1. **AI receptionist is not operational.** The backend receives and validates protocol frames but discards audio. Do not enable conditional forwarding to it.
2. **Device validation is incomplete.** The project targets Vivo Y39 and has owner-reported saved-contact verification, but exact Android/API/Funtouch build, callback timing, cold-start and background behavior still need a documented matrix.
3. **Token query-string exposure.** Ensure the WebSocket token is redacted from reverse-proxy, hosting and application access logs; rotate immediately if exposed.
4. **Provider contract remains unverified.** Exotel Voicebot Applet availability, number/account eligibility, media event details, forwarding compatibility and charges remain open.
5. **No automated end-to-end telephony test.** Parser tests do not establish real provider interoperability or bidirectional voice quality.

### Medium priority
6. **Backend event handling mismatch:** the WebSocket loop handles `disconnect`, while parser does not emit that event.
7. **UI concentration:** `MainActivity.kt` contains Activity lifecycle, permission/role state, rule management callbacks and a large Compose screen. As the product grows, split UI screens/state holders and move use cases out of the Activity while preserving the current package-level domain/telecom separation.
8. **Persistence is JSON SharedPreferences:** adequate for this small local MVP, but it lacks relational querying/migrations and explicit encryption. Avoid premature Room migration; define scale and security requirements first.
9. **Work queue/roadmap drift:** `WORK_QUEUE.md` reports CV-012 and CV-014 as in progress despite the inspected source containing rule CRUD/persistence and event history/retention implementations; CV-013 is marked device verified. The older Phase 1/2 checkboxes in `ROADMAP.md` remain unchecked even though later work queue entries say those features were implemented. Reconcile documentation with current code and actual CI/device evidence before using it as a status dashboard.
10. **Potential dead/legacy withheld matcher:** domain and storage still support it although the UI marks it unsupported by the Android callback contract. Keep migration/cleanup behavior explicit and avoid presenting it as functional.

## 8. Recommended next engineering steps

### Before enabling any carrier forwarding
1. Confirm Exotel account eligibility, Voicebot Applet enablement, destination-number support and pricing.
2. Verify current Airtel conditional-forwarding support/charges for the specific subscriber plan.
3. Complete a provider test-number media proof-of-concept.
4. Implement per-call session state, inbound STT, LLM conversation orchestration, TTS and outbound media.
5. Add AI identity/privacy disclosure, consent handling where required, retention controls and safe failure/escalation.
6. Test two-way audio, latency, barge-in, disconnect, DTMF, provider retries and fallback using a test number.
7. Obtain explicit owner approval before any forwarding from the personal SIM.

### Android hardening
1. Reconcile `WORK_QUEUE.md` and `ROADMAP.md` with implemented code and latest CI evidence.
2. Record exact Vivo Y39 model, Android API level, Funtouch OS build, SIM/carrier and manual test results.
3. Add tests for corrupted rule/event preferences, persistence ordering, cleanup worker and role revocation/reboot behavior.
4. Consider extracting Compose screens/state from `MainActivity` as UI scope grows.
5. Maintain fail-open, bounded, offline screening behavior; do not add network/LLM dependencies to the Telecom callback.

### Backend hardening
1. Resolve `disconnect` parser/handler mismatch against the real provider schema.
2. Redact query-string token in all access logs; prefer stronger provider-native authentication where available.
3. Add maximum WebSocket lifetime, idle timeout, per-call resource limits, rate limits and session isolation.
4. Add provider contract tests using synthetic media and a sandbox/test number.
5. Add metrics that do not expose phone numbers, audio, transcripts or secrets.
6. Keep `/ready` truthful until STT/LLM/TTS and safe fallback are actually operational.

## 9. Coding-agent operating rules

- Treat `cheron2000/caller` as the canonical source of truth.
- Read `AGENTS.md`, project plan, architecture, platform constraints, roadmap, decision log and work queue before changing scope.
- ChatGPT is the sole active AI agent unless the owner changes the assignment.
- Use focused direct commits to `main`; do not open PRs unless asked or staffing changes.
- Do not silently alter approved product behavior, privacy, platform strategy or major dependencies.
- Keep local screening deterministic, bounded and independent of network/LLM availability.
- Never claim CallScreeningService provides live two-way call audio.
- Never implement covert recording or undisclosed transmission.
- Keep all provider/AI secrets out of source, APKs, logs and documentation.
- Use synthetic test data.
- Do not activate carrier forwarding until the full voice pipeline is validated on a provider test number and the owner explicitly approves.
- Never claim tests, builds, provider checks or device validation unless actually executed and evidenced.
- Update decision log/work queue when scope or task state changes.

## 10. Key source files

- [README](https://github.com/cheron2000/caller/blob/main/README.md)
- [Agent instructions](https://github.com/cheron2000/caller/blob/main/AGENTS.md)
- [Android manifest](https://github.com/cheron2000/caller/blob/main/app/src/main/AndroidManifest.xml)
- [Main Activity](https://github.com/cheron2000/caller/blob/main/app/src/main/java/com/callvanta/ai/MainActivity.kt)
- [Screening policy](https://github.com/cheron2000/caller/blob/main/app/src/main/java/com/callvanta/ai/domain/screening/ScreeningPolicy.kt)
- [Number normalizer](https://github.com/cheron2000/caller/blob/main/app/src/main/java/com/callvanta/ai/domain/screening/NumberNormalizer.kt)
- [Call screening service](https://github.com/cheron2000/caller/blob/main/app/src/main/java/com/callvanta/ai/telecom/screening/CallvantaCallScreeningService.kt)
- [Voice gateway](https://github.com/cheron2000/caller/blob/main/backend/app/main.py)
- [Exotel protocol adapter](https://github.com/cheron2000/caller/blob/main/backend/app/exotel.py)
- [Architecture](https://github.com/cheron2000/caller/blob/main/docs/ARCHITECTURE.md)
- [Platform constraints](https://github.com/cheron2000/caller/blob/main/docs/ANDROID_PLATFORM_CONSTRAINTS.md)
- [Decision log](https://github.com/cheron2000/caller/blob/main/docs/coordination/DECISIONS.md)
- [Work queue](https://github.com/cheron2000/caller/blob/main/docs/coordination/WORK_QUEUE.md)

---

## Continuation handoff

Callvanta currently has a local Android call-screening implementation with deterministic rules, saved-contact support after permission, and local 30-day screening-event history. The Python/FastAPI Exotel gateway is only a protocol scaffold that discards inbound audio. The next high-value work is to reconcile documentation status, finish and validate the backend voice pipeline on a provider test number, and complete the exact target-device validation. Do not activate forwarding on the owner's personal Airtel SIM until explicit approval.
