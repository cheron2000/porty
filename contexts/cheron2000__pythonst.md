# Repository Context: cheron2000/pythonst

> Source-reviewed on 2026-11-24. This report describes the default `main` branch as inspected during this review. It is an engineering assessment, not a claim that the application has been executed or security-tested.

## 1. Executive summary

`pythonst` is a Python/Flask prototype for **SASS (Smart Alert Scoring System)**. It loads the NSL-KDD intrusion-detection dataset, trains or loads a calibrated Random Forest classifier, assigns risk scores, records analyst feedback in SQLite, and renders a browser dashboard.

The repository has a meaningful end-to-end prototype, but its current evidence supports describing it as an **offline dataset-driven alert scoring and feedback demo**, not a live network intrusion-detection or response platform. It does not capture packets, ingest live network telemetry, or perform firewall/endpoint blocking. Several dashboard actions are recorded as analyst feedback only.

**Highest-priority concerns**
1. The app's prediction path fabricates a private IPv4 address for each row rather than using a real source address from telemetry; NSL-KDD rows do not provide the actual source IP.
2. The dashboard labels and sample data imply live monitoring, but the charts and activity feed contain hard-coded/synthetic values.
3. The learned-pattern lookup is internally inconsistent: patterns are stored under an MD5 hash, then queried with a `LIKE` search for an IP-class string. That lookup cannot match the stored hash.
4. The app starts Flask with `debug=True` and `host='0.0.0.0'`, with no visible authentication or authorization layer.
5. The main app and `train_model.py` implement different feature pipelines and save different model artifacts.
6. Multiple large model/database binaries are committed to the repository; their provenance, reproducibility, and data-retention implications are not documented.

## 2. Repository inventory and evidence

Default branch: `main`  
Visibility: public  
Repository size reported by GitHub: 17,221 KB  
Tree response: complete (not truncated).

| Path | Observed role |
|---|---|
| `app.py` | Flask app, preprocessing, training, scoring, SQLite learning, API routes |
| `train_model.py` | Separate/basic Random Forest training script |
| `KDDTrain+.txt` | NSL-KDD training dataset file |
| `KDDTest+.txt` | NSL-KDD test dataset file; used by the app as its training fallback and dashboard input |
| `templates/index.html` | Main dashboard markup, inline CSS and JavaScript |
| `templates/error.html` | Error page |
| `static/advanced-dashboard.js` | Additional dashboard implementation |
| `static/styles.css` | Additional stylesheet |
| `sass_advanced_model.joblib` | Serialized advanced model bundle loaded by `app.py` |
| `sass_model.joblib` | Separate serialized basic Random Forest model |
| `sass_calibrated_model.joblib` | Additional serialized model artifact; exact consumer not established by reviewed source |
| `sass_intelligent_model.joblib` | Additional serialized model artifact; exact consumer not established by reviewed source |
| `sass_encoders.joblib` | Encoder artifact created by `train_model.py` |
| `sass_intelligent.db` | SQLite database opened by the app |
| `sass_intelligence.db` | Second SQLite database; no use found in reviewed application code |
| `README.md` | Pinned package list and a minimal run instruction |
| `.gitattributes` | Git text normalization |

There is no visible test directory, `requirements.txt`, `pyproject.toml`, Dockerfile, deployment manifest, or GitHub Actions workflow in the inspected tree. The README says to run `app.py`, but does not document data/model setup, database behavior, or deployment constraints.

## 3. Architecture and runtime flow

### Main application (`app.py`)

The `AdvancedSASSModel` class combines several responsibilities:
- NSL-KDD CSV loading and preprocessing
- categorical feature encoding
- engineered network-behavior features
- StandardScaler and SMOTE
- Random Forest training wrapped in `CalibratedClassifierCV`
- probability-to-risk-score conversion
- alert novelty checks and analyst-action learning
- SQLite schema creation and performance/learning queries
- serialization and deserialization of the model bundle

