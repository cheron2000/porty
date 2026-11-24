# Repository Context: cheron2000/verigov-ai

> Reviewed on 2026-11-24 from the complete recursive Git tree (152 entries, not truncated), README/hackathon/architecture/testing documentation, core Python modules, Lambda handlers, deployment scripts, CloudFormation, requirements, whitelist configuration, and representative tests. This is a source review; no deployment, live API call, build, or test command was executed.

## 1. Executive summary

`cheron2000/verigov-ai` is **VeriGov AI**, a government-information and public-claim verification platform. It accepts a claim, selects or receives source URLs, fetches page content, asks an LLM to assess the claim, and returns a status, confidence score, explanation, evidence, and research metadata.

The project combines:
- Python CLI/application core under `src/verigov/`.
- Groq Llama 3.3 70B-based semantic analysis.
- A domain whitelist and HTML/JSON source collector.
- Local file storage and AWS DynamoDB/S3 storage adapters.
- A Flask proxy application.
- Multiple AWS Lambda verification handlers and support handlers.
- A vanilla HTML/CSS/JavaScript dashboard hosted from S3.
- API Gateway, Lambda, DynamoDB, S3, IAM, CloudWatch, and CloudFormation deployment assets.

**Current engineering assessment:** a substantial hackathon prototype with real application layers and AWS deployment tooling, but not yet a single coherent production architecture. Several parallel verification implementations have diverged. Most importantly, the Lambda path that accepts user-supplied URLs fetches them without the local application's whitelist validation, creating an SSRF risk if exposed to untrusted callers. The system's confidence scores are model-generated and are not shown to be statistically calibrated.

## 2. Verified repository metadata

| Field | Observed value |
|---|---|
| Repository | `cheron2000/verigov-ai` |
| Visibility | Public |
| Default branch | `main` |
| GitHub size | 301 KB |
| Archived | No |
| Recursive tree | 152 entries; complete |
| Main language | Python |
| UI | HTML, CSS, vanilla JavaScript |
| AI provider in core | Groq API, Llama 3.3 70B Versatile |
| Cloud architecture | AWS Lambda, API Gateway, DynamoDB, S3 |
| Infrastructure as code | CloudFormation template for DynamoDB |
| Test assets | `tests/` with manual/API scripts and fixtures |

The repository README advertises a live S3 website and API Gateway endpoint. These URLs and deployment claims were read from the repository; their current availability and configuration were not independently probed.

## 3. Product purpose and primary user flow

The platform is intended to help users assess claims about government policy, public health, science, space, international affairs, and related topics against trusted public sources.

Documented user flow:

1. User submits a textual claim.
2. The system detects one or more topics.
3. It selects relevant domains from a trusted-source catalogue or accepts user-supplied sources.
4. It fetches content from selected pages.
5. Groq LLM analyzes the claim against the fetched text, or uses model knowledge when the Lambda route is configured for AI-only handling.
6. The response contains a status, confidence score, explanation, evidence, sources selected/checked, research method, and timestamp.
7. The system stores verification results and audit events.

The product also describes verification history, source transparency, downloadable reports, and monitoring for source-page changes. Some of these features are implemented in different layers or described in documentation, but their end-to-end deployment and current operational status were not verified.

## 4. Architecture and implementation paths

The repository currently contains several overlapping execution paths.

### 4.1 AWS-hosted dashboard path

```text
Browser
  |
  v
S3 static website (static/index.html, script.js, style.css)
  |
  v
API Gateway REST API
  |
  +--> verify-sources endpoint
  |       |
  |       v
  |    Lambda verification handler
  |       |
  |       +--> topic detection / source selection
  |       +--> external source fetching
  |       +--> Groq LLM
  |       +--> DynamoDB / S3 persistence
  |
  +--> audit endpoint --> audit_handler.py --> DynamoDB
  |
  +--> whitelist endpoint --> whitelist_handler.py --> DynamoDB
```

The README documents this as the primary deployed architecture. Several deployment scripts package and update Lambda functions, upload the frontend to S3, and configure API Gateway resources.

