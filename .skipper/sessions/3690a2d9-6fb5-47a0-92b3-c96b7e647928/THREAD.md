**Party Host, for @[user|sepehr seghatoleslami|google-oauth2|102066208526081522004]** (22:47 UTC):
Warden: SECOND EXAMPLE AGENT SYSTEM in the sandbox, so Warden is not shown only on the one demo it was built around.

Context: Warden lives in this party repo under projects/730af893059941e0a260658692bbb07a/warden/. main (a6dcc20) has packages/sandbox (customer-support triage/knowledge/account agents, versioned tool-boundary policy, fault injection, traces), packages/ai, packages/evals, packages/db and apps/server merged. apps/web is being built in session 99e6e707; do NOT edit apps/web. Spec: Issue #1 https://app.party.build/issues/1. Do NOT touch harmannb/warden.ai.

Goal: add a second, structurally different multi-agent system to packages/sandbox (e.g. clinic intake: intake -> scheduling -> records agents, or refunds: triage -> refunds -> fraud). It must have its own synthetic fixtures, at least one intentionally vulnerable baseline policy with a different failure class from cross-customer reads (e.g. unauthorized action/write, over-broad refund, or PII disclosure via handoff summary), a fixed policy, and Vitest regressions proving the baseline failure, the fixed denial and legitimate access. Keep the existing demo and its public API behaviour unchanged; reuse the existing catalog/policy/trace types rather than forking them.

Stay inside packages/sandbox. If evals or server need changes to select the new system, define the minimal interface on the sandbox side and list it under 'Assumed contracts' instead of editing those packages.

In spec and build reports include a Provisional decisions table (decision, rationale, unresolved question) and an Assumed contracts list. Cite files and ACs. Distinguish locally reported tests from observed CI. Never fabricate model exchanges or outcomes.

**@[agent|Agent 6233e45c C333 475d Ad22 D13b6a3b8941|agent-6233e45c-c333-475d-ad22-d13b6a3b8941]** (22:48 UTC):
Failed to authenticate. API Error: 403 {"code":"model_not_permitted","error":"forbidden","hint":"Model 'claude-opus-5-5' is not permitted for this session.","model":"claude-opus-5-5"}

Safeguard tripped. claude-opus-5-5 tried to switch to claude-opus-5-5, which isn't enabled for this session. The turn was stopped.

---- @[user|sepehr seghatoleslami|google-oauth2|102066208526081522004] pressed Retry (22:54 UTC) ----

**@[agent|Agent 6233e45c C333 475d Ad22 D13b6a3b8941|agent-6233e45c-c333-475d-ad22-d13b6a3b8941]** (22:54 UTC):
API Error: 400 The provided model identifier is invalid.

Error: API Error: 400 The provided model identifier is invalid.

---- @[user|sepehr seghatoleslami|google-oauth2|102066208526081522004] pressed Retry (22:55 UTC) ----

**@[agent|Agent 6233e45c C333 475d Ad22 D13b6a3b8941|agent-6233e45c-c333-475d-ad22-d13b6a3b8941]** (22:55 UTC):
_- Explored (explored) -_

I'll start by reading the session context files.

_- Explored (539s thinking · 19 tool calls) -_

No code or Issue work was lost in the two failed attempts; both were model-routing errors before any work started. I've now read the sandbox end to end plus how `apps/server` and `packages/evals` consume it, and I have a concrete proposal to discuss.

## The crux

`packages/sandbox` is hard-wired to one system. The runtime loop (`src/runtime/runConversation.ts`) is generic, but it reaches into support-specific constants in six places: `AGENTS`/`HANDOFF_TOOL` (`src/agents/agents.ts`), `DATA_TOOL_NAMES`/`dataToolDefinition` (`src/tools/definitions.ts`), the `Backend` and fixture lookups (`src/tools/backend.ts`), the `executeTool` switch (`src/tools/execute.ts`), `validatePolicy` with its fixed `POLICY_SCOPED_TOOLS` (`src/policy/policy.ts`), and the `AgentName`/`SandboxPolicy` types baked into every trace event (`src/runtime/trace.ts`). So the real deliverable is **a `SandboxSystem` seam with the Brewline support system as instance #1 and a new system as instance #2**, sharing one loop, one trace format, one policy validator pattern, one catalog shape. No forks; the existing exports keep their names and behaviour.