Startup flow:
1. Importing the module constructs a global `AdvancedSASSModel`, which opens/initializes `sass_intelligent.db`.
2. `initialize_model()` attempts to load `sass_advanced_model.joblib`.
3. If that load fails, the app reads `KDDTest+.txt`, samples up to 50,000 rows, trains the advanced model, and saves the bundle.
4. The dashboard route reads up to 2,500 rows from the same `KDDTest+.txt`, scores them, and renders the high-/low-risk lists.
5. `POST /api/user_action` stores a submitted analyst action; `GET /api/stats` and `GET /api/learning_insights` expose aggregate database/model information.

### ML feature pipeline

The app drops `difficulty`, creates `is_attack` from `attack_type != 'normal'`, engineers ratios/error/login/access/network features, log-transforms byte and count fields, encodes `protocol_type`, `service`, and `flag`, scales the feature matrix, applies SMOTE to the training split, and fits a 300-tree Random Forest with calibration.

Risk probabilities are converted to integer scores from 1 to 10 using fixed thresholds: low below 0.3, high at/above 0.7, and medium between them. The score bands are not themselves a validated operational severity taxonomy.

### Dashboard and API

The template presents alert cards and exposes actions such as Investigate, Block, Whitelist, and False Positive. In the reviewed backend, these actions are passed to the feedback-learning method; there is no firewall, router, endpoint, or network-control integration. The word “Block” therefore describes a recorded analyst action, not a real block operation.

The dashboard also contains hard-coded timeline chart series and sample activity-feed content. The additional `static/advanced-dashboard.js` has a separate dashboard class and synthetic activity feed; the reviewed `index.html` does not include that JS file or `static/styles.css`. The active page appears to rely on inline template CSS/JS plus CDN libraries. Treat the extra static files as an apparently stale or disconnected UI implementation until their intended use is confirmed.

## 4. Findings and risks

### Critical / high priority

**A. Unsafe development server configuration**
- `app.py` starts Flask using `debug=True` and binds to `0.0.0.0:5000`.
- No authentication, role checks, CSRF protection, rate limiting, or deployment server configuration was found in the reviewed routes.
- Debug mode must not be exposed to untrusted networks. The user-action endpoint can write arbitrary action labels/notes for supplied alert hashes.
- Before deployment: disable debug, bind behind a production WSGI server/reverse proxy, add authentication/authorization, validate and constrain request fields, protect state-changing requests, and restrict network access.

**B. Not a live IDS / misleading source identity**
- `predict_intelligent_alerts()` generates `192.168.x.x` source addresses with Python's random module for each prediction.
- Those generated addresses are not observed endpoints and must never be interpreted as real hosts or used for blocking.
- The input is a static NSL-KDD dataset, not a live packet/flow feed. There is no packet capture, network sensor, SIEM ingestion, or streaming collector in the inspected tree.
- Rename UI claims and labels to “dataset replay/demo” until a real, provenance-tracked telemetry source is implemented.

**C. Pattern-learning lookup cannot match its stored representation**
- `update_behavioral_patterns()` computes an MD5 digest and stores it in `ai_patterns.pattern_signature`.
- `predict_user_action()` later searches that field using `LIKE '%<ip_class>%' `, e.g. `%internal%`.
- A hexadecimal MD5 digest does not contain the string `internal` or `external`, so the similar-pattern lookup cannot return those rows. The exact-alert lookup may still work when the same alert hash is encountered.
- Store structured pattern attributes in queryable columns or store a canonical unhashed signature and query exact normalized values. Add a regression test proving a learned pattern is retrievable.

### Medium priority

**D. Training/evaluation data separation is unclear**
- The main app's fallback training path uses `KDDTest+.txt` rather than `KDDTrain+.txt`.
- The dashboard then predicts on rows from `KDDTest+.txt` as well. The training function does hold out a random split for its reported metrics, but using the benchmark test file for model fitting invalidates that file as an independent final test set.
- Establish a strict protocol: fit preprocessing/model only on the training split, tune on validation data, and evaluate once on untouched test data. Record dataset version, split strategy, random seed, and metrics.

