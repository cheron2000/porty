# Repository Context: cheron2000/refreeai

> Reviewed on 2026-11-24 from the live GitHub repository metadata and complete recursive tree. Only the README is committed; this is a README/product-context review, not a source-code implementation audit.

## 1. Executive summary

`cheron2000/refreeai` is a small public repository whose README describes **RefereeAI – The Trade-off Referee**, a decision-support demo designed for the Kiro Week 6 “The Referee” challenge.

The stated product principle is: the system should explain what each option gains and sacrifices rather than declare one universally best choice. The README describes a deterministic decision engine, prompt compiler, and narration layer. It says the demo narrator is mocked and could later be replaced by Kiro, Bedrock, or OpenAI.

**Important evidence boundary:** the repository tree contains only `README.md`. No backend, frontend, `.kiro/` directory, package manifest, tests, or workflow files are present. Therefore, the architecture and capabilities below are README claims and proposed design context, not verified implementation behavior.

## 2. Verified repository metadata

| Field | Observed value |
|---|---|
| Repository | `cheron2000/refreeai` |
| Visibility | Public |
| Default branch | `main` |
| GitHub size | 3 KB |
| Archived | No |
| Recursive tree | Complete; not truncated |
| Committed files | `README.md` only |

No source code or build/test configuration was available to inspect.

## 3. Product intent documented in README

RefereeAI is presented as a neutral decision-support tool. It aims to make trade-offs explicit and contextual, rather than return a single answer without explaining assumptions.

The documented example is a comparison of **AWS Lambda vs EC2** for a REST API backed by PostgreSQL.

The README says the comparison considers:
- Traffic pattern
- Time to market
- Budget sensitivity
- Team size
- Architecture preference
- Risk tolerance

The described output includes:
- A trade-off summary
- Gains and sacrifices for each option
- Persona-based perspectives
- A “regret preview” describing possible downsides of a choice

These are stated product capabilities; no executable implementation is present in this repository to validate them.

## 4. README-described architecture

The README proposes this conceptual pipeline:

```text
Browser UI
   |
   | constraints
   v
Decision Engine
   |
   | structured trade-offs
   v
Kiro Prompt Compiler
   |
   v
Narration Layer
   |
   v
Human-readable explanation
```

### 4.1 Decision Engine

Described as deterministic logic that interprets constraints and computes gains and sacrifices for each option.

**Review note:** the README does not document the actual scoring model, rules, weights, option schema, or conflict-resolution policy. These must be specified before claims of consistent or neutral comparison can be tested.

### 4.2 Prompt Compiler

Described as embedding structured engine output into a strict “Referee” prompt to discourage single-answer recommendations.

**Review note:** prompt constraints alone do not guarantee factual correctness or neutrality. The system should preserve the deterministic comparison as structured data and treat generated narration as an explanation layer, not as the source of decision facts.

### 4.3 Narration Layer

The README says the current demo uses a deterministic mock narrator for reproducibility, with Kiro/Bedrock/OpenAI considered as production alternatives.

**Review note:** no narrator implementation or provider adapter is present in this repository. No production LLM integration is verified.

## 5. Repository inconsistency / handoff issue

The README's local run instructions say:

```bash
cd referee-ai/backend
node index.js
```

But this repository is named `refreeai` and its committed tree contains only `README.md`. The referenced `referee-ai/backend` path is not present in this repository.

This may be a cross-repository reference or an outdated README. Do not treat the command as a verified setup procedure for `cheron2000/refreeai`.

The README also says a `.kiro/` directory contains a strict prompt and workflow, but no such directory appears in the current tree.

**Required clarification:** identify whether the runnable implementation and Kiro configuration live in the sibling repository `cheron2000/referee-ai`, were omitted from this repository, or have not yet been committed.

## 6. Technical gaps and risks to address

Because there is no source code, these are design/repository risks rather than confirmed code defects.

