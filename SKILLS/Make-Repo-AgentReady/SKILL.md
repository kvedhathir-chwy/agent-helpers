---
name: make-repo-agentready
description: >-
  Bootstraps a repository for AI coding agents by adding AGENTS.md (root),
  docs/architecture.md, a Makefile with setup/test/lint/typecheck/ci targets,
  and scripts/bootstrap.sh, run_e2e.sh, check_changed_files.sh, create_dev_db.sh.
  Covers low-entropy layout guidance and nested AGENTS.md patterns for monorepos.
  Use when the user asks to make a repo agent-ready, bootstrap agent instructions,
  scaffold AGENTS.md/Makefile/scripts, or invokes /make-repo-agentready.
---

# Make Repo Agent-Ready

Act as a **repo bootstrapper** for AI agents. Goal: add **machine-readable instructions**, **deterministic commands**, **task scaffolds**, and **documented layout** so agents spend less time guessing and more time shipping safe changes.

## When to apply

- User wants the repo “agent ready,” “Codex ready,” or aligned with **Repo-Augmentor** expectations.
- User invokes **`/make-repo-agentready`** or asks for `AGENTS.md`, `architecture.md`, `Makefile`, or `scripts/` scaffolding.
- Greenfield or brownfield: **prefer extending** existing files over blind overwrite.

## Non-negotiables

1. **Do not destroy existing behavior** — If `AGENTS.md`, `Makefile`, `docs/`, or `scripts/` already exist, **merge** sensible sections or **ask** before replacing wholesale.
2. **Ground commands in the real stack** — Inspect manifests (`package.json`, `pyproject.toml`, `requirements.txt`, `go.mod`, `pom.xml`, `build.gradle`, `Cargo.toml`, etc.) and CI (`.github/workflows`, `Jenkinsfile`) so Make targets and AGENTS.md tables reflect **actual** project commands.
3. **Create `docs/`** if missing; place **`docs/architecture.md`** there (not at repo root unless the repo already uses that convention).
4. **Reference scripts from `AGENTS.md`** after creating or updating them.
5. **Shell scripts** — Use `#!/usr/bin/env bash`, `set -euo pipefail`, and executable bits (`chmod +x`); avoid secrets in scripts; prefer `${VAR:-default}` for optional env.

---

## Step 1 — Machine-readable working instructions

### 1a. Root `AGENTS.md`

Create or update **`AGENTS.md`** at the **repository root** with at least:

- **Purpose** — What the project does in one short paragraph.
- **Stack** — Languages, frameworks, package manager (from evidence in repo).
- **Common commands** — Table mapping tasks to commands (build, test, lint, format, typecheck, run local).
- **Project layout** — High-signal paths (source roots, config, infra, `docs/`, tests).
- **Conventions** — Style, testing expectations, security or codegen notes if applicable.
- **Definition of done** — Tests/docs/migrations/changelog expectations for a change.
- **PR / changelog** — How the team expects PRs and release notes (or “follow CONTRIBUTING.md” if present).
- **Forbidden / high-risk** — Generated dirs, secrets, production configs, legacy areas agents should not edit blindly.

Add a subsection **Nested AGENTS.md (monorepos)** explaining Codex-style **closest-file-wins** behavior:

| Location        | Role                                      |
|----------------|-------------------------------------------|
| `/AGENTS.md`   | Org-wide rules and global commands        |
| `/path/pkg/AGENTS.md` | Package-specific commands and risks |

State that nested files should stay **short** and avoid duplicating the entire root guide.

### 1b. `docs/architecture.md`

Create or update **`docs/architecture.md`** with a **concise as-built** outline. If the codebase is large, start from inventory (entrypoints, modules, data flows) and fill sections; if unknown, scaffold with **`TODO`** markers and honest assumptions labeled as such.

Minimum sections (omit or rename to match repo norms):

1. **Overview** — Purpose and primary users/clients.
2. **Runtime & stack** — Languages, frameworks, build tool.
3. **Structure** — Main packages/services and responsibilities.
4. **Data & integrations** — DBs, queues, external APIs (at high level).
5. **Operational surfaces** — HTTP/gRPC/CLI/jobs; auth model if present in code/config.
6. **Testing** — How tests are organized; fast vs slow suites.
7. **Extension points** — Where new features typically land.