**E. Two incompatible training pipelines**
- `train_model.py` uses the column name `label`, LabelEncoder over categorical fields, no engineered features, and a basic Random Forest; it saves `sass_model.joblib` and `sass_encoders.joblib`.
- `app.py` uses `attack_type`, feature engineering, its own encoders/scaler, SMOTE, calibration, and the `sass_advanced_model.joblib` bundle.
- The README does not explain which path is canonical. The script also contains a machine-specific absolute Windows path.
- Consolidate training into one reusable module/CLI and make the app consume only a versioned artifact with an explicit schema.

**F. Feedback semantics and validation**
- The API checks for JSON and the presence of `alert_hash` and `action`, but does not enforce an action allowlist or validate types/ranges for response time, confidence, notes, or hash format.
- The endpoint returns a success-shaped response after calling the learning method even if the database method internally catches an error. A caller may therefore receive success without durable persistence.
- The request body is logged in full, potentially exposing free-text analyst notes.
- Define allowed actions, validate all fields, return a failure when persistence fails, avoid logging note contents, and add audit identity/authorization.

**G. Database lifecycle and concurrency**
- One SQLite connection is shared by the global model object with `check_same_thread=False`. This suppresses SQLite's thread-affinity guard; it does not by itself make concurrent use of the shared connection safe.
- Flask may handle concurrent requests. Use per-request connections or a controlled database layer, explicit transactions, busy timeout, and tested WAL/locking behavior.
- The schema has no visible migration/version mechanism. Add schema versioning and backup/retention policy.

**H. Risk score and auto-action policy**
- The code can mark repeated predicted actions as `auto_actionable` and record an `auto_...` action in its learning store. No external enforcement is performed, but the naming suggests automated response.
- Confidence derived from repeated user ratings is not equivalent to validated model confidence or a safe authorization policy.
- Keep automatic actions disabled by default. Require explicit human approval for any future enforcement and log the evidence, policy version, and outcome.

### Additional correctness / maintainability concerns

- `classify_ip()` assumes a dotted IPv4 string and uses positional splitting; malformed values or IPv6 are not handled robustly.
- Alert hashes include the risk score, so a score change creates a different identity for what may be the same host/service/protocol event.
- `generate_alert_signature()` includes a time category but is not the signature used by the stored behavioral pattern lookup.
- The dashboard uses synthetic/random response-time values and hard-coded chart data; these should be visually distinguished from measured telemetry.
- The extra static JS references many DOM elements and UI concepts that are not present in the reviewed template; loading it without reconciling the markup may cause runtime errors.
- The README is a package freeze rather than installation documentation. It omits Python version, setup commands, dataset acquisition/provenance, model artifact generation, and deployment instructions.
- `joblib.load()` deserializes Python objects. Only load model artifacts from a trusted source; do not accept user-uploaded or otherwise untrusted joblib files.

## 5. Data and artifact handling

The repository tracks two NSL-KDD text datasets (approximately 22 MB combined), several multi-megabyte `.joblib` model files, and two SQLite database files. This makes repository history and clones heavier and creates an unclear lifecycle for mutable runtime state.

Recommended separation:
- Keep source code, small test fixtures, and documented model metadata in Git.
- Store large datasets/model artifacts in a versioned artifact store or Git LFS if repository-hosted distribution is intentional.
- Keep writable runtime databases outside the source tree and exclude them from commits.
- Document dataset provenance, license/usage terms, checksums, model training parameters, and artifact compatibility.
- Review the committed SQLite files for analyst notes, IP-related records, or other operational data before further distribution; this source review did not inspect the binary database contents.

## 6. Recommended target structure

```text
pythonst/
  app/
    __init__.py
    routes/
      dashboard.py
      actions.py
      health.py
    services/
      alert_scoring.py
      feedback_learning.py
    ml/
      preprocessing.py
      training.py
      inference.py
      artifact.py
    storage/
      database.py
      repositories.py
    templates/
    static/
  configs/
  scripts/
    train.py
    evaluate.py
    replay_dataset.py
  tests/
    unit/
    integration/
    fixtures/
  artifacts/          # local only; ignored by Git
  data/               # local only; ignored by Git
  requirements.txt or pyproject.toml
  .env.example
  .gitignore
  README.md
```

