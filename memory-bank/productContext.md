# Product Context: Rapid-Test Chatbot UI

## Executive Summary
Rapid-Test is a lightweight, web-based chatbot application using Gradio to provide a fast, simple interface for testing and comparing Large Language Models (LLMs) through the OpenRouter API. It targets internal teams who need to evaluate model behavior quickly without complex tooling. The application supports streamed responses, per-session chat history, user-provided API keys (never stored server-side), and portable deployment via Docker.

## Business Objectives
- Enable rapid, low-friction experimentation with multiple LLMs via a consistent UI.
- Reduce time-to-evaluate for model comparisons (prompt parity across models).
- Ensure security by avoiding server-side storage of user-provided API keys.
- Provide responsive, streamed UX to mirror production conversational experiences.
- Make setup and deployment trivial via a single Docker container.

## Stakeholders
- Project Sponsor: Oversees delivery, ensures requirements alignment.
- ML/Research Engineers: Primary users conducting prompt/model evaluations.
- Developers: Extend or integrate additional models or features.
- DevOps: Containerize, deploy, and operate the application internally.
- Security: Validate no secrets are stored and outputs/logs are sanitized.

## In-Scope
- Gradio UI with chat history panel, input box, Send button.
- Model selection via dropdown (predefined allowlist).
- User-provided OpenRouter API key input within the UI.
- Streaming token-by-token responses in the chat UI.
- Per-session chat state (clears when session ends).
- Dockerfile for containerized deployment.

## Out-of-Scope (Initial Release)
- User authentication and multi-user accounts.
- Persistent server-side storage of chat history or keys.
- Advanced prompt tooling (templates, experiments, scoring).
- Analytics/telemetry beyond basic operational logs.
- Rate limiting/throttling policies (can be considered later).
- External databases or caches.

## Confirmed Product Decisions
- Default model: openrouter/horizon-beta
- Model allowlist (initial): 
  - meta-llama/Llama-3-8b-instruct
  - mistralai/mistral-7b-instruct
  - google/gemma-7b-it
  - openrouter/horizon-beta
- Max chat history: 25 messages per session (system-enforced cap).
- HTTP client: requests using iter_lines for streaming.
- App port: 7860.
- Docker base image: python:3.11-slim.
- No server-side storage or logging of API keys; keys confined to session memory only.

## Constraints & Assumptions
- Python 3.11 runtime for compatibility and slim image support.
- Gradio Blocks-based UI to facilitate component wiring and streaming generators.
- OpenRouter API accessible over HTTPS; environment requires outbound internet access.
- No PII collected; chat content is transient in memory for the session duration only.
- Logs must redact secrets and sensitive headers.

## Success Metrics
- Time-to-first-token < 2s p95 after sending a prompt (network permitting).
- End-to-end response latency < 8s p95 for typical prompts.
- 0 incidents of API key exposure in logs, memory dumps, or persisted storage.
- Setup to first successful streamed response in < 10 minutes for a new user.

## Competitive/Adjacent Solutions
- OpenAI Playground: Model-specific; not OpenRouter multi-model.
- Hugging Face Spaces demos: Often single model; not tailored for OpenRouter comparison.
- Custom notebooks: Flexible but slower for ad hoc non-technical users.
Rapid-Test differentiates by providing a single, minimal, multi-model streaming UI through OpenRouter with secure, user-supplied keys and simple deployment.

## Risks and Mitigations
- Risk: API key leakage via logs.
  - Mitigation: Never log headers, redact environment variables, structured error handling without echoing secrets.
- Risk: Stream handling inconsistencies.
  - Mitigation: Use requests with chunked streaming, robust parsing, graceful fallbacks to non-streaming if needed.
- Risk: Model errors or rate limits.
  - Mitigation: Clear user feedback in UI; recoverable error flows; configurable retries/backoff (cautious default).

## Future Enhancements (Backlog)
- Prompt presets and experiment tracking.
- Side-by-side model comparison view.
- Download/Export chat transcript (client-triggered).
- Optional non-secret server config (e.g., default models via env).
- Theming and UI customization.