# Repository Context: cheron2000/Chatbot

> Reviewed on 2026-11-24. The recursive Git tree contained 99 entries and was not truncated. Review covered README and architecture/status docs, Flask app factory, chat/editor routes, Bedrock and streaming services, memory/profile/vector-memory modules, skin classifier, web search, file processing, code editor, attack detector, training script, representative tests, recorded accuracy results, requirements, and ignore rules. This is a source review; no application startup, model inference, test suite, build, or deployment was executed.

## 1. Executive summary

`cheron2000/Chatbot` is a Flask-based AI assistant project named **ORACLE AI Chat** in its improvement documentation. Its main capabilities are:
- AWS Bedrock chat using NVIDIA Nemotron and intended model fallback.
- Server-sent event (SSE) streaming.
- Short-term session history, JSON long-term summaries, and ChromaDB semantic memory.
- Optional Serper web search.
- File upload and text/PDF extraction.
- Prompt enhancement and optional prompt-injection/infiltration detection.
- An EfficientNet-B0 skin-lesion image classifier trained against HAM10000.
- A code-editing workflow that proposes, previews, applies, backs up, and restores source changes.

**Engineering assessment:** a feature-rich personal/hackathon prototype with a modular Flask structure, but it has a critical security boundary problem around the editor API. The app registers editor routes without visible authentication/authorization, and the path validation in the code-editor service is not robust against path-prefix collisions or symlinks. The medical classifier also has important evaluation and clinical-safety limitations, especially weak melanoma performance in the recorded 2,000-row evaluation.

Do not expose this application to the public internet in its current state. Disable or remove the editor routes until they are protected and path handling is corrected.

## 2. Verified repository metadata

| Field | Observed value |
|---|---|
| Repository | `cheron2000/Chatbot` |
| Visibility | Public |
| Default branch | `main` |
| GitHub size | 15,307 KB |
| Archived | No |
| Recursive tree | 99 entries; complete |
| Backend | Python / Flask |
| AI runtime | AWS Bedrock Runtime |
| Configured default model list | `nvidia.nemotron-nano-12b-v2` |
| Memory | Flask session, JSON summaries, ChromaDB |
| Web search | Serper.dev |
| Image model | PyTorch + timm EfficientNet |
| Dataset | HAM10000 metadata and evaluation artifacts |
| Frontend | HTML/CSS/JavaScript in Flask templates |

The tracked tree includes `best_efficientnet_ham10000.pth` even though parts of the README describe the checkpoint as a large file that is not included. The repository's `.gitignore` explicitly says the checkpoint is included. Treat the actual tree as authoritative: the checkpoint is tracked.

## 3. Product purpose and user flows

### 3.1 Conversational assistant
1. Browser sends a message to `POST /chat`.
2. Flask validates the message and reads session state.
3. Optional web search and file context are added.
4. The route assembles short-term history, long-term summary, profile context, and semantically relevant vector memories.
5. `StreamingService` tries configured Bedrock model IDs in order.
6. The service emits SSE token events and a final formatted response.
7. Conversation history is updated and background summary/profile tasks persist context.

### 3.2 Skin-lesion classification
1. User submits an image through chat or `POST /classify`.
2. The classifier service decodes and preprocesses the image to 224×224 RGB.
3. EfficientNet-B0 predicts one of seven HAM10000 disease classes.
4. The API returns top-k predictions, softmax scores, disease descriptions, processing time, and a medical disclaimer.
5. In chat mode, classification output is inserted into the LLM prompt for an educational explanation.

The application itself includes a disclaimer that this is not a clinical diagnosis and is not FDA-approved. That disclaimer is appropriate, but the UI must not frame the output as a diagnosis or imply that the confidence score is a clinical probability.

### 3.3 Code editing
The intended flow is:
1. AI proposes a JSON change set.
2. `POST /editor/propose` parses the proposal and creates a preview.
3. User approves the changes in the frontend.
4. `POST /editor/apply` writes the changes and creates backups.
5. `POST /editor/restore` restores a backup.

The workflow has source-editing functionality, but the route layer does not show an authentication/authorization gate. This is a high-severity security issue (see Section 8).

## 4. Architecture