Do not split files mechanically before tests exist. First capture current behavior and make the data/model contract explicit; then refactor behind tested interfaces.

## 7. Phased implementation roadmap

### Phase 0 — establish a safe baseline
- Remove debug mode from any remotely reachable configuration.
- Add `.gitignore` for runtime DBs, local model outputs, caches, virtual environments, and local secrets.
- Review committed database contents and model/data provenance.
- Add pinned dependency metadata and reproducible setup instructions.
- Clearly label the app as an offline NSL-KDD demo.

### Phase 1 — make the ML pipeline reproducible
- Consolidate the two training paths.
- Use NSL-KDD training data for fitting and preserve the test split for final evaluation.
- Persist preprocessing, feature order, class mapping, dataset checksum, dependency versions, and metrics with the model.
- Add tests for column schema, unseen categorical values, missing values, score boundaries, and artifact round-trips.
- Replace silent warning suppression with targeted warning handling.

### Phase 2 — correct feedback learning and persistence
- Fix the pattern representation/query mismatch.
- Define a stable alert identity separate from changing risk score.
- Add database migrations and per-request connection management.
- Validate action names and payload types; make persistence failures visible.
- Test duplicate alerts, repeated actions, database lock behavior, and restart persistence.

### Phase 3 — harden the web application
- Add login and role-based authorization for analyst actions.
- Add CSRF protection, request limits, rate limiting, secure cookie/session configuration, and security headers.
- Remove sensitive request-body logging.
- Escape all untrusted values in HTML/JavaScript contexts and avoid constructing markup with untrusted strings.
- Run behind a production WSGI server and reverse proxy; keep debug off.

### Phase 4 — evolve beyond dataset replay (only if intended)
- Define a real event schema and ingest trusted network telemetry.
- Preserve source IP/port/protocol/time from the sensor rather than synthesizing them.
- Add event provenance, deduplication, replay controls, retention, and backpressure.
- Treat “block” as a proposed action until a separately secured enforcement adapter exists.
- Require human approval, least privilege, rollback, and audit logs before connecting any enforcement system.

## 8. Minimum acceptance criteria

- One documented command installs dependencies and one documented command starts the demo.
- A clean checkout can load a verified model artifact or train one reproducibly from the documented training data.
- Test data is not used for fitting or model selection.
- A known learned pattern can be stored and retrieved by a regression test.
- Invalid actions and malformed payloads return 4xx responses; database failures do not return success.
- Concurrent API requests do not corrupt or intermittently fail SQLite writes.
- No debug server is exposed; analyst actions require authentication and authorization.
- UI explicitly distinguishes dataset replay/synthetic visuals from live telemetry.
- No real block/whitelist action is claimed unless a tested enforcement integration exists.

## 9. Validation status and limitations

- Source review: completed for the repository tree, README, main Flask source, separate training script, main template, error template, extra static JS/CSS inventory, and Git attributes.
- Git tree response was not truncated.
- Tests/build/runtime checks: **not run**. No test suite or dependency manifest was present in the inspected tree, and no execution environment was provisioned for this repository.
- Serialized model files and SQLite database contents were not deserialized/opened during this source review. Their internal contents and exact provenance remain unverified.
- No claim is made that the application currently starts successfully or that reported model metrics are reproducible.

## 10. Coding-agent handoff

Work in this order:
1. Add a reproducible environment and a minimal test harness without changing model behavior.
2. Extract and test the canonical NSL-KDD preprocessing and inference contract.
3. Fix the hashed-pattern lookup and stable alert identity.
4. Remove synthetic IPs from any production-like alert representation; label replay data honestly.
5. Harden Flask configuration and the analyst-action API before any deployment.
6. Only then split the monolithic model class into ML, persistence, and web layers.

**Do not** wire the current “Block” button to a real network-control mechanism, treat generated private IPs as observed hosts, or describe the current dashboard as live threat monitoring.
