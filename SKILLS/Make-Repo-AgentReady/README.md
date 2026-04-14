# Make Repo Agent-Ready

Cursor skill **`make-repo-agentready`** bootstraps a repository so AI agents can work with **explicit instructions**, **one-command workflows**, and **documented structure**.

## What it adds

| Artifact | Role |
|----------|------|
| **`AGENTS.md`** (repo root) | Purpose, stack, commands table, layout, conventions, done criteria, PR expectations, risky areas, **nested `AGENTS.md` guidance** for monorepos |
| **`docs/architecture.md`** | Concise system architecture (as-built or scaffolded with TODOs) |
| **`Makefile`** | `setup`, `test`, `lint`, `typecheck`, `ci` — wired to **real** toolchain commands after repo inspection |
| **`scripts/*.sh`** | `bootstrap.sh`, `run_e2e.sh`, `check_changed_files.sh`, `create_dev_db.sh` — safe bash stubs wired from Make / `AGENTS.md` |

## When to use

- “Make this repo agent-ready,” “add AGENTS.md and Makefile,” or **`/make-repo-agentready`**
- Before onboarding agents to a brownfield codebase (skill **merges** with existing files when possible)

## Definition

- **[`SKILL.md`](./SKILL.md)** — full procedure for the agent

## Related

- **[`../references/expectations.md`](../references/expectations.md)** — principles this skill implements
- **[`../README.md`](../README.md)** — catalog of skills in **agent-helpers**
