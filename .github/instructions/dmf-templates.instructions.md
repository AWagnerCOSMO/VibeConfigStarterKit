---
name: DMF Template Rules
description: Enforces DMF JSON template conventions whenever editing module configuration templates.
applyTo: "**/Modules/**/*.json"
---

# DMF Template Authoring Rules

These rules apply to every DMF template JSON file under `Modules/`.

## Required shape
```json
{
  "Description": "Human-readable description of this template",
  "TemplateId": "NNN - Template Name",
  "SourceEntityList": [
    {
      "SourceEntityName": "Human-readable label",
      "TargetEntity": "ExactD365EntityName",
      "Description": "What this entity configures",
      "EntitySequence": "10",
      "ExecutionUnit": "1",
      "LevelInExecutionUnit": "10"
    }
  ]
}
```

## Naming conventions
- `TemplateId` format: `NNN - Descriptive Name` where `NNN` is the 3-digit DMF sequence number (e.g. `025 - General Ledger`)
- `TargetEntity`: must match the entity name exactly as it appears in `Modules/dmf_odata_mapping.json` — never guess
- Entity sets are **plural** (e.g. `LedgerJournalHeadersV2`, not `LedgerJournalHeaderV2s`)

## Sequencing rules
- `EntitySequence`: numeric string, controls load order within the template; start at `"10"`, increment by 10
- `ExecutionUnit`: numeric string; entities sharing a unit run in parallel — only valid when they have no cross-entity dependencies
- `LevelInExecutionUnit`: controls order within a unit; start at `"10"`, increment by 10
- **Never reorder existing `EntitySequence` values** without documenting the reason in the `Description` block

## Dependency order
The three-digit `NNN` prefix encodes the global wave order (010 → 650). The authoritative dependency graph is `Modules/dependency-graph.json`. Do not hand-roll wave numbers — read from JSON.

## Data files
Each entity in the template has a corresponding CSV data file: `Modules/{path}/Data/{EntitySequence}_{TargetEntity}.csv`
- Header row must match D365 entity field names exactly
- Use enum **labels**, not integer codes
- Do not commit empty CSVs (must have at least one data row)
