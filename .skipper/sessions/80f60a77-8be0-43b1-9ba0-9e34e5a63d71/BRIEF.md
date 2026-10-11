# Brief — Run a live end-to-end demo script for Warden featuring real agent interactions and security evaluators

Warden: LIVE END-TO-END DEMO RUN with a real example agent system (answers Pranav's question in https://app.party.build/session/85839168-cf96-4cb7-a975-341f3d23c4f3).

Context: Warden lives in this party repo under projects/730af893059941e0a260658692bbb07a/warden/. main (a6dcc20) already has packages/db, packages/sandbox, packages/ai, packages/evals and apps/server merged (PRs #2-#6). apps/web is being built in session 99e6e707; do NOT edit apps/web. Spec: Issue #1 https://app.party.build/issues/1. Do NOT touch harmannb/warden.ai.

Goal: prove the full loop against the bundled customer-support demo with REAL model calls, not fakes: baseline run -> security evaluator finds the cross-customer leak -> integration evaluator reproduces via handoff (causedBy link) -> fix applied (fixed-session-scoped policy, new version) -> retest reruns the exploit, the reproduction and legitimate-support regressions -> before/after comparison. Drive it through apps/server's HTTP API + SSE stream so the server path is exercised too.

Prior evidence: the Agent Sandbox session (1914694a) already ran live sandbox calls successfully through the Party worker proxy (ANTHROPIC_BASE_URL + ANTHROPIC_AUTH_TOKEN) with the real @warden/ai gateway. Reuse that route; never print credentials.

Deliver: a script under warden/e2e/ (own package.json or npm script; opt-in via WARDEN_LIVE_TESTS so normal checks stay offline), a recorded run artifact (event log + findings + comparison JSON) attached to the session, and a short report of which ACs (AC6, AC7, AC8, AC10, AC11, AC12) were observed live. If anything in the merged packages breaks under live calls, file it precisely; fix only what is needed and say which package you touched.

In spec and build reports include a Provisional decisions table (decision, rationale, unresolved question). Cite files and ACs. Distinguish locally reported tests from observed CI. Never fabricate model exchanges or outcomes: a skipped or failed live step must be reported as such.