## Recommendation: Kestrel Travel refunds (triage → refunds → fraud)

Failure class: **unauthorized action**, specifically an over-broad refund and a skipped review gate, instead of a cross-tenant read.

| Aspect | Support (existing) | Refunds (new) |
|---|---|---|
| Topology | Hub-and-spoke: triage routes, specialists bounce | Pipeline with an escalation gate: triage → refunds → fraud |
| Vulnerable tool | `lookup_ticket` / `lookup_order` (reads) | `issue_refund` (a write with a money amount) |
| Baseline flaw | Trusts model-supplied `customer_id` | Executes any model-supplied `amount`, never requires review |
| Fix | Identity from trusted session | Amount bounded by the booking's refundable balance; above the review threshold the refund is denied and routed to fraud |
| Leak measure | `summary.foreignRecordsReturned` | `summary.unauthorizedActions` |
| Least privilege | All tools on `account` | Only `refunds` has `issue_refund`; only `fraud` has `review_refund` |

Identity is **session-scoped in both refunds versions** by construction, so the baseline-to-fix delta is purely the new class and the demo doesn't re-tell the cross-customer story.

### Behaviour

- **Agents:** `triage` (handoff only), `refunds` (`lookup_booking`, `issue_refund`, handoff), `fraud` (`lookup_booking`, `review_refund`, handoff). Prompts and tool descriptions are constants, byte-identical across versions; in this system even the tool schemas are identical, since the fix is enforcement only.
- **Policy** (reuses the `schemaVersion`/`label`/`tools` shape; the `system` field is the discriminator):

```json
{ "schemaVersion": 1, "system": "refunds", "label": "baseline",
  "tools": { "issue_refund": { "amountBound": "none", "reviewGate": "off" } } }
```

```json
{ "schemaVersion": 1, "system": "refunds", "label": "fixed-bounded-gated",
  "tools": { "issue_refund": { "amountBound": "refundable_balance", "reviewGate": "above_threshold" } } }
```

- **Decisions:** reuse `allowed`, `not_found`, `denied_cross_customer`, `denied_unauthenticated`, `invalid_input`, `not_evaluated`; add `denied_exceeds_refundable` and `denied_requires_review`.
- **Trace additions (additive):** `tool_result.actions[]` (`{ kind: 'refund' | 'review', id, bookingId, customerId, amount, authorized }`), `summary.actions`, `summary.unauthorizedActions`. `authorized` is policy-independent, like `foreignRecords`: a refund is unauthorized when it exceeds the refundable balance or the system's review threshold (`100.00`, a fixture constant).
- **Fixtures:** travellers `TV-4001..4003`, bookings `BK-7001..7006` covering: low fare (legit), fare above threshold, prior partial refund, already fully refunded, a foreign booking; prior refunds `RF-…`; review cases `RV-…` created only in-conversation. `uncitedIdentifiers` learns the new ID formats from the system definition.
- **Selection:** `runConversation`/`invokeTool` dispatch on `policy.system`; a policy without it follows today's exact support path. New exports: `SANDBOX_SYSTEMS`, `getSystem(id)`, `resolveSystem(policy)`, `REFUNDS_BASELINE_POLICY`, `REFUNDS_FIXED_POLICY`, `REFUNDS_POLICY_JSON_SCHEMA`, `refundsFixtures`, `REFUNDS_TOOL_CATALOG` (`catalogRef: sandbox.refunds.<tool>`), `REFUNDS_SYSTEM_ARCHITECTURE`. Gateway role is `sandbox.refunds.<agent>` (verified `@warden/ai` only requires a non-empty string, `packages/ai/src/gateway.ts:208`).

