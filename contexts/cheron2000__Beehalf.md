# Beehalf — repository context and engineering handoff

**Repository:** `cheron2000/Beehalf`  
**Default branch:** `main`  
**Inspected:** 2026-11-23  
**Repository tree:** `f8f8c5870ad9dddceb0918bb1e380a69bc25ebd2`  
**Purpose:** Source-aware technical context for future coding agents. Verify current source before changing behavior; this document is an orientation and handoff, not a substitute for tests.

---

## 1. Executive summary

Beehalf is an Android-first, local-first AI assistant and future permission-controlled automation shell. It is implemented in Kotlin with Jetpack Compose, Room, Android Keystore, WorkManager dependency, OkHttp, coroutines, and a Groq OpenAI-compatible chat-completions endpoint using bring-your-own-key (BYOK).

The current implementation includes:
- A Compose chat UI with persistent conversations and messages.
- Groq key validation and encrypted local credential storage.
- Bounded recent-message context and local memory retrieval for prompts.
- Rolling conversation summaries and user-reviewed memory extraction.
- A local memory management screen with edit, pin, delete and retention controls.
- Storage Access Framework document references and bounded on-device TXT/Markdown parsing.
- Typed automation event/action contracts, fail-closed policy checks, Android capability discovery, and a persistent user approval queue.
- Unit tests and GitHub Actions for JVM unit tests and debug APK assembly.

The README describes the project as an **early foundation**. The Android automation execution layer and WhatsApp notification adapter are not implemented as a complete integration yet. In particular, approval records a decision only; it does not execute an Android action.

## 2. Product intent and core principles

Beehalf aims to be a private, local-first assistant with remote inference as a replaceable provider boundary:
- Conversation history, memory records, document references and automation configuration are stored locally.
- Groq is the initial inference provider, not the owner of local memory.
- Users supply their own API key; no bundled provider credential or user-managed server is intended.
- Only selected, bounded prompt context is sent to Groq. Any included message, memory, summary or future RAG chunk leaves the device.
- LLM output, retrieved documents and notification content are untrusted data.
- Device-affecting actions must pass typed validation, capability checks and explicit user approval.
- App integrations should depend on shared contracts; core logic should not depend on WhatsApp.

The first integration target is WhatsApp notification assistance, using Android-supported notification access and reply actions where available. Notification access does not grant access to the full WhatsApp database. AccessibilityService is not a general-purpose bypass and should not be introduced as an unrestricted automation mechanism.

## 3. Repository map

### Android application
- `app/src/main/AndroidManifest.xml`: Internet permission, launcher activity, cleartext traffic disabled, narrow launcher intent query. No notification listener service is registered in the inspected manifest.
- `app/src/main/java/dev/beehalf/MainActivity.kt`: application composition root and primary Compose navigation between chat, memory, documents and approvals. Constructs Room repositories, key vault and ViewModel factory.
- `app/src/main/java/dev/beehalf/chat/ChatViewModel.kt`: chat state, conversation selection, message persistence, context assembly and Groq request lifecycle.
- `app/src/main/java/dev/beehalf/ai/groq/GroqClient.kt`: OkHttp transport for Groq model listing and chat completions.
- `app/src/main/java/dev/beehalf/security/ApiKeyVault.kt`: AES-GCM encryption using a key in Android Keystore and private SharedPreferences ciphertext storage.
- `app/src/main/java/dev/beehalf/MemoryScreen.kt` and `memory/`: memory editing, candidate extraction, summaries, context construction and retention.
- `app/src/main/java/dev/beehalf/rag/TextDocumentParser.kt`: bounded TXT/Markdown text extraction.
- `app/src/main/java/dev/beehalf/data/DocumentRepository.kt`: SAF import, persistent URI grant, metadata persistence and text reading.
- `app/src/main/java/dev/beehalf/data/local/`: Room entities, DAO and database migrations.
- `app/src/main/java/dev/beehalf/automation/`: typed actions/events, action policy, Android capability checks and approval repository.
- `app/src/main/java/dev/beehalf/ui/`: document and approval queue screens.

### Build and documentation
- Root `build.gradle.kts`, `settings.gradle.kts`, `gradle.properties`: Gradle project configuration.
- `app/build.gradle.kts`: Android/Kotlin/Compose/KSP plugins, SDK levels, dependencies and test libraries.
- `.github/workflows/android-ci.yml`: JDK 17 + Android SDK 35 CI; JVM unit tests and debug APK build; uploads APK artifact.
- `docs/ARCHITECTURE.md`: module boundaries and security/data-flow invariants.
- `docs/SECURITY.md`: credential, data handling, automation and Android permission expectations.
- `docs/ROADMAP.md`: milestones and incomplete feature list.
- `CONTRIBUTING.md`: contribution guidance.

