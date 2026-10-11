# Brief — Build agent sandbox package with multi-agent support and versioned tool boundary policy

Warden component 4 of 6: AGENT SANDBOX (executable customer-support system under test).

Where to build: LOCALLY in this session's party repo under projects/730af893059941e0a260658692bbb07a/warden/packages/sandbox. Do NOT clone, push to, or open PRs against harmannb/warden.ai; we will move the code there later. Normal Flow PRs into the party repo are fine.

Context: Warden is a hackathon MVP for discovering, designing and testing multi-agent AI systems. Full spec: Issue #1 https://app.party.build/issues/1 (Build section, component 4). Parent session: https://app.party.build/session/31591058-0d2c-4bf4-8e5e-7729cdc3cadd.

Parallel work: 6 sessions build at the same time, each in its own directory under warden/. Stack: Node 24 + TypeScript, npm workspaces, React (Vite), Fastify, node:sqlite, @anthropic-ai/sdk (server only), Vitest. Ownership: Frontend = apps/web (session 99e6e707); Backend API = apps/server; AI Services = packages/ai (Claude gateway); Agent Sandbox = packages/sandbox; Evaluation Engine = packages/evals; Data & Infra = packages/db plus the warden/ root files (session cc75e5a4). Edit only your own directory. If you need something from another component (e.g. the packages/ai gateway), define a minimal interface on your side and record it under 'Assumed contracts'.

Scope: triage, knowledge and account agents making live model calls with real handoffs; synthetic tools (ticket lookup, order lookup, help-centre search, an injectable failing tool) over synthetic customers, tickets, orders and articles. Versioned tool-boundary policy: the baseline account tool trusts a model-supplied customer_id (one customer can read another's ticket); the fixed version takes identity from trusted session context and denies cross-customer reads. Output a full trace per conversation: turns, handoffs, tool calls and data actually returned. Covers AC5, AC8 (target side), AC10 (enforced policy). Vitest tests for the policy: baseline leaks; fix denies cross-customer and allows same-customer access. Never fabricate model output.

In spec and build reports include a Provisional decisions table (decision, rationale, unresolved question) and an Assumed contracts list. Cite files and ACs. Distinguish locally reported tests from observed CI.