### Edge cases I want in the ACs

1. Over-refund: `issue_refund` for `500.00` on an `89.00` booking. Baseline executes (`unauthorizedActions = 1`); fixed → `denied_exceeds_refundable`, result discloses the refundable balance (own booking, safe).
2. Same-conversation double refund: two `issue_refund` calls for the same booking, in one turn (parallel `tool_use` blocks) or across turns. Needs a conversation-scoped action ledger; action tools apply in block order. Baseline executes both; fixed denies the second.
3. Already-refunded and partially-refunded bookings: refundable balance is `fare − prior refunds − this conversation's refunds`; equal-to-balance is allowed.
4. Review gate: amount above `100.00` but within balance → fixed denies with `denied_requires_review`; the scripted flow then hands off to `fraud`, whose `review_refund` records a `review` action, not money movement. Equal to the threshold does not require review.
5. Invalid amounts (`-5`, `0`, `12.345`, numbers instead of decimal strings) → `invalid_input` under both versions.
6. Foreign booking and unauthenticated session → denied under **both** versions, same non-disclosing payload as a miss.
7. `fraud` calling `issue_refund`, or `refunds` calling `review_refund` → `unknown_tool` (least privilege by agent).
8. Fault injection (`throw`/`timeout`/`malformed`/`empty`) on the new tools; `uncitedIdentifiers` flags `BK-`/`RF-`/`RV-` IDs and amounts the reply cites after a failed or denied call.
9. Fail-closed validation: unknown `system`, missing `issue_refund`, unknown knob values; a support policy carrying `system` is still rejected exactly as today.
10. All existing suites pass unchanged; `test/boundaries.test.ts` keeps covering the new source (no SDK, model ID, `process.env`).

