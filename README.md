# integration-skills (BU → Huron person integration)

Shared agent skills and workspace-wide instructions (`CLAUDE.md`) for the BU → Huron person integration workspace.

## Purpose

This repository stores reusable skills and the workspace-wide `CLAUDE.md` instructions file that apply across multiple independent package repositories in this workspace.

The package repositories remain independently versioned and deployed:
- `integration-core`
- `integration-huron-person`
- `integration-huron-person-fargate`

### Overview of Repository Relationships:
- `integration-core`: Foundational utilities and shared logic for integration work. This comprises mainly interfaces and general use code for basic operations like S3 access, API clients, hash-based delta calculation, etc.
- `integration-huron-person`: Implements the core synchronization logic for Boston University → Huron person records. This comprises the main data mapping, and specific source/target business logic.
- `integration-huron-person-fargate`: AWS CDK infrastructure and serverless implementation for deploying the `integration-huron-person` project. This orchestrates a three-phase streaming delta synchronization pipeline using Lambda, ECS Fargate, S3, and SQS. Also primary among the orchestration is the implementation of parallel processing to speed up both the data acquistion from BU, and the pushing of mapped records into Huron.

```mermaid
flowchart LR
  subgraph C1["integration-core"]
    Ctgz["core.tgz"]
  end

  subgraph P1["integration-huron-person"]
    subgraph P2["package.json"]
      subgraph P3["core.tgz"]
        PAnchor((" "))
      end
    end
  end

  subgraph F1["integration-huron-person-fargate"]
    subgraph F2["package.json"]
      F3["huron-person.tgz"]
      F5["core.tgz"]
    end
  end

  Ctgz -->|npm pack + install| PAnchor
  PAnchor -->|npm pack + install| F2

  %% Visual styling so nested containment levels are clearly visible
  style C1 fill:#d9edf7,stroke:#31708f,stroke-width:2px,color:#111
  style P1 fill:#dff0d8,stroke:#3c763d,stroke-width:2px,color:#111
  style P2 fill:#edf7e8,stroke:#4f8a50,stroke-width:2px,color:#111
  style P3 fill:#cfe2f3,stroke:#2e5f84,stroke-width:2px,color:#111
  style F1 fill:#fcf3d6,stroke:#8a6d3b,stroke-width:2px,color:#111
  style F2 fill:#fdf7e6,stroke:#9a7f43,stroke-width:2px,color:#111
  style F3 fill:#edf7e8,stroke:#4f8a50,stroke-width:2px,color:#111
  style F5 fill:#cfe2f3,stroke:#2e5f84,stroke-width:2px,color:#111
  style Ctgz fill:#cfe2f3,stroke:#2e5f84,stroke-width:2px,color:#111

  classDef ghost fill:none,stroke:none;
  class PAnchor ghost;
```



## Layout

