# Decision Log — Rapid-Test Chatbot UI

Version: 1.0
Owner: Security Architect / SPARC Architect
Status: Updated for Architecture & Security Decisions
Last Updated: 2025-08-06
## 2025-08-06 — UI Compatibility Adapter for Gradio 5.x and DummyGradio

Context:
- Contract tests use a DummyGradio surface that diverges from real Gradio 5.x constructor names, specifically for Dropdown. Production UI built with Gradio 5.41.0 functions correctly; tests fail due to inability to construct Dropdown under DummyGradio despite multiple name attempts.
- Goals: Preserve production correctness and security, achieve test parity without leaking framework-specific variance into application UI composition, and maintain modularity (<500 LOC per file).

Options Considered
1) Adapter Boundary (UI factory module)
- Create app/gr_compat.py exposing factories: dropdown, textbox, button, chatbot, markdown, state. Internally resolve to real Gradio constructors in production and to DummyGradio-compatible constructors in tests (name probing with deterministic order).
- Benefits: Isolates variance to a single module; no behavior change in UI logic; enables future upgrades. Keeps [app/ui.py](app/ui.py) simple.
- Drawbacks: Small new module to maintain; requires unit tests for mapping.
- Complexity: Low.

2) Modify Tests to Match Real Gradio
- Benefits: No code changes.
- Drawbacks: Test contract owned externally; less control; does not help future surface drift.
- Complexity: Medium (coordination).

3) Conditional Logic Inline in UI
- Sprinkle hasattr/name-resolution in [app/ui.py](app/ui.py:1).
- Benefits: No new file.
- Drawbacks: Pollutes UI assembly, harder to test/maintain; violates modular separation.
- Complexity: Low/Medium.

4) Skip/xfail Affected Tests
- Benefits: Fast path.
- Drawbacks: Leaves integration unknown; reduces confidence; not aligned with quality gates.
- Complexity: Low.

Decision
- Chosen: Option 1 — Introduce UI Compatibility Adapter (app/gr_compat.py) and refactor [app/ui.py](app/ui.py:1) to consume adapter factories.
- Rationale: Minimal risk, clean separation, future-proof against surface drift, maintains security posture and production behavior.

Implementation Details
- Factories: dropdown(label, choices, value=None, interactive=True, visible=True, allow_custom_value=False, multiselect=False, elem_id=None); textbox(...); button(...); chatbot(...); markdown(...); state(value=None).
- Resolution strategy: Attempt canonical Gradio class names first (e.g., gr.Dropdown), then probe alternate names common in DummyGradio (Dropdown, dropdown, Select, select, Choice, choices, etc.). If constructor not found, raise NotImplementedError with clear guidance for tests.
- Production mapping: Direct to gr.* constructors (Gradio 5.41.0). No behavioral changes.
- Testing mapping: Use DummyGradio surface via injected module reference or environment flag if present; otherwise fall back to name probing.
- Observability: Optional debug log of chosen constructor name (no PII or secrets); disabled by default.

Security Measures
- No secrets introduced; adapter constructs UI components only.
- Errors from adapter are sanitized in UI layer; no stack traces or keys exposed.

Success Criteria
- tests/test_ui_contract_gradio5.py passes with DummyGradio.
- No regressions when running with real Gradio 5.41.0; streaming, event wiring, and sanitization unchanged.
- Files remain <500 LOC; adapter ~≤200 LOC.
- Memory Bank updated (decisionLog, progress; pattern to systemPatterns.md).

Risks and Mitigation
- Risk: DummyGradio uses unexpected names beyond probe list → Mitigation: expand probe list or allow injection of mapping for tests.
- Risk: Future Gradio API changes → Mitigation: single adapter boundary to update.
- Risk: Divergent behavior across environments → Mitigation: add adapter unit tests for both real and dummy surfaces.

Review Schedule
- Next Review: 2025-08-13 after adapter implementation and test re-run.
- Triggers: New failing UI contract tests due to constructor naming changes or Gradio minor upgrades.

Linked Artifacts
- Integration report: [integration-report.md](integration-report.md)
- UI module: [app/ui.py](app/ui.py:1) (to be refactored)
- Tests: [tests/test_ui_contract_gradio5.py](tests/test_ui_contract_gradio5.py)
## 2025-08-06 — Security Remediation Decisions (Dependencies + Container)

Context:
Security remediation applied to eliminate vulnerable components and harden container runtime while preserving minimal footprint and non-root execution.

Decisions:
1) Dependency: gradio&gt;=5.31.0,&lt;6.0.0
- Rationale: Addresses Safety-reported SSRF/XSS/DoS issues in 4.x/early 5.x; stays within 5.x to avoid major 6.x API shifts.
- Alternatives: Pin to latest 5.x (risk: future regressions), downgrade to 4.x with patches (insufficient).
- Implications: Potential 5.x API differences. Validate event wiring and components in [app/ui.py](app/ui.py) and [main.py](main.py). Add/enable tests to detect UI breakages.

2) Dependency: requests==2.32.4
- Rationale: Fixes CVE-2024-47081 (.netrc leakage via redirects).
- Alternatives: requests 2.32.3 or &lt;2.32.4 (vulnerable), switch to httpx (broader refactor).
- Implications: Remains synchronous, no behavioral changes expected; confirm redirect handling stays secure and consistent in [app/openrouter_client.py](app/openrouter_client.py).