### 4.2 Local CLI and core library path

```text
python -m verigov.main
  |
  v
VeriGovApp
  |
  +--> WhitelistManager
  +--> SourceCollector
  +--> IntelligenceLayer (Groq)
  +--> FactVerificationEngine
  +--> AuditLog
  +--> ChangeDetector
  |
  +--> StorageFactory
         +--> LocalStorage
         +--> AWSStorage
         +--> HybridStorage
```

The CLI supports single-claim verification, batch processing, retrieval by verification ID, audit export, and source monitoring.

### 4.3 Flask proxy path

`app.py` serves `templates/index.html` and exposes `POST /api/verify`. It forwards the claim and source list to a hard-coded API Gateway URL with a 60-second timeout. Its audit and whitelist endpoints are placeholders that return explanatory messages.

This is a proxy to the AWS verification service, not the same execution path as the local CLI. It runs with `debug=True` when started directly.

### 4.4 Multiple Lambda handler versions

The `lambda/` directory includes several iterations:
- `verify_handler.py`
- `verify_handler_simple.py`
- `verify_handler_smart.py`
- `verify_handler_smart_v2.py`
- `verify_handler_with_sources.py`
- `audit_handler.py`
- `whitelist_handler.py`

The deployment scripts reference different handler versions and deployment package layouts. This makes it necessary to identify the currently deployed handler and treat it as the actual production entry point; a file existing in the repository does not prove that AWS is running it.

## 5. Core Python components

### 5.1 `FactVerificationEngine`

File: `src/verigov/verification/fact_verification_engine.py`

Responsibilities:
- Creates a UUID for each verification.
- Logs verification start and completion.
- Collects each supplied URL through `SourceCollector`.
- Sends collected sources to `IntelligenceLayer`.
- Adds claim, source count, source URLs, and verification ID.
- Persists the result through `StorageFactory`.
- Supports batch verification and retrieval.

**Behavioral distinction:** when no valid sources are collected, this core engine returns `NO_SOURCES` directly. It does not invoke the intelligence layer's AI-knowledge-base path. This differs from the README's dual-mode product description and from the smart Lambda handlers, which may call the model without fetched sources.

Storage errors are logged, but a failed result write does not change the verification response to a failure. The caller may receive a result that was not durably stored.

### 5.2 `IntelligenceLayer`

File: `src/verigov/verification/intelligence_layer.py`

- Uses the Groq Python client.
- Calls `llama-3.3-70b-versatile`.
- Classifies claim text into biographical, statistical, factual, or general categories using keyword heuristics.
- Builds prompts with source content and asks the model for JSON containing status, confidence, explanation, and evidence.
- Attempts multiple JSON parsing strategies and normalizes fields/status/confidence.

Important limitations:
- Claim classification is keyword-based; it is not a trained classifier.
- The model assigns its own confidence score. No calibration dataset, reliability curve, or measured accuracy is present in the inspected materials.
- A model-generated confidence number should not be presented as a statistical probability that a claim is true.
- When source content is absent, some analysis branches ask the model to use its knowledge base. That is not equivalent to source-backed verification.
- Prompt context contains fetched page text. The implementation should explicitly treat retrieved content as untrusted data and guard against instructions embedded in pages.
- The code's output normalization constrains field shape, but does not independently validate factual correctness or evidence provenance.

### 5.3 `SourceCollector`

File: `src/verigov/collection/source_collector.py`

- Extracts a hostname/domain from a URL.
- Checks the hostname against `WhitelistManager`.
- Fetches using a persistent `requests.Session`, TLS verification enabled, redirects enabled, and a configurable timeout.
- Parses HTML with BeautifulSoup, removes common navigation/script/style elements, extracts title/text/summary/article links, and truncates extracted text.
- Supports JSON and generic text response handling.

