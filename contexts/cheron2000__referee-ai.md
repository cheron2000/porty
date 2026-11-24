# Repository Context: cheron2000/referee-ai

> Reviewed on 2026-11-24 from the complete recursive Git tree and the committed README, backend, frontend, and Kiro prompt/workflow files. No build or test execution was performed.

## 1. Executive summary

`cheron2000/referee-ai` is the implementation repository for **RefereeAI – The Trade-off Referee**, a decision-support demo comparing AWS Lambda and EC2 for a REST API backed by PostgreSQL.

The core idea is to expose contextual trade-offs rather than declare a universal winner. The current implementation has a deterministic rule-based decision engine, a Node.js HTTP server, an embedded HTML interface, a prompt-template builder, and a deterministic mock narrator.

**Implementation status:** an early, narrow prototype. The backend decision path is present, but the React frontend is incomplete/disconnected, the LLM/Kiro integration is not active, and the repository has no dependency manifest, automated tests, or CI workflow.

This repository resolves the cross-reference in `cheron2000/refreeai`: the README there points to `referee-ai/backend`, and this repository contains that backend path. However, the `refreeai` README also describes a Kiro workflow directory; here the prompt file exists but the workflow JSON is empty.

## 2. Verified repository metadata

| Field | Observed value |
|---|---|
| Repository | `cheron2000/referee-ai` |
| Visibility | Public |
| Default branch | `main` |
| GitHub size | 160 KB |
| Archived | No |
| Recursive tree | Complete; not truncated |
| Main implementation | `backend/` |
| React pages | `frontend/pages/` |
| Kiro assets | `.kiro/prompts/`, `.kiro/workflows/` |
| Screenshots | `screenshots/` |

### Committed file inventory

- `.kiro/prompts/tradeoffPrompt.txt`
- `.kiro/workflows/decisionWorkflow.json` (0 bytes)
- `README.md` (0 bytes)
- `backend/decisionEngine.js`
- `backend/index.js`
- `backend/personas.js`
- `frontend/pages/index.js`
- `frontend/pages/result.js`
- Several screenshots documenting the demo and architecture

No `package.json`, lockfile, test files, CI workflow, or environment template appears in the tree.

## 3. Architecture and runtime flow

The implemented backend path is:

```text
Browser
  |
  | GET /
  v
Node.js built-in HTTP server
  |
  | POST /decide (JSON constraints)
  v
evaluateDecision() in decisionEngine.js
  |
  +--> buildPrompt() using .kiro/prompts/tradeoffPrompt.txt
  |
  +--> dynamicNarrative() deterministic mock
  |
  v
JSON response: promptSentToKiro + result
```

The server uses Node's built-in `http` and `fs` modules. The default user interface is HTML embedded as a template string inside `backend/index.js`; it is not served from the React page files.

## 4. Backend component review

### 4.1 `backend/index.js`

Responsibilities:
- Loads `.kiro/prompts/tradeoffPrompt.txt` relative to the module.
- Builds a prompt containing serialized user constraints and decision-engine output.
- Defines a deterministic `dynamicNarrative()` function.
- Serves an embedded HTML form at `GET /`.
- Accepts JSON at `POST /decide`.
- Returns both the generated prompt and mock narrative.
- Listens on port 3000.

**Important observation:** `promptSentToKiro` is a misleading response field name. The prompt is constructed and returned to the client; there is no Kiro, Bedrock, or OpenAI network call in this implementation.

The HTML form embedded in the server is the functional interface path currently wired to `/decide`. It collects six constraints and displays the response text.

### 4.2 `backend/decisionEngine.js`

Exports `evaluateDecision(inputs)`. It creates two structured result objects, `lambda` and `ec2`, each with:
- `gains`
- `sacrifices`
- `shortTerm`
- `longTerm`

The current rule branches consider:
1. Traffic pattern
2. Time to market
3. Budget sensitivity
4. Team size
5. Architecture preference
6. Risk tolerance

The rules append qualitative trade-off statements to the two options. For example, spiky traffic favors Lambda's autoscaling/pay-per-use characteristics, while predictable traffic adds EC2 gains around stable performance and cost estimation.

**Rule-engine limitations:**
- The engine is a fixed set of conditional statements, not a quantitative scoring or optimization model.
- The `shortTerm` and `longTerm` arrays are returned but never populated.
- Inputs are not validated against allowed values. Any unexpected value falls into an `else` branch and may silently be treated as a valid opposite choice.
- Missing fields also enter the `else` branches, producing misleading output instead of a validation error.
- There is no explicit representation of uncertainty, evidence, assumptions, or conflicting constraints.
- The engine only compares Lambda and EC2; the option set and workload are hard-coded.

