# Repository Context: cheron2000/TimeTableExtractor

> Reviewed on 2026-11-24. The complete recursive Git tree contained 13 entries. Review covered the FastAPI entry point, parser engine, Pydantic models, logging helpers, requirements, static HTML/CSS/JavaScript frontend, README, case study, fix log, and ignore rules. This is a source review; no uploaded workbook, API server, browser session, or test/build command was executed.

## 1. Executive summary

**Timetable Extractor** is a Python/FastAPI application that accepts Excel timetables and turns them into queryable schedules for individual teachers, divisions, and batches. Its main differentiator is a heuristic parser designed for messy institutional spreadsheets: merged cells, variable time formats, embedded teacher codes, room numbers, and lab blocks.

The architecture is compact:
- FastAPI handles uploads, session management, and schedule-query endpoints.
- `parser_engine.py` loads workbooks with openpyxl, expands merged cells, detects day/time/class columns, discovers teacher codes, extracts cell entities, and builds schedule views.
- Pydantic models describe API response structures.
- A static HTML/CSS/vanilla-JavaScript frontend uploads files, selects a view, renders schedules, and exports JSON or prints.

**Highest-priority finding: stored/reflected XSS risk.** The frontend inserts uploaded workbook-derived values (filename, subject, room, class/division, faculty display data) into `innerHTML` without escaping. A malicious or untrusted timetable can therefore inject markup/script into the browser of whoever views that session. Use DOM APIs and `textContent`, or a well-defined HTML sanitizer where rich markup is truly required.

**Highest-priority backend exposure:** the API has wildcard CORS and no authentication/authorization. Session IDs are random UUIDs, but any party who obtains one can query that timetable; `DELETE /cache/clear` is a global unauthenticated destructive operation. The upload route reads the entire file into memory and writes it to disk without a server-side size cap. These are important if deployed beyond a trusted local environment.

Other key engineering issues:
- Rich teacher/division/batch schedule lab merging marks the next lecture slot as skipped whenever the current slot contains any lab; it does not verify the next slot contains the same lab. This can hide a legitimate class.
- Session state and expiry metadata are process-local, limiting reliability across multiple workers/instances.
- The parser is heavily heuristic and hard-coded to a fixed set of divisions, batches, days, and slot times.
- The case study's performance/accuracy claims are documentation claims, not independently verified test results.

## 2. Verified repository metadata

| Field | Observed value |
|---|---|
| Repository | `cheron2000/TimeTableExtractor` |
| Visibility | Public |
| Default branch | `main` |
| GitHub size | 41 KB |
| Archived | No |
| Recursive tree | 13 entries; complete |
| Backend | Python 3.11+, FastAPI, Uvicorn |
| Excel parser | openpyxl |
| Schemas | Pydantic 2 |
| Frontend | Static HTML, CSS, vanilla JavaScript |
| Session store | In-memory dictionaries + temporary files |
| Session TTL | 3,600 seconds (1 hour) |
| API default port | 8000 |
| Dependencies | FastAPI, Uvicorn, openpyxl, python-multipart, Pydantic |
| Tests / CI | No test suite or GitHub Actions workflow found in the reviewed tree |

## 3. Repository structure

```text
TimeTableExtractor/
├── main.py              # FastAPI routes, uploads, sessions, cache cleanup
├── parser_engine.py     # Excel loading, heuristics, extraction, schedule building
├── models.py            # Pydantic response schemas
├── utils.py             # Logging configuration and truncate helper
├── requirements.txt     # Pinned Python dependencies
├── index.html           # Static frontend structure
├── script.js            # Upload, API calls, rendering, export, session UI
├── style.css            # Dark responsive UI and timetable styling
├── README.md            # Setup, API, features, deployment guidance
├── CASE_STUDY.md        # Product/architecture case study and claimed results
├── CASE_STUDY.html      # HTML version of the case study
├── fix_needed.md        # Historical issue/fix checklist
└── .gitignore           # Python artifacts, environments, uploads, secrets
```

No sample Excel workbook, automated test directory, deployment manifest, container definition, CI workflow, or license file appeared in the recursive tree, even though README links to a LICENSE file.

## 4. Architecture and data flow

```text
Browser (index.html + script.js + style.css)
  |
  | multipart upload / JSON REST calls
  v
FastAPI (main.py, port 8000)
  |
  +-- Upload validation by extension
  +-- Temporary file storage
  +-- Session ID (UUID)
  +-- In-memory parser cache + TTL
  |
  v
TimetableParser (parser_engine.py)
  |
  +-- openpyxl loads active worksheet
  +-- Expand merged-cell values into flat grid
  +-- Detect day/time columns
  +-- Detect division/class columns
  +-- Build day-to-row map
  +-- Discover teacher codes
  +-- Extract teacher/division/room from cell text
  +-- Build teacher/division/batch schedules
  +-- Merge consecutive lab entries
  |
  v
Pydantic response models -> JSON -> frontend timetable renderer
```