Risks and improvement areas:
- Whitelist validation is an exact domain comparison; normalize case, trailing dots, IDNs, and subdomain policy deliberately.
- Redirects are followed after the initial domain check. Revalidate every redirect destination and resolved IP before following.
- Add response-size/content-type limits and explicit scheme restrictions.
- Avoid fetching private, loopback, link-local, and cloud metadata addresses.
- Add DNS-rebinding-resistant validation at connection time.
- HTML extraction is heuristic and may capture irrelevant content or miss client-rendered pages.
- The collector records fetch time and word count, but should also preserve a stable content hash, retrieval timestamp, canonical URL, and extraction quality indicators for reproducibility.

### 5.4 `WhitelistManager`

Loads source domains from the selected storage backend and merges metadata from `config/whitelist.json`. It supports adding/removing approved sources through its storage abstraction.

The configuration contains government, scientific, health, and international organizations. A domain being present in the whitelist is not proof that every page on that domain is authoritative for every claim; source selection and claim relevance still need validation.

### 5.5 Change monitoring

`ChangeDetector` periodically fetches configured URLs and compares current content with an in-memory cache. It logs a medium-impact change event and invokes a callback.

Limitations:
- Cache is in memory and resets on process restart.
- It uses a blocking loop and `time.sleep`, not a durable scheduler or distributed worker.
- It detects any content difference, including dynamic timestamps/ads/layout noise; no semantic diff or content normalization strategy is present.
- The `monitor_sources` wrapper accepts an interval but does not pass it to the detector's `monitor()` call, so the wrapper's interval argument is not applied by that path.

## 6. Storage and AWS integration

### 6.1 Storage abstraction

`StorageInterface` defines methods for audit logs, verification results, whitelist sources, and batch results.

Implementations:
- **LocalStorage:** JSON files under `logs/`, `data/verifications/`, and `data/batch_results/`, plus `config/whitelist.json`.
- **AWSStorage:** DynamoDB for verification/audit/whitelist operations and S3 archival for audit records, verification results, and batch results.
- **HybridStorage:** writes to local and AWS and reads from AWS first with local fallback.

### 6.2 AWS configuration consistency

There is a notable region default mismatch:
- `AWSStorage` and `DynamoDBClient` default to `us-east-1`.
- `.env.example` also sets `AWS_REGION=us-east-1`.
- Hackathon and architecture documentation describe deployment in `ap-south-1`.
- Some Lambda handlers hard-code `ap-south-1`.

This can cause the local AWS storage path to look for tables/buckets in a different region than the deployed stack unless the environment is configured consistently. Use one explicit region setting and avoid hard-coded regional clients.

### 6.3 DynamoDB and S3

CloudFormation defines three DynamoDB tables:
- Verifications (partition key `verification_id`, status/timestamp GSI).
- Audit logs (timestamp/event-type keys and GSIs).
- Trusted-source whitelist (domain key).

The AWS storage adapter archives audit events and verification results to S3 with server-side encryption. Batch results are stored in S3. Actual deployed table names, indexes, policies, and bucket settings were not queried during this review.

### 6.4 Audit semantics

The application describes audit logging as immutable/append-only, but the reviewed implementation does not establish tamper-proof audit guarantees. Local files can be edited by a process/user with filesystem access, and DynamoDB records can be changed by principals with write permissions. For stronger audit integrity, use restrictive IAM, append-only access patterns, retention controls, and optionally signed/hash-chained records or a dedicated immutable archive.

## 7. Lambda and API security review

This is the highest-priority review area.

### 7.1 SSRF risk in user-provided source fetching

The smart Lambda handlers accept `sources` from the request and pass them to a network fetch function. The inspected `verify_handler_smart_v2.py` uses `urllib.request.urlopen()` directly; the other smart handler uses `requests.get()`. These Lambda fetch paths do not call the local `WhitelistManager` before fetching user-supplied URLs.

If the API is reachable by untrusted callers, a user may be able to make Lambda request internal or sensitive network destinations. Redirects and DNS resolution also need to be constrained.

**Required controls:** allow only HTTP/HTTPS; enforce exact approved hostnames; resolve and block private/loopback/link-local/multicast/reserved IPs; revalidate redirects; protect against DNS rebinding; cap response size, redirects, and time; and preferably route fetches through a dedicated egress-controlled service.

