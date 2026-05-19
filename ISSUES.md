# Agentic Coding Issues Audit

Verified repo issues against current VS Code Copilot customization guidance,
GitHub Copilot custom-instructions support, the open Agent Skills specification,
AGENTS.md conventions, and MCP interoperability guidance.

Audit date: 2026-05-12

## Scope

This document records the issues found during a repo review of the agentic
customization surfaces in this workspace:

- always-on instructions
- path-scoped instructions
- prompt files
- skills
- subagent design
- custom agents
- MCP configuration
- validation and CI coverage

## Verification Snapshot

- Built-in repo validation passed: `tools/validate_repo.py` reported `0 error(s), 0 warning(s)` once its missing Python dependency was installed.
- Skill metadata and governed JSON are internally consistent.
- Worker skills are already correctly marked as subagent-only and use forked context, but helper and infrastructure skills do not encode invocation intent consistently.
- The main issues are not random breakage; they are gaps in portability, enforcement, and use of newer VS Code/Copilot customization primitives.

## Standards Used

- VS Code: custom instructions, prompt files, custom agents, agent skills, subagents, and agent overview
- GitHub Copilot: repository instructions support matrix and precedence rules
- Agent Skills open specification: folder layout, frontmatter, progressive disclosure, portability
- AGENTS.md convention for cross-agent instructions
- MCP as the interoperability layer for external tools and data

## Issue Index

| ID | Severity | Category | Title |
|---|---|---|---|
| I-01 | HIGH | Cross-agent compatibility | No `AGENTS.md` compatibility layer |
| I-02 | HIGH | Custom agents | No `.agent.md` custom agents defined |
| I-03 | HIGH | Skills structure | Skills rely on a non-default `skills/` path |
| I-04 | MEDIUM | Prompt design | Prompt files do not declare execution context |
| I-05 | MEDIUM | Validation | Repo validator ignores major agentic surfaces |
| I-06 | MEDIUM | CI enforcement | Quality gate only partially covers agentic assets |
| I-07 | MEDIUM | Bootstrap | Local validation bootstrap is incomplete |
| I-08 | LOW | Documentation accuracy | README overstates portability and uses dated framing |
| I-09 | LOW | MCP hygiene | MCP config is template-like and unvalidated |
| I-10 | MEDIUM | Skill metadata | Internal/helper skill invocation controls are inconsistent |

---

## I-01: No `AGENTS.md` compatibility layer

Severity: HIGH
Category: Cross-agent compatibility

### Problem

The repo defines [.github/copilot-instructions.md](.github/copilot-instructions.md)
and scoped instruction files, but it does not define a root-level `AGENTS.md`.

That means the repo is missing the most direct cross-agent instruction surface
supported across modern GitHub Copilot environments and other agentic tooling.

### Evidence

- No `AGENTS.md` file exists in the repo root or subfolders.
- The repo currently relies on [.github/copilot-instructions.md](.github/copilot-instructions.md) as its only always-on shared instruction surface.

### Why This Matters

- VS Code and GitHub Copilot cloud agent both support `AGENTS.md` as agent instructions.
- Copilot CLI also supports repository-wide instructions plus `AGENTS.md`.
- Without `AGENTS.md`, the repo loses a portable, agent-neutral compatibility layer and stays more VS Code-specific than it needs to be.

### Recommended Fix

Add a root-level `AGENTS.md` that captures:

- repo purpose and structure
- how to validate changes locally
- where skills, prompts, instructions, and schemas live
- what must always be checked before edits or deployment-oriented tasks
- which artefacts are authoritative

Keep [.github/copilot-instructions.md](.github/copilot-instructions.md) as the
VS Code/Copilot-specific companion, not the only top-level control surface.

---

## I-02: No `.agent.md` custom agents defined

Severity: HIGH
Category: Custom agents

### Problem

The repo implements a rich multi-phase workflow in prose and skills, but it has
no `.agent.md` files and no `.github/agents/` directory.

The workflow therefore lacks first-class encoded personas for planning,
implementation, deployment, review, or orchestration.

### Evidence