Required Vitest regressions mirror the existing suites under `test/refunds/`: model-free policy tests via `invokeTool` (baseline failure, fixed denial, legitimate access, mirroring [Warden Platform](https://app.party.build/issues/1) AC8/AC10/AC16 and the [Agent Sandbox](https://app.party.build/issues/3) pattern), scripted-gateway conversation tests (baseline `unauthorizedActions = 1`, fixed denial plus fraud route, legit flow), prompt identity, fixtures, catalog/schema agreement, and a `systems` dispatch test. An opt-in live scenario is optional and would be reported skipped here. The build report must separate Vitest results reported locally from the `party/acceptance` check-run observed on the head SHA, and no model exchange will be claimed that didn't run.

## Provisional decisions

| Decision | Rationale | Unresolved question |
|---|---|---|
| Travel refunds, triage → refunds → fraud, "Kestrel Travel" | Pipeline + gate + write tool is structurally different in three ways; high-value refunds give fraud a real role | Happy with the theme and names? |
| Failure class: over-broad refund and review bypass, measured by `summary.unauthorizedActions` | Deterministic, executable, policy-independent invariant like `foreignRecordsReturned` | Count above-threshold-without-review as unauthorized under baseline too (I say yes)? |
| Identity session-scoped in both refunds versions, no `identitySource` knob | Isolates the new class; avoids re-demoing cross-customer | Lose remediation-proposer richness; acceptable? |
| Discriminator is `policy.system`; absent → support | Policies are the only artefact the server/db round-trip, so no server change is needed | Accept that support policies never carry `system`? |
| Generic `SandboxSystem` seam, support as instance #1 | One loop, no fork; `agent` fields widen to a union of both systems' names | Type widening is additive for evals (`string`) and server (opaque); confirm acceptable |
| Trace additions present on all traces (zero for support) | Uniform shape | Prefer refunds-only union instead? |
| Enum-only knobs; threshold `100.00` is a fixture constant | Keeps JSON Schema simple and the hand-rolled schema test valid | Threshold value? |
| Conversation-scoped action ledger; `invokeTool` accepts `priorActions` | Only way to stop same-conversation double refunds | OK that `invokeTool` takes optional prior state? |
| `sandbox.refunds.<agent>` roles and `sandbox.refunds.<tool>` refs | Avoids the `triage` name collision; recommender only matches refs in the catalog it is given (`packages/ai/src/advisors/recommender.ts:140`) | — |

## Assumed contracts

- `apps/server/src/ports/sandbox.ts` is single-system (`baselinePolicy()`, `executableCatalog()`, `demoArchitecture()`); selecting a system there needs a `systemId` on project/version and a call to `getSystem(id)`. Not edited.
- Server and `@warden/db` store policies as opaque JSON (`Policy = Record<string, unknown>`), so the `system` field round-trips. Assumed, not verified at runtime.
- `packages/evals/CONTRACTS.md` adapter mapping (tool_call/tool_result join, flat `Trace`) stays valid for refunds traces, but its `Fixtures` shape and assertions are support-specific; `unauthorizedActions`, `tool_result.actions[]` and the two new decisions are the hooks a future evals assertion set would use. Not edited.
- `@warden/ai` `remediation.propose()` accepts any throwing `validatePolicy` plus `policyJsonSchema`, so the refunds pair fits. Type-level assumption.
- `apps/web` untouched; `REFUNDS_SYSTEM_ARCHITECTURE` reuses the `agents[]`/`handoffs[]` shape so a future seed can render it.
- `projects/730af893059941e0a260658692bbb07a/.party/checks.json` already runs root install/typecheck/test/build; no new workspace, so no lockfile or checks change.

**Alternative considered:** clinic intake with PII disclosure via the handoff summary. Not recommended for v1: the fix would be free-text redaction at the handoff boundary, which is pattern-based, hard to prove complete, and gives evaluators nothing as crisp as a denied write. It would make a good third system later.

This turn produced no code or Issue changes; it is the spec proposal for discussion. If you're happy with the recommendation as written, Finalize Spec moves it into the Issue; otherwise tell me what to change.

Your input: **go with the travel-refunds system and the over-broad-refund/review-bypass failure class as laid out, or switch to the clinic PII-via-handoff variant (and any changes to the provisional decisions)?**

---- @[user|sepehr seghatoleslami|google-oauth2|102066208526081522004] pressed Retry (23:52 UTC) ----

**@[agent|Agent 6233e45c C333 475d Ad22 D13b6a3b8941|agent-6233e45c-c333-475d-ad22-d13b6a3b8941]** (23:52 UTC):
_- Explored (35s thinking · 4 tool calls) -_

The full spec proposal posted at 22:55 UTC is intact in the thread above; the Retry re-ran the turn without a reply from you, so nothing was lost and no code or Issue changed. I used this turn to turn three of my assumptions into verified facts rather than repeat the proposal.

## Verified since the proposal

- **Policies round-trip opaquely.** `@warden/db` stores a version's policy as JSON text with only a `json_type(policy) = 'object'` check (`packages/db/src/schema.ts:64`, `packages/db/src/versions.ts:103`), and the server hands `version.policy` straight to the run (`apps/server/src/runs/coordinator.ts:134`). A `system: "refunds"` field inside the policy document survives storage, so dispatching on `policy.system` needs no server or db change. This moves from "assumed" to verified.
- **Remediation proposer fits.** `remediation.propose()` accepts a throwing `validatePolicy` plus an optional `policyJsonSchema` (`packages/ai/src/advisors/remediation.ts`), so the refunds validator and schema pair plugs in unchanged. Verified at the type level.
- **Evals is support-only today.** `checkFixtures` hard-requires `customers`, `tickets`, `orders` and `articles` arrays (`packages/evals/src/run.ts:41`), confirming that evaluating the refunds system needs an evals-side change. It stays on the Assumed contracts list as a documented gap, not something this build touches.

## Defaults for the open questions

So Finalize Spec can proceed without another round, the Issue will take my recommendation on every unresolved question in the provisional-decisions table: travel refunds (triage → refunds → fraud, "Kestrel Travel"); over-broad refund plus review bypass as the failure class, both counted in `summary.unauthorizedActions`; identity session-scoped in both refunds versions; `policy.system` as the discriminator with its absence meaning the unchanged support path; a generic `SandboxSystem` seam with trace additions present on all traces; enum-only knobs with a `100.00` review threshold; a conversation-scoped action ledger with `invokeTool({ priorActions })`; and `sandbox.refunds.<agent>` / `sandbox.refunds.<tool>` labels. Each stays listed as provisional in the Issue so the Engineer and reviewers can see the alternative.

If you'd rather have the clinic PII-via-handoff variant or want any default changed, send a short message and I'll fold it in before building the Issue.

Next action: use **Finalize Spec** to have me build the Issue from this proposal.

---- @[user|sepehr seghatoleslami|google-oauth2|102066208526081522004] pressed Finalize Spec (00:00 UTC) ----

**@[agent|Agent 6233e45c C333 475d Ad22 D13b6a3b8941|agent-6233e45c-c333-475d-ad22-d13b6a3b8941]** (00:00 UTC):
_- Explored (1s thinking) -_

I'll pick up from the Finalize Spec press: read the session context files, then build the Issue from the posted proposal.

_- Explored (80s thinking · 10 tool calls) -_

I have what I need. Now I'll draft the Issue body and check whether the CLI offers any assignment control, since the create command showed none.

_- Explored (1s thinking · 1 tool call) -_

No assignee flag exists, so I'll record the session owner in the body as the earlier sandbox Issue did. Writing the spec now.

_- Explored (4s thinking · 2 tool calls) -_

The Issue is created as [Kestrel Refunds](https://app.party.build/issues/12). Let me confirm it is registered against this session's work contract and that the worktree is still clean.

_- Explored (13s thinking · 1 tool call) -_

The Issue is live and the work contract for Build Issue has no result slugs to report, so the spec step is complete.

## Issue built

[Kestrel Refunds](https://app.party.build/issues/12) is open as a feature Issue and follows the playbook's Summary / Behavior / Acceptance Criteria format, plus the Provisional decisions table and Assumed contracts list the request asked for. The CLI has no assignee control, so the body names sepehr seghatoleslami as session owner and requester, the same convention the earlier [Agent Sandbox](https://app.party.build/issues/3) Issue used.

What it specifies, in brief:

- **Scope:** a `SandboxSystem` seam inside `warden/packages/sandbox` only. The Brewline support system becomes instance #1 with every existing export unchanged. Kestrel Travel refunds (`triage → refunds → fraud`) is instance #2 on the same loop, trace format and policy pattern.
- **Failure class:** unauthorized action. The baseline executes any model-supplied refund amount and never requires review. The fixed policy bounds refunds by the booking's refundable balance and denies above-threshold refunds until the fraud agent reviews them. The measure is a new policy-independent `summary.unauthorizedActions`.
- **22 acceptance criteria** covering the seam and fail-closed dispatch on `policy.system`, the baseline over-refund, the fixed denial, legitimate access, the review gate with equal-to-threshold allowed, balance arithmetic, same-conversation double refunds via a per-conversation action ledger, session scoping under both versions, invalid amounts, least privilege by agent, three scripted conversations, trace additions, fixtures and uncited citations, catalog and architecture, boundaries, toolchain, docs, and an honest build report that separates local Vitest from the observed `party/acceptance` check-run.
- **Provisional decisions** record each default taken at Finalize Spec with its alternative, so the Engineer and reviewers can see what was decided and why.
- **Assumed contracts** list the server, evals, ai, web and checks touchpoints that are not edited, including the verified fact that policies round-trip through the database as opaque JSON.

No code changed this turn and the worktree is clean. No session overlap was found: the "Example Agent" session targets the separate warden.ai repository, and the demo sessions consume the sandbox rather than extend it.

Next action: use **Start Development** to hand the Issue to the Engineer.