The workbook is loaded with `data_only=True` and the active worksheet is parsed. All sheets are reported in upload metadata, but parsing logic shown uses only `wb.active`; users cannot select another sheet through the current API.

## 5. Backend: FastAPI and session lifecycle

File: `main.py`.

### Implemented behavior
- Creates a temporary upload directory at `tempfile.gettempdir()/timetable_uploads`.
- Accepts `.xlsx` and `.xlsm` extensions.
- Generates a UUID hex session ID.
- Saves the uploaded bytes to a temporary file.
- Parses the workbook immediately and caches the `TimetableParser` instance.
- Returns filename, sheet names, row/column counts, discovered teachers, and divisions.
- Expires inactive sessions after one hour when `get_parser()` is called.
- Supports session deletion and global cache clearing.
- Reconstructs a parser from the temporary file if the in-memory parser object is absent.
- Provides Swagger/OpenAPI docs through FastAPI defaults.

### API surface observed
| Method | Endpoint | Purpose |
|---|---|---|
| GET | `/` | Health/status response |
| POST | `/upload` | Upload and parse an Excel file |
| GET | `/extract/cell?text=...` | Debug one cell's entity extraction |
| GET | `/extract/full/{session_id}` | Full per-teacher/per-division split |
| GET | `/extract/{session_id}` | Compact master schedule |
| GET | `/debug/{session_id}` | Internal parser state and sample grid |
| GET | `/debug/row/{session_id}/{row_index}` | Raw parsed row |
| GET | `/faculty` | Faculty code/name mapping |
| GET | `/teachers/{session_id}` | Detected teacher list |
| GET | `/timetable/teacher/{session_id}/{code}` | Teacher schedule |
| GET | `/timetable/division/{session_id}/{div}` | Division schedule |
| GET | `/timetable/batch/{session_id}/{div}/{batch}` | Batch schedule |
| DELETE | `/session/{session_id}` | Delete one session |
| DELETE | `/cache/clear` | Clear all in-memory sessions |

### Session-management risks
- `_session_cache` and `_session_ts` are process-local. Multiple Uvicorn workers or multiple instances will not share cache/TTL metadata. The temporary file may be shared only if the deployment filesystem is shared and persistent.
- Expiry is lazy: `_evict_expired()` is called when a session is accessed, not by a dedicated periodic cleanup task. Expired files can remain on disk indefinitely if no later request triggers eviction.
- `DELETE /cache/clear` clears memory but does not delete uploaded files. It also has no authentication and can disrupt every active user.
- `DELETE /session/{session_id}` returns success even if the session did not exist.
- No ownership boundary exists. A session ID acts as a bearer capability. UUID entropy makes guessing difficult, but URLs/logs/browser storage can leak the ID.
- Session restoration in the frontend only restores the ID from localStorage; it cannot guarantee that the corresponding session still exists or that a different server instance can access its file.

Recommended direction: store sessions and ownership in Redis or a database, persist uploaded files in controlled object storage or a managed temporary volume, use an authenticated owner/session mapping, and run a periodic TTL cleanup worker.

## 6. Parser engine

File: `parser_engine.py` (approximately 42 KB; main body of the project).

### 6.1 Workbook loading and merged cells
- Loads workbook with openpyxl and `data_only=True`.
- Uses the active worksheet.
- Expands merged ranges by copying the top-left cell's value into every grid coordinate in the range.
- Tracks merged-cell owner coordinates and row spans so later schedule logic can avoid duplicate continuation cells and infer duration.

Potential improvements:
- Explicitly close the workbook after extracting values.
- Define behavior for formula cells without cached results; `data_only=True` can return `None` if the workbook has not been recalculated/saved by a spreadsheet application.
- Add workbook limits for maximum rows, columns, sheets, merged ranges, and decompressed size.
- Report which worksheet was parsed and allow selecting a sheet.
- Add handling/tests for hidden sheets, blank sheets, malformed workbooks, and unsupported Excel features.

### 6.2 Structure detection
- Day column and time column are selected by frequency-based voting across all grid cells.
- Day names accept short and full variants.
- Class/division headers are inspected in the first eight rows.
- Day rows are mapped to fixed lecture slots using a ±20-minute tolerance.
- Only Monday through Saturday are included in `FIXED_DAYS`.

