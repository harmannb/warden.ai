# Warden components

Warden is split into six components. Each one owns a slice of the
[Warden Platform](https://app.party.build/issues/1) acceptance criteria,
exposes a narrow interface to its neighbours, and can be built and tested on
its own.

```text
 Describe ──► Discover ──► Visualize ──► Run tests ──► Identify ──► Fix ──► Retest
   [1]          [2]           [2]          [3]+[4]       [5]         [6]     [4]+[5]
```

| # | Component | Owns | Acceptance criteria |
|---|-----------|------|---------------------|
| 1 | Discovery Studio | Conversational onboarding and the editable requirements record | AC1, AC4 (entry to demo) |
| 2 | Architecture Advisor | Agent/skill/tool recommendations and the interactive canvas | AC2, AC3, AC4 |
| 3 | Support Sandbox | The executable customer-support system under test | AC5, AC8 (target side), AC10 (policy) |
| 4 | Evaluation Team | Collaborating evaluators, orchestrator, assertions, run engine | AC6, AC7, AC8, AC12 (aggregation), AC14 |
| 5 | Evidence Dashboard | Live run feed, findings, before/after comparison | AC11, AC12, AC15 |
| 6 | Remediation Loop | Fix proposal, review/apply, versioning, retest, history, reset | AC9, AC10, AC11, AC13 |

AC16 (automated checks) and AC17 (docs, preview, visuals) apply to every
component.

## Shared core (plumbing, not a product component)

Small modules every component uses; built first, alongside component 3.

- **Model gateway** — server-side Claude client. Reads credentials from the
  environment only, enforces per-run call and time budgets, validates
  structured JSON output, and maps missing keys, provider errors, malformed
  output, and timeouts to typed errors (AC14).
- **Store** — SQLite (Node's built-in `node:sqlite`) for projects,
  requirements, architectures, system versions, runs, findings, and events
  (AC13).
- **Event stream** — Server-Sent Events channel per run; every event is
  persisted before it is emitted, so a refresh replays the same history.

## 1. Discovery Studio

Turns a free-text project idea into a structured, user-approved requirements
record.

- Chat that asks contextual follow-ups about goals, workflows, success
  criteria, data, integrations, and constraints.
- Requirements panel the user can edit before requesting an architecture.
- Banner explaining the executable boundary, with a **Try the support demo**
  path that loads the bundled project.
- **Outputs:** `Requirements { goals[], workflows[], successCriteria[], data[], integrations[], constraints[] }`
  consumed by components 2 and 4 (acceptance tests trace to these).

## 2. Architecture Advisor

Recommends an agent team for the requirements and renders it as a graph.

- Model-generated agents, each with responsibilities, skills, tools,
  integration needs, and a rationale that cites requirement IDs.
- Every tool and integration is tagged `executable` (configured in Warden)
  or `proposed` (recommendation only), shown with a text label as well as
  colour.
- Canvas with directed handoff edges; selecting a node (mouse or keyboard)
  opens a detail panel. Tabs keep project context across Requirements,
  Architecture, and Testing.
- **Outputs:** `Architecture { agents[], edges[] }` persisted per project.

## 3. Support Sandbox

The real, executable target system: three agents making live model calls
over synthetic data.

- **Triage agent** classifies a request and hands off; **Knowledge agent**
  answers from a synthetic help-centre corpus; **Account agent** reads
  tickets and orders through tools.
- Synthetic fixtures: customers, tickets, orders, articles. Includes a
  tool that can fail on demand for reliability tests.
- **Versioned policy config.** The baseline account tool trusts a
  `customer_id` argument supplied by the model (the intentional
  vulnerability). The fixed version takes customer identity from the
  trusted session context and denies cross-customer reads at the tool
  boundary.
- Every call returns a trace: agent turns, handoffs, tool calls, tool
  results, and data the caller actually received.
- **Interface:** `runConversation(versionId, session, messages) → Trace`.

## 4. Evaluation Team

Specialised AI testers that collaborate through a shared findings board.

- **Evaluators** (each a distinct prompt and model call): unit,
  integration, user-acceptance, and security/reliability.
- **Orchestrator** plans the run, dispatches evaluators, and consolidates
  the report. It never writes findings itself.
- **Findings board.** Evaluators publish structured findings; peers
  subscribe. The integration evaluator consumes the security finding and
  generates a follow-up multi-agent reproduction. Each finding records its
  `causedBy` parent so the collaboration chain is traceable.
- **Executable assertions** decide verdicts for concrete invariants:
  cross-customer data never reaches the caller, same-customer access
  succeeds, a failed tool yields a safe fallback. A model's opinion cannot
  pass an access-control check.
- Verdicts: `passed | failed | errored | skipped | incomplete`. Only
  `passed` counts as a pass.
- **Interface:** `startRun(projectId, versionId) → runId`; streams events
  through the shared core.

## 5. Evidence Dashboard

Shows what actually happened during a run.

- Live collaboration feed: evaluator messages, finding hand-offs, and
  expandable technical detail (inputs, tool calls, trace IDs).
- Results grouped by category, each showing status (icon and text),
  severity, affected agent/tool, expected vs observed, reproduction input,
  evidence, and recommendation.
- Before/after comparison of two runs on different versions, scenario by
  scenario, linked to evidence.
- Empty, loading, setup-required, and error states; responsive layout;
  full keyboard navigation.

## 6. Remediation Loop

Turns a failed finding into a reviewed, versioned fix and a fresh retest.

- **Fix proposal** generated from the observed failure: rationale, affected
  behaviour, and a diff of the policy/config change.
- **Apply** creates a new immutable system version; the baseline and its
  findings are preserved. Runs are pinned to the version they tested, so
  editing cannot change an in-flight run.
- **Retest** reruns the original exploit, the collaborator's reproduction,
  and legitimate-access and normal-support regression scenarios.
- **History and reset:** run history survives refresh; **Reset demo**
  returns the active system to the baseline as a new version label, so
  older results remain accurately labelled.

## Build order

Ordered by demo risk. The executable target and the real collaboration are
the hardest parts and everything else depends on them.

1. Shared core + **3 Support Sandbox**
2. **4 Evaluation Team**
3. **6 Remediation Loop**
4. **5 Evidence Dashboard**
5. **1 Discovery Studio**
6. **2 Architecture Advisor**

## Stack

- Node 24 + TypeScript in one repository: a Fastify API server and a React
  (Vite) client.
- `@anthropic-ai/sdk` on the server only.
- `node:sqlite` for persistence, so there is no native build step.
- Vitest for assertion, aggregation, and versioning regressions.