### 7.2 Public endpoints and authorization

The inspected support Lambda handlers return wildcard CORS headers and the deployment script creates API Gateway methods with `authorizationType='NONE'`. CORS is not authentication. If these endpoints are public, audit history and verification data may be readable by anyone who can reach the API.

Add authentication/authorization or explicitly limit the data exposed by public endpoints. Do not rely on CORS to protect audit or whitelist routes.

### 7.3 Claim and source input validation

Some handlers validate only that claim text is non-empty and coerce invalid source lists to empty lists. Define a shared request schema with:
- Maximum claim length.
- Maximum number and length of URLs.
- Strict URL parsing and allowed schemes.
- Explicit handling of invalid types.
- Request-body size limits.
- Safe error responses that do not leak internals.

### 7.4 Error status semantics

The smart handlers return HTTP 200 even when the verification process fails internally and the body contains `status: ERROR`. This makes API-level monitoring and client retry behavior unreliable. Use an explicit error contract and appropriate 4xx/5xx status codes for failures.

### 7.5 Sensitive data in logs

The smart Lambda handler logs the full event payload. That can include user claims and supplied URLs. Establish a logging policy, redact sensitive fields, limit retention, and avoid logging full request bodies by default.

### 7.6 Prompt injection and source integrity

Fetched web pages are untrusted input. A page can contain text designed to influence the model. Clearly delimit source content, instruct the model to treat it as evidence rather than instructions, and validate citations/evidence against the actual fetched text. Do not let the model invent a source or quote.

## 8. Frontend and UX

The deployed static frontend is `static/index.html`, `static/script.js`, and `static/style.css`; a separate `templates/index.html` serves the Flask path.

The README documents:
- Claim submission and result display.
- Light/dark mode persisted in localStorage.
- Verification statistics/history.
- Source and research-method transparency.
- Responsive dashboard and downloadable reports.

There are multiple UI assets/paths, so verify which UI is actively deployed before changing templates. The Flask app's template path is separate from the S3 static site. No frontend build system is required by the inspected tree; the primary frontend is plain HTML/CSS/JS.

## 9. Documentation and claim reliability

The repository has extensive documentation: deployment guides, AWS integration plans, setup guides, architecture diagrams, source statistics, testing summaries, and hackathon submission material.

Treat some documentation metrics as historical project claims rather than independently measured current facts. Examples include 99.9% uptime, millions of requests, unlimited concurrency, sub-500ms API latency, and sub-$1/month operating cost. The reviewed source does not include load-test reports or monitoring exports that substantiate all of these figures.

The README's live demo URL uses S3 static website hosting over HTTP, while it separately claims HTTPS on API Gateway. Do not describe the static website itself as HTTPS-enabled unless a CloudFront/custom-domain TLS layer is configured.

The testing summary is dated March 3, 2026 and records successful manual setup/CLI/API checks at that time. It also explicitly acknowledges weak page-content extraction. It is historical evidence, not a current CI result.

## 10. Tests, deployment, and developer workflow

### Test assets observed

The `tests/` tree includes:
- API endpoint scripts that call the deployed API.
- Smart endpoint scenarios for space, health, science, and general claims.
- Local verification and source collector scripts.
- Groq/API/Lambda integration checks.
- HTML fixture/display checks.
- A Lambda event fixture.

Many inspected tests are executable/manual integration scripts that make network calls and print results, rather than isolated deterministic unit tests with assertions. The tree includes test files, but the review did not establish a comprehensive automated regression suite or CI workflow.

No `.github/workflows/` files appeared in the recursive tree. There is no repository-level CI workflow visible in the reviewed tree.

### Dependencies and packaging

- `requirements.txt`: Flask, requests, BeautifulSoup, dotenv, Groq, pytest, and developer tools.
- `requirements-base.txt`: local application dependencies and test/lint tools.
- `requirements-aws.txt`: broad AWS, Bedrock/LangChain, auth, deployment, and test extras.
- `lambda/requirements.txt`: Lambda-specific packages.
- `setup.py`: installable `verigov` CLI package with a console entry point.