```text
Browser (templates/index.html)
  |
  | HTTP + Server-Sent Events
  v
Flask application (app_refactored.py)
  |
  +-- routes/main.py
  |     +-- GET /
  |
  +-- routes/chat.py
  |     +-- POST /chat
  |     +-- POST /clear and session utilities
  |     +-- web search / file context / skin classification
  |     +-- optional infiltration test endpoints
  |
  +-- routes/editor.py
  |     +-- POST /editor/propose
  |     +-- POST /editor/apply
  |     +-- POST /editor/restore
  |     +-- GET /editor/backups
  |
  +-- services/
  |     +-- bedrock.py       -> AWS Bedrock Runtime
  |     +-- streaming.py     -> model iteration + SSE
  |     +-- websearch.py     -> Serper.dev
  |     +-- skin_classifier.py -> EfficientNet / PyTorch
  |     +-- code_editor.py   -> filesystem edits / backups
  |
  +-- models/
  |     +-- memory.py        -> JSON long-term summary
  |     +-- vector_memory.py -> ChromaDB
  |     +-- user_profile.py  -> inferred profile JSON
  |
  +-- utils/
        +-- validators.py
        +-- file_processor.py
        +-- prompt_enhancer.py
        +-- attack_detector.py
        +-- injection_tester.py
        +-- search_detector.py
```

The modular split is a meaningful improvement over the legacy monolithic app. The current app factory creates Bedrock, streaming, classifier, vector memory, rate limiter, and blueprints.

## 5. Core components

### 5.1 Flask application and configuration

Files: `app_refactored.py`, `config.py`.

- Application factory pattern through `create_app()`.
- Server-side filesystem sessions using Flask-Session.
- Flask-Limiter defaults to 200 requests/day and chat-specific 20/minute.
- Rate-limit storage defaults to in-memory; documentation recommends external storage for production.
- Default Bedrock region is `us-east-1`.
- Default configured model list contains only `nvidia.nemotron-nano-12b-v2`.
- Classifier settings include model path, device selection, cache, image-size, and rate-limit settings.
- Startup warns when the default Flask secret key is used.

**Documentation mismatch:** README advertises Claude 3 Haiku plus NVIDIA Nemotron with automatic fallback, but the inspected default model list contains only the NVIDIA model. The fallback loop only has another model to try if another ID is configured. The default configuration therefore does not provide the advertised two-model fallback.

**Deployment concern:** running the module directly binds to `0.0.0.0` and enables Flask debug mode. This is suitable only for local development. Use a production WSGI server, disable debug, configure HTTPS, and set a strong secret key before any shared deployment.

### 5.2 Bedrock and streaming

Files: `services/bedrock.py`, `services/streaming.py`.

- Bedrock Runtime client uses configured region, connect/read timeouts, and retry settings.
- Anthropic stream handler consumes Bedrock content-block/message events.
- NVIDIA stream handler consumes an OpenAI-compatible `choices[].delta.content` response shape and supports image payloads.
- Streaming service iterates configured models in order.
- It retries the next model only if the previous model fails before emitting tokens.
- If a model fails after emitting part of a response, the service sends an error event and stops rather than mixing two model responses.

This mid-stream behavior is sensible: it avoids silently concatenating two different model outputs. Add integration tests for both pre-stream and mid-stream failure paths.

### 5.3 Conversation memory

File: `models/memory.py`.

- Short-term history is held in Flask session and limited by configuration.
- Long-term memory is a JSON summary file per session.
- Summary generation runs in a background thread and invokes a Bedrock model.
- Summary includes prior summary, latest user message, and assistant response.

Risks:
- Background threads are not durable jobs; work can be lost on process restart or worker termination.
- Multiple concurrent requests for the same session can race and overwrite the same JSON summary.
- Summary generation is model-produced and can retain incorrect or prompt-injected content.
- No retention/expiry policy is apparent for memory files.
- Filesystem-backed session and memory storage are not shared between multiple app instances unless a shared persistent filesystem is configured.

### 5.4 Vector memory

File: `models/vector_memory.py`.

- ChromaDB `PersistentClient` stores conversation messages.
- Each record has session ID, role, UTC timestamp, and optional metadata.
- Semantic search can be scoped to a session.
- The service supports history retrieval, deletion, and relevant-context formatting.

Risks:
- ChromaDB's default embedding function/model behavior should be pinned and documented for reproducibility.
- The same persistent collection stores all sessions; session filtering is essential on every query.
- Metadata can be overridden by caller-supplied metadata because it is expanded after the standard keys in the dictionary literal. Do not allow arbitrary caller metadata to overwrite `session_id` or `role`.
- Deletion and retention should be tested across both vector records and JSON/session memory so that a "clear history" action does not leave residual context.

### 5.5 User profile inference

File: `models/user_profile.py`.

- Profiles are stored as JSON files keyed by session ID.
- A background heuristic examines recent user messages and infers expertise level, communication style, and topic interests.
- Profile data is added to the system context.

