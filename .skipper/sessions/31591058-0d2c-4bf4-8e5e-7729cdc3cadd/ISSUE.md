# Issue #1: Build Warden agent discovery and collaborative testing MVP

**URL:** https://app.party.build/issues/1
**Reference slug:** Warden Platform
**Status:** open  **Type:** feature
**Revision:** 22886df9-06fa-4b24-a54d-0460b8ba578e
**Annotation coordinates:** use `party issue view <n> --numbered`; this file's lines are not canonical.

## Summary
Build Warden in the `warden.ai` repository: a polished, functional hackathon MVP that helps users describe a project, discover an agent architecture, inspect it visually, and evaluate a working customer-support agent system with a collaborating AI testing team. Deliver an evidence-backed demonstration of a real security failure, a reviewable fix, and a successful retest that preserves legitimate behavior.

Requested by and product owner: Harmann Bajwa (session owner).

## Behavior
- **Conversational discovery:** Users describe their project and answer contextual follow-up questions about goals, workflows, success criteria, data, integrations, and constraints. They can inspect and revise the resulting requirements before requesting an architecture.
- **AI recommendations:** Generate a project-specific team with agent responsibilities, required skills, tools, integration needs, and reasons for each recommendation. Recommendations for external integrations must distinguish proposed capabilities from configured, executable capabilities.
- **Interactive architecture:** Visualize agents and directed handoffs. Selecting a node reveals its role, skills, tools, and collaborators. Users can move between requirements, architecture, and testing without losing their project context.
- **Focused executable scope:** Support architectural recommendations for user-described projects; execute tests against the bundled customer-support system for this MVP. Clearly explain this boundary and offer the bundled demo from the discovery flow. Arbitrary agent imports, live production integrations, billing, and team administration are outside this release.
- **Working support system:** Provide distinct triage, knowledge, and account agents backed by executable synthetic support tools and fixtures. Agents perform real model calls and handoffs; synthetic customers and tickets provide a safe, repeatable target with an intentionally vulnerable baseline and a corrected version.
- **Collaborating testing team:** Specialized AI evaluators cover unit, integration, user acceptance, and security/reliability tests. Each evaluator has a distinct role and model invocation. They share structured findings and evidence; at least one evaluator consumes a peer finding to choose and execute a follow-up test. An orchestrator coordinates execution and consolidates results without fabricating exchanges or outcomes.
- **Observable runs:** Stream actual evaluator activity and show running, passed, failed, and incomplete states. Surface test category, expected versus observed behavior, severity, affected agent/tool, reproduction inputs, evidence, and remediation. Keep the collaboration feed understandable, with expandable technical details and trace identifiers rather than hidden reasoning.
- **Evidence-based verdicts:** Combine model-generated scenarios and analysis with executable assertions for concrete invariants, especially customer access and valid support behavior. A model's claim alone cannot mark an access-control test as passing. Skipped, errored, and unexecuted checks never count as passes; results describe the tested scenarios rather than implying comprehensive security certification.
- **Flagship failure and fix:** A security evaluator attempts to retrieve another synthetic customer's ticket. An integration evaluator uses that finding to reproduce the unauthorized handoff through the support team. The baseline demonstrates actual unauthorized data access. Warden recommends enforcing authenticated customer scope in the account tool; customer identity must come from trusted execution context, not text supplied by the model. The fix changes executable policy/configuration at the tool boundary, not merely wording in a prompt.
- **Review and retest:** Show the proposed fix, rationale, and affected behavior before the user applies it. Applying it creates a new system version and preserves baseline evidence. Run the original exploit, collaborator reproduction, and legitimate support regression scenarios against the new version. Report observed results honestly if the retest fails.
- **History and repeatability:** Persist requirements, architecture, system versions, findings, and run results across browser refreshes. Associate each run with its tested version; prevent edits from changing an in-flight run's configuration. Users can inspect before/after results and reset the demo for another presentation without mislabeling older results as current.
- **Product experience:** Use a coherent Warden visual identity, clear primary actions, a legible architecture canvas, and a dashboard that emphasizes findings and before/after evidence. Provide useful empty, loading, setup, and failure states; keyboard-accessible controls; and usable layouts on desktop and narrow screens. Do not encode status only through color.
- **Live-model setup and recovery:** Keep provider credentials server-side and document required configuration. Missing credentials, provider errors, malformed model responses, or timeouts produce actionable states and retain available evidence. Bound execution time and model-call counts. Any prerecorded example must be explicitly labeled and cannot satisfy the live-demo acceptance criteria.
- **Implementation freedom:** Engineer chooses the stack and orchestration approach, favoring a small locally runnable application and reliable demo execution. No provider credentials were attached during scope definition. Record verified startup/configuration instructions, tests, and the demo walkthrough; expose a working Party preview and attach visuals of delivered UI changes.