Architecture documentation describes future `core`, `integrations`, and `background` module boundaries, but the current Gradle tree is a single Android app module. Do not assume those described modules exist yet.

## 4. Chat and inference flow

1. `MainActivity` creates the Room database and obtains `BeehalfDao`.
2. It constructs `ConversationRepository`, `MemoryRepository`, `ConversationSummaryRepository`, `DocumentRepository`, and `ActionApprovalRepository`.
3. `ChatViewModel` observes conversation records and message streams. It creates a conversation if none exists and supports selecting/creating chats.
4. On send, it reads the BYOK key from `ApiKeyVault`, persists the user message, fetches up to 40 recent messages, and builds a bounded chronological context window.
5. It retrieves up to 12 memory records for context and inserts them into the system prompt as untrusted reference data.
6. It calls `GroqClient.complete` with model ID and chat messages, then persists the assistant response.
7. Summary refresh runs best-effort after the assistant response. A summary failure should not invalidate an already-saved reply.
8. UI state exposes generic connection/model errors without displaying raw provider response bodies.

### Context window
`ContextWindowBuilder` defaults to a maximum of 40 messages and an approximate 6,000-token budget, estimating tokens as roughly one per four characters. It selects newest messages, truncates an oversized single newest message, then returns selected messages in chronological order. This is an approximation, not provider tokenizer accounting.

### Groq client
- Base URL defaults to `https://api.groq.com/openai/v1`.
- `validate` calls `GET /models` using bearer authentication and returns sorted model IDs.
- `complete` calls `POST /chat/completions`, validates required inputs and temperature range, and returns the first assistant message content.
- OkHttp connect/read/call timeouts are configured (15s/60s/75s).
- Non-success responses throw an error containing the HTTP status only, not the response body.
- The client documentation explicitly prohibits logging headers, API keys or conversation contents.

Potential improvement: expose a selectable/validated model picker rather than allowing an arbitrary free-form model ID; distinguish rate-limit, authentication, network, invalid-model and malformed-response errors safely.

## 5. Credential storage and privacy boundary

`ApiKeyVault`:
- Generates/retrieves an AES key from Android Keystore.
- Uses `AES/GCM/NoPadding` with randomized encryption.
- Stores Base64 ciphertext and IV in app-private SharedPreferences.
- Reads by decrypting with the Keystore key.
- Clears preferences and deletes the key alias when removing credentials.

Important review points:
- `clear()` currently clears the entire `secure_provider_config` preferences file, not only the Groq key fields. This is safe only while the file is dedicated to this vault; preserve this invariant or narrow the deletion.
- `read()` returns null on decryption/key errors, which makes a corrupted vault indistinguishable from a missing key. Consider safe diagnostics and a recovery path without exposing secrets.
- Device backup is disabled in the manifest; keep backup policy aligned with the encrypted credential and local-data threat model.
- No secret should enter logs, analytics, crash reports, build config, source control or prompts.
- Groq inference is remote. Explain clearly that selected context is transmitted; local-first does not mean inference is fully offline.

## 6. Memory subsystem

### Persistent memory
`MemoryRepository` supports observing memories, category filtering, bounded context retrieval, save/update, pin/unpin, delete and retention cleanup of old unpinned records. Each memory stores category, source, confidence, timestamps and pinned state.

Categories: working, episodic, semantic, procedural and knowledge. User-created memories use source `user`; extracted records should retain their originating conversation/document source.

### Candidate extraction
`MemoryCandidateExtractor`:
- Selects the latest 20 user/assistant messages and bounds transcript size.
- Requests JSON-only candidate memories from Groq.
- Instructs the model not to follow transcript instructions and to omit sensitive data.
- Caps output at five items, requires known category, nonblank content up to 300 characters, and confidence 0.65–1.0.
- Returns proposals only; it does not persist them.

`MemoryScreen` presents each candidate with rationale/confidence and explicit “Approve & save” or “Dismiss” controls. This is a useful human-in-the-loop boundary, but model-generated confidence is not calibrated truth. Users must be able to inspect/edit/delete memories and should not be pressured to accept candidates.

