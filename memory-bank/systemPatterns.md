# System Patterns — Rapid-Test Chatbot UI

## UI Factory Adapter Pattern

Intent
Abstract UI component construction behind a thin compatibility layer to isolate framework surface variance (e.g., Gradio vs DummyGradio) from application UI composition.

Context
- Contract tests rely on a DummyGradio with constructor naming that may diverge from real Gradio 5.x, notably for Dropdown.
- Production uses Gradio 5.41.0 with correct behavior.
- We need test parity without polluting [app/ui.py](app/ui.py:1) with conditional logic.

Forces
- Maintain production correctness and security guarantees.
- Keep UI module under 500 LOC and focused on composition, not framework probing.
- Enable minor upgrades and alternative shims with minimal churn.
- Provide deterministic fallbacks in tests without changing app semantics.

Pattern
- Introduce adapter module [app/gr_compat.py](app/gr_compat.py) exporting factories:
  - dropdown(label, choices, value=None, interactive=True, visible=True, allow_custom_value=False, multiselect=False, elem_id=None)
  - textbox(label, value="", lines=1, placeholder=None, interactive=True, visible=True, elem_id=None, password=False)
  - button(label, variant="primary", visible=True, elem_id=None)
  - chatbot(label=None, visible=True, elem_id=None, height=None)
  - markdown(value, visible=True, elem_id=None)
  - state(value=None)
- Primary resolution path maps to real Gradio: gr.Dropdown/Textbox/Button/Chatbot/Markdown/State.
- Secondary resolution path probes alternate constructor names common in DummyGradio (e.g., Dropdown, dropdown, Select, select, Choice, choices, etc.) in a deterministic order.
- Optional injection: allow tests to inject the dummy surface or explicit name mapping for precise control.

Structure
- app/gr_compat.py: ≤200 LOC, pure functions returning component instances; no secrets; optional debug logging (disabled by default).
- app/ui.py: imports factories and composes the UI; no direct gr.* usage; retains behavior and security invariants.

Consequences
Positive:
- Single boundary to update for future framework drift.
- Cleaner UI assembly, easier testing.
- Enables running the same composition against both environments.

Negative:
- Additional small module and unit tests to maintain.
- Requires careful argument normalization to avoid subtle differences.

Implementation Notes
- Prefer capability detection via hasattr over exception-driven probing for clarity and performance.
- Keep argument names consistent with Gradio 5.x; adapt internally for DummyGradio if parameters differ.
- Raise NotImplementedError with explicit guidance when a component cannot be resolved in the dummy environment.

Testing Strategy
- Unit tests for adapter mapping against:
  - Real Gradio 5.41.0 (happy path).
  - A minimal DummyGradio shim exposing names used by contract tests.
- Integration: Re-run [tests/test_ui_contract_gradio5.py](tests/test_ui_contract_gradio5.py) to verify parity.
- Ensure zero behavior change in production paths.

Security Considerations
- Adapter must not log API keys or sensitive data.
- Errors are surfaced as safe messages; stack traces not exposed to UI.
- No network or file I/O in adapter.

When to Use
- When UI framework abstractions vary across environments (prod vs tests/mocks).
- When you need to maintain stable app-facing APIs amid library evolution.

When Not to Use
- If tests already mirror the production surface.
- If a single environment is authoritative and alternatives are not required.

Related Decisions
- See: “UI Compatibility Adapter for Gradio 5.x and DummyGradio” in [memory-bank/decisionLog.md](memory-bank/decisionLog.md:7)

Status
- Approved and pending implementation in [app/gr_compat.py](app/gr_compat.py) with refactor of [app/ui.py](app/ui.py:1).

# System Patterns — Rapid-Test Chatbot UI

Version: 1.0
Owner: SPARC Orchestrator
Status: Active
Last Updated: 2025-08-06

## Index
- [Container Hardening Patterns](#container-hardening-patterns)
- [Healthcheck Patterns](#healthcheck-patterns)
- [Dependency Hygiene Patterns](#dependency-hygiene-patterns)
- [Validator and Error Sanitization Patterns](#validator-and-error-sanitization-patterns)
- [Testing Patterns for Streaming UI](#testing-patterns-for-streaming-ui)

## Container Hardening Patterns

### Minimal, Non-Root, Deterministic-UTF8 Runtime
Pattern:
- Base: python:3.11-slim
- User: non-root
- Environment:
  - PIP_NO_CACHE_DIR=1
  - PYTHONHASHSEED=random
  - PYTHONUTF8=1

Rationale:
- Reduce cache footprint and attack surface (pip cache off)
- Randomized hash seed reduces certain hash-collision exploit predictability
- UTF-8 default avoids locale-induced encoding issues

Implementation Reference:
- See [Dockerfile](Dockerfile) for canonical configuration

### Network Egress and Proxy Expectations
Pattern:
- Rely on upstream gateway/proxy to enforce:
  - Block IMDS/metadata and RFC1918/RFC4193 egress
  - Normalize Host/Forwarded headers
  - Apply basic rate limiting to POST / and streaming endpoints

Rationale:
- SSRF risk reduction at network edge
- Header canonicalization eliminates spoofing pivots
- Rate limit to reduce DoS impact

## Healthcheck Patterns

### Python Stdlib Socket Healthcheck (No Extra Packages)
Pattern:
- HEALTHCHECK uses python -c with socket to verify TCP port availability (7860)
- Keep dependency-less health verification to preserve slim image size

Reference:
- See HEALTHCHECK in [Dockerfile](Dockerfile)

Guidelines:
- Prefer simple TCP connect for liveness (fast failure)
- Use application-level /health endpoint readiness in orchestrator where supported

## Dependency Hygiene Patterns

### Security-Driven Pinning
Pattern:
- Pin security-sensitive libraries to versions with known fixes:
  - gradio: &gt;=5.31.0,&lt;6.0.0
  - requests: ==2.32.4

Rationale:
- Addresses Safety advisories (SSRF/XSS/DoS for gradio; CVE-2024-47081 for requests)

Workflow:
- Run Safety scans on requirements.txt
- Capture exceptions at account policy-level separately; ensure project file is clean
- Document decisions in [memory-bank/decisionLog.md](memory-bank/decisionLog.md)

## Validator and Error Sanitization Patterns

### Centralized User-Facing Error Messages
Pattern:
- Map transport/auth/network errors to safe, actionable messages
- Never include headers/body or secrets in messages or logs

Reference:
- Implemented in [app/validators.py](app/validators.py:36-81) sanitize_error_for_user

Guidelines:
- 401/403 → “Invalid key or permission” (no details)
- 4xx/5xx → “Model request failed”
- Network timeouts → “Network issue; try again”
- Always scrub potential secrets before rendering/logging

## Testing Patterns for Streaming UI

### Gradio 5.x Compatibility Guardrails
Pattern:
- Add/enable tests that validate:
  - Component initialization and event bindings in [app/ui.py](app/ui.py)
  - Entry-point wiring in [main.py](main.py)
  - Streaming generator behavior and chunk iteration
  - Negative paths: invalid key, timeout, server error → sanitized messages

Targets:
- [tests/test_validators.py](tests/test_validators.py)
- [tests/test_state.py](tests/test_state.py)
- [tests/test_openrouter_client.py](tests/test_openrouter_client.py)
- Add minimal UI contract tests to assert structure/wiring under Gradio 5.x

Rationale:
- Current suite shows 26 skipped; enable coverage to detect regressions introduced by major UI library upgrades

Quality Gates:
- R1: Tests must run and pass (no skips) for Gradio 5.x
- R3: Negative tests cover sanitization and SSRF/timeout surfaces