# Warden components

Mirrors the **Build** section of [Warden Platform](https://app.party.build/issues/1). AC numbers refer to that issue's acceptance criteria, in order.

## 1. Frontend — web app
- **Scope:** React (Vite) client for the full journey: discovery chat and editable requirements, interactive architecture canvas with agent detail panel, live run feed, findings dashboard, fix review, and before/after comparison.
- **Standards:** Persistent project tabs (Requirements, Architecture, Testing); empty, loading, setup-required, and error states; keyboard navigation; desktop and narrow layouts; status shown with icon and text, not colour alone.
- **Covers:** AC1, AC3, AC4, AC12, AC15 (UI), AC11 (comparison view).

## 2. Backend API — application server
- **Scope:** Fastify service exposing REST endpoints for projects, requirements, architectures, system versions, runs, findings, fixes, and demo reset, plus a Server-Sent Events stream per run.
- **Rules:** Request validation; runs pinned to an immutable version at start; one active run per project; secrets never serialized to responses or logs.
- **Covers:** AC13 (run/version binding), AC14 (no secrets in browser payloads), shared transport for all other ACs.

## 3. AI Services — model gateway and advisors
- **Scope:** Server-side Claude client with per-run call and time budgets, structured JSON output validation, and typed errors for missing credentials, provider failures, malformed output, and timeouts. Hosts the discovery interviewer, architecture recommender (agents, skills, tools, integrations, rationale, `executable` vs `proposed` tags), and remediation proposer.
- **Covers:** AC1, AC2, AC9, AC14.

## 4. Agent Sandbox — executable support system
- **Scope:** Triage, knowledge, and account agents making live model calls with real handoffs; synthetic tools (ticket lookup, order lookup, help-centre search, an injectable failing tool) over synthetic customers, tickets, orders, and articles.
- **Policy:** Versioned tool-boundary config. The baseline account tool trusts a model-supplied `customer_id`; the fixed version takes identity from the trusted session context and denies cross-customer reads.
- **Output:** A full trace per conversation: turns, handoffs, tool calls, and the data actually returned.
- **Covers:** AC5, AC8 (target side), AC10 (enforced policy).

## 5. Evaluation Engine — collaborating testing team
- **Scope:** Unit, integration, user-acceptance, and security/reliability evaluators, each with its own role and model call, plus an orchestrator that plans, dispatches, and consolidates without authoring findings.
- **Collaboration:** A findings board. The security evaluator publishes the cross-customer finding; the integration evaluator consumes it and runs a follow-up reproduction through the support team. Findings record `causedBy` links, and the report cites the resulting evidence.
- **Verdicts:** Executable assertions decide concrete invariants (cross-customer denial, same-customer access, safe tool-failure fallback). Statuses are `passed | failed | errored | skipped | incomplete`, and only `passed` counts as a pass. Retests rerun the original exploit, the collaborator reproduction, and legitimate-support regressions.
- **Covers:** AC6, AC7, AC8, AC11, AC12 (aggregation).

## 6. Data & Infrastructure
- **Scope:** SQLite schema and repositories for projects, requirements, architectures, immutable system versions, runs, events, and findings. Apply-fix creates a new version; demo reset restores the baseline policy under a new version label without relabelling history.
- **Ops:** Environment config (`ANTHROPIC_API_KEY`, model and budget variables, names only), dev and start scripts, `.party/preview.md`, and Party preview.
- **Quality:** Vitest regressions for cross-customer denial, legitimate access, honest aggregation, and version-linked retests; README with startup, credentials, demo/reset walkthrough, and known limits; recorded live demo evidence.
- **Covers:** AC10 (version retention), AC13, AC16, AC17.

## Build order
1. Data & Infrastructure + AI Services gateway
2. Agent Sandbox
3. Evaluation Engine
4. Backend API
5. Frontend
6. AI Services advisors (discovery, recommendations, remediation) wired into the Frontend
