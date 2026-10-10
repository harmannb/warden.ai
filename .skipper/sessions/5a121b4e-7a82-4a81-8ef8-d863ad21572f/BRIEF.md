# Brief — Build the evaluation engine package with collaborating AI testing team and assertion-backed verdicts

Warden component 5 of 6: EVALUATION ENGINE (collaborating AI testing team).

Where to build: LOCALLY in this session's party repo under projects/730af893059941e0a260658692bbb07a/warden/packages/evals. Do NOT clone, push to, or open PRs against harmannb/warden.ai; we will move the code there later. Normal Flow PRs into the party repo are fine.

Context: Warden is a hackathon MVP for discovering, designing and testing multi-agent AI systems. Full spec: Issue #1 https://app.party.build/issues/1 (Build section, component 5). Parent session: https://app.party.build/session/31591058-0d2c-4bf4-8e5e-7729cdc3cadd.

Parallel work: 6 sessions build at the same time, each in its own directory under warden/. Stack: Node 24 + TypeScript, npm workspaces, React (Vite), Fastify, node:sqlite, @anthropic-ai/sdk (server only), Vitest. Ownership: Frontend = apps/web (session 99e6e707); Backend API = apps/server; AI Services = packages/ai (Claude gateway); Agent Sandbox = packages/sandbox (support agents + versioned tool policy + traces); Evaluation Engine = packages/evals; Data & Infra = packages/db plus the warden/ root files (session cc75e5a4). Edit only your own directory. If you need something from another component, define a minimal interface on your side and record it under 'Assumed contracts'.

Scope: unit, integration, user-acceptance and security/reliability evaluators, each with its own role and model call, plus an orchestrator that plans, dispatches and consolidates but never writes findings. Findings board: the security evaluator publishes the cross-customer finding; the integration evaluator consumes it and runs a follow-up reproduction through the support team. Findings carry causedBy links; the report cites resulting evidence. Executable assertions, not model claims, decide cross-customer denial, same-customer access and safe tool-failure fallback. Statuses passed | failed | errored | skipped | incomplete; only passed counts. Retests rerun the original exploit, collaborator reproduction and legitimate-support regressions. Emit a run-event stream the API can relay. Covers AC6, AC7, AC8, AC11, AC12 (aggregation). Vitest tests for honest aggregation and assertion-backed verdicts. Never fabricate exchanges or outcomes.

In spec and build reports include a Provisional decisions table (decision, rationale, unresolved question) and an Assumed contracts list. Cite files and ACs. Distinguish locally reported tests from observed CI.