This is heuristic personalization, not a reliable user model. Provide a way to inspect/reset profile data and avoid inferring sensitive traits. The repository currently tracks several profile JSON artifacts under `user_profiles/` despite `.gitignore` listing that directory. Since ignore rules do not untrack already committed files, remove any unintended personal/session data from Git history and keep profile files out of future commits.

### 5.6 Web search

File: `services/websearch.py`.

- Serper.dev Google Search API integration.
- Uses `SERPER_API_KEY`, configurable result count and retries.
- Defaults to `gl=us` and `hl=en`.
- Returns formatted title, snippet, publication date, and source URL for prompt context.
- Failures return an empty string rather than crashing the chat.

Recommendations:
- Make region/language configurable; US results are a poor default for India-focused queries.
- Preserve structured search result objects rather than flattening them into one prompt string.
- Treat search snippets as untrusted content and defend against prompt injection.
- Add request budget/timeouts and observability for search failures/costs.

### 5.7 File upload and parsing

File: `utils/file_processor.py`.

- 10 MB maximum file size.
- Supports common image types, PDFs, and a list of code/text extensions.
- PDF text is extracted with pypdf.
- Text is capped at 12,000 characters.
- Images are base64-encoded for vision-model use.

The `/upload` route reads the uploaded file into memory before calling the processor, so enforce Flask request-body limits as well as the downstream 10 MB check. MIME type should not be trusted as proof of file type. Add decompression/page-count limits for PDFs and robust handling of malformed/oversized images.

## 6. Skin-lesion classifier and model evidence

Files: `services/skin_classifier.py`, `models/classification.py`, `data/diseases.json`, `datasets/train_efficientnet.py`, `tests/`.

- PyTorch + timm EfficientNet-B0.
- Seven HAM10000 classes: `akiec`, `bcc`, `bkl`, `df`, `mel`, `nv`, `vasc`.
- 224×224 input, ImageNet mean/std normalization.
- Lazy model loading, singleton instance, lock around model loading/unloading.
- Softmax top-k predictions and disease metadata.
- API route limits decoded image size to 10 MB and top_k to 1–7.
- Classifier endpoint is rate-limited to 10 requests/minute if the limiter is configured.

### 6.1 Recorded 2,000-row evaluation

I independently counted the rows in the committed `tests/model_accuracy_test_results.csv` and recomputed the summary:

<badge color="warning">Recorded result, not independently rerun</badge>

- Samples: 2,000
- Correct: 1,705
- Accuracy: 85.25%
- Mean inference time: approximately 39.15 ms

Per-class accuracy from that CSV:

| Class | Correct / total | Accuracy |
|---|---:|---:|
| Melanocytic nevi (`nv`) | 1,288 / 1,374 | 93.74% |
| Melanoma (`mel`) | 88 / 179 | 49.16% |
| Benign keratosis (`bkl`) | 170 / 226 | 75.22% |
| Basal cell carcinoma (`bcc`) | 85 / 107 | 79.44% |
| Actinic keratoses (`akiec`) | 43 / 74 | 58.11% |
| Vascular lesions (`vasc`) | 25 / 27 | 92.59% |
| Dermatofibroma (`df`) | 6 / 13 | 46.15% |

The sample is imbalanced: 1,374 of 2,000 examples (68.7%) are melanocytic nevi. Overall accuracy therefore hides weaker results on several clinically important classes. The melanoma result (49.16%) is particularly concerning.

The separate `docs/MODEL_ACCURACY_TEST_FINAL_REPORT.md` reports 83.33% on a 30-image sample, while README mentions 81.73% validation accuracy and the 2,000-row summary reports 85.25%. These are different evaluation snapshots and must not be presented as interchangeable.

### 6.2 Evaluation validity concerns

The training script uses a stratified random row-level 80/20 train/validation split over the HAM10000 metadata. The recorded 2,000-row test script/results need a clearly documented provenance showing that these images were excluded from training and model selection. HAM10000 can contain multiple images of the same lesion; a row-level split can place images from one lesion in both partitions. Use lesion-level grouped splits and a truly held-out test set.

The current evidence does not establish external validation, clinical validation, or suitability for screening. Softmax confidence is not calibrated clinical confidence. Do not make diagnosis, triage, or reassurance decisions from this model.

### 6.3 Classifier implementation concerns