### Rolling summaries
`ConversationSummaryRepository` keeps one working summary per conversation. It summarizes messages older than the recent message window, only after a threshold, and stores a source watermark. The summary is pinned and later included as untrusted context. Summarization is best-effort.

**Potential correctness issue to investigate:** the watermark is based on `createdAt` timestamps and DAO selection uses strict `createdAt > afterExclusive` and `createdAt < beforeExclusive`. If multiple messages share a timestamp (possible with millisecond timestamps under fast inserts), messages at the watermark can be skipped permanently. Use a stable ordered cursor (e.g. createdAt + message ID or monotonic sequence) and test equal-timestamp messages. Also verify concurrent refreshes cannot race and overwrite a newer summary.

### Context assembly and privacy
Memory and summary content are explicitly described to the model as untrusted reference data. Preserve that separation. Add tests for prompt injection in saved memories and summaries, stale/conflicting memories, and sensitive-data exclusion at every summarization/extraction stage.

## 7. Local document ingestion and RAG status

### Implemented
- User picks documents via Android Storage Access Framework.
- Repository takes a persistable read permission and stores URI, display name, MIME type, optional size and import timestamp in Room.
- Duplicate URI imports update the existing row.
- Removing a document deletes metadata and attempts to release the persisted URI grant.
- Text content is opened only when requested.
- `TextDocumentParser` supports TXT and Markdown, enforces a 2 MB input limit, handles UTF-8/UTF-16 BOMs, normalizes line endings and rejects NUL-containing binary-like content.
- Markdown is preserved as source text.
- UI preview is capped at 12,000 characters.

### Not implemented yet
- PDF, CSV, JSON, DOCX and XLSX parsing.
- Chunking and metadata.
- On-device embedding runtime.
- Hybrid full-text/vector retrieval.
- Source citations and retrieval evaluation.
- Integration of retrieved chunks into chat context.

Do not describe Beehalf as having a complete RAG pipeline. It currently has document references and a bounded plain-text parser foundation; no embedding/index/retrieval pipeline is present in the inspected source.

### Document ingestion risks
- `importFromUri` requests a persistable grant before all metadata/database operations succeed. Consider rollback/release on failed import.
- URI providers can become unavailable, permissions can be revoked, and content may change. Handle stale URIs and changed metadata explicitly.
- Parser input is bounded, but tests should cover oversized streams, malformed encodings, unusual MIME/extensions, NUL bytes and SAF provider failures.
- Parsed content is not persisted or uploaded in this slice. Keep that privacy statement accurate until chunk/index storage is implemented.

## 8. Safe automation foundation

### Contracts and event validation
`ActionContracts.kt` defines data-only proposal types:
- Open a named app package.
- Create a reminder at a future timestamp.
- Open an absolute HTTPS URL.
- Unsupported action marker.

`AutomationEventPolicy` validates event IDs/timestamps and source consistency for user commands, Android notification events and scheduled reminder events. Notification event payloads carry package/key metadata rather than notification text. Acceptance means “eligible for routing”; it does not authorize or execute an action.

### Action policy
`ActionPolicy` fails closed on invalid IDs, summaries, timestamps, package names, reminder times and URLs. Only HTTPS absolute URLs with a host are allowed. A valid result is `APPROVAL_REQUIRED`, never auto-approved.

### Capability discovery
`AndroidCapabilityChecker` performs read-only discovery for launchable app packages, alarm handlers and HTTPS handlers. It does not start an activity or schedule an alarm. Handler availability is not user consent.

### Approval repository and UI
- Proposals are persisted in Room with typed action details, summary, status, created/expiry/decision timestamps.
- Default expiry is 24 hours.
- Stale pending proposals are marked expired and cannot be approved.
- Approve/reject is a single-use state update.
- UI displays the exact package, HTTPS target or reminder title/time before asking for approval.
- `ActionApprovalRepository` has no Android execution dependency; approval records the user's decision only.

**Critical current boundary:** no action executor or idempotent side-effect/audit pipeline is implemented in the inspected source. Do not add execution directly inside the approval repository or Compose callbacks. Build a separate executor with capability re-check, explicit approved-state check, idempotency key, permission checks, cancellation/kill switch, result state and durable audit record. Only then wire Android intent/alarm handoffs.

## 9. WhatsApp and Android integration status

The README and roadmap identify WhatsApp notification assistance as the first integration, but M5 is unchecked:
- No `NotificationListenerService` implementation is present in the inspected tree.
- The manifest does not declare/register a notification listener service.
- Notification parsing, conversation matching, response suggestions and supported quick-reply flow remain roadmap work.
- No full WhatsApp database access is available through standard notification APIs.
- Any reply must use a supported notification action and explicit user approval.