The dependency lists overlap and are not fully aligned. The package metadata says Python >=3.8 while the README and Lambda deployment target Python 3.11; reconcile supported Python versions and pin/lock deployment dependencies.

### Deployment automation

The `scripts/` directory includes scripts for Lambda packaging/updating, API Gateway endpoint creation, DynamoDB provisioning, S3 frontend upload, full-stack deployment, billing alerts, and status checks.

Some deployment scripts use fixed AWS account/region/resource names. Treat them as environment-specific operator scripts, not safe generic deployment tooling. Before running them, review target account, region, IAM role, resource names, and destructive/update behavior.

## 11. Prioritized engineering roadmap

### P0 — secure network fetching and API exposure
- Apply the same allowlist policy to every Lambda and local source-fetch path.
- Enforce redirect/IP/DNS protections and response limits to close SSRF paths.
- Add authentication/authorization for audit and management endpoints.
- Stop logging full event/request payloads.
- Validate all request fields and set correct HTTP status codes for internal failures.

### P1 — consolidate verification execution
- Choose one canonical verification engine and one canonical Lambda handler.
- Extract shared topic detection, source selection, fetch, prompt, parsing, and response schemas into reusable modules.
- Remove or clearly archive superseded handler variants and deployment scripts.
- Align local CLI, Flask proxy, and Lambda behavior—especially the no-source/AI-knowledge fallback.
- Make deployment scripts target the canonical handler only.

### P1 — improve verification trustworthiness
- Distinguish source-backed verification from model-knowledge assessment in the API contract and UI.
- Store source URL, title, publisher/domain, retrieval time, content hash, and quoted evidence spans.
- Require evidence references for VERIFIED results.
- Calibrate confidence against a labeled evaluation dataset; otherwise label it as model confidence/estimate.
- Add adversarial prompt-injection tests and evidence-grounding checks.
- Add human review/escalation for high-impact government, health, and legal claims.

### P2 — data and operations reliability
- Unify AWS region configuration and remove hard-coded regions/account IDs.
- Define retention, deletion, and access policies for claims, results, and audit data.
- Improve audit integrity beyond the current append-only convention.
- Make source monitoring durable and schedule-driven, with normalized/semantic content comparison.
- Add structured logs, metrics, tracing, request IDs, and cost alarms.

### P2 — reproducible engineering
- Add a locked dependency set and supported Python version policy.
- Add CI for formatting, linting, type checking, unit tests, and packaging.
- Replace network-dependent print scripts with mocked unit tests; keep a smaller, opt-in integration suite for live services.
- Add deterministic tests for whitelist bypass, redirects, private IPs, malformed URLs, empty sources, model JSON errors, storage failures, and DynamoDB pagination.
- Add deployment smoke tests that identify the exact Lambda version/alias behind the API.

## 12. Validation status and limitations

- Repository metadata and recursive tree: inspected; 152 entries, complete.
- README, hackathon submission, architecture, project summary, testing summary: inspected.
- Core CLI, verification engine, intelligence layer, collector, whitelist, monitor: inspected.
- Local/AWS/hybrid storage abstractions and DynamoDB client: inspected.
- Representative Lambda handlers and support handlers: inspected.
- CloudFormation DynamoDB template and deployment scripts: inspected.
- Requirements, package setup, environment example, source whitelist: inspected.
- Representative tests: inspected.
- Full source tree: not line-by-line reviewed.
- Current AWS deployment/configuration and live endpoint: not queried.
- Test suite, build, lint, type check, deployment, and runtime: not executed.
- Documentation metrics and claimed uptime/load: not independently verified.

## 13. Coding-agent handoff

Treat VeriGov AI as a hackathon prototype with a useful end-to-end concept and several implementation generations. Do not add another handler variant. First identify the exact deployed Lambda and consolidate around one canonical verification pipeline.

Prioritize SSRF protection for user-supplied URLs, endpoint authorization, evidence grounding, and truthful confidence semantics. Then align local CLI and AWS behavior, standardize AWS configuration, and add deterministic tests/CI before expanding sources or adding more AI features.
