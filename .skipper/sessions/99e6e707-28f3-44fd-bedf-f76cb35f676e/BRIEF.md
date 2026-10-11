# Brief — Build the Warden frontend React web app with discovery chat, canvas, and dashboard

Warden component 1 of 6: FRONTEND (React web app).

Where to build: LOCALLY in this session's party repo under projects/730af893059941e0a260658692bbb07a/warden/apps/web. Do NOT clone, push to, or open PRs against harmannb/warden.ai; we will move the code there later. Normal Flow PRs into the party repo are fine.

Context: Warden is a hackathon MVP for discovering, designing and testing multi-agent AI systems. Full spec: Issue #1 https://app.party.build/issues/1 (Build section, component 1). Parent session: https://app.party.build/session/31591058-0d2c-4bf4-8e5e-7729cdc3cadd.

Parallel work: 6 sessions build at the same time, each in its own directory under warden/. Stack: Node 24 + TypeScript, npm workspaces, React (Vite), Fastify, node:sqlite, @anthropic-ai/sdk (server only), Vitest. Ownership: Frontend = apps/web; Backend API = apps/server (REST + per-run SSE); AI Services = packages/ai; Agent Sandbox = packages/sandbox; Evaluation Engine = packages/evals; Data & Infra = packages/db plus the warden/ root files (session cc75e5a4). Edit only your own directory. Talk to the backend only through a typed API client inside apps/web, and record the endpoint shapes you assume under 'Assumed contracts'. Never call model providers from the browser.

Scope: a React (Vite) client covering the whole journey: discovery chat with editable requirements; an interactive architecture canvas with directed handoffs and an agent detail panel (role, skills, tools, collaborators; executable vs proposed tools clearly distinguished); a clear entry into the bundled customer-support demo; a live run feed; a findings dashboard (category, status, severity, expected vs observed, reproduction, evidence, recommendation); fix review; and a before/after comparison linked to evidence. Standards: persistent project tabs (Requirements, Architecture, Testing); empty, loading, setup-required and error states; keyboard navigation; desktop and narrow layouts; status shown with icon and text, never colour alone; a coherent Warden visual identity. Covers AC1, AC3, AC4, AC12, AC15 (UI), AC11 (comparison view). Never show fabricated live results; label fixture data clearly. Provide a Party preview and screenshots.

In spec and build reports include a Provisional decisions table (decision, rationale, unresolved question) and an Assumed contracts list. Cite files and ACs. Distinguish locally reported tests from observed CI.