Other Android integrations, workflow configuration, kill switch, idempotency and audit log are also incomplete. Do not use AccessibilityService as a general-purpose autonomous control mechanism.

## 10. Database and data model

`BeehalfDatabase` uses Room with database name `beehalf-local.db`, schema version 3 and migrations:
- Migration 1→2 creates `documents` and indexes URI/import time.
- Migration 2→3 creates `action_proposals` and indexes status/expiry.
- Current entities: conversations, messages, memories, documents and action proposals.
- `BeehalfDao` provides reactive Flow observation and suspend persistence/query operations.

Schema export is disabled (`exportSchema = false`). This is a release-quality risk: enable Room schema export, commit schema snapshots, and test every migration from prior versions. Add migration tests using actual Room database snapshots rather than relying only on DAO fakes.

Potential data lifecycle gaps to review:
- Conversation deletion methods exist in DAO, but inspect UI/repository pathways to ensure dependent messages/summaries are removed consistently.
- User-facing export and complete deletion verification are explicitly roadmap items.
- Persisted SAF grants must be reconciled if a document row is deleted or database migration fails.
- Sensitive data retention and memory cleanup need clear user controls.

## 11. Build, dependencies and CI

Build configuration:
- Android Gradle plugin/Kotlin/Compose/KSP plugins configured in Gradle.
- Namespace/application ID: `dev.beehalf`.
- Compile SDK 35, target SDK 35, minimum SDK 29.
- Java/Kotlin target 17.
- Compose BOM 2025.02.00; Room 2.7.0; DataStore 1.1.3; WorkManager 2.10.0; OkHttp 4.12.0; coroutines Android 1.10.1.
- Unit tests use JUnit 4, MockWebServer and kotlinx-coroutines-test.

CI workflow:
1. Checkout.
2. Set up Temurin JDK 17.
3. Install Android SDK platform/build tools 35.
4. Set up Gradle 8.11.1.
5. Run `gradle :app:testDebugUnitTest`.
6. Run `gradle :app:assembleDebug`.
7. Upload debug APK artifact for 14 days.

Tests inspected include Groq HTTP behavior/auth/error handling, summary watermark behavior, memory context/retention, text parser, automation event/action policy and approval lifecycle. The tests are JVM unit tests; they do not replace instrumentation/device/OEM tests for Android notification access, URI grants, Keystore behavior or intent handoff.

No build or tests were run during this Porty inspection. CI configuration exists, but do not claim current branch CI is green without checking an actual workflow run.

## 12. Engineering risks and recommended priorities

### P0 — Preserve safe automation boundary
- Keep proposal validation, capability discovery, user approval and execution as distinct stages.
- Do not let model output or notification text directly invoke Android APIs.
- Add executor only as a separate component with idempotency, revalidation, audit, failure handling and a kill switch.
- Require explicit consent and narrow permissions for every integration.

### P1 — Memory correctness and data governance
- Replace timestamp-only summary watermark with a stable cursor and cover same-timestamp messages.
- Serialize or transactionally coordinate summary refreshes per conversation.
- Add prompt-injection and sensitive-data tests for memories, summaries and future RAG chunks.
- Provide export, deletion and retention controls with verified database/URI cleanup.
- Treat model-extracted confidence as heuristic, not a calibrated probability.

### P1 — Finish local RAG before advertising it
- Add parsers for the planned formats with strict size/resource limits.
- Chunk while preserving source offsets, headings and metadata.
- Implement on-device embedding and hybrid retrieval.
- Add source citations and a small retrieval evaluation set.
- Keep documents and embeddings local by default; disclose any context selected for remote inference.

### P1 — Room schema/migration discipline
- Enable Room schema export and commit snapshots.
- Add migration tests from every supported schema version.
- Define deletion cascades or explicit transactional repository operations for conversations/messages/summaries.
- Handle SAF permission rollback and stale URI references.

### P2 — Architecture and UX
- Extract Compose screens and dependency construction from the large `MainActivity.kt`.
- Introduce explicit dependency injection or a small application container.
- Separate Groq transport, prompt construction and provider configuration.
- Add a proper model picker and safe diagnostics.
- Build feature modules only when useful; docs describe future layers that are not separate modules today.
- Add device/OEM coverage for notification and background execution once those integrations exist.

