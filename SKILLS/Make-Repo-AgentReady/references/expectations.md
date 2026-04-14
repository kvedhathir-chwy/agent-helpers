Use this document as a **checklist** when improving a repo. 

---

## 1. Machine-readable working instructions

Agents do better when the repo states **how to work in it** in a file they can load reliably. Examples include **AGENTS.md** (OpenAI Codex and nested layouts), repository instruction files (GitHub Copilot), and **CLAUDE.md** (Claude Code). These files should answer navigation, commands, standards, and boundaries—not tribal knowledge buried in chat.

A strong instruction file usually covers:

- Repo purpose and a concise architecture map
- Canonical **build**, **test**, **lint**, and **typecheck** commands
- Code style and naming expectations
- Forbidden or high-risk areas (secrets, generated code, legacy modules)
- How to run local services or dependencies
- What “done” means for a change (tests, docs, migrations)
- How you expect PRs, changelogs, or migrations to look

**Examples:** 
[Industry recommendations: https://agents.md/#examples]
[`examples/cid-insight-service/AGENTS.md`](examples/cid-insight-service/AGENTS.md), [`examples/firebird-promo-service/AGENTS.md`](examples/firebird-promo-service/AGENTS.md)


---

## 2. Deterministic setup and validation

If a human needs undocumented steps to run the project, an agent will struggle too. Prefer **one-command setup** and **one-command validation**, with the same entrypoints documented in your agent instructions:

```text
make setup
make test
make lint
make typecheck
make ci
```

(Substitute `pnpm`, `gradle`, or `just`—what matters is **discoverability** and **repeatability**.) Clear dev environments and reliable tests improve agent outcomes measurably.

---

## 3. Low-entropy layout

Reduce time spent inferring where code and config live. Favor:

- Clear package or service boundaries
- Predictable folder names and ownership (CODEOWNERS, README per package)
- Examples and fixtures **next to** the code they exercise
- A small set of **sanctioned entrypoints** (API surface, CLI, jobs)

The goal is **searchability** and fast mental mapping—not cosmetic tidiness alone.

---

## 4. Trustworthy tools and context (MCP)

When agents need databases, internal docs, feature flags, or deployment context, avoid one-off prompt hacks. Prefer **[Model Context Protocol (MCP)](https://modelcontextprotocol.io/)** so tools, resources, and prompts are **explicit and versionable** in the environment.

Typical uses:

- **Resources:** schemas, configs, runbooks
- **Tools:** safe, scoped internal actions
- **Workflows:** repeatable operations (e.g. run integration suite, open a ticket, generate a migration plan)

---

## 5. Guardrails in the repo, not only in the prompt

Prompt-only rules are easy to forget. **Repo-enforced** rules stick:

- Pre-commit hooks
- Linters and formatters
- Strict typing where applicable
- Unit and integration tests
- Policy checks (secrets, licenses)
- Branch protections and required CI gates

The agent can experiment; the pipeline should make **unsafe or sloppy changes hard to merge**.

---

## 6. Task scaffolds agents can run

Decompose repeat work into **scripts** so the agent does not reverse-engineer prose from a wiki:

```text
scripts/bootstrap.sh
scripts/run_e2e.sh
scripts/check_changed_files.sh
scripts/create_dev_db.sh
```

Name them consistently and reference them from `AGENTS.md` (or equivalent).

---

## 7. Cheap, targeted verification

An agent is only useful if it can **prove** a change works. Aim for:

- Fast unit tests (roughly minutes, not half-hour loops)
- Narrow targets (by package, file, or tag)
- Deterministic fixtures
- Reproducible local CI parity
- Snapshot or golden tests where they reduce flake

Long, flaky validation makes the agent loop expensive and encourages shallow fixes.

---

## 8. Repository memory at the right granularity

Codex supports **nested `AGENTS.md`** files: the closest file in the directory tree applies, which scales well in **monorepos**.

A common pattern:

| Location | Role |
|----------|------|
| `/AGENTS.md` | Org-wide rules and global commands |
| `/services/payments/AGENTS.md` | Domain-specific commands and risks |
| `/frontend/AGENTS.md` | UI conventions and test commands |

Keep local files **short** and **specific**; avoid duplicating the entire top-level guide.

---

## 9. A safe delivery lane

For sustained agentic workflows, pair the repo with **least-privilege** and **disposable** environments:

- Ephemeral dev environments
- Scoped credentials and short-lived tokens
- Sandboxed CI runners
- Preview deployments for PRs
- Automated PR creation from agent branches

The agent should be able to **read, change, test, and propose** without broad production access.

---

## Related

- **[`../SKILLS/README.md`](../SKILLS/README.md)** — reusable agent skills (e.g. PR security review, auditing third-party skills) that complement repo structure.

---

## Disclaimer

Recommendations here are **patterns**, not a guarantee of security or correctness. Adapt commands, paths, and tooling to your stack and organizational policies.
