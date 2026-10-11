# Brief — Add a second multi agent system to packages sandbox with synthetic fixtures and regression tests

Warden: SECOND EXAMPLE AGENT SYSTEM in the sandbox, so Warden is not shown only on the one demo it was built around.

Context: Warden lives in this party repo under projects/730af893059941e0a260658692bbb07a/warden/. main (a6dcc20) has packages/sandbox (customer-support triage/knowledge/account agents, versioned tool-boundary policy, fault injection, traces), packages/ai, packages/evals, packages/db and apps/server merged. apps/web is being built in session 99e6e707; do NOT edit apps/web. Spec: Issue #1 https://app.party.build/issues/1. Do NOT touch harmannb/warden.ai.

Goal: add a second, structurally different multi-agent system to packages/sandbox (e.g. clinic intake: intake -> scheduling -> records agents, or refunds: triage -> refunds -> fraud). It must have its own synthetic fixtures, at least one intentionally vulnerable baseline policy with a different failure class from cross-customer reads (e.g. unauthorized action/write, over-broad refund, or PII disclosure via handoff summary), a fixed policy, and Vitest regressions proving the baseline failure, the fixed denial and legitimate access. Keep the existing demo and its public API behaviour unchanged; reuse the existing catalog/policy/trace types rather than forking them.

Stay inside packages/sandbox. If evals or server need changes to select the new system, define the minimal interface on the sandbox side and list it under 'Assumed contracts' instead of editing those packages.

In spec and build reports include a Provisional decisions table (decision, rationale, unresolved question) and an Assumed contracts list. Cite files and ACs. Distinguish locally reported tests from observed CI. Never fabricate model exchanges or outcomes.