## 13. Coding-agent rules

1. Read `docs/ARCHITECTURE.md`, `docs/SECURITY.md`, `docs/ROADMAP.md` and the relevant source before changing behavior.
2. Treat the current code as source of truth where docs describe future architecture.
3. Never add API keys or credentials to source, Gradle configuration, logs, analytics, crash reports or prompts.
4. Keep the BYOK key in Android Keystore-backed storage; do not expose plaintext to UI logs or diagnostics.
5. Explicitly disclose that selected context is sent to Groq.
6. Treat messages, summaries, memories, documents and notifications as untrusted prompt data.
7. Do not persist extracted memory candidates without explicit user approval.
8. Do not execute an action merely because policy validation or capability discovery succeeds.
9. Keep approval repository decision-only; add a separate executor with idempotency and audit before any side effect.
10. Do not claim complete RAG or WhatsApp integration until parsers/retrieval and notification service/reply flow actually exist.
11. Preserve data migrations and add migration tests for every Room schema change.
12. Run `gradle :app:testDebugUnitTest` and `gradle :app:assembleDebug`; report exact outcomes. Add device/instrumentation tests where platform behavior is involved.
13. Avoid adding broad Android permissions or AccessibilityService without a narrowly justified use case, user disclosure and policy review.
14. Update roadmap and security/architecture docs in the same change when implementation status changes.

## 14. Suggested implementation sequence

1. Add Room schema export and migration tests; verify deletion/retention semantics.
2. Fix summary watermark stability and concurrent refresh behavior with regression tests.
3. Improve credential-vault recovery and narrow preference clearing.
4. Complete document import failure/permission rollback and parser robustness tests.
5. Implement a distinct idempotent action executor and durable audit/kill switch; retain explicit approval.
6. Complete bounded on-device RAG, citations and evaluation before calling it a RAG feature.
7. Add opt-in notification listener and supported reply integration with clear consent, device/OEM tests and policy review.
8. Refactor UI/dependency construction and improve provider/model diagnostics.
9. Complete export/deletion, threat model, privacy and release-signing work before production release.

## 15. Source references inspected

- Repository: https://github.com/cheron2000/Beehalf
- README: https://github.com/cheron2000/Beehalf/blob/main/README.md
- Android entry/UI: https://github.com/cheron2000/Beehalf/blob/main/app/src/main/java/dev/beehalf/MainActivity.kt
- Chat ViewModel: https://github.com/cheron2000/Beehalf/blob/main/app/src/main/java/dev/beehalf/chat/ChatViewModel.kt
- Groq client: https://github.com/cheron2000/Beehalf/blob/main/app/src/main/java/dev/beehalf/ai/groq/GroqClient.kt
- API key vault: https://github.com/cheron2000/Beehalf/blob/main/app/src/main/java/dev/beehalf/security/ApiKeyVault.kt
- Memory screen: https://github.com/cheron2000/Beehalf/blob/main/app/src/main/java/dev/beehalf/MemoryScreen.kt
- Memory summary/extraction: https://github.com/cheron2000/Beehalf/tree/main/app/src/main/java/dev/beehalf/memory
- Document parser: https://github.com/cheron2000/Beehalf/blob/main/app/src/main/java/dev/beehalf/rag/TextDocumentParser.kt
- Automation policy/approval: https://github.com/cheron2000/Beehalf/tree/main/app/src/main/java/dev/beehalf/automation
- Room database: https://github.com/cheron2000/Beehalf/blob/main/app/src/main/java/dev/beehalf/data/local/BeehalfDatabase.kt
- Architecture: https://github.com/cheron2000/Beehalf/blob/main/docs/ARCHITECTURE.md
- Security: https://github.com/cheron2000/Beehalf/blob/main/docs/SECURITY.md
- Roadmap: https://github.com/cheron2000/Beehalf/blob/main/docs/ROADMAP.md
- CI: https://github.com/cheron2000/Beehalf/blob/main/.github/workflows/android-ci.yml

---

## Continuation handoff

Continue from the current `main` branch of `cheron2000/Beehalf`. Preserve the local-first data boundary, BYOK credential protection, bounded remote context, untrusted-memory/document handling, and explicit approval requirement. Highest-value correctness task is replacing the timestamp-only summary watermark with a stable cursor and testing equal timestamps. Do not add device-action execution until an idempotent, auditable executor and kill switch exist. Do not claim complete RAG, WhatsApp integration, or production readiness from the current foundation.