### 4.3 `backend/personas.js`

Defines three persona records:
- Solo developer
- Startup team
- Enterprise team

Each contains a focus description and a `likelyChoice`.

**Integration gap:** this module is not imported or used by `backend/index.js`. The narrative currently contains hard-coded persona statements, so changing `personas.js` will not change the actual response.

## 5. Prompt and narration review

### 5.1 `.kiro/prompts/tradeoffPrompt.txt`

The prompt defines RefereeAI as a neutral technical decision referee. It instructs the model to:
- Compare Lambda and EC2 for a PostgreSQL-backed REST API.
- Avoid a single best-option recommendation.
- Explain gains and sacrifices.
- Compare short-term and long-term impacts.
- Include monolith/microservices implications.
- Use a five-section output format.

The backend substitutes a combined block of user constraints and engine output into the `{{constraints}}` placeholder.

**Current status:** prompt construction is implemented; LLM execution is not. The deterministic mock narrative is what the user sees.

### 5.2 `.kiro/workflows/decisionWorkflow.json`

The file exists but is empty (0 bytes). It does not define an executable or documented workflow.

### 5.3 Narration consistency

The mock narrator uses engine gains/sacrifices but also inserts fixed generic statements about personas and regret. Those fixed statements do not depend on the actual persona configuration or a structured regret model. Consequently, the prose may not faithfully reflect all six selected constraints.

The mock narrator is useful for reproducible demos, but should be labeled clearly as a deterministic template rather than an LLM-generated answer.

## 6. Frontend review

### 6.1 `frontend/pages/index.js`

This is a React component using `useState`. It initializes six form values and renders one select per key.

However:
- Each select renders only one option: its current value.
- There is no complete list of valid choices for the user to select.
- There is no submit/compare handler.
- It does not call the backend API.
- It does not navigate to the result page or render a result.
- No framework or package configuration is committed to establish how these pages are built or served.

This page is a partial UI scaffold, not a functional counterpart to the embedded HTML interface.

### 6.2 `frontend/pages/result.js`

Displays a static example of Lambda/EC2 trade-offs, persona perspective, and regret preview.

It does not receive live decision data, fetch the backend, or consume route/query state. It is a static mock result page.

### 6.3 Frontend/backend split

There are currently two separate UI paths:
- The embedded HTML page in `backend/index.js`, which calls the backend and is the functional demo path.
- The React page files, which are not connected to the backend and are incomplete.

Choose one canonical frontend. If retaining React, remove the duplicate embedded UI after the React app has a working API integration and verified build. If the project is intended as a minimal Node demo, remove or clearly label the unused React scaffold.

## 7. API and reliability review

The server exposes:
- `GET /` — returns embedded HTML.
- `POST /decide` — parses JSON, evaluates the rules, builds a prompt, and returns a narrative.
- Other routes — 404.

Observed hardening gaps:
1. **No request-body size limit:** a client can send an arbitrarily large body.
2. **No schema validation:** the endpoint accepts missing, unexpected, or wrong-typed fields.
3. **No content-type enforcement:** JSON parsing is attempted regardless of request headers.
4. **No method/path routing abstraction:** the small server is manageable now, but will become harder to maintain as endpoints grow.
5. **No explicit exception boundary around evaluation/prompt generation:** malformed input or unexpected runtime errors may terminate the request without a controlled JSON error response.
6. **No CORS/authentication/rate limiting:** these may be unnecessary for a localhost-only demo, but must be deliberately designed before network exposure.
7. **No graceful shutdown or server error handling:** operational lifecycle behavior is not addressed.
8. **No request timeout or cancellation policy:** relevant if a remote model provider is added later.
9. **No API contract documentation:** field names and allowed values are embedded in code rather than a schema.

The server listens on the default host binding rather than explicitly restricting itself to loopback. Review deployment exposure before running it on an untrusted network.

## 8. Product and decision-quality risks

- **Hard-coded comparison domain:** the product concept sounds general, but the engine only compares AWS Lambda and EC2 for one API workload.
- **Oversimplified infrastructure trade-offs:** cost and performance depend on workload shape, region, architecture, database connectivity, concurrency, cold starts, scaling configuration, and operational requirements. Generic statements should not be represented as universal facts.
- **No evidence provenance:** technical claims have no linked source, date, or confidence.
- **No assumptions surfaced:** users cannot inspect what the engine assumes about traffic, database behavior, or workload duration.
- **Persona overgeneralization:** the static idea that enterprise teams prefer EC2 or solo developers prefer Lambda can be misleading without contextual caveats.
- **No sensitivity analysis:** small changes in constraints cannot be traced to the exact rules that changed the result.
- **No clear distinction between descriptive comparison and recommendation:** the UI says it is not a recommendation, but the persona module uses `likelyChoice`. Align the product language and behavior.