This is a practical heuristic for a known timetable template, not a general-purpose Excel schema detector. A different institution's workbook can be misclassified silently if columns or header placement differ.

### 6.3 Teacher discovery
- Starts from a hard-coded `FACULTY_MAP`.
- Also detects 2–3 character uppercase tokens that appear at least four times.
- Uses skip-token lists to reduce false positives.
- Parenthesized known faculty codes are treated as stronger signals during cell extraction.

Risks:
- A subject abbreviation or room/administrative code can be mistaken for a teacher if it appears frequently enough.
- New or renamed faculty are not centrally configurable without code changes.
- The static map contains institution-specific names and division assumptions; separate configuration from parser logic.
- The frontend now fetches `/faculty`, avoiding the previously duplicated map described in `fix_needed.md`.

### 6.4 Cell entity extraction
The parser tries to extract:
- Teacher codes.
- Division identifiers (FE/SE/TE/BE and A–H variants).
- Batch identifiers (A1–H3).
- Room identifiers and lab room labels.
- Lab/practical indicators.
- Cleaned subject text.

It supports newline/comma splitting outside parentheses and normalizes common time formats. Entity assignment is heuristic: when counts of teachers/divisions/rooms differ, it pairs by position or reuses the first/last division/room. This can produce incorrect pairings for cells containing several independent classes or multiple rooms.

### 6.5 Division and batch model
- `DIVISION_CONFIG` is hard-coded for divisions A–H, with three batches per division.
- `FIXED_DAYS` is Monday–Saturday.
- `SLOT_CONFIG` is a fixed sequence of nine slots including break and lunch.
- The UI generates batch names A1/A2/A3 etc. rather than asking the backend for actual batch definitions.

This strongly suggests the parser is tailored to a specific institutional timetable format. Generalization should move division, batch, slot, and faculty configuration into a per-institution configuration or workbook-derived metadata.

### 6.6 Time normalization
- Handles colon/dot time strings, optional seconds, and AM/PM.
- Converts dot notation such as `9.3` to `9:30`.
- Uses a heuristic that adds 12 hours for any hour less than 7, to interpret afternoon values like `1:30`.
- Matches to known slots within ±20 minutes.

Potential correctness issues:
- The hour-under-7 rule makes a true early-morning time such as `6:30` become `18:30`.
- Explicit AM values below 7 are also shifted because `time_to_mins()` has no AM/PM context.
- Parsed hour/minute ranges are not explicitly validated before matching.
- Ambiguous dot forms like `9.3` are interpreted as 9:30; this convention should be configurable and tested.
- The slot matching tolerance can map a malformed or off-schedule time to a nearby valid slot.

Use a normalized time representation with explicit 24-hour conversion and strict range checks; configure timetable slot definitions per workbook/institution.

### 6.7 Lab merging
Two separate lab merge paths exist:

**Compact output:** `_merge_compact_labs()` checks whether the identical compact string appears in consecutive lecture slots before merging. This is comparatively safe, though it only recognizes entries containing the literal word "lab".

**Rich teacher/division/batch output:** `_merge_labs()` marks the next lecture slot as skipped whenever the current slot contains any lab entry. It does not verify that the next slot contains the same lab, same subject, same teacher, same division, or same room. It also applies a two-slot span to all entries in the current slot when any entry is a lab.

This can hide an unrelated lecture or lab in the next slot and can incorrectly stretch non-lab entries alongside a lab. It is a concrete schedule correctness defect. Reuse identity-based consecutive matching from compact merging, and only merge the matching entries while preserving unrelated entries in the next slot.

### 6.8 Full compact split
`build_full_timetable_json()` splits each compact string by whitespace and assumes token 1 is teacher and token 2 is division. If an entry has a teacher but no division and does have a room, the room can be misclassified as the division key. Parse structured entity objects instead of reconstructing fields from display strings.

## 7. Frontend behavior and security

Files: `index.html`, `script.js`, `style.css`.

### Implemented user flow
1. User uploads an `.xlsx` or `.xlsm` file.
2. Frontend checks extension and client-side file size (10 MB).
3. Browser sends multipart form data to `http://localhost:8000/upload`.
4. Session ID is saved in localStorage.
5. User chooses teacher, division, or batch.
6. Frontend fetches a timetable JSON response and renders a weekly grid.
7. User can print or export the JSON schedule.
8. User can clear the active session.

### Critical: unsafe HTML insertion
`displayFileInfo()` interpolates `data.filename` into `fileInfo.innerHTML`. The filename is supplied by the uploader.

