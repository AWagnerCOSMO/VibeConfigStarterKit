---
name: phase-orchestrator
description: Top-level dispatcher for the D365 F&O implementation lifecycle. Use FIRST on any new user request — it classifies the request (Phase 1 analysis, Phase 1 config, Phase 2 deployment, Phase 2 testing, Phase 3 docs, troubleshooting), checks prerequisite gates (e.g. Phase 1 approval before Phase 2), picks the right phase skill, and manages hand-offs between phases. Drives the `module-fanout` skill when a phase needs per-module parallelism. Keeps the main agent's context lean by delegating heavy work to sub-agents instead of loading everything inline.
license: Proprietary
metadata:
  domain: dynamics-365-fo
  layer: "0"
  version: "1.0"
---

# Phase Orchestrator (Layer 0)

> The brain that decides **which phase skill to run, in what order, with what gates**. It does not execute the work itself — it dispatches.

---

## Decision flow

```
                       USER REQUEST
                            │
                            ▼
                ┌───────────────────────┐
                │  CLASSIFY             │
                │  (intent + artefacts) │
                └───────────┬───────────┘
                            │
        ┌─────────┬─────────┼─────────┬─────────┬─────────┐
        ▼         ▼         ▼         ▼         ▼         ▼
     Analysis  Config    Deploy    Testing    Docs   Trouble-
     (1.1)    (1.2-1.5)  (2.1)     (2.2)      (3)    shoot
        │         │         │         │         │         │
        ▼         ▼         ▼         ▼         ▼         ▼
   d365-req-  d365-      d365-      d365-     d365-    reinforce-
   analysis   config-    deploy-    valida-   docu-    ment-
              builder    ment       tion-     menta-   learning
                                    testing   tion
        │         │         │         │
        │         ▼         ▼         ▼
        │     module-fanout (Layer 2) → spawns Layer 3 workers
```

---

## Classification table

| Signal in request | Phase | Skill to invoke |
|---|---|---|
| "analyse / analyze requirements", new docs in `Requirements/` | 1.1 | `d365-requirements-analysis` |
| "build config", "DMF", "rollout plan", "test plan", "approval gate" | 1.2–1.5 | `d365-config-builder` |
| "deploy", "push to F&O", "go-live" | 2.1 | `d365-deployment` |
| "test", "validate", "E2E", "scenario", "fix-loop" | 2.2 | `d365-validation-testing` |
| "rollout report", "environment HTML", "final docs" | 3 | `d365-documentation` |
| Error, failure, "doesn't work", unexpected behaviour | — | `reinforcement-learning` (always pair) |

When ambiguous, ask the user **one** clarifying question.

---

## Prerequisite gates

Before invoking each phase skill, verify the prior phase gate has passed:

| To start | Required artefacts | Required state |
|---|---|---|
| `d365-config-builder` | `Documentation/requirement-profile.md`, `requirement-matrix.json` | All matrix rows have a non-`Open` status |
| `d365-deployment` | `rollout-plan.md`, `e2e-test-plan.md`, `validation-report.md`, `gap-resolutions.md` | All 9 approval-gate items pass (see `d365-config-builder` §1.5) |
| `d365-validation-testing` | `deployment-log.md` | All in-scope modules ✅ Deployed or ⚠️ Partial-with-documented-gap |
| `d365-documentation` | `e2e-test-results.md` | Coverage check passes (every applicable E2E process has ≥1 passing scenario) |

If a gate fails, **do NOT advance**. Report the failing items and either fix them in-phase or return to the prior phase.

---

## Hand-off protocol

When a phase completes:
1. The phase skill writes a one-line completion record to `Documentation/phase-handoff.md`:
   ```
   - 2026-04-29 09:30 | Phase 1.5 complete | Gate: 9/9 passed | Next: Phase 2.1
   ```
2. Phase orchestrator re-evaluates: classify the next user instruction OR auto-advance if the user said "do everything".
3. **Never auto-advance past a failed gate.** Always surface to the user.

---

## When to use sub-agents

The orchestrator itself stays in the main context. It triggers sub-agents only when invoking the `module-fanout` skill. Rule of thumb:

| Work shape | Approach |
|---|---|
| Single module, simple ask | Stay in main context, call leaf skills directly |
| Multiple modules, repetitive per-module work (config building, deployment, module-scoped testing) | **Spawn sub-agents via `module-fanout`** |
| Cross-module reasoning (gap analysis, dependency check, validation report) | Stay in main context |
| Long-running operation that may produce a lot of intermediate output | Sub-agent (use the VS Code `agent` tool) |

---

## Layer model

```
Layer 0 — phase-orchestrator           classify + gate-check + dispatch
Layer 1 — phase skill                  d365-config-builder | d365-deployment | d365-validation-testing | …
Layer 2 — module-fanout                spawn one sub-agent per module (parallel within wave, sequential across)
Layer 3 — worker skill (sub-agent)     module-config-worker | module-deployment-worker | module-validation-worker
Layer 4 — leaf skills                  fo-mcp-server, reinforcement-learning, d365-knowledge-routing
```

---

## Skills index

### Orchestration
| Skill | Use when |
|---|---|
| `phase-orchestrator` | **Always load first.** Classifies the request and picks the phase skill. Manages gates and hand-offs. |
| `module-fanout` | Pattern + contract for spawning per-module sub-agents. Used by every phase skill that does repetitive per-module work. |