- The README describes image processing as in-memory/no image storage, but chat history/session/vector memory behavior must be checked to ensure image bytes or derived sensitive content are not persisted.
- The model is loaded from a checkpoint using `torch.load`; only load trusted, integrity-checked checkpoints. Pin the exact model artifact hash and library versions.
- The classifier's `auto_unload_minutes`, cache size, and max image size appear in configuration; the inspected service path does not demonstrate a full TTL-based unload/cache implementation.
- A failed or missing model returns service errors; deployment readiness should verify checkpoint presence and compatible PyTorch/timm versions.
- The classifier is instantiated in the app factory and also retrieved through the singleton in route handlers; singleton initialization behavior should be tested under Flask reloads and worker processes.

## 7. Prompt-injection and infiltration tooling

Files: `utils/attack_detector.py`, `utils/injection_tester.py`, and the optional infiltration routes.

- Regex patterns classify prompt injection, jailbreak, SQL/command/XSS patterns, data extraction, authority impersonation, session manipulation, and memory poisoning.
- Infiltration mode defaults to disabled in `config.py`.
- The chat route only runs detection when the session's infiltration mode is enabled.
- Injection tester generates a set of adversarial prompt variations for testing.

This is a heuristic detection layer, not a security boundary. Regexes can be bypassed and can false-positive on ordinary security discussions. Keep model/system security independent of this detector. The injection test route currently generates and returns test cases with `tested: false`; it does not itself execute all cases against the model despite the endpoint name/description.

## 8. Critical security findings

### P0 — editor endpoints are not visibly protected

Files: `app_refactored.py`, `routes/editor.py`, `services/code_editor.py`.

- The app factory unconditionally registers `editor_bp`.
- `/editor/apply`, `/editor/restore`, `/editor/backups`, and `/editor/propose` have no visible authentication or authorization decorator.
- The chat route's developer mode is hardcoded off, but this does not disable the separately registered editor blueprint.
- `/editor/apply` accepts a client-provided list of file changes and writes them to the project filesystem.
- `_safe_path()` uses `abs_path.startswith(PROJECT_ROOT)`, which is not a safe path-containment test. A sibling path with a common prefix can pass, and symlinks are not resolved before the check.
- The service allows editing Python, JavaScript, HTML, CSS, JSON, YAML, shell, and other files; a malicious write to executable application code can become code execution when imported/restarted.
- The Python syntax check is applied to replace/create, but not append. It is not a security sandbox.

**Action:** immediately remove/disable editor blueprint registration in any network-accessible deployment. If the feature is required, place it behind strong authentication, authorization, CSRF protection, explicit allowlisted workspace roots, atomic file operations, and a reviewed change-approval flow. Use `Path.resolve()` plus `os.path.commonpath()` or equivalent after resolving symlinks; reject symlinks and non-regular files. Do not trust a frontend approval as server-side authorization.

### P0 — development server is exposed on all interfaces

The module entry point uses `app.run(host="0.0.0.0", ..., debug=True)`. The Flask development server/debugger must never be exposed to untrusted networks. Disable debug and run behind a production WSGI server with TLS and a reverse proxy.

### P1 — CSRF and session hardening

The inspected configuration uses filesystem sessions but does not show explicit secure cookie settings such as `SESSION_COOKIE_SECURE`, `SESSION_COOKIE_HTTPONLY`, and `SESSION_COOKIE_SAMESITE`. The editor routes are state-changing POST endpoints and no CSRF protection is visible. Add CSRF tokens/origin checks, secure cookie settings, and appropriate session lifecycle controls.

### P1 — upload and image handling

- Enforce request size limits before reading complete uploads.
- Validate actual image format, dimensions, and decoded pixel count.
- Validate base64 strictly; use `base64.b64decode(..., validate=True)`.
- Avoid returning raw internal exception details from classification endpoints.
- Apply consistent per-user and per-IP limits to all expensive routes.

### P1 — persistent data privacy

- Profile JSON files appear in the tracked tree despite the ignore rule.
- Conversation summaries and vector records can contain personal or sensitive user content.
- Define retention, deletion, export, encryption-at-rest, and access-control behavior.
- Make clear-history delete all relevant stores, including profile, JSON summary, session, and ChromaDB records.

## 9. Code-editor implementation defects

In addition to missing route protection:
- Path containment uses string prefix rather than resolved path ancestry.
- Symlink traversal is not rejected.
- Protected-file checks compare only the basename, which is brittle and may not protect all sensitive paths.
- Backup names use second-level timestamps and basename only; two files with the same name edited within the same second can collide.
- Restore searches the entire project for the first matching basename, which is ambiguous if files share names.
- A replacement validates the candidate Python syntax before checking that the exact old string exists. This can validate a file state that will not actually be written.
- Multi-file apply is not transactional: earlier changes remain applied if a later change fails.
- Append operations are not syntax-checked for Python.
- No maximum number/size of changes is visible in the route.