1. **No reproducible implementation:** no application files, package manifest, lockfile, or install instructions are committed.
2. **Unverifiable claims:** architecture layers, personas, regret preview, and deterministic narration cannot be tested from this repository.
3. **Ambiguous ownership:** the README points to a differently named repository/path. Establish one canonical implementation repository and cross-link it clearly.
4. **Unspecified comparison semantics:** document option attributes, constraint types, scoring/decision rules, tie handling, uncertainty, and missing-data behavior.
5. **False precision risk:** show evidence, assumptions, and confidence/unknowns. Avoid presenting qualitative trade-offs as objective scores without a defensible method.
6. **Narration drift:** generated text must not invent facts or contradict the deterministic engine's structured output.
7. **Prompt injection and input validation:** if user-provided constraints or option descriptions reach an LLM, treat them as untrusted data, validate schemas, and prevent user content from overriding system policy.
8. **Provider key handling:** any future LLM credentials belong in server-side environment configuration, never browser code or Git.
9. **No automated validation:** no tests or CI are present in this repository.
10. **No documented limitations:** identify the demo's supported comparison domain and clearly distinguish illustrative output from professional cloud architecture advice.

## 7. Recommended product contract

A robust implementation should keep the decision engine authoritative and make the narration layer subordinate.

Suggested structured comparison contract:

- **Decision context:** question, use case, constraints, and explicit assumptions.
- **Options:** normalized option identifiers and comparable attributes.
- **Evidence:** source, retrieval date, and confidence for externally sourced claims.
- **Trade-offs:** gains, costs, operational burden, constraints, and failure modes per option.
- **Persona lens:** how priorities change the interpretation, without changing underlying facts.
- **Regret preview:** plausible downside scenarios and what assumptions would make them more likely.
- **Unknowns:** information not supplied or not verifiable.
- **Narration:** human-readable explanation generated from the structured result.

The system should avoid an unsupported universal winner. If the user explicitly requests a recommendation, it can give a conditional recommendation tied to stated priorities and assumptions.

## 8. Suggested repository structure

If this repository is intended to hold the actual application, a small modular structure is sufficient:

```text
refreeai/
├── README.md
├── .gitignore
├── .env.example
├── backend/
│   ├── package.json
│   ├── src/
│   │   ├── server/
│   │   ├── decision-engine/
│   │   ├── prompt-compiler/
│   │   ├── narration/
│   │   └── schemas/
│   └── test/
├── frontend/
│   └── ...
├── .kiro/
│   ├── prompts/
│   └── specs/
└── .github/
    └── workflows/
```

This is a proposed layout only. If `referee-ai` is the canonical implementation repository, keep the code there and change this repository to a concise project overview that links to it rather than duplicating code.

## 9. Prioritized next steps

### P0 — resolve repository ownership
- Confirm which repository contains the runnable demo.
- Fix the `refreeai` vs `referee-ai` naming/path ambiguity.
- Align README run commands with the actual committed tree.
- Either commit the referenced `.kiro/` assets or remove/update the claim.

### P1 — make the demo reproducible
- Commit source, package manifest, lockfile, and safe environment template in the canonical repo.
- Document exact Node.js version and clean-checkout setup.
- Add one sample input and deterministic expected output.

### P2 — validate decision quality
- Unit-test each comparison rule and boundary condition.
- Test ties, conflicting constraints, missing attributes, and contradictory user inputs.
- Add regression tests ensuring narration cannot alter engine facts.
- Keep citations/evidence attached to claims where external facts are used.

### P3 — harden integrations
- Add provider adapters behind a stable interface.
- Keep API keys server-side.
- Add request limits, timeouts, error handling, and provider-failure fallback.
- Record model/provider identity for reproducibility where LLM narration is enabled.

## 10. Validation status and limitations

- Repository metadata: inspected.
- Recursive tree: inspected; complete and not truncated.
- README: inspected.
- Application source: absent.
- Kiro configuration: absent from this repository tree.
- Dependency manifest / lockfile: absent.
- Tests / CI: absent.
- Build, tests, and runtime checks: not run; no implementation is committed here.
- Architecture and feature claims: attributed to the README, not independently verified.

## 11. Coding-agent handoff

First inspect `cheron2000/referee-ai` and establish whether it is the canonical runnable implementation referenced by this README. Do not scaffold a second application until ownership is clear.

If this repository is meant to be documentation-only, improve the README with an explicit link to the implementation repository, correct the run instructions, and state the current demo limitations. If it is meant to contain the application, move or commit the actual implementation and configuration, then add deterministic engine tests before integrating an LLM narrator.