- No `.agent.md` files are present anywhere in the workspace.
- No `.github/agents/` directory exists.
- Role separation is described in [skills/phase-orchestrator/SKILL.md](skills/phase-orchestrator/SKILL.md) and [skills/module-fanout/SKILL.md](skills/module-fanout/SKILL.md), but not enforced through custom agent definitions.

### Why This Matters

Custom agents are the current VS Code-native way to define:

- tool restrictions
- allowed subagents
- model preferences
- handoffs between roles
- persistent personas for repeated workflows

Without them:

- read-only phases are not protected by least-privilege tool boundaries
- multi-phase handoffs are not encoded as interactive agent transitions
- users must rely on prompt wording instead of runtime controls

### Recommended Fix

Create `.github/agents/` and add at least:

- `d365-orchestrator.agent.md`
- `d365-planner.agent.md`
- `d365-implementer.agent.md`
- `d365-reviewer.agent.md`

Use them to enforce tool boundaries and define handoffs between planning,
implementation, deployment, and validation.

---

## I-03: Skills rely on a non-default `skills/` path

Severity: HIGH
Category: Skills structure

### Problem

The repo stores skills in a top-level [skills](skills) directory rather than a
default project skill location such as `.github/skills/`.

This works in this workspace only because [.vscode/settings.json](.vscode/settings.json)
contains a custom skill discovery override.

### Evidence

- Skills are stored under [skills](skills).
- [.vscode/settings.json](.vscode/settings.json) contains:

```json
{
  "chat.agentSkillsLocations": {
    "skills": true
  }
}
```

### Why This Matters

- Default-discovery clients look in `.github/skills/`, `.agents/skills/`, or `.claude/skills/`.
- The current layout weakens the repo's portability claim in [README.md](README.md).
- The repo becomes dependent on a VS Code workspace setting rather than default standard locations.

### Recommended Fix

Preferred:

- move or mirror [skills](skills) to `.github/skills/`
- remove the custom override from [.vscode/settings.json](.vscode/settings.json)

If the top-level folder must stay, document clearly that it is a workspace-local
override and not a portable default.

---

## I-04: Prompt files do not declare execution context

Severity: MEDIUM
Category: Prompt design

### Problem

The two root prompt files are syntactically valid, but they do not declare
`agent`, `tools`, or model preferences.

That means they inherit whatever the currently active agent session provides.

### Evidence

- [.github/prompts/new-project.prompt.md](.github/prompts/new-project.prompt.md)
  declares `name` and `description`, but no `agent` or `tools`.
- [.github/prompts/rerun-module.prompt.md](.github/prompts/rerun-module.prompt.md)
  declares `name`, `description`, and `argument-hint`, but no `agent` or `tools`.

### Why This Matters

- Prompt files are more reliable when they encode their intended execution mode.
- `new-project.prompt.md` is expected to create state and run workflow logic.
- `rerun-module.prompt.md` is expected to load context and perform targeted re-run behavior.
- If launched from a read-only or mismatched agent, behavior becomes implicit instead of reproducible.

### Recommended Fix

Add explicit frontmatter where appropriate, for example:

```yaml
agent: agent
tools: ['codebase', 'editFiles']
```

Or, preferably, point prompts at named custom agents once those agents exist.

---

## I-05: Repo validator ignores major agentic surfaces

Severity: MEDIUM
Category: Validation

### Problem

The built-in validator is useful, but its scope is narrow. It validates governed
JSON, skill frontmatter, dependency graph consistency, and mapping drift. It does
not validate prompt files, instruction files, custom agents, always-on repo
instructions, or MCP config.

### Evidence

[tools/validate_repo.py](tools/validate_repo.py) currently validates:

- governed JSON artefacts
- `SKILL.md` frontmatter and local links
- dependency-graph cross-checks

It does not include rules for:

- `.github/prompts/**/*.prompt.md`
- `.github/instructions/**/*.instructions.md`
- `.github/agents/**/*.agent.md`
- root `AGENTS.md`
- [.github/copilot-instructions.md](.github/copilot-instructions.md)
- [.vscode/mcp.json](.vscode/mcp.json)

