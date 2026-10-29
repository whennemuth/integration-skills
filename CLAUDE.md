# Integration Workspace Knowledge Base

Workspace-wide instructions for every integration project. This file lives at `<workspace root>/.claude/CLAUDE.md`
(versioned in the integration-skills repository) and is loaded automatically by Claude Code, GitHub Copilot CLI, and
VS Code Copilot Chat. Paths below are relative to the workspace root.

## Architecture Overview

This multi-project workspace synchronizes Boston University person records from the BU CDM API (source) into the
Huron research administration platform (target, via its HRS API) using a three-phase streaming delta pipeline
(chunking → processing → post-processing). Each project is its own git repository:

- **integration-core**: Shared abstractions (delta engines, DataSource/DataTarget base classes, `TestEnvironment`)
- **integration-huron-person**: BU → Huron person sync logic (data sources, mapping, data targets, delta strategy, ConfigManager)
- **integration-huron-person-fargate**: AWS CDK infrastructure and ECS Fargate pipeline, plus the management dashboard in `dashboard/`
- **integration-huron-person-dashboard**: Retired; the dashboard moved into `integration-huron-person-fargate/dashboard/`
- **integration-huron-file-drop**: S3-based file ingestion pipeline
- **basic-readonly-npm-registry**: Private npm registry for package hosting

Packages flow core → huron-person → huron-person-fargate as packed `.tgz` npm dependencies.

## Shared Patterns

Pattern details live in skills under `.claude/skills/`, which load on demand:

- Harness environment variables (`TestEnvironment('PREFIX')`, `.env` grouping, `example-env.md`):
  `testenvironment-pattern`, `environment-variable-organization`, `harness-initialization-patterns`
- Config and credential resolution chain: `configmanager-chain-pattern`
- Chunking, processing, and post-processing phases: `three-phase-pipeline`
- DataSource (API key) and DataTarget (JWT) implementations: `datasource-datatarget-abstraction`
- ECS Fargate Docker entry points: `docker-entrypoint-pattern`
- Unfamiliar abstractions or flags: `verify-abstractions-before-implementation`
- Commit messages (problem-first): `commit-message-conventions`

## Project-Specific Knowledge

Project and folder-level CLAUDE.md files hold detailed patterns. If your tool has not already loaded the relevant one,
read it before changing code in that area:

- `integration-core/CLAUDE.md`
- `integration-huron-person/CLAUDE.md`, plus `src/data-source/`, `src/data-target/`, `src/data-mapper/`, and
  `src/delta-strategy/`
- `integration-huron-person-fargate/CLAUDE.md`, plus `lib/`, `docker/`, `src/`, `src/chunking/`, `src/processing/`,
  `src/post-processing/`, `src/runner/`, and `dashboard/`

## Change and Commit Preferences

- Do not automatically stage and/or commit any changes, but prompt the user when you want to do so.
- Keep this file current. When a change makes anything here inaccurate (architecture, project list, skill pointers,
  CLAUDE.md map, verification steps), update this file as part of the same work and tell the user. Keep only
  workspace-wide guidance here: put project detail in that project's CLAUDE.md and pattern detail in a skill instead of
  repeating it, and avoid details that go stale quickly, such as line numbers and counts.

## Task Verification Protocol

For ALL code implementation tasks, the agent (Claude Code or Copilot) must:

1. **Build after changes**
   - Run `npm run build` or `tsc`
   - Show build output (errors or success)

2. **Run relevant tests**
   - Execute affected test files
   - Show test results (pass/fail counts, output)

3. **Verify the fix**
   - Confirm changes address the original problem
   - Check for unintended side effects

4. **Use verification template format**

### Response Template

```
FILES CHANGED:
- [list with brief descriptions]

BUILD STATUS:
✓/✗ [command + output summary]

TESTS RUN:
$ [command]
[test results or "N/A with justification"]

VERIFICATION:
- [what was checked]
- [edge cases considered]

STATUS: [Complete/Blocked/Needs Review]
```

### When to Apply

- All code edits (production or test files)
- Refactoring operations
- Bug fixes
- New feature implementations

### When NOT to Apply

- Pure documentation updates (README, CLAUDE.md)
- Simple questions with no code changes
- Exploratory searches or analysis tasks
- Quick clarifications