## 10. Tests, CI, and developer workflow

Observed test assets include:
- `tests/test_classifier_integration.py` — live local Flask health/classify/chat integration script.
- `tests/test_model_accuracy.py` — model evaluation script using HAM10000 images/metadata.
- `tests/check_model.py` — checkpoint/model sanity checks.
- `test_websearch.py`, `test_search_simple.py`, `test_vector_memory.py`.
- A committed 2,000-row prediction CSV and summary.

Many scripts require a running local server, external AWS/Serper credentials, or local dataset images. They are not equivalent to deterministic unit tests. No `.github/workflows/` CI files appeared in the reviewed tree.

No tests, lint, type checks, model inference, or app startup were executed for this report.

### Dependency and packaging notes
- Main `requirements.txt` pins Flask, Flask-Session, Flask-Limiter, boto3/botocore, ChromaDB, httpx, BeautifulSoup, pypdf, and dotenv.
- PyTorch, torchvision, timm, Pillow, and NumPy use lower-bound versions rather than exact pins.
- Platform-specific PyTorch installation and CUDA compatibility should be documented separately.
- Model artifact hash and training metadata should be stored alongside the checkpoint.
- `datasets/requirements.txt` is a separate training environment dependency list.

## 11. Prioritized roadmap

### P0 — contain the editor attack surface
1. Disable `editor_bp` registration on public deployments immediately.
2. Add authentication and authorization on every editor route.
3. Fix canonical path validation with resolved paths and symlink rejection.
4. Add CSRF protection and strict change schema validation.
5. Make apply operations atomic and backups collision-safe.

### P0 — production server hardening
1. Remove `debug=True` and the Flask development server from deployment.
2. Use a production WSGI server behind TLS.
3. Configure secure session cookies, trusted proxy handling, and request-size limits.
4. Move rate-limit storage to Redis or another shared backend for multi-worker deployments.

### P1 — medical model safety and evaluation
1. Create lesion-level grouped train/validation/test splits.
2. Evaluate on a genuinely held-out and preferably external dataset.
3. Publish confusion matrix, per-class sensitivity/specificity, precision/recall, macro-F1, calibration, and confidence intervals.
4. Prioritize melanoma sensitivity and establish conservative abstention/referral behavior.
5. Keep the product strictly educational until clinical validation and appropriate regulatory review.

### P1 — data privacy and memory correctness
1. Remove tracked profile/session artifacts and review Git history for sensitive data.
2. Add retention/deletion policies and verify clear-history removes data from every store.
3. Prevent metadata from overriding vector-memory session/role fields.
4. Make background memory/profile updates durable or serialize writes per session.
5. Document whether images and file contents can enter conversation/vector memory.

### P2 — architecture and reliability
1. Align documented Claude/NVIDIA fallback with the actual configured model list.
2. Add deterministic tests for SSE failures, memory races, vector session isolation, file upload limits, editor path traversal, and classifier edge cases.
3. Make web search region/language configurable and retain structured citations.
4. Add CI for formatting, linting, type checking, unit tests, and packaging.
5. Add observability for model latency, token usage, Bedrock errors, search cost, classifier load/inference, and rate-limit behavior.

## 12. Validation status and limitations

- Repository metadata/tree: inspected; 99 entries, complete.
- README, architecture, project status, improvement docs, model evaluation report: inspected.
- Flask app factory, config, chat/editor routes: inspected.
- Bedrock streaming, memory, vector memory, profile inference: inspected.
- Web search, file processor, classifier, attack detector, code editor: inspected.
- Training script, integration tests, evaluation CSV and summary: inspected.
- Evaluation CSV independently recounted: 2,000 rows, 1,705 correct, 85.25% overall, approximately 39.15 ms mean recorded inference.
- Full source tree: not line-by-line reviewed.
- Live AWS Bedrock calls, Serper API, server behavior, and deployment configuration: not tested.
- Test suite, build, lint, model inference, and app startup: not executed.
- Clinical performance and regulatory suitability: not established.

## 13. Coding-agent handoff

Treat this repository as a modular personal AI assistant prototype with an integrated but non-clinical skin-lesion classifier. Do not add new features before closing the editor route exposure. The first task should be to disable the editor API in any network-accessible deployment, then implement authentication, CSRF protection, and robust filesystem path containment.

After the editor surface is secured, validate data privacy across session/JSON/vector/profile stores and rebuild the skin-classifier evaluation around lesion-level splits and class-sensitive metrics. The committed 85.25% overall accuracy is not sufficient evidence of clinical reliability, particularly with the recorded 49.16% melanoma accuracy.