`renderTimetable()` interpolates API values including subject, faculty names, room, class/division, and slot labels into an HTML string, then assigns it to `table.innerHTML`. Most of those fields ultimately originate from uploaded workbook contents or parser output.

A crafted workbook can therefore carry HTML/script payloads into the UI. If another user obtains/views that session, the content can execute in their browser. This is a stored/reflected XSS class of issue across the upload-to-render path.

Required remediation:
- Construct cells/elements with `document.createElement()`.
- Assign all untrusted strings with `textContent`.
- Do not interpolate workbook-derived data into HTML templates.
- Add CSP as defense in depth; do not treat CSP as a substitute for safe DOM APIs.
- Add a security regression test using a workbook cell and filename containing HTML metacharacters/script payloads.

The static UI currently uses `innerHTML` safely for fixed option placeholders and icon templates in some places, but data-bearing paths must be changed.

### Other frontend issues
- `API_BASE` is hard-coded to `http://localhost:8000`; deployed users need source edits. Use environment/build configuration.
- The frontend file-size limit is only client-side; direct API callers bypass it.
- Upload progress is simulated with a timer rather than measured from actual transfer progress. The progress interval is cleared only after a response; a rejected fetch can leave it running.
- `loadSchedule()` discards API error detail and reports a generic message.
- The frontend stores session IDs in localStorage without an explicit expiration check.
- Session restoration shows "Previous session restored" before verifying the session is still valid.
- `clearSession()` only clears local state after a successful DELETE; server cache clear endpoint is separate and not exposed in the UI.
- The frontend generates batch options A1–A3 etc. from a fixed assumption rather than retrieving per-division batch metadata.
- Rendering and API logic are concentrated in a single script file; no module boundary or automated frontend tests exist.
- The HTML includes Font Awesome from a CDN; no integrity attribute is visible in the reviewed link.

## 8. Security and operational assessment

### Positive controls
- Random UUID session identifiers.
- File extension allowlist for `.xlsx` and `.xlsm`.
- Upload filename is not used as the filesystem path; server generates the stored filename.
- Parser errors during upload are caught and converted to HTTP 422.
- `.gitignore` excludes local Excel uploads, logs, Python artifacts, virtual environments, and `.env`.

### High-priority gaps
1. **No authentication or authorization:** session data and destructive cache endpoints are unauthenticated.
2. **Wildcard CORS:** `allow_origins=["*"]`, with all methods and headers. CORS is not authorization, but the broad policy increases exposure when the API is reachable from browsers.
3. **No server-side upload size cap:** `await file.read()` reads the entire upload into memory; then `write_bytes()` writes it to disk. The 10 MB limit exists only in JavaScript.
4. **No upload concurrency/rate limits:** a public endpoint can be abused for CPU, memory, and disk exhaustion.
5. **XSS through workbook-derived content:** see frontend section.
6. **Debug endpoints disclose raw timetable data:** `/debug/{session_id}` and `/debug/row/...` expose internal grid contents to anyone with a session ID.
7. **Global cache clear is unauthenticated:** any caller can disrupt all active sessions.
8. **No file-content validation beyond extension:** use file signatures/ZIP structure checks, parse in a constrained worker, and set limits for workbook complexity and decompression.
9. **No explicit retention guarantee:** TTL is lazy and uploaded files can remain after process restart until eviction is triggered.
10. **No production deployment hardening:** no trusted-host policy, auth, rate limiting, request timeouts, reverse-proxy configuration, or structured health/readiness separation observed.

Recommended deployment model: do not expose this version publicly with sensitive institutional schedules until authentication, per-user session ownership, upload limits, safe rendering, and cache-control authorization are implemented.

## 9. Documentation accuracy

The README provides useful setup and endpoint documentation, but several claims should be treated cautiously:
- "Session-based caching (1-hour expiry)" is implemented, but expiry is lazy and process-local, not a durable distributed session service.
- "Zero-config frontend" still hardcodes localhost API base and requires the backend to be separately running.
- "Upload once, query infinitely" is bounded by a one-hour inactivity TTL and local process/file availability.
- README says 10 MB validation under production checklist, but only the browser enforces a 10 MB check; backend currently reads any upload size.
- Case study reports specific performance, accuracy, file-size, concurrent-user, and time-savings figures. These were not accompanied by benchmark scripts or test artifacts in the reviewed tree and must be labeled as targets/observations until independently reproduced.
- README links to a LICENSE file, but no LICENSE file appears in the tree.
- `fix_needed.md` lists eight historical issues as fixed, including cross-platform temp path, route ordering, extension mismatch, dead code, cache TTL, and logging setup. The reviewed source confirms a temp-directory-based path, `/extract/full` declared before the parameterized `/extract/{session_id}`, and logging configured at startup. The checklist is useful history but not a substitute for regression tests.

