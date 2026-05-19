# Reinforcement Learning — Deduplication, KPI Tracking & Supersede Chains

Reference detail extracted from the main `SKILL.md` to keep the body concise.
Load this file when you need the full procedure for deduplication, KPI accounting, or managing supersede chains.

---

## Deduplication (mandatory before every insert)

1. Compute `dedupHash = SHA-1(module + "|" + category + "|" + symptom.operation + "|" + rootCause)`.
2. Search `challenge_journal.json` for an entry with the same `dedupHash`.
3. **If found** → do NOT insert a new entry. Increment `recurrenceCount` on the existing entry, append today's date to `recurrenceDates[]`, and update `lastSeen`.
4. **If not found** → insert a new entry with `recurrenceCount = 1`.

---

## Supersede chains (when knowledge changes)

When a new fix contradicts a previously documented one (e.g., MS Learn updated, F&O version bump):

1. Mark the old entry: `resolution.status = "superseded"`, `supersededBy = <new id>`.
2. The new entry lists `supersedes: [<old id>, ...]`.
3. Both entries remain queryable; pre-flight lookup **ignores** `status = "superseded"` entries.

Do not delete old entries — the audit trail must remain complete.

---

## KPI tracking

Each successful pre-flight match (a journal entry actively prevented a recurrence) bumps:

- `entry.preventionEffective.preventedCount += 1`
- `entry.preventionEffective.lastPreventedAt = <today>`
- `_metadata.kpi.preventedRecurrences += 1` (file-level counter)

On resolution of new entries, update `_metadata.kpi.averageTimeToResolveMinutes` using a running average:

```
new_avg = ((old_avg × (n - 1)) + new_minutes) / n
```

where `n` = total resolved entries in the journal.

---

## Project-ID namespacing (multi-tenant)

When the agent is operating under a `projectId` (set in `run-state.json`):

- New journal entries get `projectId = <id>`.
- Pre-flight lookup filters first by exact `projectId`, then falls back to `projectId = null` (global learnings) if no project-scoped match is found.
- See `skills/d365-knowledge-routing/SKILL.md` §5 for the full namespacing convention covering `Documentation/`, `ChallengeJournal/`, and per-project `.discoveries/` overlays.

---

## Canonical schema

`schemas/challenge-journal.schema.json` (version `2.0`).

Minimum required fields per entry: `id`, `date`, `module`, `phase`, `category`, `severity`, `symptom`, `rootCause`, `resolution`, `sourceFilesUpdated[]`, `knowledgeUpdates[]`, `status`, `dedupHash`.

Optional but encouraged: `projectId`, `recurrenceCount`, `recurrenceDates[]`, `lastSeen`, `supersedes[]`, `supersededBy`, `preventionEffective`.
