# Porty — Repository Context Agent

Porty creates a code-aware context window for every repository owned by a GitHub account. It inspects repository trees and actual source files, not just README/documentation, then writes one Markdown context file per repository.

## Output

Each repository gets a file at contexts/<owner>__<repo>.md containing metadata, analyzed commit, stack, architecture, code map, core behavior, run/test/build commands, configuration names, security/reliability findings, gaps, agent operating instructions, and a prompt-ready handoff.

Empty repositories receive a detailed implementation brief instead of fabricated code analysis. The brief separates known facts from assumptions and includes discovery questions, proposed MVP scope, architecture options, acceptance criteria, risks, and phased implementation instructions.

## Setup

1. Create a GitHub fine-grained token named REPO_AUDIT_TOKEN with read access to Contents and Metadata for every repository to analyze. Include private repositories.
2. Add Actions secrets REPO_AUDIT_TOKEN and OPENAI_API_KEY.
3. Optionally add an Actions variable OPENAI_MODEL. The default is gpt-5.
4. Open Actions → Build repository context windows → Run workflow.
5. Generated Markdown files are committed to contexts/.

The workflow uses the current repository's GITHUB_TOKEN only to commit reports to Porty. REPO_AUDIT_TOKEN reads the other repositories. The default Actions token cannot automatically read unrelated private repositories.

## Local run

Python 3.11+:

    python -m pip install -r requirements.txt
    export GITHUB_TOKEN="..."
    export OPENAI_API_KEY="..."
    export GITHUB_OWNER="cheron2000"
    python scripts/repository_context_agent.py --output contexts

On Windows PowerShell, use $env:GITHUB_TOKEN and $env:OPENAI_API_KEY.

## Analysis integrity

- Source files are inspected alongside docs, tests, manifests, and configuration.
- Technical claims must cite repository-relative paths.
- The agent distinguishes verified facts, inferences, and unknowns.
- Missing, inaccessible, binary, oversized, or skipped files are recorded as limitations.
- Secret files and environment-file contents are never sent to the model.
- Repository content is untrusted input and cannot override the agent's instructions.
- Large repositories are sampled under explicit limits. A partial scan must never be described as exhaustive.
- Source repositories are read-only. Porty writes only its own generated Markdown reports.