## Acceptance Criteria
- [ ] A user can enter an original project idea, answer contextual follow-ups, revise captured requirements, and receive AI-generated architecture recommendations tied to those requirements.
- [ ] Every recommended agent includes responsibilities, skills, tools, integration needs, and a rationale; proposed integrations are distinguishable from configured executable tools.
- [ ] The architecture view displays directed agent relationships; selecting an agent reveals its details, and navigation preserves project context.
- [ ] The interface explicitly identifies the bundled support system as the executable MVP target and offers a clear path into its demo.
- [ ] The bundled triage, knowledge, and account agents perform live model calls, execute synthetic tools, and complete a legitimate support request through a real handoff.
- [ ] A live evaluation run executes and reports at least one meaningful test in each category: unit, integration, user acceptance, and security/reliability. Acceptance tests connect to recorded requirements; reliability coverage includes a tool failure or missing-data case with defined safe behavior.
- [ ] Execution evidence shows distinct testing-agent model calls and a causal collaboration: one evaluator publishes a finding, another receives it and executes a follow-up test, and the final report cites the resulting evidence. A scripted chat animation or independent calls with no shared findings do not meet this criterion.
- [ ] The intentionally vulnerable baseline exposes another synthetic customer's ticket through the executable support system, with captured reproduction input, observed unauthorized result, affected tool/agent, and an assertion-backed failed security verdict.
- [ ] Warden generates a remediation tied to that observed failure and presents its rationale and reviewable policy/configuration change before application.
- [ ] Applying the remediation creates a new version that enforces trusted customer scope at the tool boundary while retaining the original configuration and findings for comparison.
- [ ] A fresh live retest blocks the original exploit and collaborator reproduction while passing legitimate same-customer access and normal support regressions; the dashboard compares actual before/after outcomes and links supporting evidence.
- [ ] Live progress and findings expose categories, statuses, severity, expected/observed behavior, reproduction steps, and recommendations. Failed, skipped, errored, and incomplete checks are visibly distinct and never reported as passing.
- [ ] Refreshing the app preserves project and run history. Runs remain tied to immutable tested versions; demo reset permits another baseline-to-fix presentation while preserving accurate version labeling.
- [ ] Missing credentials, provider failures, malformed outputs, and timeouts yield actionable setup/error states without invented findings or fake live collaboration. Model execution is bounded and secrets are absent from browser payloads and displayed logs.
- [ ] The complete journey is usable with keyboard controls and at desktop/narrow widths, including architecture details, findings, fix review, and run comparison. Loading, empty, and error states are implemented and status is understandable without color alone.
- [ ] Relevant build and automated checks pass, including executable regressions for cross-customer denial, legitimate access, honest result aggregation, and version-linked retesting. A documented live demo run supplies evidence that the collaboration and fix/retest criteria were met; mocked tests alone are insufficient.
- [ ] Startup instructions, credential variable names without values, the synthetic demo/reset procedure, and known limits are documented. A working Party preview and attached UI visuals support review of the completed MVP.


