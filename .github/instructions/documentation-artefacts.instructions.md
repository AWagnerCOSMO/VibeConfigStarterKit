---
name: Documentation Artefact Standards
description: Enforces output artefact conventions for all files written to the Documentation/ folder.
applyTo: "**/Documentation/**/*.md"
---

# Documentation Artefact Standards

All artefacts written to `Documentation/` must follow these conventions.

## File naming
| Phase | Artefact | Canonical filename |
|---|---|---|
| 1.1 | Requirement profile | `requirement-profile.md` |
| 1.2 | Per-module config summary | `config-{module-slug}.md` |
| 1.2 | Parameter settings | `parameter-settings.md` |
| 1.3 | Rollout plan | `rollout-plan.md` |
| 1.3 | E2E test plan | `e2e-test-plan.md` |
| 1.4 | Validation report | `validation-report.md` |
| 1.4 | Gap resolutions | `gap-resolutions.md` |
| 2.1 | Deployment log | `deployment-log.md` |
| 2.2 | E2E test results | `e2e-test-results.md` |
| any | Phase hand-off record | `phase-handoff.md` |

## Continuous-write rule
**Never batch writes.** After every mapping decision or step completion, update the relevant artefact immediately. Documents must reflect the current state at all times.

## Multi-tenant namespacing
When a `projectId` is active (see `Documentation/run-state.json`), all artefacts are written under `Documentation/<projectId>/` — not directly in `Documentation/`. The flat paths above are the leaf filenames within that folder.

## Per-worker status files
Workers write interim status to `Documentation/_status/<phase>/<module>.json` — these are never read directly by the user. The orchestrator merges them into the phase-level aggregate artefact (e.g. `deployment-log.md`).

## Traceability
Every artefact section that addresses a requirement must cite the requirement ID (e.g. `REQ-001`). Do not write configuration decisions without linking them to their source requirement.
