# Issue #6: Build the Warden Evaluation Engine (packages/evals): collaborating AI testing team with assertion-backed verdicts

**URL:** https://app.party.build/issues/6
**Reference slug:** Evaluation Engine
**Status:** closed  **Type:** feature
**Revision:** edfa33a8-87a0-4a49-92fb-4b3b2550eac8
**Annotation coordinates:** use `party issue view <n> --numbered`; this file's lines are not canonical.

## Summary
Build `warden/packages/evals` (under `projects/730af893059941e0a260658692bbb07a/` in the party repo): a pure TypeScript library that runs Warden's collaborating AI testing team against the bundled customer-support system. Four evaluators (unit, integration, user-acceptance, security/reliability) each make their own model call through an injected AI gateway and share findings on a per-run board; an orchestrator plans, dispatches, and consolidates without ever authoring a finding. Executable assertions over sandbox traces — never model claims — decide cross-customer denial, same-customer access, and safe tool-failure fallback. Only `passed` counts. Retests replay the original exploit, the collaborator reproduction, and legitimate-support regressions. The engine emits a run-event stream the Backend API relays over SSE.

Carries parent [Warden Platform](https://app.party.build/issues/1) acceptance criteria AC6, AC7, AC8, AC11, and AC12 (aggregation). Dispatched by sepehr seghatoleslami (session owner); product owner Harmann Bajwa. Build locally only — do not clone, push to, or open PRs against `harmannb/warden.ai`; normal Flow PRs into the party repo are fine. Edit only `warden/packages/evals` plus this project's `.party/checks.json`.

## Behavior
- **Pure library boundary:** `packages/evals` has no HTTP, no database, and no direct `@anthropic-ai/sdk` import. The AI gateway and the sandbox are injected interfaces defined in `src/contracts/`, so the engine is fully testable with stubs on this branch.
- **Scenario catalog:** Stable `scenarioKey`s describe every test: `security.cross_customer_ticket_read`, `integration.unauthorized_handoff_repro`, `integration.legitimate_handoff`, `unit.triage_classifies_intent`, `unit.knowledge_answers_from_kb`, `unit.account_same_customer_lookup`, `acceptance.<requirementId>`, `reliability.tool_failure_fallback`. Each declares its category, `verdictSource: 'assertion' | 'model'`, and the assertions it needs. Access-control and tool-failure scenarios are assertion-only by type.
- **Four evaluators, four model calls:** Each evaluator has a distinct role prompt and makes its own gateway call. The model designs inputs, interprets traces, ranks severity, and recommends; it never sets the status of an assertion-backed scenario.
- **Findings board and causal collaboration:** A per-run board offers `publish`, `consume`, and bounded `waitFor`. The security evaluator publishes the cross-customer finding; the integration evaluator consumes it and runs a follow-up reproduction through the full support team. The follow-up carries `causedBy`, the source carries `consumedBy`, and the report cites the follow-up's own evidence.
- **Executable assertions decide invariants:** Pure functions over a sandbox `Trace` and fixtures — `crossCustomerDenied`, `sameCustomerAccessGranted` (positive: the customer's own ticket must appear), `safeToolFailureFallback` (no fabricated order data, graceful reply or escalation, conversation completes), `handoffOccurred(from, to)`. Leak detection inspects both tool results and the final reply for victim identifiers drawn from fixtures.
- **Evidence must resolve:** Every evidence entry points at a real trace entry (`traceId` plus path) with an excerpt. Unresolved refs cause the orchestrator to record the finding as `errored` (`reason: evidence_unresolved`), never to accept it.
- **Honest statuses and aggregation:** Finding status is `passed | failed | errored | skipped | incomplete`. Run status: any `failed` → `failed`; otherwise any `errored`, `skipped`, or `incomplete` → `incomplete`; otherwise `passed`. Orchestrator failure before any scenario runs → `errored`. Zero executed scenarios is never `passed`. The report's `scope` lists exactly what was tested and claims nothing beyond it.
- **Orchestrator never writes findings:** Only evaluators hold a publish handle; the orchestrator receives a read-only board view, enforced by type and proven by a test.
- **Budgets and failure handling:** Per-run call and time budgets come from the gateway. Budget exhaustion, provider error, timeout, or a model declining to craft an exploit mark in-flight scenarios `incomplete` and unstarted ones `skipped` with a reason (`budget_exhausted`, `dependency_unavailable`, `model_declined`, `no_requirements`), retain all findings so far, and still emit `run.finished`.
- **Model-claim mismatches are recorded:** When a model asserts a leak the trace does not show (or the reverse), the assertion result stands and the discrepancy is attached as `modelClaimMismatch` evidence.
- **Retests replay pinned inputs:** A `kind: 'retest'` run replays the original exploit and the collaborator reproduction using the exact `reproduction.inputs` saved on the prior findings, plus legitimate regressions (same-customer lookup, legitimate handoff, acceptance scenarios). Retest findings link `retestOf` to the prior finding. A retest that still leaks is reported `failed` with evidence.
- **Run-event stream:** An async iterable with monotonic `seq` emits `run.started`, `plan.created`, `evaluator.started`, `evaluator.model_call`, `evaluator.message`, `scenario.started`, `tool.call`, `assertion.result`, `finding.published`, `finding.consumed`, `scenario.finished`, `evaluator.finished`, `run.finished`. Every event carries `runId`, `ts`, optional `evaluator`, `traceId`, and `scenarioKey`.
- **Acceptance ties to requirements:** Acceptance scenarios carry `requirementIds[]` from the requirements supplied for the run. With no requirements supplied they are `skipped`, not invented.
- **Never fabricate:** No exchange, tool result, model output, or verdict is synthesized. Stubs exist only in tests and are never shipped as fallbacks.
- **Self-contained package and real checks:** `@warden/evals` ships its own `package.json`, lockfile, `tsconfig.json`, and `vitest.config.ts` so it installs, tests, and builds before the root workspace scaffold exists, and remains workspace-ready. This project's `.party/checks.json` runs the package's real install, test, and build.

## Acceptance Criteria
- [ ] `warden/packages/evals` installs, typechecks, tests, and builds on its own via `npm ci`, `npm test`, and `npm run build`, with no runtime dependency on `@anthropic-ai/sdk`, `node:sqlite`, or any HTTP framework. [annotation:A1]
- [ ] `runEvaluation({ runId, versionId, kind, retestOf?, requirements, gateway, sandbox })` returns a `RunReport` and exposes the event stream; the gateway and sandbox interfaces live in `src/contracts/` and are documented in `CONTRACTS.md` with the assumed shapes below. [annotation:A2]
- [ ] A baseline run plans and executes at least one scenario in each category — unit, integration, acceptance, security/reliability — including `reliability.tool_failure_fallback` with asserted safe behaviour; every finding and event carries a stable `scenarioKey`. (AC6) [annotation:A3]
- [ ] Each of the four evaluators makes its own gateway call under a distinct role; `evaluator.model_call` events carry `callId` and role; a test proves all four roles called the gateway during a full run. (AC7) [annotation:A4]
- [ ] The security evaluator publishes `security.cross_customer_ticket_read`; the integration evaluator consumes it from the board and executes `integration.unauthorized_handoff_repro` through the support team; the follow-up finding carries `causedBy`, the source carries `consumedBy`, and the report cites the follow-up's own evidence. (AC7) [annotation:A5]
- [ ] Cross-customer denial, positive same-customer access, and safe tool-failure fallback are decided by assertions in `src/assertions/` over the sandbox trace; those scenario kinds are assertion-only at the type level, and a model verdict cannot set their status. (AC8, AC11) [annotation:A6]
- [ ] Leak detection checks both tool results and the final reply for victim identifiers taken from sandbox fixtures; against the vulnerable baseline the security finding is `failed` with reproduction inputs, the observed unauthorized data, the affected agent and tool, and resolvable evidence refs. (AC8) [annotation:A7]
- [ ] Every evidence entry resolves to a real trace entry; an unresolved ref causes the finding to be recorded `errored` with `reason: evidence_unresolved`. [annotation:A8]
- [ ] Finding status is one of `passed | failed | errored | skipped | incomplete`; run status derives exactly as specified in Behavior; zero executed scenarios never yields `passed`; the report's `scope` lists the tested scenarios. (AC12) [annotation:A9]
- [ ] The orchestrator holds a read-only board view and cannot publish; a test proves no finding is authored by the orchestrator. [annotation:A10]
- [ ] Budget exhaustion, provider error, timeout, or model refusal mid-run marks in-flight scenarios `incomplete` and unstarted ones `skipped` with a reason, retains findings so far, and still emits `run.finished` with the derived run status. [annotation:A11]
- [ ] A `kind: 'retest'` run replays the original exploit and collaborator reproduction using the pinned `reproduction.inputs` from `retestOf`, runs the legitimate regressions, and links each retest finding to its prior finding via `retestOf`. (AC11) [annotation:A12]
- [ ] The event stream emits the listed event types in order with monotonic `seq`, is consumable as an async iterable, and includes `finding.published` and `finding.consumed` events that expose the causal link. (AC7, AC12) [annotation:A13]
- [ ] Acceptance scenarios carry `requirementIds[]` from the supplied requirements; with none supplied they are `skipped` with `reason: no_requirements`. (AC6) [annotation:A14]
- [ ] A model claim that contradicts an assertion result is recorded as `modelClaimMismatch` evidence and the assertion result stands. [annotation:A15]
- [ ] Vitest with stubbed gateway and sandbox covers: honest aggregation (mixed statuses never pass, zero scenarios never pass, skipped not counted), each assertion (leak via tool result, leak via reply only, denial, positive same-customer access, safe and unsafe fallback), board causality, orchestrator-never-authors, evidence-resolution rejection, budget-exhaustion marking, and retest planning. A live-model smoke test is gated behind `WARDEN_LIVE_TESTS=1` and excluded from CI. [annotation:A16]
- [ ] `projects/730af893059941e0a260658692bbb07a/.party/checks.json` runs this package's install, test, and build; the build report separates locally reported test results from observed CI, includes the Provisional decisions table and Assumed contracts, and states plainly that live end-to-end collaboration evidence awaits integration with the other components. [annotation:A17]
- [ ] The package `README.md` documents the scenario catalog, status semantics, contracts, and how the Backend should invoke the engine and relay its events; no secret values appear anywhere in code, docs, or reports. [annotation:A18]

## Provisional decisions
| Decision | Rationale | Unresolved question |
|---|---|---|
| Evals is a pure library: no DB, no HTTP, no `@anthropic-ai/sdk` import; gateway and sandbox are injected interfaces | Testable with stubs on this branch; the Backend owns persistence and transport | Confirm the Backend calls `runEvaluation()` in-process rather than over a queue |
| Verdict source is declared per scenario; access-control and tool-failure scenarios are assertion-only at the type level | Makes AC8 and AC11 unfakeable by construction | Should model-judged acceptance scenarios report `passed`, or only `failed`/`flagged`? Current choice: allow `passed` labelled `verdictSource: 'model'` |
| Evidence refs must resolve to real trace entries or the finding is `errored` | Enforces "never fabricate" mechanically | Exact trace path format depends on the Sandbox's `Trace` shape |
| Orchestrator holds a read-only board view; only evaluators publish | Spec requires the orchestrator never write findings; a test proves it | None |
| Retests replay pinned inputs rather than regenerating the exploit | A retest must test the same exploit for a valid before/after comparison | Add an optional fresh-variant attempt under a separate `scenarioKey`? Deferred |
| Emit `scenarioKey` on every finding and event | The Frontend composes the before/after comparison by `scenarioKey` | None |
| Event and finding shapes follow the Frontend's assumed contract | Avoids a three-way renegotiation at merge | SSE envelope wrapping (`id`, `event`, `data`) is the Backend's call |
| Self-contained `@warden/evals` with own `package.json`, lockfile, `tsconfig.json`, `vitest.config.ts`; devDeps `typescript` + `vitest`; no runtime deps | Must install and test before the root scaffold exists; workspace-ready afterwards | Package naming and `tsconfig.base.json` extends from Data & Infra |
| Live-model smoke test gated behind `WARDEN_LIVE_TESTS=1`, excluded from CI; CI uses stubs only | No credential is attached here; CI must not depend on a live provider | Who performs the documented live demo once the components meet |
| Add `--prefix warden/packages/evals` install/test/build commands to this project's shared `.party/checks.json` | Checks must be real on this branch | Six sessions edit the same file; Data & Infra should consolidate at merge |

## Assumed contracts
- **AI gateway (`packages/ai`):** `complete<T>({ role, system, messages, parse: (raw: unknown) => T, budgetKey }) → Promise<{ data: T, callId, model, usage }>`, throwing typed `SetupRequiredError | ProviderError | MalformedOutputError | TimeoutError | BudgetExhaustedError`. Evals supplies its own parsers, so it is agnostic to zod versus JSON Schema inside the gateway.
- **Sandbox (`packages/sandbox`):** `runConversation({ versionId, session: { customerId }, messages: [{ role: 'customer', content }], faults?: { failTool?: string } }) → Promise<Trace>` where `Trace = { traceId, versionId, turns: [{ agent, content }], handoffs: [{ from, to, reason }], toolCalls: [{ id, agent, tool, args, result, error? }], finalReply, status }`; `fixtures() → { customers, tickets, orders, articles }` so assertions know the victim ticket and leak indicators.
- **Backend API (`apps/server`):** invokes `runEvaluation(...)` in-process with the run's `runId`, pinned `versionId`, `kind: 'baseline' | 'retest'`, `retestOf` findings when retesting, and the project's recorded `requirements: [{ id, text }]`; consumes the event iterable for SSE; persists findings and events. One active run per project is enforced by the Backend.
- **Data & Infra (`packages/db`):** stores findings and events exactly as emitted; evals never reads or writes storage.
- **Frontend (`apps/web`):** consumes the event names above and the finding shape `{ id, scenarioKey, category, status, severity, title, expected, observed, affectedAgent, affectedTool, reproduction: { inputs, steps[] }, evidence: [{ type, ref, traceId, excerpt }], recommendation, causedBy?, consumedBy?, retestOf?, requirementIds[], verdictSource }`.

## Out of scope
- Gateway implementation (`packages/ai`), support agents and tools (`packages/sandbox`), persistence (`packages/db`), HTTP and SSE transport (`apps/server`), and UI (`apps/web`).
- Remediation proposal generation (AI Services); evals supplies the failed finding the proposer consumes.
- Pushing to or opening PRs against `harmannb/warden.ai`.


## Annotations

### Annotation A1 · current · open
**Range:** `24:7-24:217`

> `warden/packages/evals` installs, typechecks, tests, and builds on its own via `npm ci`, `npm test`, and `npm run build`, with no runtime dependency on `@anthropic-ai/sdk`, `node:sqlite`, or any HTTP framework.

#### A1.1 · implemented
agent-920fa686-ba54-405a-883d-e9153d30c261 · 2026-10-10T21:57:37.710622+00:00

`@warden/evals` has no runtime dependencies (devDependencies only, same ranges as the root). From the package folder `npm ci` (resolves the root lockfile, where the workspace is now recorded), `npm test` (typecheck + 93 Vitest tests) and `npm run build` (emits dist/) pass locally. test/boundary.test.ts fails if src/ imports the SDK, node:sqlite, an HTTP framework or any non-relative module.

**Evidence:**
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/warden/packages/evals/package.json`
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/warden/package-lock.json`
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/warden/packages/evals/test/boundary.test.ts`

### Annotation A2 · current · open
**Range:** `25:7-25:273`

> `runEvaluation({ runId, versionId, kind, retestOf?, requirements, gateway, sandbox })` returns a `RunReport` and exposes the event stream; the gateway and sandbox interfaces live in `src/contracts/` and are documented in `CONTRACTS.md` with the assumed shapes below.

#### A2.1 · implemented
agent-920fa686-ba54-405a-883d-e9153d30c261 · 2026-10-10T21:57:41.423108+00:00

`runEvaluation(input)` returns an awaitable `EvaluationRun` that resolves to the `RunReport` (never rejects) and exposes `.events` as an async iterable. Gateway and sandbox ports live in src/contracts/ and are documented in CONTRACTS.md. Deviation: the gateway port is `structured({ runId, role, system, messages, parse })` to match AI Services #8 instead of `complete({ budgetKey })`; `runId` is the budget key.

**Evidence:**
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/warden/packages/evals/src/run.ts`
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/warden/packages/evals/src/contracts/gateway.ts`
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/warden/packages/evals/src/contracts/sandbox.ts`
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/warden/packages/evals/CONTRACTS.md`

### Annotation A3 · current · open
**Range:** `26:7-26:273`

> A baseline run plans and executes at least one scenario in each category — unit, integration, acceptance, security/reliability — including `reliability.tool_failure_fallback` with asserted safe behaviour; every finding and event carries a stable `scenarioKey`. (AC6)

#### A3.1 · implemented
agent-920fa686-ba54-405a-883d-e9153d30c261 · 2026-10-10T21:57:47.823503+00:00

A baseline plans 7 catalog scenarios plus `acceptance.<id>` per requirement across unit, integration, acceptance, security and reliability. `reliability.tool_failure_fallback` forces lookup_order to fail and is decided by `safeToolFailureFallback`. Every finding carries `scenarioKey`; scenario-scoped events always carry it and model-call events carry `payload.scenarioKeys` (test/baseline.test.ts, test/events.test.ts).

**Evidence:**
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/warden/packages/evals/src/catalog.ts`
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/warden/packages/evals/src/orchestrator.ts`
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/warden/packages/evals/test/baseline.test.ts`

### Annotation A4 · current · open
**Range:** `27:7-27:212`

> Each of the four evaluators makes its own gateway call under a distinct role; `evaluator.model_call` events carry `callId` and role; a test proves all four roles called the gateway during a full run. (AC7)

#### A4.1 · implemented
agent-920fa686-ba54-405a-883d-e9153d30c261 · 2026-10-10T21:57:53.945323+00:00

Each evaluator calls `gateway.structured` under its own role label (`evals.unit`, `evals.integration`, `evals.acceptance`, `evals.security`) with its own role prompt; `evaluator.model_call` carries `callId`, `role`, `purpose`, `model` and `usage`. test/baseline.test.ts asserts all four roles called the gateway with distinct callIds.

**Evidence:**
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/warden/packages/evals/src/evaluators/engine.ts`
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/warden/packages/evals/src/evaluators/prompts.ts`
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/warden/packages/evals/test/baseline.test.ts`

### Annotation A5 · current · open
**Range:** `28:7-28:343`

> The security evaluator publishes `security.cross_customer_ticket_read`; the integration evaluator consumes it from the board and executes `integration.unauthorized_handoff_repro` through the support team; the follow-up finding carries `causedBy`, the source carries `consumedBy`, and the report cites the follow-up's own evidence. (AC7)

#### A5.1 · implemented
agent-920fa686-ba54-405a-883d-e9153d30c261 · 2026-10-10T21:57:58.804483+00:00

Security publishes `security.cross_customer_ticket_read`; integration waits on the board (bounded), consumes it (`consumedBy: ['integration']`, `finding.consumed` event), designs a triage→account reproduction with the same session and target, and publishes `integration.unauthorized_handoff_repro` with `causedBy`. `report.collaboration[].followUpEvidence` is the follow-up's own trace evidence (test/baseline.test.ts, test/board.test.ts).

**Evidence:**
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/warden/packages/evals/src/evaluators/team.ts`
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/warden/packages/evals/src/board.ts`
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/warden/packages/evals/src/orchestrator.ts`

### Annotation A6 · current · open
**Range:** `29:7-29:278`

> Cross-customer denial, positive same-customer access, and safe tool-failure fallback are decided by assertions in `src/assertions/` over the sandbox trace; those scenario kinds are assertion-only at the type level, and a model verdict cannot set their status. (AC8, AC11)

#### A6.1 · implemented
agent-920fa686-ba54-405a-883d-e9153d30c261 · 2026-10-10T21:58:02.502728+00:00

`crossCustomerDenied`, `sameCustomerAccessGranted` and `safeToolFailureFallback` in src/assertions/ decide those scenarios. The catalog `satisfies` a mapped type that forbids `verdictSource: 'model'` on assertion-only keys, and `decideStatus` has no overload that takes a model verdict for them. test/types.test.ts holds `@ts-expect-error` proofs checked by tsc; a forged catalog entry was confirmed to fail compilation under TypeScript 7.

**Evidence:**
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/warden/packages/evals/src/assertions/index.ts`
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/warden/packages/evals/src/catalog.ts`
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/warden/packages/evals/src/status.ts`
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/warden/packages/evals/test/types.test.ts`

### Annotation A7 · current · open
**Range:** `30:7-30:306`

> Leak detection checks both tool results and the final reply for victim identifiers taken from sandbox fixtures; against the vulnerable baseline the security finding is `failed` with reproduction inputs, the observed unauthorized data, the affected agent and tool, and resolvable evidence refs. (AC8)

#### A7.1 · implemented
agent-920fa686-ba54-405a-883d-e9153d30c261 · 2026-10-10T21:58:06.519742+00:00

`findLeaks` matches victim fixture identifiers (name, email, ticket ids and subjects, order ids and items) in tool results and the final reply, plus records returned with another customer's `customerId`; typed-in, own and public help-centre values are excluded. Against the vulnerable test double the security finding is `failed` with `reproduction.inputs`, `observedData`, `affectedAgent: account`, `affectedTool: lookup_ticket` and resolvable evidence (test/assertions.test.ts, test/baseline.test.ts).

**Evidence:**
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/warden/packages/evals/src/assertions/identifiers.ts`
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/warden/packages/evals/test/assertions.test.ts`
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/warden/packages/evals/test/baseline.test.ts`

### Annotation A8 · current · open
**Range:** `31:7-31:157`

> Every evidence entry resolves to a real trace entry; an unresolved ref causes the finding to be recorded `errored` with `reason: evidence_unresolved`.

#### A8.1 · implemented
agent-920fa686-ba54-405a-883d-e9153d30c261 · 2026-10-10T21:58:09.824925+00:00

The board's publish gate checks every evidence entry (recorded trace, existing path, excerpt present) and every traceId. An unresolved entry stores the finding `errored` with `reason: evidence_unresolved`, keeping `proposedStatus` and `unresolvedEvidence`; a verdict with no evidence is `errored` with `evidence_missing` (test/board.test.ts, test/model-claims.test.ts).

**Evidence:**
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/warden/packages/evals/src/board.ts`
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/warden/packages/evals/src/evidence.ts`
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/warden/packages/evals/test/board.test.ts`

### Annotation A9 · current · open
**Range:** `32:7-32:239`

> Finding status is one of `passed | failed | errored | skipped | incomplete`; run status derives exactly as specified in Behavior; zero executed scenarios never yields `passed`; the report's `scope` lists the tested scenarios. (AC12)

#### A9.1 · implemented
agent-920fa686-ba54-405a-883d-e9153d30c261 · 2026-10-10T21:58:13.386834+00:00

`deriveRunStatus`: any failed → failed; else any errored, skipped or incomplete → incomplete; else passed. No findings, or nothing executed, is never passed; an orchestrator failure before any scenario → errored. `report.scope.tested` lists exactly the executed scenarios and `notTested` the rest with reasons (test/aggregation.test.ts).

**Evidence:**
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/warden/packages/evals/src/status.ts`
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/warden/packages/evals/src/orchestrator.ts`
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/warden/packages/evals/test/aggregation.test.ts`

### Annotation A10 · current · open
**Range:** `33:7-33:130`

> The orchestrator holds a read-only board view and cannot publish; a test proves no finding is authored by the orchestrator.

#### A10.1 · implemented
agent-920fa686-ba54-405a-883d-e9153d30c261 · 2026-10-10T21:58:16.083952+00:00

The orchestrator is constructed with `board.view()`, a frozen object exposing only `list` and `get`. Evaluator handles are created in run.ts and the orchestrator only receives `run()` closures. Tests prove the view has no publish or consume and that every finding is authored by an evaluator (test/board.test.ts, test/baseline.test.ts, test/types.test.ts).

**Evidence:**
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/warden/packages/evals/src/orchestrator.ts`
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/warden/packages/evals/src/board.ts`
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/warden/packages/evals/src/run.ts`

### Annotation A11 · current · open
**Range:** `34:7-34:243`

> Budget exhaustion, provider error, timeout, or model refusal mid-run marks in-flight scenarios `incomplete` and unstarted ones `skipped` with a reason, retains findings so far, and still emits `run.finished` with the derived run status.

#### A11.1 · implemented
agent-920fa686-ba54-405a-883d-e9153d30c261 · 2026-10-10T21:58:18.959507+00:00

A failure stops the current phase: running scenarios become `incomplete` and unstarted ones `skipped`, both with the reason (budget_exhausted, timeout, provider_error, model_declined). Budget, setup and abort stop the whole run. Findings so far are kept and `run.finished` is always emitted. A design-call failure happens before `scenario.started`, so those scenarios are `skipped`. test/failures.test.ts covers each kind.

**Evidence:**
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/warden/packages/evals/src/evaluators/engine.ts`
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/warden/packages/evals/test/failures.test.ts`

### Annotation A12 · current · open
**Range:** `35:7-35:251`

> A `kind: 'retest'` run replays the original exploit and collaborator reproduction using the pinned `reproduction.inputs` from `retestOf`, runs the legitimate regressions, and links each retest finding to its prior finding via `retestOf`. (AC11)

#### A12.1 · implemented
agent-920fa686-ba54-405a-883d-e9153d30c261 · 2026-10-10T21:58:22.166911+00:00

`planRetest` replays the exploit and collaborator reproduction from the prior findings' exact `reproduction.inputs` (skipped with `no_prior_reproduction` without them) and adds same-customer lookup, legitimate handoff and acceptance regressions; each finding carries `retestOf`. Against the fixed double both replays pass, the sandbox receives the pinned messages and no design calls are made; a fix that still leaks is `failed` (test/retest.test.ts).

**Evidence:**
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/warden/packages/evals/src/retest.ts`
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/warden/packages/evals/src/orchestrator.ts`
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/warden/packages/evals/test/retest.test.ts`

### Annotation A13 · current · open
**Range:** `36:7-36:223`

> The event stream emits the listed event types in order with monotonic `seq`, is consumable as an async iterable, and includes `finding.published` and `finding.consumed` events that expose the causal link. (AC7, AC12)

#### A13.1 · implemented
agent-920fa686-ba54-405a-883d-e9153d30c261 · 2026-10-10T21:58:24.950602+00:00

`RunEventStream` assigns monotonic, gap-free `seq`, replays from the first event for every iterator, and closes after `run.finished`. test/events.test.ts checks every listed type, first-appearance order, per-scenario lifecycle order, late-subscriber replay, and that `finding.consumed` precedes the follow-up's `finding.published` with `causedBy`.

**Evidence:**
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/warden/packages/evals/src/events.ts`
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/warden/packages/evals/test/events.test.ts`

### Annotation A14 · current · open
**Range:** `37:7-37:160`

> Acceptance scenarios carry `requirementIds[]` from the supplied requirements; with none supplied they are `skipped` with `reason: no_requirements`. (AC6)

#### A14.1 · implemented
agent-920fa686-ba54-405a-883d-e9153d30c261 · 2026-10-10T21:58:28.059128+00:00

Acceptance scenarios are generated per supplied requirement as `acceptance.<id>` with `requirementIds: [id]`. With none, a single `acceptance.requirements` finding is `skipped` with `reason: no_requirements` and no acceptance model call is made (test/baseline.test.ts).

**Evidence:**
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/warden/packages/evals/src/catalog.ts`
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/warden/packages/evals/src/orchestrator.ts`
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/warden/packages/evals/test/baseline.test.ts`

### Annotation A15 · current · open
**Range:** `38:7-38:135`

> A model claim that contradicts an assertion result is recorded as `modelClaimMismatch` evidence and the assertion result stands.

#### A15.1 · implemented
agent-920fa686-ba54-405a-883d-e9153d30c261 · 2026-10-10T21:58:31.884664+00:00

For assertion scenarios the review claim is compared with the decided status; a contradiction adds `modelClaimMismatch` evidence that points at the deciding trace entry with the review `callId`, and the assertion result stands. Model citations are not admitted as evidence on assertion scenarios, so they cannot change those statuses (test/model-claims.test.ts).

**Evidence:**
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/warden/packages/evals/src/evaluators/engine.ts`
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/warden/packages/evals/test/model-claims.test.ts`

### Annotation A16 · current · open
**Range:** `39:7-39:483`

> Vitest with stubbed gateway and sandbox covers: honest aggregation (mixed statuses never pass, zero scenarios never pass, skipped not counted), each assertion (leak via tool result, leak via reply only, denial, positive same-customer access, safe and unsafe fallback), board causality, orchestrator-never-authors, evidence-resolution rejection, budget-exhaustion marking, and retest planning. A live-model smoke test is gated behind `WARDEN_LIVE_TESTS=1` and excluded from CI.

#### A16.1 · implemented
agent-920fa686-ba54-405a-883d-e9153d30c261 · 2026-10-10T21:58:35.622090+00:00

Ten suites (93 tests) using the scripted gateway and rule-based sandbox doubles under test/fakes/ cover the listed cases. test/live/smoke.test.ts runs only with WARDEN_LIVE_TESTS=1 and an adapter module and is excluded by vitest.config.ts otherwise; without an adapter it fails rather than passing.

**Evidence:**
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/warden/packages/evals/vitest.config.ts`
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/warden/packages/evals/test/live/smoke.test.ts`
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/warden/packages/evals/test/fakes/gateway.ts`

### Annotation A17 · current · resolved
**Range:** `40:7-40:363`

> `projects/730af893059941e0a260658692bbb07a/.party/checks.json` runs this package's install, test, and build; the build report separates locally reported test results from observed CI, includes the Provisional decisions table and Assumed contracts, and states plainly that live end-to-end collaboration evidence awaits integration with the other components.

#### A17.1 · partial
agent-920fa686-ba54-405a-883d-e9153d30c261 · 2026-10-10T21:58:40.580775+00:00

Data & Infra's consolidated checks.json (merged on main) already installs, typechecks and tests every workspace; this branch appends `npm --prefix warden run build --workspace @warden/evals`. All four commands pass locally from a clean install (130 tests across db and evals). CI is not observed yet because no PR exists. This turn's build report separates the two and includes the Provisional decisions table, Assumed contracts and the live-evidence caveat.

**Evidence:**
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/.party/checks.json`

#### A17.2 · partial
agent-920fa686-ba54-405a-883d-e9153d30c261 · 2026-10-10T22:12:04.080011+00:00

Correction after rebasing onto main (b9a5893): this branch no longer edits checks.json. Main's shared checks now end with a root-level `npm --prefix warden run build`, which builds @warden/evals along with every other workspace, so my workspace-scoped command was dropped as redundant. All four shared commands pass locally from a clean install against main with db, sandbox, ai and server (474 tests passed, 3 skipped). The PR is open; the repository has no GitHub Actions workflows, so the Party project checks are the branch build, and their run on the PR has not been observed yet.

**Evidence:**
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/.party/checks.json`

#### A17.3 · verified
agent-6233e45c-c333-475d-ad22-d13b6a3b8941 · 2026-10-10T22:20:48.353787+00:00

Product wrap-up: the observed check is now available. GitHub check-run party/acceptance on PR #6 head b9a5893 completed with conclusion success, and the PR merged to main as a6dcc20 (22:14 UTC). Main's shared checks.json (install, typecheck, test, build over all workspaces) covers @warden/evals without a package-specific edit. Independently re-ran npm test in warden/packages/evals from the merged tree: typecheck passed, 11 files / 105 tests passed (locally reported).

### Annotation A18 · current · open
**Range:** `41:7-41:218`

> The package `README.md` documents the scenario catalog, status semantics, contracts, and how the Backend should invoke the engine and relay its events; no secret values appear anywhere in code, docs, or reports.

#### A18.1 · implemented
agent-920fa686-ba54-405a-883d-e9153d30c261 · 2026-10-10T21:58:45.680198+00:00

README.md documents the scenario catalog, assertions, status semantics, evidence gate, collaboration, budgets, retests, the event stream, and a Backend invocation and relay example. CONTRACTS.md documents the gateway, sandbox (with a ConversationTrace adapter mapping), Backend, db and Frontend contracts. Only variable names appear; no secret values.

**Evidence:**
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/warden/packages/evals/README.md`
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/warden/packages/evals/CONTRACTS.md`

Reply with `party issue annotation reply <n> <A1> ...`; resolve with `party issue annotation resolve <n> <A1>`.

## Comments

### Agent 920fa686 Ba54 405a 883d E9153d30c261 (2026-10-10T22:12:08.667324+00:00)

PR opened for the Evaluation Engine: [PR #6](https://github.com/Party-Internal/party-962bc9f6a03f4affbff6eef3aa742a6e/pull/6). Before opening it I rebased onto main twice (Agent Sandbox, AI Services and Backend API merged during the session), so the commits are now a0372ea (engine) and b9a5893 (alignment). The alignment commit makes the engine accept the merged sandbox's order shape (items[].name, total.amount, trackingNumber), so its exported fixtures pass straight through to leak and fabrication checks. It also classifies the sandbox's missing_credentials code, and CONTRACTS.md now documents the real ConversationTrace field mapping. createGateway() and the sandbox fixtures were both checked with tsc to fit the engine's ports unchanged, and the Backend's merged EvalsPort.startRun() is compatible through a thin adapter. Locally reported: shared checks pass on main plus this branch (474 passed, 3 skipped; evals 105).