## 9. Security considerations

Current demo appears to have no user authentication, database, or secrets. Do not add these without a clear requirement.

Before adding an LLM provider:
- Keep provider keys on the server, in environment variables or a secrets manager.
- Never expose keys in React/browser bundles or commit them.
- Treat all user-supplied constraints and option text as untrusted input.
- Keep system instructions separate from user data; delimit and validate structured input.
- Bound request sizes and model token budgets.
- Add provider timeouts, rate limits, and safe error handling.
- Avoid logging secrets or unnecessary personal data.
- Ensure model output is escaped/safely rendered if rich HTML or Markdown is later introduced.

For a local hackathon demo, make the intended bind address explicit and document whether LAN access is supported.

## 10. Recommended target structure

A modest modular Node application is sufficient; microservices are not justified by the current scope.

```text
referee-ai/
├── README.md
├── package.json
├── package-lock.json
├── .gitignore
├── .env.example
├── backend/
│   ├── src/
│   │   ├── server.js
│   │   ├── routes/
│   │   ├── schemas/
│   │   ├── decision-engine/
│   │   ├── prompt/
│   │   └── narration/
│   └── test/
├── frontend/
│   ├── package.json
│   └── ...
├── .kiro/
│   ├── prompts/
│   └── workflows/
└── .github/
    └── workflows/
```

If using one Node package for the backend only, keep the structure smaller. Do not introduce a monorepo or frontend build pipeline unless React is actually selected as the primary UI.

## 11. Prioritized implementation roadmap

### P0 — make the current prototype reproducible
- Add a README with the actual setup and run instructions.
- Add a Node.js engine requirement, `package.json`, and lockfile if dependencies are introduced.
- Document the current embedded-HTML demo path.
- Decide whether React pages are active or archival.
- Correct the `promptSentToKiro` field name or actually integrate a provider.

### P1 — make decision logic testable
- Define a schema for the six input dimensions and allowed values.
- Reject missing/invalid values with a 400 response.
- Add unit tests for every rule branch, defaults, missing fields, and invalid values.
- Populate or remove the unused short/long-term arrays.
- Add structured explanations that identify which input rule produced each gain/sacrifice.

### P2 — align the UI and architecture
- Select one frontend.
- If React is retained, add complete option lists, submit handling, loading/error states, API calls, and result rendering.
- Remove duplicate or stale UI implementations.
- Integrate `personas.js` or remove it until used.
- Add API contract documentation.

### P3 — improve decision quality
- Make assumptions explicit.
- Attach evidence/source metadata to factual cloud claims.
- Add uncertainty and unknowns rather than false precision.
- Add workload profiles and more options only when supported by documented rules and evidence.
- Keep deterministic facts separate from generated narration.

### P4 — integrate LLM narration safely
- Define a provider interface and a mock implementation.
- Add a real provider adapter only when configured.
- Ensure the model explains, but does not modify, structured engine output.
- Add timeout, retry, budget, and provider-failure handling.
- Test prompt-injection attempts and contradictory or malformed model output.

### P5 — CI and release hygiene
- Add formatting/lint/test/build scripts.
- Add CI for clean installation and checks.
- Keep `.env`, secrets, build output, and local artifacts out of Git.
- Document demo limitations and deployment boundaries.

## 12. Validation status and limitations

- Repository metadata: inspected.
- Recursive tree: inspected; complete and not truncated.
- Backend source: inspected.
- Decision engine and persona module: inspected.
- React page files: inspected.
- Kiro prompt: inspected.
- Kiro workflow JSON: confirmed empty.
- README: confirmed empty.
- Package manifest / lockfile: absent.
- Tests / CI: absent.
- Build, tests, and runtime checks: not run.
- Screenshots: inventoried but not visually audited; conclusions above are based on source files.

## 13. Coding-agent handoff

Treat `backend/index.js` plus `decisionEngine.js` as the current runnable prototype's core. First add reproducible project metadata and tests around the decision rules, then validate the embedded HTML flow. Decide whether React is the intended frontend before wiring or deleting the current scaffold.

Do not claim Kiro/LLM integration is live: the prompt is only assembled and returned, while the actual narrative is generated by a deterministic local function. Keep the deterministic engine authoritative and make any future LLM a bounded explanation layer.