### Why This Matters

The repo can report green status while key agentic entry points drift out of
date, violate frontmatter expectations, or become inconsistent with the intended
workflow.

### Recommended Fix

Extend [tools/validate_repo.py](tools/validate_repo.py) to cover:

- prompt frontmatter presence and basic schema
- instruction-file frontmatter and `applyTo` sanity
- custom-agent frontmatter and tool declarations
- root-level instruction compatibility files such as `AGENTS.md`
- MCP config structure and placeholder detection

---

## I-06: Quality gate only partially covers agentic assets

Severity: MEDIUM
Category: CI enforcement

### Problem

The CI workflow is present and valuable, but it only enforces a subset of the
repo's agentic surfaces.

### Evidence

[.github/workflows/quality-gate.yml](.github/workflows/quality-gate.yml)
currently does the following:

- installs Python and `jsonschema`
- runs [tools/validate_repo.py](tools/validate_repo.py)
- link-checks the `skills` folder only
- allows markdown link check failures with `continue-on-error: true`

### Why This Matters

- prompt files, instructions, README, and future agents are not link-checked
- broken docs can pass CI
- agentic repo quality is only partially enforced even though the repo is built around reusable AI configuration assets

### Recommended Fix

Expand CI to:

- link-check `.github/`, `README.md`, and any future `.github/agents/`
- fail on broken links once the baseline is clean
- run the expanded validator from I-05

---

## I-07: Local validation bootstrap is incomplete

Severity: MEDIUM
Category: Bootstrap

### Problem

The repo includes a validator, but it does not declare the Python dependency
needed to run it locally.

### Evidence

- No `requirements.txt`, `requirements-dev.txt`, `pyproject.toml`, or equivalent Python dependency manifest exists in the repo root.
- Running [tools/validate_repo.py](tools/validate_repo.py) initially failed because `jsonschema` was not installed.
- The workflow installs `jsonschema` in CI, but local bootstrap instructions are not encoded in repo metadata.

### Why This Matters

For an agentic repo, bootstrap should be explicit. Agents and contributors
should not have to discover how to satisfy local validation prerequisites.

### Recommended Fix

Add one of the following:

- `requirements-dev.txt`
- `pyproject.toml`
- a dedicated `tools/requirements.txt`

Then document a single local validation command in [README.md](README.md) and,
ideally, in `AGENTS.md` once added.

---

## I-08: README overstates portability and uses dated framing

Severity: LOW
Category: Documentation accuracy

### Problem

The README presents the repo as broadly portable and describes the design as
being built on "Anthropic Skills". That is no longer the best framing for the
current ecosystem.

### Evidence

- [README.md](README.md) describes the repo as built on "Anthropic Skills architecture".
- The repo currently depends on a VS Code-specific skill-path override in [.vscode/settings.json](.vscode/settings.json).
- The repo does not yet provide a cross-agent `AGENTS.md` layer or any `.agent.md` custom agents.

### Why This Matters

- The current open standard is broader Agent Skills portability, not just Anthropic-specific framing.
- The repo is portable in intent, but not yet fully portable in structure.
- Overstating that portability makes the repo look more standards-complete than it currently is.

### Recommended Fix

Update [README.md](README.md) after structural fixes to say:

- the repo is aligned to the open Agent Skills model
- some workflows are currently optimized for VS Code Copilot
- full cross-client portability depends on default skill locations and shared instruction surfaces

---

## I-09: MCP config is template-like and unvalidated

Severity: LOW
Category: MCP hygiene

### Problem

The MCP config exists, but it behaves like an environment template rather than a
fully governed repo asset.

### Evidence

- [.vscode/mcp.json](.vscode/mcp.json) contains a placeholder-style entry for the Finance & Operations server URL.
- There is no validation in [tools/validate_repo.py](tools/validate_repo.py) for placeholder values or required structure beyond raw JSON syntax.
- There is no documented sample-vs-local strategy for environment-specific MCP configuration.

### Why This Matters

