---
name: Re-run Single Module
description: Re-deploys or re-validates a single named module without touching other modules. Use after fixing a module-level configuration error to avoid re-running the entire wave.
argument-hint: "[module name] [phase: 2.1 or 2.2]"
---

Re-run a single module in an ongoing deployment or validation phase.

**Usage**: `/rerun-module General Ledger 2.1`

Load the `/phase-orchestrator` skill. Tell it:
> "Re-run module `<module>` for phase `<phase>` only. Apply a `moduleFilter` to the `module-fanout` dispatch table so only this module's DMF sequence is included. Skip all other modules."

**Pre-flight (mandatory)**
Before re-running, load `/reinforcement-learning` and query `ChallengeJournal/challenge_journal.json` for any resolved entries for this module. Apply all documented preventive measures from the resolution notes.

**What this does**
- Phase 2.1: Dispatches a single `module-deployment-worker` sub-agent for the named module only. Sets `moduleFilter: [<dmfSeq>]` in the fan-out dispatch table.
- Phase 2.2: Dispatches a single `module-validation-worker` sub-agent with the filtered scenario slice.

**After completion**
Merge the worker's `_inbox` journal entries and update `Documentation/_status/<phase>/<module>.json`. Log any new findings via `/reinforcement-learning`.
