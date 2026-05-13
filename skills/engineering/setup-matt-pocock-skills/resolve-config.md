# Resolve per-repo config

Engineering skills read this doc before touching the issue tracker or domain docs. It describes where the config lives and how to discover the right paths for the current repo.

## Two storage modes

1. **Repo-local config** (GitHub / GitLab / local-markdown / "other" trackers) — config files live in `docs/agents/*.md` inside the repo.
2. **User-personal config** (Obsidian backend) — config lives in `~/.config/matt-pocock-skills/repos.toml` outside the repo. The repo itself is untouched.

A given repo uses exactly one mode. Both can coexist across different repos on the same machine.

## Resolution algorithm

Run these in order. Stop at the first that resolves.

### Step 1 — Personal config (TOML)

1. Get the absolute repo root: `git rev-parse --show-toplevel`.
2. Read `~/.config/matt-pocock-skills/repos.toml`. If missing, skip to Step 2.
3. Find the section whose key equals the repo root. If none, skip to Step 2.
4. Read `backend`. Supported value: `"obsidian"`. For an Obsidian section, use:
   - `vault` — absolute path to the Obsidian vault root.
   - `repo_root_in_vault` — subpath inside the vault where this repo's files live. Join: `<vault>/<repo_root_in_vault>`.
   - `domain_docs_in_vault` — boolean. If `true`, `CONTEXT.md` is at `<vault>/<repo_root_in_vault>/CONTEXT.md` and ADRs are under `<vault>/<repo_root_in_vault>/ADRs/`. If `false`, fall back to the repo for domain docs (`CONTEXT.md` at the repo root, `docs/adr/` under it).
   - `triage_labels` — map of canonical role → actual label string. If a role is missing, use the canonical name.
   - Conventions for reading, writing, and querying issues / PRDs in this layout: see [issue-tracker-obsidian.md](./issue-tracker-obsidian.md).

### Step 2 — Repo-local config (`docs/agents/`)

1. Read `docs/agents/issue-tracker.md` if it exists. Follow its conventions (GitHub `gh`, GitLab `glab`, local `.scratch/`, or freeform "other").
2. Read `docs/agents/triage-labels.md` for the label vocabulary.
3. Read `docs/agents/domain.md` to learn the single-context / multi-context layout for `CONTEXT.md` and `docs/adr/`.

### Step 3 — Neither found

Tell the user: "I couldn't find a per-repo config for this repo. Run `/setup-matt-pocock-skills` first so I know where to publish issues and find domain docs." Stop.

## Cheat sheet for callers

| Skill needs to…                | Obsidian backend                                          | Repo-local backend                       |
| ------------------------------ | --------------------------------------------------------- | ---------------------------------------- |
| Publish an issue               | New file under `<vault>/<repo_root_in_vault>/Issues/`     | Per `docs/agents/issue-tracker.md`       |
| Publish a PRD                  | New file under `<vault>/<repo_root_in_vault>/PRDs/`       | Per `docs/agents/issue-tracker.md`       |
| Fetch an issue by `#42`        | `<vault>/<repo_root_in_vault>/Issues/0042-*.md`           | `gh issue view 42` / `glab issue view 42` / file lookup |
| List issues by triage state    | `rg -l '^status: <role>$' <vault>/<repo_root_in_vault>/Issues/` | Per tracker (`gh issue list --label …`)  |
| Read `CONTEXT.md`              | `<vault>/<repo_root_in_vault>/CONTEXT.md` if `domain_docs_in_vault`, else repo root | Repo root (or per `CONTEXT-MAP.md`)      |
| Read ADRs                      | `<vault>/<repo_root_in_vault>/ADRs/` if `domain_docs_in_vault`, else `docs/adr/` | `docs/adr/`                              |
| Apply a triage label           | Mutate `status:` field + `status/<role>` tag in frontmatter | Per tracker                              |

When in doubt, the operational details for the Obsidian layout (frontmatter schema, ID assignment, comment append format, etc.) live in [issue-tracker-obsidian.md](./issue-tracker-obsidian.md).