## 10. Testing and CI status

Observed:
- Pinned dependencies in `requirements.txt`.
- Pydantic response models for upload, teacher list, timetable, and compact schedule.
- Historical `fix_needed.md` issue checklist.
- Case study describes manual test scenarios.

Not observed:
- pytest/unittest test files.
- Test fixtures or sample workbooks.
- Automated parser regression suite.
- GitHub Actions workflows.
- Docker/Compose or deployment manifests.
- Automated security or dependency scanning.
- Reproducible benchmark scripts.

No tests, API requests, parser runs, or performance benchmarks were executed in this review. Case-study checkmarks and numerical metrics are documentation assertions, not verified test outcomes.

Recommended test coverage:
- Parser fixtures for merged cells, blank cells, multiple teachers, multiple divisions, multi-room entries, batch filters, lab spans, and malformed time values.
- Golden expected JSON outputs for teacher/division/batch queries.
- Property/fuzz tests for entity extraction and time parsing.
- Upload tests for invalid ZIPs, oversized files, malformed workbooks, huge dimensions, and formula-only cells.
- Session tests for expiry, cleanup, multiple workers, deletion, and global-cache authorization.
- Browser tests proving workbook content is displayed as text and cannot execute script.
- Lab merge regression tests where a lab is followed by an unrelated lecture.
- Tests for entries with missing division but present room in full compact split.

## 11. Prioritized roadmap

### P0 — security before public deployment
1. Replace all data-bearing `innerHTML` rendering with DOM construction and `textContent`.
2. Add authentication and per-user ownership for uploads and session queries.
3. Protect or remove `DELETE /cache/clear`; do not expose global cache mutation to ordinary users.
4. Enforce server-side streaming upload size limits and rate limiting.
5. Restrict CORS to actual frontend origins.
6. Disable or protect raw debug endpoints in production.
7. Validate workbook structure and process uploads in a resource-constrained worker.

### P1 — schedule correctness
1. Fix rich `_merge_labs()` to merge only matching consecutive lab entries and preserve unrelated next-slot classes.
2. Fix `build_full_timetable_json()` to use structured entities rather than token-position parsing.
3. Replace heuristic hour-under-7 conversion with strict explicit time normalization.
4. Validate time ranges and make slot configuration configurable.
5. Add parser fixtures and expected-output regression tests.
6. Define deterministic behavior for multiple teachers/divisions/rooms in one cell.

### P1 — session reliability
1. Move session metadata/cache to Redis or a database.
2. Add periodic TTL cleanup that deletes both cache metadata and files.
3. Use a controlled storage location and enforce quotas.
4. Define behavior across worker restarts and multi-instance deployments.
5. Verify session existence before restoring the frontend session.

### P2 — maintainability and productization
1. Separate parser configuration from code (faculty map, divisions, batches, slot times, days).
2. Split frontend API, upload, state, rendering, and export modules.
3. Make API base URL environment-configurable.
4. Add selected-sheet support and report the active sheet.
5. Add observability for parse duration, workbook dimensions, rejection reasons, and session lifecycle without logging sensitive cell contents.
6. Add CI for formatting, linting, unit tests, and dependency scanning.
7. Add an actual LICENSE file or remove the README license claim.
8. Reproduce and document case-study performance/accuracy metrics with a representative anonymized workbook corpus.

## 12. Coding-agent handoff

Treat `main.py` + `parser_engine.py` as the active backend and `index.html` + `script.js` + `style.css` as the active frontend.

First fix the upload-to-render XSS path and add backend upload limits. Before public deployment, add authentication/session ownership and protect the global cache-clear route. Then fix the rich schedule lab-merging defect and add golden parser tests so changes do not silently corrupt schedules.

Avoid broad parser rewrites before creating representative workbook fixtures. The current heuristics are tied to a particular institution's divisions, faculty codes, days, and slot timings; preserve that behavior with regression tests while making configuration external.

## 13. Validation status and limitations

- Repository metadata and complete recursive tree: inspected (13 entries).
- FastAPI entry point, parser engine, Pydantic models, utility helpers, requirements: inspected.
- Static frontend HTML, JavaScript, CSS: inspected.
- README, case study, historical fix checklist, gitignore: inspected.
- Runtime, parser against real workbook, browser rendering, API security, tests, and performance: not executed.
- No real timetable workbook was present in the repository tree.
- No test suite or CI workflow was found.
