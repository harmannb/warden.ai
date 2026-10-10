# Issue #7: Build the Warden frontend web app: discovery, architecture canvas, live runs, findings, fix review, and comparison

**URL:** https://app.party.build/issues/7
**Reference slug:** Warden Frontend
**Status:** open  **Type:** feature
**Revision:** 11a1eacd-d93d-4716-9668-48f00ab8f32e
**Annotation coordinates:** use `party issue view <n> --numbered`; this file's lines are not canonical.

## Summary
Build the Warden React (Vite) web client — component 1 of the [Warden Platform](https://app.party.build/issues/1) Build section — locally in this party repo at `projects/730af893059941e0a260658692bbb07a/warden/apps/web`. It covers the full journey: discovery chat with editable requirements, an interactive architecture canvas with an agent detail panel, a clear entry into the bundled customer-support demo, a live run feed, a findings dashboard, fix review, and a before/after comparison linked to evidence. It carries parent AC1, AC3, AC4, AC12, AC15 (UI) and AC11 (comparison view).

The client talks to the backend only through a typed API client, ships an explicit and clearly labelled fixture mode so the UI is reviewable before the other five components land, and never shows fabricated live results. Do not clone, push to, or open PRs against `harmannb/warden.ai`; edit only `warden/apps/web` (plus this project's `.party/` files).

Requested by sepehr seghatoleslami (session owner); product owner Harmann Bajwa.

## Behavior
- **Project home and demo entry:** List projects, create a new one, and pin a bundled customer-support demo card that states the executable-scope boundary (architecture recommendations for any project; test execution only against the bundled support system) and opens the demo project in one step.
- **Persistent project tabs:** Requirements, Architecture, and Testing tabs under `/projects/:id/...`; tab, selected agent, and selected run live in the URL so refresh, back/forward, and deep links preserve project context.
- **Discovery chat:** A conversational panel that sends the user's project description and answers to the server interviewer, shows follow-up questions, and surfaces the requirements it extracts.
- **Editable requirements:** Requirements grouped by category (goals, workflows, success criteria, data, integrations, constraints) with inline add/edit/delete and save; drafts survive tab switches with a discard prompt on leave.
- **Generate architecture:** A primary action on the Requirements tab requests recommendations; a requirements change after generation marks the architecture **Stale** with a regenerate action.
- **Architecture canvas:** An SVG canvas with auto-layout showing agents as nodes and directed handoffs as arrows; nodes are keyboard-focusable (arrow keys move, Enter selects, Esc clears) and a list view toggle exists for narrow screens.
- **Agent detail panel:** Selecting a node shows role, responsibilities, skills, tools, integrations, collaborators (incoming/outgoing handoffs), and rationale; each tool is tagged **Executable** or **Proposed — not configured** with icon and text.
- **Version and run controls:** The Testing tab shows system versions (baseline, fix, reset-baseline) with labels, lets the user start a baseline run or retest against a chosen version, and surfaces a 409 "run already active" conflict with a link to that run.
- **Live run feed:** Per-run SSE stream rendering evaluator started/message events, tool calls, finding published and finding consumed links, assertion results, and trace IDs; technical detail is expandable; the feed reconnects with `Last-Event-ID`, shows "Connection lost — retrying", and only marks a run complete on a terminal event.
- **Findings dashboard:** Filterable table and detail view with category, status, severity, affected agent/tool, expected vs observed, reproduction inputs and steps, evidence (type, ref, trace ID, excerpt), recommendation, linked requirements, and `causedBy` link; `passed`, `failed`, `errored`, `skipped`, and `incomplete` are visibly distinct and only `passed` reads as a pass.
- **Fix review:** Shows the proposed policy/configuration change as before/after, its rationale, affected behaviour, and the source finding before the user applies it; applying yields a new version and offers a retest; a stale fix (409) is surfaced, not retried silently.
- **Before/after comparison:** Side-by-side verdicts per scenario between a baseline run and a retest, each cell linking to its evidence and trace; a failed retest shows **failed**; a scenario without evidence shows "Evidence unavailable", never a blank cell.
- **Demo reset:** A reset action that reports the new baseline version label and leaves older runs showing the versions they actually tested; blocked with an explanation while a run is active.
- **Honest states everywhere:** Empty, loading, setup-required (names `ANTHROPIC_API_KEY`, never values), and error states for `setup_required`, `provider_error`, `malformed_output`, `timeout`, `validation`, `conflict`, and `not_found`; partial results from an errored run are kept and labelled **Incomplete**.
- **Fixture mode, explicit and labelled:** `VITE_WARDEN_DATA_MODE=fixture` serves a bundled dataset behind a persistent "Fixture data — not live results" banner; live mode never falls back to fixtures on error.
- **Accessibility and layout:** ARIA tabs pattern, roving tabindex on the canvas, `aria-live="polite"` feed with a pause control, visible focus, status always icon + text, usable at desktop and 390 px widths (drawer detail panel, scrollable tabs).
- **Warden visual identity:** A small token-based design system (colour, type, spacing, status iconography) in plain CSS applied consistently across all screens; no UI framework.
- **Typed API client and assumed contracts:** All server access goes through `src/api/` with `zod`-validated responses; the assumed endpoint and event shapes are recorded in `src/api/CONTRACTS.md` for the Backend API session to confirm; the browser never calls a model provider.
- **Self-contained workspace package:** `apps/web` has its own `package.json` (`@warden/web`), `package-lock.json`, and `dev`, `build`, `test`, `typecheck` scripts so it installs, tests, and builds on this branch alone and slots into the root npm workspace when Data & Infra adds it.
- **Verification and delivery:** Vitest + Testing Library unit tests, Playwright screenshots at desktop and narrow widths attached to the thread, a Party preview in fixture mode, `.party/preview.md` filled with the verified startup command, and `projects/730af893059941e0a260658692bbb07a/.party/checks.json` replaced with real `npm ci`, `test`, and `build` commands scoped to `warden/apps/web`.

## Acceptance Criteria
- [ ] Routes `/projects/:id/requirements`, `/projects/:id/architecture`, and `/projects/:id/testing` render persistent tabs; refreshing or using back/forward keeps the project, active tab, selected agent, and selected run. (AC3) [annotation:A1]
- [ ] The project home lists projects, creates a new project, and shows a bundled customer-support demo card whose text states the executable-scope boundary and opens the demo project. (AC4) [annotation:A2]
- [ ] The discovery chat sends messages through the typed client, renders assistant follow-up questions, and shows extracted requirements; the requirements list supports inline add, edit, delete, and save with drafts preserved across tab switches. (AC1) [annotation:A3]
- [ ] Requesting an architecture produces a canvas of agent nodes with directed handoff arrows; editing requirements afterwards shows a Stale badge and a regenerate action. (AC1, AC3) [annotation:A4]
- [ ] Selecting a canvas node by mouse or keyboard (Tab/arrow keys, Enter, Esc) opens a detail panel with role, responsibilities, skills, tools, integrations, collaborators, and rationale; every tool shows Executable or Proposed with both icon and text. (AC3, AC2 display) [annotation:A5]
- [ ] The Testing tab lists versions with their labels and kinds, starts a baseline run or retest against a chosen version, and surfaces an active-run 409 conflict with a link to the running run. (AC13 UI side) [annotation:A6]
- [ ] The live feed consumes the per-run SSE stream, renders evaluator activity, tool calls, published and consumed finding links, assertion results, and trace IDs with expandable detail; it reconnects with `Last-Event-ID`, shows a visible "Connection lost — retrying" state, and marks the run complete only on a terminal event. (AC12, AC7 display) [annotation:A7]
- [ ] The findings dashboard shows category, status, severity, affected agent/tool, expected vs observed, reproduction, evidence with trace references, recommendation, linked requirements, and `causedBy`; `passed`, `failed`, `errored`, `skipped`, and `incomplete` render with distinct icons and text, and summaries count only `passed` as passing. (AC12) [annotation:A8]
- [ ] Fix review shows the before/after policy change, rationale, affected behaviour, and source finding; Apply creates a new version, offers a retest, and a stale-fix 409 is surfaced to the user. (AC9 display, AC10) [annotation:A9]
- [ ] The comparison view pairs a baseline run and a retest per scenario, links each verdict to its evidence, shows a failed retest as failed, and renders "Evidence unavailable" when evidence is missing. (AC11) [annotation:A10]
- [ ] Demo reset shows the new baseline label, older runs keep their tested version labels, and reset is refused with an explanation while a run is active. (AC13 UI side) [annotation:A11]
- [ ] Setup-required, loading, empty, and error states exist for every data-bearing screen; the setup state names `ANTHROPIC_API_KEY` without a value, and no credential or raw header is ever rendered. (AC14, AC15) [annotation:A12]
- [ ] Fixture mode is enabled only by `VITE_WARDEN_DATA_MODE=fixture`, shows a persistent "Fixture data — not live results" banner, and live mode shows an error state instead of fixture data when the API fails. (AC12, AC14) [annotation:A13]
- [ ] The whole journey (discovery, architecture detail, run feed, findings, fix review, comparison) is operable by keyboard alone with visible focus and works at desktop and 390 px widths; status is never conveyed by colour alone. (AC15) [annotation:A14]
- [ ] All server access goes through `src/api/` with `zod`-validated responses, `src/api/CONTRACTS.md` documents every assumed endpoint, event, and error shape, and no browser code references a model provider or SDK. (AC14) [annotation:A15]
- [ ] `warden/apps/web` installs, type-checks, tests, and builds on its own with `npm ci`, `npm run typecheck`, `npm test`, and `npm run build`; `projects/730af893059941e0a260658692bbb07a/.party/checks.json` runs those commands and the project checks pass. (AC16 share) [annotation:A16]
- [ ] Vitest covers the status badge, SSE event reducer and reconnect logic, findings filtering and pass counting, API schema parsing (including rejected malformed payloads), and canvas keyboard navigation; the build report lists locally reported results separately from observed CI. [annotation:A17]
- [ ] A Party preview runs in fixture mode, `.party/preview.md` records the verified startup command, and desktop plus narrow screenshots of every major screen are attached to the thread. (AC17 share) [annotation:A18]
- [ ] The build report includes a Provisional decisions table (decision, rationale, unresolved question) and an Assumed contracts list, and states plainly that live-backend integration (AC11, AC12 end to end) is verified only when the components meet in `warden.ai`. [annotation:A19]

## Provisional decisions
| Decision | Rationale | Unresolved question |
|---|---|---|
| URL-based tab, agent, and run state | Refresh and deep links preserve context | None |
| Explicit `VITE_WARDEN_DATA_MODE=fixture` with banner; no fallback on error | Reviewable before the backend exists without faking live results | Run the Party preview in fixture mode until the server lands (assumed yes) |
| Hand-rolled SVG canvas with layered auto-layout and roving tabindex | Few nodes; full keyboard control; no heavy graph dependency | Is drag-to-reposition needed, or select + pan/zoom enough? |
| Self-contained `@warden/web` package with own lockfile | Must build alone on this branch; workspace-ready for the root scaffold | Package naming convention from Data & Infra |
| Vite proxy `/api` → `http://localhost:3000`, overridable via `VITE_WARDEN_API_BASE` | Backend port unknown | Server port and `/api` prefix |
| TanStack Query + `zod` + plain CSS tokens | Loading/error states for free; malformed payloads become visible errors; small dependency set | None |
| `EventSource` with `Last-Event-ID`; complete only on terminal event | No silent completion or implied progress | Event retention window for replay |
| Stale badge after requirement edits; edits locked during an active run | Outdated recommendations never look current; matches run/version immutability | Does the server expose a requirements `revision`? |
| Comparison composed client-side by `scenarioKey` | Avoids a bespoke endpoint | Will Evaluation Engine emit `scenarioKey`? |
| `.party/checks.json` scoped to `warden/apps/web` via `--prefix` | Checks must be real and pass on this branch | Six sessions share this file; Data & Infra consolidates at merge |

## Assumed contracts
- `GET /api/health` → `{ ok, setup: { anthropicConfigured } }`.
- `GET|POST /api/projects`, `GET /api/projects/:id` → `{ id, name, kind: 'custom'|'demo', createdAt }`.
- `GET /api/projects/:id/discovery` → `{ messages[] }`; `POST .../discovery/messages { content }` → `{ assistantMessage, requirements?, complete }`.
- `GET|PUT /api/projects/:id/requirements` → `{ revision, items: [{ id, category, text, source }] }`.
- `POST /api/projects/:id/architecture` → `{ id, requirementsRevision, agents: [{ id, name, role, responsibilities[], skills[], tools: [{ name, kind: 'executable'|'proposed', description }], integrations[], rationale }], handoffs: [{ from, to, label }] }`.
- `GET /api/projects/:id/versions` → `[{ id, label, kind: 'baseline'|'fix'|'reset-baseline', parentVersionId, createdAt, policySummary }]`.
- `POST /api/projects/:id/runs { versionId, kind: 'baseline'|'retest', retestOfRunId? }` → `{ id, versionId, status }`; 409 `conflict` when a run is active.
- `GET /api/runs/:id`, `GET /api/runs/:id/findings`, `GET /api/runs/:id/events?after=<id>`.
- `GET /api/runs/:id/stream` SSE with monotonic `id`; events `run.status`, `evaluator.started`, `evaluator.message`, `tool.call`, `finding.published`, `finding.consumed`, `assertion.result`, `run.finished` (terminal status `passed|failed|errored|incomplete`); data `{ runId, ts, traceId, evaluator?, payload }`.
- Finding `{ id, scenarioKey, category: 'unit'|'integration'|'acceptance'|'security'|'reliability', status: 'passed'|'failed'|'errored'|'skipped'|'incomplete', severity: 'critical'|'high'|'medium'|'low'|'info', title, expected, observed, affectedAgent, affectedTool, reproduction: { inputs, steps[] }, evidence: [{ type, ref, traceId, excerpt }], recommendation, causedBy?, requirementIds[] }`.
- `GET /api/projects/:id/fixes` → `[{ id, findingId, rationale, change: { before, after }, affectedBehavior[] }]`; `POST /api/fixes/:id/apply` → `{ versionId }`; 409 when stale.
- `POST /api/projects/:id/demo/reset` → `{ versionId }`; 409 while a run is active.
- Error envelope `{ error: { code: 'setup_required'|'provider_error'|'malformed_output'|'timeout'|'validation'|'conflict'|'not_found', message, details? } }`.
- Browser payloads never include credentials.

## Out of scope
- Any file outside `warden/apps/web` and this project's `.party/` files; the `warden/` root scaffold belongs to Data & Infra (session `cc75e5a4`).
- Server, AI, sandbox, evaluation, and persistence code; live-backend integration is verified when the six components meet in `warden.ai`.
- Arbitrary agent imports, production integrations, billing, and team administration (per the parent spec).


## Annotations

### Annotation A1 · current · open
**Range:** `31:7-31:229`

> Routes `/projects/:id/requirements`, `/projects/:id/architecture`, and `/projects/:id/testing` render persistent tabs; refreshing or using back/forward keeps the project, active tab, selected agent, and selected run. (AC3)

#### A1.1 · implemented
agent-920fa686-ba54-405a-883d-e9153d30c261 · 2026-10-10T22:27:47.211758+00:00

Routes in App.tsx with ARIA route tabs in ProjectLayout; selected agent (?agent, ?layout) and run state (?run, ?view, ?finding, ?fix, ?baseline, ?retest) live in the URL. App.test.tsx restores an agent from the URL; verified in Chromium.

**Evidence:**
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/warden/apps/web/src/App.tsx`
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/warden/apps/web/src/routes/ProjectLayout.tsx`
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/warden/apps/web/src/features/runs/testingNav.ts`

### Annotation A2 · current · open
**Range:** `32:7-32:191`

> The project home lists projects, creates a new project, and shows a bundled customer-support demo card whose text states the executable-scope boundary and opens the demo project. (AC4)

#### A2.1 · implemented
agent-920fa686-ba54-405a-883d-e9153d30c261 · 2026-10-10T22:27:55.402696+00:00

ProjectsHome lists custom projects, creates projects, and pins the demo card with the executable-scope text; it opens or seeds the demo (POST /api/projects kind=demo, verified against the Backend fake-adapter build).

**Evidence:**
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/warden/apps/web/src/routes/ProjectsHome.tsx`

### Annotation A3 · current · open
**Range:** `33:7-33:254`

> The discovery chat sends messages through the typed client, renders assistant follow-up questions, and shows extracted requirements; the requirements list supports inline add, edit, delete, and save with drafts preserved across tab switches. (AC1)

#### A3.1 · implemented
agent-920fa686-ba54-405a-883d-e9153d30c261 · 2026-10-10T22:28:04.803652+00:00

DiscoveryChat sends through WardenApi and shows follow-ups and extracted requirements. RequirementsEditor supports inline add, edit, recategorise, delete, and save; the draft lives in ProjectLayout so it survives tab switches, and leaving the project asks first (App.test.tsx). The PUT round trip was verified live.

**Evidence:**
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/warden/apps/web/src/features/requirements/DiscoveryChat.tsx`
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/warden/apps/web/src/features/requirements/RequirementsEditor.tsx`
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/warden/apps/web/src/features/requirements/draft.ts`

### Annotation A4 · current · open
**Range:** `34:7-34:184`

> Requesting an architecture produces a canvas of agent nodes with directed handoff arrows; editing requirements afterwards shows a Stale badge and a regenerate action. (AC1, AC3)

#### A4.1 · implemented
agent-920fa686-ba54-405a-883d-e9153d30c261 · 2026-10-10T22:28:15.789618+00:00

Generate/Regenerate produces a layered SVG canvas with directed arrows (skip edges arc over, back edges loop under). The Stale badge and regenerate action come from revision comparison; verified live after a requirements save.

**Evidence:**
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/warden/apps/web/src/features/architecture/layout.ts`
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/warden/apps/web/src/features/architecture/ArchitectureCanvas.tsx`
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/warden/apps/web/src/features/architecture/stale.ts`

### Annotation A5 · current · open
**Range:** `35:7-35:273`

> Selecting a canvas node by mouse or keyboard (Tab/arrow keys, Enter, Esc) opens a detail panel with role, responsibilities, skills, tools, integrations, collaborators, and rationale; every tool shows Executable or Proposed with both icon and text. (AC3, AC2 display)

#### A5.1 · implemented
agent-920fa686-ba54-405a-883d-e9153d30c261 · 2026-10-10T22:28:27.173121+00:00

Nodes are buttons with a roving tabindex (arrows, Home/End, Enter, Esc). AgentDetail shows role, responsibilities, skills, tools, integrations, collaborators, and rationale; ToolKindTag pairs icon and text for Executable and Proposed. Covered by ArchitectureCanvas.test.tsx.

**Evidence:**
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/warden/apps/web/src/features/architecture/ArchitectureCanvas.tsx`
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/warden/apps/web/src/features/architecture/AgentDetail.tsx`
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/warden/apps/web/src/components/Tags.tsx`

### Annotation A6 · current · open
**Range:** `36:7-36:211`

> The Testing tab lists versions with their labels and kinds, starts a baseline run or retest against a chosen version, and surfaces an active-run 409 conflict with a link to the running run. (AC13 UI side)

#### A6.1 · implemented
agent-920fa686-ba54-405a-883d-e9153d30c261 · 2026-10-10T22:28:34.584575+00:00

The Testing tab lists versions with label, kind, and current marker; it starts baseline runs or retests against a chosen version, and a 409 active-run conflict renders with a link to the running run (App.test.tsx; screenshot 05).

**Evidence:**
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/warden/apps/web/src/features/runs/RunsPanel.tsx`
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/warden/apps/web/src/routes/TestingTab.tsx`

### Annotation A7 · current · open
**Range:** `37:7-37:349`

> The live feed consumes the per-run SSE stream, renders evaluator activity, tool calls, published and consumed finding links, assertion results, and trace IDs with expandable detail; it reconnects with `Last-Event-ID`, shows a visible "Connection lost — retrying" state, and marks the run complete only on a terminal event. (AC12, AC7 display)

#### A7.1 · implemented
agent-920fa686-ba54-405a-883d-e9153d30c261 · 2026-10-10T22:28:41.041055+00:00

A fetch-based SSE reader sends Last-Event-ID on every reconnect, backs off, and drops replayed ids. RunFeed renders every event type with trace IDs and expandable detail, shows "Connection lost — retrying", has a pause control, and completes only on run.finished. Covered by sse.test.ts and feedReducer.test.ts and exercised in the fixture journey. Live SSE is unverified because the Backend fake mode needs a credential to start runs.

**Evidence:**
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/warden/apps/web/src/api/sse.ts`
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/warden/apps/web/src/features/runs/feedReducer.ts`
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/warden/apps/web/src/features/runs/RunFeed.tsx`

### Annotation A8 · current · open
**Range:** `38:7-38:354`

> The findings dashboard shows category, status, severity, affected agent/tool, expected vs observed, reproduction, evidence with trace references, recommendation, linked requirements, and `causedBy`; `passed`, `failed`, `errored`, `skipped`, and `incomplete` render with distinct icons and text, and summaries count only `passed` as passing. (AC12)

#### A8.1 · implemented
agent-920fa686-ba54-405a-883d-e9153d30c261 · 2026-10-10T22:28:45.808779+00:00

The findings table and detail cover every listed field, plus causedBy, consumedBy, and retestOf links. StatusBadge gives each status a distinct icon shape and text, and summaries count only passed (filter.test.ts, StatusBadge.test.tsx).

**Evidence:**
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/warden/apps/web/src/features/findings/FindingsDashboard.tsx`
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/warden/apps/web/src/features/findings/FindingDetail.tsx`
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/warden/apps/web/src/features/findings/filter.ts`
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/warden/apps/web/src/components/StatusBadge.tsx`

### Annotation A9 · current · open
**Range:** `39:7-39:217`

> Fix review shows the before/after policy change, rationale, affected behaviour, and source finding; Apply creates a new version, offers a retest, and a stale-fix 409 is surfaced to the user. (AC9 display, AC10)

#### A9.1 · implemented
agent-920fa686-ba54-405a-883d-e9153d30c261 · 2026-10-10T22:28:49.836568+00:00

FixReview shows the before/after policy diff with +/− markers, rationale, affected behaviour, risks, expected outcome, and the source finding. Apply asks for confirmation, creates a version, and offers a retest of that version. A 409 for an already-applied or stale fix is shown and never retried. Verified end to end in the fixture journey.

**Evidence:**
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/warden/apps/web/src/features/fixes/FixReview.tsx`
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/warden/apps/web/src/features/fixes/diff.ts`

### Annotation A10 · current · open
**Range:** `40:7-40:211`

> The comparison view pairs a baseline run and a retest per scenario, links each verdict to its evidence, shows a failed retest as failed, and renders "Evidence unavailable" when evidence is missing. (AC11)

#### A10.1 · implemented
agent-920fa686-ba54-405a-883d-e9153d30c261 · 2026-10-10T22:28:53.476806+00:00

Comparison pairs findings by scenarioKey. Only failed → passed reads as Fixed; failed → failed is Still failing; other non-pass pairs are Inconclusive. Each verdict links to its evidence and trace IDs, and missing evidence says "Evidence unavailable" (compare.test.ts, App.test.tsx).

**Evidence:**
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/warden/apps/web/src/features/comparison/compare.ts`
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/warden/apps/web/src/features/comparison/ComparisonView.tsx`

### Annotation A11 · current · open
**Range:** `41:7-41:171`

> Demo reset shows the new baseline label, older runs keep their tested version labels, and reset is refused with an explanation while a run is active. (AC13 UI side)

#### A11.1 · implemented
agent-920fa686-ba54-405a-883d-e9153d30c261 · 2026-10-10T22:28:57.884805+00:00

DemoReset confirms, then reports the new baseline label; earlier runs keep their tested version labels. It is disabled with an explanation while a run is active, and a 409 is surfaced. Verified in fixture mode and live against the Backend fake-adapter build.

**Evidence:**
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/warden/apps/web/src/features/runs/RunsPanel.tsx`

### Annotation A12 · current · open
**Range:** `42:7-42:214`

> Setup-required, loading, empty, and error states exist for every data-bearing screen; the setup state names `ANTHROPIC_API_KEY` without a value, and no credential or raw header is ever rendered. (AC14, AC15)

#### A12.1 · implemented
agent-920fa686-ba54-405a-883d-e9153d30c261 · 2026-10-10T22:29:03.381790+00:00

Shared Loading, Empty, Error, and SetupRequired views are used on every data screen. Setup names only env-shaped variable names (ANTHROPIC_API_KEY by default) and lists missing components; error details render code, message, issue paths, and trace ID only.

**Evidence:**
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/warden/apps/web/src/components/StateViews.tsx`
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/warden/apps/web/src/components/AppShell.tsx`
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/warden/apps/web/src/api/errors.ts`

### Annotation A13 · current · open
**Range:** `43:7-43:224`

> Fixture mode is enabled only by `VITE_WARDEN_DATA_MODE=fixture`, shows a persistent "Fixture data — not live results" banner, and live mode shows an error state instead of fixture data when the API fails. (AC12, AC14)

#### A13.1 · implemented
agent-920fa686-ba54-405a-883d-e9153d30c261 · 2026-10-10T22:29:06.713143+00:00

Only VITE_WARDEN_DATA_MODE=fixture selects FixtureWardenApi, behind a persistent banner and header chip. Live failures render network or HTTP errors and never fixture data (config.test.ts; App.test.tsx; screenshot 11).

**Evidence:**
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/warden/apps/web/src/api/config.ts`
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/warden/apps/web/src/components/AppShell.tsx`
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/warden/apps/web/src/api/fixtures/fixtureClient.ts`

### Annotation A14 · current · open
**Range:** `44:7-44:239`

> The whole journey (discovery, architecture detail, run feed, findings, fix review, comparison) is operable by keyboard alone with visible focus and works at desktop and 390 px widths; status is never conveyed by colour alone. (AC15)

#### A14.1 · partial
agent-920fa686-ba54-405a-883d-e9153d30c261 · 2026-10-10T22:29:14.897068+00:00

Done: native buttons, links, and selects throughout; keyboard ARIA tabs; canvas roving tabindex (tested); focus-visible rings; a focus-trapped confirm dialog; icon+text status everywhere; 390 px layouts verified (drawer, stacked tables, run-first Testing tab; screenshot 09). Not yet done: a full keyboard-only walkthrough of the whole journey.

**Evidence:**
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/warden/apps/web/src/components/Tabs.tsx`
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/warden/apps/web/src/components/ConfirmDialog.tsx`
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/warden/apps/web/src/styles/app.css`

### Annotation A15 · current · open
**Range:** `45:7-45:224`

> All server access goes through `src/api/` with `zod`-validated responses, `src/api/CONTRACTS.md` documents every assumed endpoint, event, and error shape, and no browser code references a model provider or SDK. (AC14)

#### A15.1 · implemented
agent-920fa686-ba54-405a-883d-e9153d30c261 · 2026-10-10T22:29:25.412428+00:00

All access goes through WardenApi (src/api/) with zod-validated responses. CONTRACTS.md lists every endpoint, SSE event, error code, and the Backend-shape tolerances. No browser code references a model SDK or provider (grep clean).

**Evidence:**
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/warden/apps/web/src/api/schemas.ts`
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/warden/apps/web/src/api/http.ts`
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/warden/apps/web/src/api/CONTRACTS.md`

### Annotation A16 · current · open
**Range:** `46:7-46:270`

> `warden/apps/web` installs, type-checks, tests, and builds on its own with `npm ci`, `npm run typecheck`, `npm test`, and `npm run build`; `projects/730af893059941e0a260658692bbb07a/.party/checks.json` runs those commands and the project checks pass. (AC16 share)

#### A16.1 · partial
agent-920fa686-ba54-405a-883d-e9153d30c261 · 2026-10-10T22:29:37.968422+00:00

checks.json runs npm ci, typecheck, test (JUnit to PARTY_OUTPUT_DIR), and build scoped to warden/apps/web; all four pass locally exactly as written. Not yet done: the Party project-check result is not observed at the time of this annotation.

**Evidence:**
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/.party/checks.json`

#### A16.2 · partial
agent-920fa686-ba54-405a-883d-e9153d30c261 · 2026-10-10T22:43:09.309705+00:00

Update after rebasing onto main (commit f9a89f5): main's root project checks (npm install, typecheck, test, build across every workspace) now cover @warden/web; the package joined the workspace and its own lockfile was removed, matching the other packages. All four root commands pass locally: 567 tests passed and 3 skipped across all workspaces, including the 93 web tests; typecheck and build include @warden/web. Not yet done: the Party check result is not observed at this point.

**Evidence:**
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/.party/checks.json`
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/warden/package-lock.json`
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/warden/apps/web/package.json`

### Annotation A17 · current · open
**Range:** `47:7-47:284`

> Vitest covers the status badge, SSE event reducer and reconnect logic, findings filtering and pass counting, API schema parsing (including rejected malformed payloads), and canvas keyboard navigation; the build report lists locally reported results separately from observed CI.

#### A17.1 · implemented
agent-920fa686-ba54-405a-883d-e9153d30c261 · 2026-10-10T22:29:48.067774+00:00

93 Vitest tests, locally reported: StatusBadge, feedReducer, sse (reconnect with Last-Event-ID, 204, malformed), filter (pass counting), http and schemas (malformed payloads rejected), layout/navigation, ArchitectureCanvas keyboard, compare, stale, config, and App integration.

**Evidence:**
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/warden/apps/web/src/api/sse.test.ts`
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/warden/apps/web/src/features/runs/feedReducer.test.ts`
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/warden/apps/web/src/api/http.test.ts`
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/warden/apps/web/src/features/architecture/ArchitectureCanvas.test.tsx`

### Annotation A18 · current · open
**Range:** `48:7-48:201`

> A Party preview runs in fixture mode, `.party/preview.md` records the verified startup command, and desktop plus narrow screenshots of every major screen are attached to the thread. (AC17 share)

#### A18.1 · implemented
agent-920fa686-ba54-405a-883d-e9153d30c261 · 2026-10-10T22:29:55.225142+00:00

The Party preview runs in fixture mode on the session port. .party/preview.md records the verified command. Eleven screenshots are attached: eight desktop screens, the 390 px contact sheet of all screens, and two live-mode states.

**Evidence:**
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/.party/preview.md`

#### A18.2 · implemented
agent-920fa686-ba54-405a-883d-e9153d30c261 · 2026-10-10T22:43:11.254984+00:00

Update after rebasing onto main (commit f9a89f5): the verified preview commands now live in the canonical repository-root .party/preview.md, in place of the removed project-level file. That covers the fixture-mode web preview and the web-live plus server-fake pairing. The fixture preview was restarted from the workspace install and is live.

**Evidence:**
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:.party/preview.md`
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:.party/warden-web.md`

### Annotation A19 · current · open
**Range:** `49:7-49:267`

> The build report includes a Provisional decisions table (decision, rationale, unresolved question) and an Assumed contracts list, and states plainly that live-backend integration (AC11, AC12 end to end) is verified only when the components meet in `warden.ai`.

#### A19.1 · implemented
agent-920fa686-ba54-405a-883d-e9153d30c261 · 2026-10-10T22:30:02.489517+00:00

The Build turn report includes the Provisional decisions table and Assumed contracts, separates local results from CI, and states that live end-to-end integration (AC11, AC12) is verified only when the components meet in warden.ai.

**Evidence:**
- File: `party-962bc9f6a03f4affbff6eef3aa742a6e:projects/730af893059941e0a260658692bbb07a/warden/apps/web/src/api/CONTRACTS.md`

Reply with `party issue annotation reply <n> <A1> ...`; resolve with `party issue annotation resolve <n> <A1>`.