Link **`AGENTS.md`** to **`docs/architecture.md`** (“See `docs/architecture.md` for system structure.”) and vice versa if useful.

---

## Step 2 — Deterministic setup and validation (`Makefile`)

Create or extend a root **`Makefile`** with **phony** targets so agents and humans share one vocabulary:

| Target      | Intent |
|------------|--------|
| `setup`    | Install deps, tooling, git hooks if applicable; may call `./scripts/bootstrap.sh`. |
| `test`     | Run the canonical test command for the repo. |
| `lint`     | Run linters / format check. |
| `typecheck`| Static types (TypeScript, mypy, etc.); if N/A, document as no-op or skip with a message. |
| `ci`       | What CI approximates: often `lint` + `typecheck` + `test` (order per project norms). |

**Implementation rules:**

- Start with `.PHONY: setup test lint typecheck ci` (add others as needed).
- **Map targets** to real commands, e.g. `npm ci && npm run prepare`, `./gradlew check`, `poetry install && poetry run pytest`, `go test ./...`.
- If multiple apps exist in a monorepo, prefer **documenting** the root orchestration in `AGENTS.md` and delegating from Make to package scripts (`pnpm -r test`) or per-package `make -C subdir test`.
- If the stack has **no** typechecker, make `typecheck` print one line and exit 0, or document “not applicable” in `AGENTS.md`—do not invent a fake tool.

---

## Step 3 — Low-entropy layout (guidance, not blind moves)

In **`AGENTS.md`** (and optionally a short **“Layout”** section in `docs/architecture.md`), capture:

- Clear **package/service boundaries** and where new code belongs.
- **Predictable** top-level folders; call out any non-obvious names.
- **Examples/fixtures** next to code when that’s the project pattern.
- **Sanctioned entrypoints** (main API module, CLI entry, job runner).

**Do not** mass-rename directories without explicit user request; focus on **documenting** the as-is layout and naming conventions.

---

## Step 4 — Task scaffolds (`scripts/`)

Create **`scripts/`** (if missing) and add or update these **four** scripts. Wire them from **`Makefile`** and **`AGENTS.md`** where appropriate.

| Script | Purpose |
|--------|---------|
| `scripts/bootstrap.sh` | One-shot local setup: deps, env file copy from example, tool versions. Exit non-zero with a clear message if prerequisites are missing. |
| `scripts/run_e2e.sh` | Entry point for E2E or slow integration tests; may be a no-op stub that prints “configure E2E command” until the project defines one. |
| `scripts/check_changed_files.sh` | Run fast checks on **git changed** files only (lint/format on diff) to keep agent loops cheap; fall back to full lint if no diff tooling exists. |
| `scripts/create_dev_db.sh` | Create/reset local DB (docker compose, migrations, seed); safe defaults; no destructive prod behavior. |

Each script should:

- Be **idempotent** where reasonable (safe to re-run).
- Print **what it runs** at verbose level or in comments at top.
- Avoid **hardcoded secrets**; use env vars and document them in `AGENTS.md` or `.env.example`.

---

## Step 5 — Deliverables checklist

Before finishing, confirm:

- [ ] `AGENTS.md` at repo root with nested-`AGENTS.md` guidance.
- [ ] `docs/architecture.md` exists and is linked from `AGENTS.md`.
- [ ] `Makefile` with `setup`, `test`, `lint`, `typecheck`, `ci` (commands match the repo).
- [ ] `scripts/bootstrap.sh`, `scripts/run_e2e.sh`, `scripts/check_changed_files.sh`, `scripts/create_dev_db.sh` exist and are referenced.
- [ ] User gets a **short summary** in chat of what was created or updated and any **TODOs** left for them.

---

## What not to do

- Do not add real API keys, tokens, or internal URLs to templates.
- Do not claim **LGTM** security or production readiness; this is **developer experience** scaffolding.
- Do not skip **`docs/architecture.md`** when the user asked for full bootstrap—create a scaffold with TODOs if the system is unknown.

## Reference

Aligns with **[`../references/expectations.md`](../references/expectation.md)** and **[`SKILLS/README.md`](../README.md)**.