3) Dockerfile Hardening (ENV + HEALTHCHECK)
- ENV: PIP_NO_CACHE_DIR=1, PYTHONHASHSEED=random, PYTHONUTF8=1
- HEALTHCHECK: Python stdlib socket check to avoid extra packages
- Non-root user retained; base remains python:3.11-slim
- Rationale: Reduce attack surface, nondeterministic hash defense-in-depth, UTF-8 stability, liveness detection without adding binaries.
- Implications: CI must respect HEALTHCHECK; observability should surface health status. Keep image lean; avoid cache bloat.

Consequences:
- Positive: Vulnerable components remediated; container posture improved; no added runtime deps for health checks.
- Negative: Gradio 5.x compatibility risk until validated; local disk constraints blocked full pip install in dev environment (not expected in CI).

Follow-ups:
- Tests: Unskip/add functional tests for Gradio 5.x wiring and validators.
- Runtime: E2E container run to verify streaming, allowlist, and sanitized errors.
- Proxy: Apply gateway-level SSRF blocking, header normalization, and rate limiting.

Quality Gates:
- R1 Implementation: Deps and Docker hardening completed; tests pending.
- R3 Security: Safety scan clean for requirements.txt; runtime negative-path validations pending.
- C1 Completion: HEALTHCHECK and non-root image in place; CI build and proxy hardening pending.

References:
- Dependencies: [requirements.txt](requirements.txt)
- Container: [Dockerfile](Dockerfile)
- Validators: [app/validators.py](app/validators.py:36-81)
- Client: [app/openrouter_client.py](app/openrouter_client.py)
- UI: [app/ui.py](app/ui.py), [main.py](main.py)

## 2025-08-06 — Core Technology and Security Decisions

Context:
Rapid-Test is an internal, lightweight Gradio app to test multiple LLMs via OpenRouter with streaming responses, session-scoped state, and no server-side secret storage.

Decisions:
1) UI Framework: Gradio Blocks
- Rationale: Simple composition, native support for generator-based streaming updates, minimal boilerplate.
- Alternatives: Gradio Interface (less flexible), custom Flask/FastAPI + frontend (more complexity).
- Implications: Keep UI logic modular in app/ui.py.

2) HTTP Client: requests with iter_lines
- Rationale: Mature, straightforward streaming via stream=True and iter_lines(decode_unicode=True).
- Alternatives: httpx async (added complexity with event loop), SSE clients (unnecessary for minimal scope).
- Implications: Implement synchronous streaming generator in app/openrouter_client.py.

3) Default Model and Allowlist
- Default: openrouter/horizon-beta
- Allowlist: [ meta-llama/Llama-3-8b-instruct, mistralai/mistral-7b-instruct, google/gemma-7b-it, openrouter/horizon-beta ]
- Rationale: Sponsor-specified default; curated internal list for comparison.
- Implications: Server-side validation required in validators.validate_model.

4) Session State and History Cap
- Cap: 25 messages per session (total entries user+assistant).
- Rationale: Bound memory, reduce DoS surface, meet functional requirement for ephemeral testing.
- Implications: Enforce trimming on every append in state.append_message.

5) API Key Handling (No Storage)
- Policy: Never persist API keys to disk/logs; confine to session memory only.
- Rationale: Security objective to prevent secret leakage.
- Implications: No logging of headers; sanitize all errors; no echoing of secrets. Add test checks for leakage.

6) Transport Security
- Policy: HTTPS-only for OpenRouter; TLS verification enabled; reject cleartext.
- Rationale: Prevent MITM and leakage.
- Implications: Hardcode base URL to https://openrouter.ai/api/v1/chat/completions; validators to assert scheme if configurable.

7) Error Handling and Logging Hygiene
- Policy: Sanitize user-facing errors (401 → invalid key; 4xx/5xx → model request failed; network → retry message).
- Rationale: Avoid information disclosure while staying actionable.
- Implications: Centralize mapping in validators.sanitize_error_for_user; no headers/body in logs.

8) Containerization
- Base Image: python:3.11-slim
- Port: 7860
- Rationale: Small footprint, fast boot; consistent with Gradio defaults.
- Implications: Minimal dependencies in requirements.txt; optional non-root user.

9) Modularity Constraint
- Max 500 LOC per file; modules: settings.py, validators.py, state.py, openrouter_client.py, ui.py, main.py.
- Rationale: SPARC modularity and testability.
- Implications: Enforce via CI check and code reviews.

Consequences:
- Positive: Simple, secure-by-default architecture with minimal moving parts; easy to deploy; testable.
- Negative: Sync streaming may tie a worker during long streams; acceptable for internal usage.
- Follow-ups: Consider non-root container user and optional prompt-size cap.

Quality Gates:
- Security: No secrets in logs; HTTPS enforced; allowlist validation present.
- Testing: Unit tests for validators/state/client; integration mocks for streaming and error paths.
- Documentation: Threat model and security architecture created.

Linked Artifacts:
- Specification: specification.md
- Pseudocode: pseudocode.md
- Architecture: architecture.md
- Threat Model: docs/security/threat-model.md
- Security Architecture: docs/security/security-architecture.md
