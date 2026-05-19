---
name: Start New Project
description: Initialize a new D365 F&O implementation project — sets up the project ID, drops the run-state file, and guides the user through Phase 1.1 pre-flight.
---

You are starting a new D365 F&O implementation project.

**Step 1 — Collect project details**
Ask the user for:
- `projectId` (short slug, e.g. `acme-uat`)
- Modules in scope (list)
- Target F&O environment URL
- Any existing requirement documents ready to upload to `Requirements/`

**Step 2 — Initialize run-state**
Create `Documentation/run-state.json` with the projectId, current phase (`1.0`), and empty module status map.

**Step 3 — Source document pre-flight**
Load the `/source-document-validator` skill and run it against every file in `Requirements/`. If the gate is blocked, surface blockers and pause.

**Step 4 — Begin Phase 1.1**
Once the gate is open, load the `/phase-orchestrator` skill. It will classify this as a Phase 1.1 request and invoke `/d365-requirements-analysis`.