This repository is cloned **as the `.claude` directory of the workspace root**, so each skill ends up at `<workspace root>/.claude/skills/<skill-name>/SKILL.md` and the workspace instructions end up at `<workspace root>/.claude/CLAUDE.md`. Both are native locations that VS Code (GitHub Copilot), GitHub Copilot CLI, and Claude Code discover without custom settings (see [Workspace Instructions](#workspace-instructions-claudemd)).

Repository layout:

```text
integration-skills/
  CLAUDE.md
  README.md
  skills/
    <skill-name>/
      SKILL.md
```

Workspace layout after cloning:

```text
<workspace root>/
  .claude/                  <- this repository
    CLAUDE.md
    skills/
      <skill-name>/
        SKILL.md
  integration-core/
  integration-huron-person/
  integration-huron-person-fargate/
  ...
```

Skills must stay directly under `skills/` in this repository. Do not add a `.claude/` directory inside the repository.

## Installation

```bash
cd <workspace root>
git clone https://github.com/whennemuth/integration-skills.git .claude
```

`git clone` fails if `<workspace root>/.claude` already exists and is not empty (for example, Claude Code may already have created `.claude/settings.local.json`). In that case, initialize the repository in place:

```bash
cd <workspace root>/.claude
git init
git remote add origin https://github.com/whennemuth/integration-skills.git
git fetch origin
git checkout -t origin/master
```

Claude Code's personal `settings.local.json` is ignored by this repository's `.gitignore`.

## VS Code Setup (multi-root `.code-workspace`)

VS Code discovers project skills in `.claude/skills/` only at the top of each **root folder** of a [**multi-root**](https://code.visualstudio.com/docs/editing/workspaces/workspaces#_multiroot-workspaces) workspace. Skills found in any root are available to chat across the whole workspace.

Because the skills live in the workspace root rather than in a package repository, add the workspace root itself as a root folder of your `.code-workspace` file (which is saved in the workspace root):

```json
{
  "folders": [
    { "path": ".", "name": "workspace" },
    { "path": "integration-core", "name": "core" },
    { "path": "integration-huron-person", "name": "huron-person" },
    { "path": "integration-huron-person-fargate", "name": "huron-person-fargate" }
  ],
  "settings": {
    "chat.tools.terminal.autoApprove": {
      "npm test": true
    }
  }
}
```

The `workspace` root also contains every package folder, so hide them inside that root to avoid listing each repository twice in the Explorer and search results. Create `<workspace root>/.vscode/settings.json`:

```json
{
  "files.exclude": {
    "integration-*": true,
    "basic-readonly-npm-registry": true
  }
}
```

Folder-level settings apply only to their own root folder, so this hides the package folders in the `workspace` root without affecting their own roots. Each package repository keeps its own `.vscode/launch.json`, which VS Code only reads when the repository is a root folder.

Notes:
- Do not open the workspace root as a single folder (File > Open Folder). Discovery would still work, but VS Code ignores the `.vscode/launch.json` files of nested package repositories.
- Do not add same-named skills to the package repositories. When roots contain duplicate skill names, only the primary root's copy is used.
- GitHub Copilot CLI and the GitHub Copilot app do not use VS Code workspaces. They discover project skills only in the directory or repository they are started in.

See [Use Agent Skills in VS Code](https://code.visualstudio.com/docs/agent-customization/agent-skills).

## Claude Code Setup (started in the workspace root)

Claude Code loads project skills from `.claude/skills/` in the directory where it starts, so no extra configuration is needed:

```bash
cd <workspace root>
claude
```

Notes:
- When starting Claude Code inside a package repository, it stops searching parent directories at that repository's root. Add the workspace root with `--add-dir` (or `/add-dir` in a running session) to load the skills, e.g. `claude --add-dir ..` from `integration-huron-person-fargate/`.
- `--add-dir` does not load `CLAUDE.md` files from the added directory by default. To also load the workspace `.claude/CLAUDE.md`, set `CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD=1` (e.g. `CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD=1 claude --add-dir ..`).
- `permissions.additionalDirectories` in `settings.json` only grants file access. It does not load skills.
- Claude Code cloud sessions do not have access to this local clone. They only load skills committed to the checked-out repository or enabled on your claude.ai account.

See [Extend Claude with skills](https://code.claude.com/docs/en/skills).

## Workspace Instructions (`CLAUDE.md`)

`CLAUDE.md` at the root of this repository becomes `<workspace root>/.claude/CLAUDE.md`. It holds workspace-wide instructions that apply to all integration projects (core, huron-person, fargate, dashboard, file-drop, etc.): the architecture overview, pointers to the shared-pattern skills and project CLAUDE.md files, change and commit preferences, and the task verification protocol (build verification, test execution, and completion reporting on all code implementation tasks).

**Discovery**:
- **Claude Code**: `.claude/CLAUDE.md` is a native project instructions location, loaded at launch when Claude Code starts in the workspace root. Run `/context` and check **Memory files** to confirm.
- **GitHub Copilot CLI**: loads `.claude/CLAUDE.md` when started in the workspace root.
- **VS Code (GitHub Copilot Chat)**: loads `.claude/CLAUDE.md` from the workspace root folder (`"path": "."`) when the `chat.useClaudeMdFile` setting is enabled.

Project-specific conventions stay in each package repository's own `CLAUDE.md`.

**Scope**: These instructions complement (not replace) the skills in this repository. Skills define domain-specific technical patterns that load on demand, while `CLAUDE.md` defines workspace-wide context and workflow and verification requirements that load in every session.

## Maintenance Rules

1. Keep each skill in its own folder with `SKILL.md` directly under `skills/`.
2. Keep `CLAUDE.md` at the root of this repository. Do not add a second workspace `CLAUDE.md` at the workspace root, since both would load and duplicate instructions.
3. Use one canonical skill location path in docs and examples.
4. Document major skill changes in this repo history.
5. Do not assume workspace-level source control for package repos.