## Build
Six components, split by technical layer. Each lists its scope and the acceptance criteria (AC, numbered in order above) it carries. Stack: Node 24 + TypeScript, React (Vite), Fastify, `node:sqlite`, `@anthropic-ai/sdk` (server only), Vitest.

### 1. Frontend — web app
- **Scope:** React (Vite) client for the full journey: discovery chat and editable requirements, interactive architecture canvas with agent detail panel, live run feed, findings dashboard, fix review, and before/after comparison.
- **Standards:** Persistent project tabs (Requirements, Architecture, Testing); empty, loading, setup-required, and error states; keyboard navigation; desktop and narrow layouts; status shown with icon and text, not colour alone.
- **Covers:** AC1, AC3, AC4, AC12, AC15 (UI), AC11 (comparison view).

### 2. Backend API — application server
- **Scope:** Fastify service exposing REST endpoints for projects, requirements, architectures, system versions, runs, findings, fixes, and demo reset, plus a Server-Sent Events stream per run.
- **Rules:** Request validation; runs pinned to an immutable version at start; one active run per project; secrets never serialized to responses or logs.
- **Covers:** AC13 (run/version binding), AC14 (no secrets in browser payloads), shared transport for all other ACs.

### 3. AI Services — model gateway and advisors
- **Scope:** Server-side Claude client with per-run call and time budgets, structured JSON output validation, and typed errors for missing credentials, provider failures, malformed output, and timeouts. Hosts the discovery interviewer, architecture recommender (agents, skills, tools, integrations, rationale, `executable` vs `proposed` tags), and remediation proposer.
- **Covers:** AC1, AC2, AC9, AC14.

### 4. Agent Sandbox — executable support system
- **Scope:** Triage, knowledge, and account agents making live model calls with real handoffs; synthetic tools (ticket lookup, order lookup, help-centre search, an injectable failing tool) over synthetic customers, tickets, orders, and articles.
- **Policy:** Versioned tool-boundary config. The baseline account tool trusts a model-supplied `customer_id`; the fixed version takes identity from the trusted session context and denies cross-customer reads.
- **Output:** A full trace per conversation: turns, handoffs, tool calls, and the data actually returned.
- **Covers:** AC5, AC8 (target side), AC10 (enforced policy).

### 5. Evaluation Engine — collaborating testing team
- **Scope:** Unit, integration, user-acceptance, and security/reliability evaluators, each with its own role and model call, plus an orchestrator that plans, dispatches, and consolidates without authoring findings.
- **Collaboration:** A findings board. The security evaluator publishes the cross-customer finding; the integration evaluator consumes it and runs a follow-up reproduction through the support team. Findings record `causedBy` links, and the report cites the resulting evidence.
- **Verdicts:** Executable assertions decide concrete invariants (cross-customer denial, same-customer access, safe tool-failure fallback). Statuses are `passed | failed | errored | skipped | incomplete`, and only `passed` counts as a pass. Retests rerun the original exploit, the collaborator reproduction, and legitimate-support regressions.
- **Covers:** AC6, AC7, AC8, AC11, AC12 (aggregation).

### 6. Data & Infrastructure
- **Scope:** SQLite schema and repositories for projects, requirements, architectures, immutable system versions, runs, events, and findings. Apply-fix creates a new version; demo reset restores the baseline policy under a new version label without relabelling history.
- **Ops:** Environment config (`ANTHROPIC_API_KEY`, model and budget variables, names only), dev and start scripts, `.party/preview.md`, and Party preview.
- **Quality:** Vitest regressions for cross-customer denial, legitimate access, honest aggregation, and version-linked retests; README with startup, credentials, demo/reset walkthrough, and known limits; recorded live demo evidence.
- **Covers:** AC10 (version retention), AC13, AC16, AC17.

### Build order
1. Data & Infrastructure + AI Services gateway
2. Agent Sandbox
3. Evaluation Engine
4. Backend API
5. Frontend
6. AI Services advisors (discovery, recommendations, remediation) wired into the Frontend