- MCP config is operationally important for a repo built around external tool access.
- Placeholder drift or accidental commits of environment-specific values can create fragile onboarding and inconsistent behavior.

### Recommended Fix

Adopt one of these patterns:

- keep a checked-in sample config and a local ignored override
- validate placeholder usage explicitly in the repo validator
- document which fields must be replaced locally before using deployment-oriented workflows

---

## I-10: Internal/helper skill invocation controls are inconsistent

Severity: MEDIUM
Category: Skill metadata

### Problem

The repo's three worker skills correctly encode that they are internal
subagents, but the same clarity is not applied consistently to helper and
infrastructure skills.

That means some skills that read like background helpers or coordinator-only
plumbing are still implicitly user-invocable because they omit frontmatter such
as `user-invocable: false`.

### Evidence

The current skill metadata state is:

| Skill | Current metadata | Assessment |
|---|---|---|
| `module-config-worker` | `user-invocable: false`, `disable-model-invocation: true`, `context: fork` | Correct for internal worker |
| `module-deployment-worker` | `user-invocable: false`, `disable-model-invocation: true`, `context: fork` | Correct for internal worker |
| `module-validation-worker` | `user-invocable: false`, `disable-model-invocation: true`, `context: fork` | Correct for internal worker |
| `module-fanout` | none of the above | Likely should be hidden; infrastructure skill |
| `d365-knowledge-routing` | none of the above | Likely should be hidden; helper lookup skill |
| `fo-mcp-server` | none of the above | Arguable; reads like a companion rules skill |
| `reinforcement-learning` | none of the above | Arguable; reads like a companion/background skill |
| `source-document-validator` | none of the above | Reasonable as user-invocable |
| `financial-compliance-guard` | none of the above | Reasonable as user-invocable |
| phase and top-level workflow skills | none of the above | Reasonable as user-invocable |

The clearest concrete gap is [skills/module-fanout/SKILL.md](skills/module-fanout/SKILL.md), which is described as dispatch infrastructure rather than a user-facing workflow.

### Why This Matters

- The slash menu becomes noisier and exposes skills that are not intended as end-user entry points.
- Invocation intent is encoded inconsistently across the skill graph.
- Helper/coordinator behavior is left to prose rather than frontmatter controls.

### Recommended Fix

At minimum:

- add `user-invocable: false` to [skills/module-fanout/SKILL.md](skills/module-fanout/SKILL.md)

Strong candidates to consider hiding as well:

- [skills/d365-knowledge-routing/SKILL.md](skills/d365-knowledge-routing/SKILL.md)
- [skills/fo-mcp-server/SKILL.md](skills/fo-mcp-server/SKILL.md)
- [skills/reinforcement-learning/SKILL.md](skills/reinforcement-learning/SKILL.md)

Leave the phase skills and clearly user-triggerable gate skills user-invocable
unless you later replace them with custom agents or prompt-driven entry points.

---

## Verified Strengths

The following items were checked and are not current issues:

- Worker skills already use `user-invocable: false`, `disable-model-invocation: true`, and `context: fork`.
- The worker-skill metadata issue does not apply repo-wide; it is limited to the helper/infrastructure layer and not the actual worker subagents.
- The scoped instruction files are narrow and appropriately targeted:
  [.github/instructions/dmf-templates.instructions.md](.github/instructions/dmf-templates.instructions.md) and [.github/instructions/documentation-artefacts.instructions.md](.github/instructions/documentation-artefacts.instructions.md).
- The repo has a real validation and CI story, even though that coverage should be expanded.
- The core coordinator/worker skill architecture is coherent and internally consistent.

## Recommended Remediation Order

1. Add `AGENTS.md` and `.github/agents/`.
2. Move or mirror skills into `.github/skills/`.
3. Make prompt files explicit about agent and tool scope.
4. Expand the validator to cover prompts, instructions, agents, and MCP config.
5. Normalize helper/infrastructure skill invocation metadata.
6. Expand CI to enforce the broader validation surface.
7. Add local Python dependency metadata for the validator.
8. Update README portability claims once the structural fixes are complete.