### Phase skills (Layer 1)
| Skill | Use when |
|---|---|
| `d365-requirements-analysis` | Phase 1.1 — ingest `Requirements/`, classify, build profile + matrix. |
| `d365-config-builder` | Phase 1.2–1.5 — config files, rollout plan, test plan, approval gate. Delegates per-module work to `module-config-worker`. |
| `d365-deployment` | Phase 2.1 — push to F&O via MCP. Delegates per-module work to `module-deployment-worker`. |
| `d365-validation-testing` | Phase 2.2 — E2E tests + fix loop. Delegates scenario slices to `module-validation-worker`. |
| `d365-documentation` | Phase 3 — single-file HTML deliverables. |

### Worker skills (Layer 3 — run as sub-agents only)
| Skill | Spawned by |
|---|---|
| `module-config-worker` | `d365-config-builder` via `module-fanout`. Builds all artefacts for ONE module. |
| `module-deployment-worker` | `d365-deployment` via `module-fanout`. Deploys ONE module to F&O. |
| `module-validation-worker` | `d365-validation-testing` via `module-fanout`. Runs scenario slice for ONE module. |

### Leaf skills (Layer 4 — load on demand)
| Skill | Use when |
|---|---|
| `source-document-validator` | **Hard gate at start of Phase 1.1.** Verifies every file in `Requirements/` is readable + extractable. Blocks on encrypted / corrupt / OCR-required documents. |
| `financial-compliance-guard` | **Non-negotiable gate** whenever any Finance module is in scope (GL/AP/AR/FA/Cash/Tax/Budget/PrjAcct/Expense). Captures frameworks at 1.1; validates at 1.4 / 1.5 / 2.2. Blocks on compliance failures. |
| `d365-knowledge-routing` | Look up the file path for any module, DMF template, or process. |
| `fo-mcp-server` | **Always before any `data_*` / `form_*` / `api_*` MCP tool call.** |
| `reinforcement-learning` | Before risky operations and after any failure. Challenge Journal feedback loop. |

---

## Workspace layout

```
VibeConfigStarterKit/
├── .github/
│   ├── copilot-instructions.md    ← always-on operating rules (3 rules)
│   ├── instructions/              ← file-scoped instruction rules
│   └── prompts/                   ← reusable slash-command workflows
├── skills/                        ← 15 skills across 4 layers
├── Modules/                       ← 47 module knowledge files + DMF templates
├── Business Process/              ← 5,767-item APQC process catalogue
├── ChallengeJournal/              ← Running challenge + resolution log
├── Requirements/                  ← Drop input requirement docs here
├── Documentation/                 ← All project output artefacts
├── SourceFiles/                   ← Raw OData metadata + prompt templates
└── schemas/                       ← JSON Schema files for all contract types
```

---

## Cardinal rules (apply across ALL phases and skills)

1. **Source files are authoritative.** If the environment ever drifts from what the source files say, fix the source first, then re-deploy. Never patch the environment directly and call it done.
2. **Document continuously.** Write artefacts in `Documentation/` as you go — never batch. After every mapping decision or deployment step, update the relevant document immediately.
3. **Follow processes start-to-finish.** E2E tests must trace complete business processes. Never test fragments. A test that starts at step 3 of a 7-step process is not a valid E2E test.
4. **Consult module knowledge first.** When the system behaves unexpectedly, open the module `.md` file (sections: "Critical Configuration Rules", "Implementation Discoveries") before attempting any workaround. The answer is usually already there.
5. **Feed back every discovery.** Use the `reinforcement-learning` skill — every challenge, workaround, and non-obvious finding becomes a future shortcut via the Challenge Journal.
6. **Respect dependency order.** DMF template numbers (010 → 650) encode the global load order derived from `Modules/dependency-graph.json`. Never skip ahead or deploy a module before its upstream dependencies.
7. **Never guess entity names.** Verify against `Modules/dmf_odata_mapping.json` or the module knowledge file. Entity set names are plural and version-suffixed exactly as D365 defines them.
8. **Project-ID namespacing for multi-tenant work.** When operating under a `projectId` (set in `Documentation/run-state.json`), scope ALL writes under `<projectId>/` prefixes:
   - `Documentation/<projectId>/...` for project artefacts
   - `ChallengeJournal/<projectId>/challenge_journal.json` for project-scoped journal
   - `Modules/.discoveries/<projectId>/<module>.md` for project-specific learnings
   - **Never mutate canonical module `.md` files** in `Modules/` themselves
   - Pre-flight journal lookup: prefer `projectId` match, fall back to global (`projectId = null`)
9. **Financial compliance is a defect class, not a preference (NON-NEGOTIABLE).** Whenever the engagement touches General Ledger, AP, AR, Fixed Assets, Cash & Bank, Tax, Budgeting, Project Accounting, or Expense, the `financial-compliance-guard` skill MUST run at:
   - Phase 1.1 — capture every applicable framework (US GAAP, IFRS, local GAAPs, SOX, ASC 606 / IFRS 15, ASC 842 / IFRS 16, SAF-T, GoBD, MTD-VAT, ESG/CSRD, etc.) explicitly from the customer.
   - Phase 1.4 — validate proposed config against every framework check.
   - Phase 1.5 — approval gate cannot pass while any check is `fail` + `blocker`.
   - Phase 2.2 — re-validate against the deployed environment.
   Never assume a framework. Never waive a blocker on convenience. Waivers require written sign-off (`waivedBy`, `waivedAt`) from a qualified controller, CFO, or external auditor. LIFO under IFRS, missing audit trail under SOX, missing mandated e-invoicing — these are immediate blockers.

---

## Orchestrator-level cardinal rules
- **One phase at a time.** Never interleave Phase 1 and Phase 2 work.
- **Always check the gate before advancing.**
- **Always pair `reinforcement-learning`** when error signals appear.
- **Never load module knowledge in the main context** if you can fan it out to a sub-agent — keeps the orchestration light.
