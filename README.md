Here is the translated README, with all formatting preserved:

---

# Claude Code Framework

A framework for AI-First development with [Claude Code](https://docs.anthropic.com/en/docs/claude-code).

At its core is a spec-driven pipeline: first we plan the work in detail through a user interview and codebase research, align on specifications, and only then write code. Each stage is verified by specialized validator agents. Code is written using TDD.

## Quick Start

**New project:**

`/init-project` → `/init-project-knowledge` → start features

**New feature:**

`/new-user-spec` → `/new-tech-spec` → `/decompose-tech-spec` → `/do-feature` (or `/do-task`) → `/done`

**Quick task without a spec:**

`/write-code`

---

## How the Methodology Works

### Project Documentation — Project Knowledge

All project documentation is stored not in CLAUDE.md, but in a dedicated skill — **Project Knowledge** — a set of files in `.claude/skills/project-knowledge/references/`:

| File | Contents |
|---|---|
| `project.md` | Project purpose, audience, key features, scope |
| `architecture.md` | Tech stack, project structure, dependencies, data model |
| `patterns.md` | Code conventions, git workflow, testing strategy, business rules |
| `deployment.md` | Platform, environment variables, CI/CD pipeline, monitoring |
| `ux-guidelines.md` | UI language, tone of voice, domain glossary (optional) |

CLAUDE.md stays minimal — project name, link to Project Knowledge, default branch. The agent loads only the files from Project Knowledge that are needed for the current task (just-in-time context), not the entire context at once.

To create Project Knowledge for a new project, the **`project-planning`** skill is used (command `/init-project-knowledge`) — it conducts a user interview, fills in all Project Knowledge files, and creates a project backlog (features + roadmap). To update documentation during development — the **`documentation-writing`** skill.

---

### Feature Development Pipeline

The full path from idea to production. At each step, automatic validators run, and a git commit is made after each validation round (you can roll back to any intermediate state).

#### Step 1. User Spec — what we're building (`/new-user-spec`)

The agent loads the **`user-spec-planning`** skill, reads Project Knowledge, scans the codebase, and conducts a structured 3-cycle interview with the user:

1. **General questions** — what we want to build, why, and for whom
2. **With code context** — the agent has already studied the project and asks follow-up questions about integration, existing patterns, and dependencies
3. **Edge cases** — boundary conditions, errors, what-if scenarios

After the interview, the `interview-completeness-checker` agent verifies there are no gaps. Then the agent produces user-spec.md — a requirements specification in English, understandable to a non-technical person.

Two validators check the result (up to 3 correction iterations):
- **`userspec-quality-validator`** — document structure, testability of acceptance criteria
- **`userspec-adequacy-validator`** — feasibility of the solution, absence of over/underengineering

The user reads and approves the spec.

**Output:** `work/{feature}/user-spec.md` (status: approved)

#### Step 2. Tech Spec — how we're building it (`/new-tech-spec`)

The agent loads the **`tech-spec-planning`** skill, takes the approved user-spec, and translates it into a technical specification: architecture, key decisions, testing strategy, implementation plan broken down into tasks.

Written in English — this is a document for the agent, not for humans. If you're not a developer, you won't understand much here — and that's fine, that's what the previous step was for.

At this stage the agent researches the codebase (via the `code-researcher` agent), checks dependencies, and uses Context7 MCP to get up-to-date documentation for external libraries.

5 validators run in parallel (up to 3 correction iterations):
- **`skeptic`** — looks for mirages: references to non-existent files, functions, or APIs
- **`completeness-validator`** — bidirectional tracing: are all requirements from the user-spec covered, is there anything extra
- **`security-auditor`** — architectural review against OWASP Top 10
- **`test-reviewer`** — adequacy of the testing strategy
- **`tech-spec-validator`** — template compliance, task quality, dependency conflicts

The user approves the tech-spec.

**Output:** `work/{feature}/tech-spec.md` (status: approved)

#### Step 3. Task Decomposition (`/decompose-tech-spec`)

The agent loads the **`task-decomposition`** skill, takes the approved tech-spec, and creates a separate file for each task in the implementation plan. Tasks are created in parallel by the `task-creator` agent.

Each task file contains: acceptance criteria, a TDD anchor (which tests to write first), a list of context files, required skills, reviewers, execution wave, and dependencies on other tasks.

2 validators check the result (up to 3 iterations):
- **`task-validator`** — template compliance, description quality
- **`reality-checker`** — whether the referenced files, functions, and dependencies actually exist in the codebase

**Output:** `work/{feature}/tasks/*.md`

#### Step 4. Development and QA

Two modes to choose from:

**`/do-task`** — one task at a time, manual control. The agent reads the task file, loads the specified skills (usually `code-writing`), writes tests, then code, and goes through review. After each review round — a commit with fixes. Best for debugging, complex tasks, iterative work.

**`/do-feature`** — parallel execution of all tasks by a team of agents. The **`feature-execution`** skill spins up a team lead who creates a team via TeamCreate and distributes tasks in waves. In each wave, tasks are executed in parallel: one agent = one task. Each agent independently commits code, goes through review (up to 3 rounds), and fixes comments. The team lead coordinates and commits statuses.

Both modes use TDD: tests first, then code. After the code — automatic code review and security audit.

The final part of any feature's development is QA. QA tasks are automatically added at the end of the tech-spec:
- **Pre-deploy QA** — runs tests, verifies acceptance criteria from user-spec and tech-spec
- **Post-deploy QA** — verification in a live environment via MCP tools (Playwright, curl, Telegram MCP, etc.)

There is also **`/write-code`** — for writing code without a specification. Quick task, bug fix, experiment. Uses the `code-writing` skill directly: plan → tests → code → code review + security audit. No upfront planning via user-spec/tech-spec.

#### Step 5. Finalization (`/done`)

Closes out the feature: reads user-spec, tech-spec, and decisions.md (decisions made during development), updates the affected Project Knowledge files (architecture.md, patterns.md, etc.), and archives `work/{feature}/` into `work/completed/{feature}/`.

---

### Work Directory Structure

```
work/{feature}/
├── user-spec.md       # What we're building (English, for humans)
├── tech-spec.md       # How we're building it (English, for the agent)
├── decisions.md       # Decisions made during development
├── tasks/
│   ├── 1.md           # Atomic tasks
│   ├── 2.md
│   └── 3.md
└── logs/              # Work logs (interviews, research, reviews)
```

Completed features are archived in `work/completed/{feature}/`.

---

## Project Initialization

For a new project:

1. **`/init-project`** — creates the project structure from a template: a `.claude/` folder with Project Knowledge files, `CLAUDE.md`, `.gitignore` with rules for secrets and dependencies. Initializes a git repository, creates a private GitHub repository via the `gh` CLI, makes an initial commit, and creates `main` and `dev` branches. If there are already files in the folder — it will offer to move them to `old/`.

2. **`/init-project-knowledge`** — runs the `project-planning` skill, which conducts a detailed interview about the project and fills in all Project Knowledge files (`project.md`, `architecture.md`, `patterns.md`, `deployment.md`), and also creates a project backlog with features and a roadmap.

After this, feature development can begin with `/new-user-spec`.

---

## Creating New Skills

The framework is extended by creating new skills in the same style:

- **`skill-master`** — a guide and rules for creating skills: structure, patterns, types (procedural and informational), templates
- **`skill-test-designer`** — designing test scenarios for skills via interview
- **`skill-tester`** — running test scenarios with parallel runners (demo version, in development)

---

## Reference

### All Commands

| Command | What it does |
|---|---|
| `/init-project` | Creates project structure from a template, initializes git, creates a private GitHub repository, configures branches (main + dev) |
| `/init-project-knowledge` | Conducts a project interview, fills in all Project Knowledge files, and creates a backlog (features + roadmap) |
| `/new-user-spec` | Conducts a user interview, researches the code, creates a requirements specification with validation (2 validators, up to 3 iterations) |
| `/new-tech-spec` | Researches the codebase, creates a technical specification with architecture, decisions, testing strategy, and implementation plan (5 validators) |
| `/decompose-tech-spec` | Breaks down the tech-spec into atomic tasks with acceptance criteria, TDD anchors, and dependencies (2 validators) |
| `/do-task` | Executes one task: TDD (tests → code), code review, security audit. Commit after implementation and after each review round |
| `/do-feature` | Creates a team of agents, distributes tasks in waves, each agent executes a task in parallel with TDD and review |
| `/write-code` | Writes code without a specification: plan → tests → code → code review + security audit |
| `/done` | Reads specs and decisions.md, updates Project Knowledge, archives the feature to `work/completed/` |

### All Agents

Agents are isolated subprocesses with their own context. They receive a task, do one job, and return a structured result.

#### Validators and Creators
| Agent | What it does |
|---|---|
| `userspec-quality-validator` | Checks user-spec structure, coverage, testability of acceptance criteria |
| `userspec-adequacy-validator` | Checks feasibility of the solution, scope, absence of over/underengineering |
| `interview-completeness-checker` | Looks for gaps in the user-spec interview |
| `tech-spec-validator` | Checks tech-spec structure, template compliance, task quality |
| `skeptic` | Looks for mirages — references to non-existent files, functions, or APIs |
| `completeness-validator` | Bidirectional requirements tracing user-spec ↔ tech-spec, over/underengineering detection |
| `task-creator` | Creates task files from the tech-spec implementation plan |
| `task-validator` | Checks task files for template compliance and description quality |
| `reality-checker` | Verifies that files, functions, and dependencies referenced in tasks actually exist |
| `skill-checker` | Checks skills for compliance with skill-master standards |

#### Reviewers
| Agent | What it checks |
|---|---|
| `code-reviewer` | Code quality: architecture, readability, error handling, tests |
| `code-researcher` | Researches the codebase: files, patterns, tests, integrations, risks |
| `documentation-reviewer` | Project Knowledge quality: completeness, relevance, absence of bloat |
| `test-reviewer` | Test quality: finds issues and suggests specific fixes |
| `security-auditor` | Security per OWASP Top 10: SQL injection, XSS, auth, cryptography |
| `deploy-reviewer` | CI/CD pipeline: GitHub Actions, secrets management, deployment configuration |
| `infrastructure-reviewer` | Infrastructure: project structure, Docker, pre-commit hooks, .gitignore |
| `prompt-reviewer` | Quality of LLM prompts per prompt-master principles |

#### QA
| Agent | What it does |
|---|---|
| `pre-deploy-qa` | Runs tests, verifies acceptance criteria from user-spec and tech-spec |
| `post-deploy-qa` | Post-deploy verification in a live environment via MCP tools (Playwright, curl, Telegram MCP) |

### All Skills

#### Planning
| Skill | Purpose |
|---|---|
| `methodology` | Description of the entire methodology: pipeline, structure, principles |
| `project-planning` | Interview about a new project → filling in Project Knowledge + backlog (features, roadmap) |
| `user-spec-planning` | User interview → user-spec.md with requirements |
| `tech-spec-planning` | Code research → tech-spec.md with architecture and implementation plan |
| `task-decomposition` | Tech-spec → atomic task files with TDD anchors |

#### Development
| Skill | Purpose |
|---|---|
| `code-writing` | The code writing process: plan → tests → code → review |
| `feature-execution` | Feature orchestration: team lead creates an agent team, distributes tasks in waves |
| `prompt-master` | Writing, improving, and reviewing LLM prompts |

#### Quality
| Skill | Purpose |
|---|---|
| `code-reviewing` | Code review methodology across 11 dimensions: architecture, security, performance, etc. |
| `test-master` | Testing strategy: test pyramid, when to use unit/integration/e2e, how to ensure test quality |
| `security-auditor` | Security audit per OWASP Top 10: injections, authentication, cryptography |
| `pre-deploy-qa` | Acceptance testing: running tests, checking acceptance criteria without a live environment |
| `post-deploy-qa` | Post-deploy verification in a live environment via MCP tools |

#### Infrastructure and Documentation
| Skill | Purpose |
|---|---|
| `infrastructure-setup` | Setting up new project infrastructure: framework, Docker, pre-commit hooks (gitleaks), tests |
| `deploy-pipeline` | CI/CD setup: GitHub Actions, deployment (Vercel, Railway, Fly.io, AWS, VPS), secrets management |
| `documentation-writing` | Project Knowledge management: auditing, updating, consistency checks |

#### Meta
| Skill | Purpose |
|---|---|
| `skill-master` | Creating and updating skills: structure, patterns, rules |
| `skill-test-designer` | Designing test scenarios for skills via interview |
| `skill-tester` | Running test scenarios (demo version) |

---

## Shared — Templates and Scripts

The `shared/` folder contains source materials used by skills and commands:

| Folder | Contents |
|---|---|
| `templates/new-project/` | New project template: `.claude/` structure, Project Knowledge files, CLAUDE.md, .gitignore. Used by the `/init-project` command |
| `templates/infrastructure/` | Infrastructure templates (Docker, CI/CD configs). Used by the `infrastructure-setup` skill |
| `work-templates/` | Work document templates: `user-spec.md.template`, `tech-spec.md.template`, `task.md.template`, `decisions.md.template`, `checkpoint.yml.template`, `execution-plan.md.template`. Skills copy them when creating new specs and tasks |
| `interview-templates/` | Interview structures for planning skills: `feature.yml` (for user-spec), `skill.yml` (for skill-test-designer) |
| `scripts/` | Helper scripts: `init-feature-folder.sh` — creates the work directory for a new feature |

---

## Hooks — Automation

The `hooks/` folder contains Claude Code hooks that automatically trigger on specific events:

| Hook | Event | What it does |
|---|---|---|
| `post-compact-restore.sh` | SessionStart (compact) | Restores the feature-execution context after compaction: finds the checkpoint, verifies the current session is the team lead, outputs instructions for resuming work |

---

## Requirements

- [Claude Code CLI](https://docs.anthropic.com/en/docs/claude-code)
- [Context7 MCP server](https://github.com/upstash/context7) — the agent uses it to get up-to-date library documentation instead of relying on training data

---

## License

MIT License — use freely.
