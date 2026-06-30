# AI Guide

This guide describes how we use AI coding agents at MindStudios. Our standard tool is **[Claude Code](https://claude.com/claude-code)** — the CLI agent from Anthropic. The goal of this guide is to make every engineer productive with the same conventions, the same shared tooling, and the same guardrails, so that AI-assisted work looks and reads like the rest of our codebase.

The same principles apply if you use Claude through the IDE extension (VS Code / JetBrains), the desktop app, or `claude.ai/code` — the configuration described here travels with the repository.

## Principles

1. **The agent follows our handbook, not its own defaults.** Service objects, query objects, policies, error handling, code style — the agent is expected to match what is already documented here. Point it at the relevant guide when it drifts.
2. **Every change goes through the same review gate as human work.** AI does not get a fast lane. Conventional Commits, Jira-tagged branches, merge requests, and CI all apply (see [Gitflow](gitflow.md)).
3. **Context is configuration.** A good `CLAUDE.md` and the right shared skills do more for output quality than clever prompting. Invest in them.
4. **Secrets never reach the model.** Treat prompts as if they will be logged. See [Store Secrets](store_secrets.md).
5. **You own the output.** The agent drafts; you read, run, and verify. A green pipeline is the minimum bar, not the finish line.

## Setup

### 1. Install Claude Code

```bash
npm install -g @anthropic-ai/claude-code
```

Authenticate on first run with your MindStudios account.

### 2. Recommended global settings

Settings live in `~/.claude/settings.json`. Our baseline:

```json
{
  "model": "opus[1m]",
  "effortLevel": "high",
  "theme": "auto"
}
```

`opus[1m]` gives the 1M-token context window, which matters on our larger Rails apps. Drop to a faster/cheaper model for throwaway tasks if you prefer.

### 3. Install the shared skills plugin

We maintain a private plugin marketplace, **`tms-skills`**, that ships the team's skills and conventions. Clone the skills repo and install it:

```bash
claude plugin marketplace add /path/to/skills
claude plugin install tms-skills@tms-skills
```

Restart Claude Code after installing. Invoke a skill with `/<skill-name>` or just describe the task — skills are matched by description.

## Project context: `CLAUDE.md`

Every repository must have a `CLAUDE.md` at its root. It is the first thing the agent reads, and it is the single highest-leverage file for output quality. Commit it like any other source file — it is shared team context, not personal config.

Keep it skimmable. Our house structure:

```markdown
# CLAUDE.md

This file provides guidance to Claude Code when working with code in this repository.

## Project Overview
<one paragraph: what the product is, stack, key business domains>

## Development Commands
<the exact commands to run the app, tests, linters, console — copy-pasteable>

## Architecture Overview
<core models, key services/workers, API namespaces, database notes>

## Conventions
<commit/branch rules, where things live, testing strategy>

## Codebase Gotchas
<the non-obvious traps a newcomer would hit>
```

Guidelines that have worked well for us:

- **Make commands copy-pasteable and environment-correct.** If the app runs in Docker, every command should be the `docker-compose run --rm ...` form — the agent will run exactly what is written.
- **Document the "Gotchas".** The single most valuable section. Known bugs, disabled-but-present libraries, SSR guards, branch-specific behavior — these stop the agent from "fixing" things that are intentional.
- **Describe the domain in nouns.** A short map of the core models and services lets the agent navigate without re-reading the whole tree.
- **Keep it current.** Update `CLAUDE.md` in the same MR that changes the architecture it describes.

> Tip: run `/init` in a new repo to have the agent draft a first `CLAUDE.md` from the codebase, then trim and correct it.

## Shared skills

Skills are reusable, named workflows that encode our conventions. The `tms-skills` plugin currently ships:

| Skill | What it does |
|-------|--------------|
| `create-branch` | Creates a git branch following our naming convention. Given a Jira issue, names it `type/ISSUE-KEY_description` (see [Gitflow](gitflow.md)). |
| `commit` | Generates a [Conventional Commit](https://www.conventionalcommits.org/) message for staged changes. |
| `create-mr` | Opens/updates a GitLab Merge Request with our title, description, and issue-reference conventions. Always use this rather than creating an MR by hand. |
| `iterate-mr` | Runs the feedback-fix-push-wait loop on an MR until the pipeline is green and review comments are addressed. |
| `rails-audit` | Full Rails code audit across 8 categories (quality, performance, security, testing, docs, localisation, CI/CD, error monitoring). Mirrors our [Backend Audit](backend_audit.md). |
| `security-review` | OWASP-style security review of the pending diff, with confidence-rated findings. |
| `sentry-fix-issues` | Pulls a Sentry issue (events, stack traces, breadcrumbs, traces) and works toward a root cause. |
| `implement-design` | Translates a Figma design into production code with 1:1 fidelity. Requires the Figma MCP server. |
| `skill-scanner` | Audits a skill for prompt injection, malicious scripts, excessive permissions, and secret exposure before you install it. |

If a workflow is becoming a copy-pasted prompt, it should become a skill. See [Contributing skills](#contributing-skills).

## Engineering skills (recommended)

Beyond the team's `tms-skills` baseline, we recommend layering on a set of *process* skills that encode a disciplined "plan → spec → build → review" loop. These are installed per-developer, not committed to the repo, and they're a matter of personal workflow — but the team has converged on two options.

### Recommended: Superpowers

[**Superpowers**](https://github.com/obra/superpowers) is our main recommendation, and what most of the team uses today. It's a plugin that ships a large library of skills built around a **brainstorm → plan → implement** loop, with a strong bias toward TDD, subagent delegation, and writing decisions down before any code is written. Install it via the plugin marketplace:

```bash
claude plugin marketplace add obra/superpowers-marketplace
claude plugin install superpowers@superpowers-marketplace
```

It's an opinionated, batteries-included framework — once installed, describe your task and it walks you through the phases. Start here unless you have a reason not to.

### Alternative: Matt Pocock's engineering skills

If you prefer a lighter, à-la-carte set rather than one framework, [Matt Pocock's engineering skills](https://github.com/mattpocock/skills) are a good alternative — several of us still use them. They're installed individually into `~/.claude/skills/`, so you take only the ones you want. The simplest way to add one is:

```bash
npx skills add <skill-name>
```

The ones we actually use, by phase:

**Plan & spec — before any code**

| Skill | What it does |
|-------|--------------|
| `grilling` | Interviews you relentlessly to stress-test a plan or design before you build. Surfaces the holes while they're still cheap to fix. |
| `grill-with-docs` | Same interview, but writes ADRs and a glossary as you go — use it when the decisions are worth recording. |
| `to-prd` | Turns the conversation you just had into a PRD and publishes it to the issue tracker. No second interview, just synthesis. |
| `implement` | Implements a piece of work from a PRD or a set of issues. |

**Design & architecture**

| Skill | What it does |
|-------|--------------|
| `domain-modeling` | Pins down the ubiquitous language and records architectural decisions. Pairs with our domain-first `CLAUDE.md`. |
| `codebase-design` | Shared vocabulary for designing deep modules — use when improving an interface or deciding where a seam goes. |
| `improve-codebase-architecture` | Scans the codebase for "deepening" opportunities and presents them as a visual report, then grills through the one you pick. |

**Build & verify**

| Skill | What it does |
|-------|--------------|
| `tdd` | The red-green-refactor loop — write the failing test first, then the code. |
| `diagnosing-bugs` | A structured diagnosis loop for hard bugs and performance regressions, instead of guess-and-check. |
| `thermo-nuclear-review` | An extremely strict security & correctness audit of a branch's diff (bugs, breaking changes, feature-gate leaks). |
| `thermo-nuclear-code-quality-review` | The maintainability counterpart — abstraction quality, giant files, spaghetti conditionals. |
| `thermos` | Runs both thermo-nuclear reviews in parallel and synthesises the findings. Reach for this before a big MR. |

**Session hygiene**

| Skill | What it does |
|-------|--------------|
| `handoff` | Compacts a long conversation into a handoff document so a fresh session (or a teammate) can pick up without losing context. |
| `teach` | Walks you through a new concept or skill inside the current workspace. |

> Run `setup-matt-pocock-skills` once per repo before first use — it wires these skills to our issue tracker (GitLab), triage labels, and domain-doc layout.

Recommended path: `grilling` → `to-prd` → `implement` (with `tdd`) → `thermos`. The first two cost minutes and routinely save hours of building the wrong thing.

Whichever you pick, the team baseline in `tms-skills` (branching, commits, MRs, reviews) stays the same underneath — these process skills sit on top of it.

## MCP servers

[Model Context Protocol](https://modelcontextprotocol.io) servers connect the agent to our tooling. Configure them per-project in `.claude/mcp.json` (or globally), and provide the tokens via environment variables — never hardcode them.


| Server | Used for |
|--------|----------|
| **Sentry** (self-hosted) | Reading production errors, traces, and breadcrumbs — pairs with `sentry-fix-issues`. |
| **GitLab** (self-hosted) | MRs, pipelines, issues — pairs with `create-mr` / `iterate-mr`. |
| **Figma** | Design hand-off — pairs with `implement-design`. Run the Figma desktop app's local MCP server. |
| **Atlassian (Jira)** | Reading tickets, creating tasks, status reports. Available as the official `atlassian` plugin. |
| **SonarQube** | Pulling static-analysis findings into the agent. |

Project-specific official plugins we use where relevant: `datadog` (metrics/APM, see [Metrics and Logging](metrics_and_logging.md)), `expo` (React Native apps), `skill-creator` (authoring skills).

## A typical workflow

For a standard feature or bug fix, the loop maps cleanly onto our skills:

1. **Branch** — `/create-branch` with the Jira key → `task/UNIGHT-760_update_login_screen`.
2. **Implement** — describe the task. Point the agent at the relevant handbook guide ("follow our [Service Objects](service_objects.md) pattern") and let `CLAUDE.md` carry the rest. Write or update specs as you go.
3. **Verify** — run the linters and the suite using the exact commands from `CLAUDE.md`. The agent fixes what it broke.
4. **Commit** — `/commit` for a Conventional Commit message.
5. **Open the MR** — `/create-mr`. It references the Jira issue and follows our description format.
6. **Get it green** — `/iterate-mr` watches the pipeline and addresses review comments until everything passes.
7. **Review** — before requesting human review, run `/security-review` on the diff (and `/rails-audit` for larger backend changes).

Human review still happens. The agent makes the diff smaller and the description better; it does not replace the reviewer.

## Sandboxed execution

We run Claude Code with **OS-level sandboxing enabled and enforced** for the whole team. This is pushed centrally as managed settings — it is not something each developer opts into. The relevant policy:

```json
{
  "sandbox": {
    "enabled": true,
    "allowUnsandboxedCommands": false,
    "excludedCommands": ["git", "docker", "lefthook"],
    "filesystem": {
      "allowRead": [".env", ".env.local", ".env.development", ".env.test"],
      "allowWrite": ["~/.agent-browser"]
    },
    "network": {
      "allowLocalBinding": true,
      "allowUnixSockets": ["~/.docker/run/docker.sock"],
      "allowedDomains": ["localhost", "127.0.0.1", "postgres", "db",
                         "github.com", "*.npmjs.org", "registry.yarnpkg.com", "*.themindstudios.com"]
    }
  }
}
```

### Why we sandbox

The sandbox confines every command the agent runs to an allow-listed set of files and network hosts. Two things come out of that, and they reinforce each other:

1. **Speed.** Because the agent physically *cannot* read outside the project, write outside a tiny allow-list, or reach an unapproved host, it is safe to let it run commands without a permission prompt for each one. Without a sandbox, working safely means approving (or denying) command after command; with it, the agent stays in flow and you stay out of the prompt loop.
2. **Containment.** A confused agent, a prompt-injection payload in a web page, or a malicious dependency cannot exfiltrate data or trash your machine. Network egress is limited to the hosts we actually need — our registries, GitHub, and `*.themindstudios.com` — so even if something tries to phone home, it can't.

### How the policy is shaped

- **`enabled: true` + `allowUnsandboxedCommands: false`** — sandboxing is mandatory; the agent cannot escape it.
- **`excludedCommands: [git, docker, lefthook]`** — these legitimately need broader access (git talks to remotes, docker to its daemon socket, lefthook runs our git hooks), so they run outside the sandbox by design. Everything else is confined.
- **`filesystem`** — sandboxed commands may read the dev env files needed to run the app (`.env`, `.env.local`, `.env.development`, `.env.test`) but are otherwise read-confined to the project, and may only write to a single scratch directory (`~/.agent-browser`). Production env files (`.env.production`) are *not* in the read list. Note that these files being readable by *commands* does not mean Claude can read them into the conversation — the permission rules below block that (see the note on `.env` appearing in both lists).
- **`network.allowedDomains`** — only our package registries, GitHub, local services (Postgres/Redis under `localhost`/`db`/`postgres`), and our own domain. Anything else is blocked.
- **`allowUnixSockets`** — grants the Docker socket so the agent can drive our `docker-compose` workflows without leaving the sandbox.

### Reinforced by managed permission rules

The same managed settings layer adds hard guardrails that sit *outside* anyone's local config:

```json
{
  "allowManagedPermissionRulesOnly": true,
  "permissions": {
    "deny": [
      "Read(./.env)", "Read(./.env.production)", "Read(./.env.local)",
      "Read(./secrets/**)", "Read(./config/credentials.*)",
      "Bash(rm -rf *)", "Bash(git push --force *)"
    ],
    "disableBypassPermissionsMode": "disable"
  }
}
```

- Reading production secrets and force-pushing are **denied for everyone** — a developer cannot grant them locally (`allowManagedPermissionRulesOnly` and `disableBypassPermissionsMode` lock this down).
- Reminder of the standing org rule, surfaced in-session: **all code requires review before merge**, and **AI-related security concerns go to the security team**.

> **Why `.env` appears in both lists.** The sandbox `allowRead` and the permission `deny` act at different layers and don't conflict. The sandbox `allowRead` lets *commands the agent runs* (the app, the test suite) load `.env*` at runtime via dotenv; the permission `deny` stops the *Read tool* from pulling those same files into the conversation. So the app can boot with its env, but the secret values never enter the model's context.

If a command you legitimately need is being blocked, raise it with the team rather than working around the sandbox — the allow-lists are meant to grow deliberately.

## Security & guardrails

- **Never paste secrets into a prompt.** API keys, tokens, customer data, and `.env` contents stay out of the conversation. Provide credentials to MCP servers via environment variables. See [Store Secrets](store_secrets.md).
- **Scan skills before installing.** Run `/skill-scanner` on any skill from outside `tms-skills`. Skills can run code — treat them as dependencies.
- **Use permission scoping.** Pre-approve only safe, read-only commands in `.claude/settings.json`. Example:

  ```json
  {
    "permissions": {
      "allow": ["Bash(git status:*)", "Bash(git log:*)", "Bash(git diff:*)",
                "Bash(git branch:*)", "Bash(gh:*)", "Bash(ls:*)"]
    }
  }
  ```

- **Review every diff.** The agent is confident even when wrong. Read the change, run it, and check it against this handbook before it leaves your machine.
- **`.claude/settings.local.json` is personal** and git-ignored; shared, committed config goes in `.claude/settings.json` and `.claude/mcp.json`.

## Contributing skills

Shared skills live in the skills repo under `plugins/tms-skills/skills/<skill-name>/SKILL.md`. To add or improve one:

- Use lowercase-hyphenated directory names: `skill-name/`.
- Each skill needs a `SKILL.md` with YAML frontmatter (`name`, `description`); scope `allowed-tools` as narrowly as the task allows.
- Write the `description` for *matching* — list the trigger phrases a teammate would actually type.
- Keep skill docs under ~500 lines; put scripts and executables alongside `SKILL.md`.
- The `skill-creator` plugin scaffolds the structure for you.

Open a merge request like any other change. If a prompt is being reused across the team, it belongs here.
