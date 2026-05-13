---
name: setup-matt-pocock-skills
description: Sets up an `## Agent skills` block in AGENTS.md/CLAUDE.md and `docs/agents/` so the engineering skills know this repo's issue tracker (GitHub, GitLab, local markdown, or an Obsidian vault), triage label vocabulary, and domain doc layout. Run before first use of `to-issues`, `to-prd`, `triage`, `diagnose`, `tdd`, `improve-codebase-architecture`, or `zoom-out` — or if those skills appear to be missing context about the issue tracker, triage labels, or domain docs.
disable-model-invocation: true
---

# Setup Matt Pocock's Skills

Scaffold the per-repo configuration that the engineering skills assume:

- **Issue tracker** — where issues live (GitHub by default; GitLab, local markdown, and a personal Obsidian vault are also supported)
- **Triage labels** — the strings used for the five canonical triage roles
- **Domain docs** — where `CONTEXT.md` and ADRs live, and the consumer rules for reading them

This is a prompt-driven skill, not a deterministic script. Explore, present what you found, confirm with the user, then write.

> **Two storage modes.** For GitHub / GitLab / local-markdown, configuration lives in `docs/agents/*.md` inside the repo. For Obsidian, configuration lives in the user's personal config at `~/.config/matt-pocock-skills/repos.toml` and **nothing is written to the repo** — designed for work repos where you can't commit agent-generated files.

## Process

### 1. Explore

Look at the current repo to understand its starting state. Read whatever exists; don't assume:

- `git remote -v` and `.git/config` — is this a GitHub repo? Which one?
- `AGENTS.md` and `CLAUDE.md` at the repo root — does either exist? Is there already an `## Agent skills` section in either?
- `CONTEXT.md` and `CONTEXT-MAP.md` at the repo root
- `docs/adr/` and any `src/*/docs/adr/` directories
- `docs/agents/` — does this skill's prior output already exist?
- `.scratch/` — sign that a local-markdown issue tracker convention is already in use

### 2. Present findings and ask

Summarise what's present and what's missing. Then walk the user through the three decisions **one at a time** — present a section, get the user's answer, then move to the next. Don't dump all three at once.

Assume the user does not know what these terms mean. Each section starts with a short explainer (what it is, why these skills need it, what changes if they pick differently). Then show the choices and the default.

**Section A — Issue tracker.**

> Explainer: The "issue tracker" is where issues live for this repo. Skills like `to-issues`, `triage`, `to-prd`, and `qa` read from and write to it — they need to know whether to call `gh issue create`, write a markdown file under `.scratch/`, or follow some other workflow you describe. Pick the place you actually track work for this repo.

Default posture: these skills were designed for GitHub. If a `git remote` points at GitHub, propose that. If a `git remote` points at GitLab (`gitlab.com` or a self-hosted host), propose GitLab. Otherwise (or if the user prefers), offer:

- **GitHub** — issues live in the repo's GitHub Issues (uses the `gh` CLI)
- **GitLab** — issues live in the repo's GitLab Issues (uses the [`glab`](https://gitlab.com/gitlab-org/cli) CLI)
- **Local markdown** — issues live as files under `.scratch/<feature>/` in this repo (good for solo projects or repos without a remote)
- **Obsidian (local vault)** — issues live as markdown files inside a personal Obsidian vault on disk, outside the repo. Nothing is written to the repo. Good for work repos where you can't commit issue files or can't access the native tracker. See the Obsidian sub-flow below.
- **Other** (Jira, Linear, etc.) — ask the user to describe the workflow in one paragraph; the skill will record it as freeform prose

#### Obsidian sub-flow (only if user picks Obsidian)

Run these steps inline before moving to Section B.

**A.1 — Detect vaults.**

Read the canonical Obsidian vault registry first:

- macOS: `~/Library/Application Support/obsidian/obsidian.json`
- Linux: `~/.config/obsidian/obsidian.json`
- Windows: `%APPDATA%\obsidian\obsidian.json`

Parse the `vaults` object — each entry has `{ path, ts, open? }`. Collect those.

If the file is missing or `vaults` is empty, **fall back to a filesystem scan**: look for `.obsidian/` directories up to depth 4 under these roots, in this order:

- `~/Documents`
- `~/Notes`
- `~/Obsidian`
- `~/Vaults`
- `~/Library/Mobile Documents/iCloud~md~obsidian/Documents` (macOS iCloud Obsidian)
- `~` (depth 2 only)

Exclude `node_modules`, `.git`, `.Trash`, and the macOS `Library` tree (except the iCloud path above). Use `realpath` to dedupe, since iCloud + symlinks can produce duplicates. Tag each result with its origin: `registered` (from `obsidian.json`) or `found` (from the scan).

**A.2 — Pick a vault.**

Present the vaults as a numbered list. Sort: vaults currently open first (`open: true`), then by `ts` descending (most recently opened first). For each row, show:

- Index (1-based)
- Basename of the path (e.g. `MyVault`)
- Absolute path
- Origin tag (`registered` / `found`)
- Whether it's currently open

The default selection is `1` (top of the list). Also offer `0` for "none of these, let me paste a path".

**A.3 — Pick a subpath base.**

Ask the user where inside the vault the per-repo folders should live. Default suggestion: `Work/Issues`. Anything they type is taken verbatim and joined to the vault path.

**A.4 — Pick a repo slug.**

Default: `basename` of `git rev-parse --show-toplevel`. Confirm with the user — they can override.

The full target inside the vault becomes `<vault>/<subpath_base>/<repo_slug>/` (this string is stored as `repo_root_in_vault` in the TOML).

**A.5 — Dataview install offer.**

Look for `<vault>/.obsidian/community-plugins.json`. If it does not list `"dataview"`, offer to install it. With the user's explicit confirmation:

1. Download the latest release tarball from `https://github.com/blacksmithgu/obsidian-dataview/releases/latest` — files needed: `main.js`, `manifest.json`, `styles.css`.
2. Place them in `<vault>/.obsidian/plugins/dataview/`.
3. Add `"dataview"` to the array in `<vault>/.obsidian/community-plugins.json` (create the file with `["dataview"]` if absent).
4. Tell the user to reload Obsidian (Cmd-R inside Obsidian, or close and reopen) for the plugin to activate.

If the user declines, continue — Dataview is only for nicer browsing of `_index.md`; the skills do not depend on it.

If the vault has `restrictedMode: true` set in `<vault>/.obsidian/app.json`, do **not** flip it silently — ask the user to disable Restricted Mode manually in Obsidian's Community plugins settings.

**A.6 — Domain docs decision.**

When the backend is Obsidian, also ask up-front whether `CONTEXT.md` and ADRs should live in the vault (next to the issues) or stay in the repo. Default: **in the vault**. If they go in the vault, they live at `<vault>/<repo_root_in_vault>/CONTEXT.md` and `<vault>/<repo_root_in_vault>/ADRs/`. Record this in the TOML as `domain_docs_in_vault = true|false`. Section C below is treated as confirmed by this decision — don't re-ask there.

**Section B — Triage label vocabulary.**

> Explainer: When the `triage` skill processes an incoming issue, it moves it through a state machine — needs evaluation, waiting on reporter, ready for an AFK agent to pick up, ready for a human, or won't fix. To do that, it needs to apply labels (or the equivalent in your issue tracker) that match strings *you've actually configured*. If your repo already uses different label names (e.g. `bug:triage` instead of `needs-triage`), map them here so the skill applies the right ones instead of creating duplicates.

The five canonical roles:

- `needs-triage` — maintainer needs to evaluate
- `needs-info` — waiting on reporter
- `ready-for-agent` — fully specified, AFK-ready (an agent can pick it up with no human context)
- `ready-for-human` — needs human implementation
- `wontfix` — will not be actioned

Default: each role's string equals its name. Ask the user if they want to override any. If their issue tracker has no existing labels, the defaults are fine.

**Section C — Domain docs.**

> Explainer: Some skills (`improve-codebase-architecture`, `diagnose`, `tdd`) read a `CONTEXT.md` file to learn the project's domain language, and `docs/adr/` for past architectural decisions. They need to know whether the repo has one global context or multiple (e.g. a monorepo with separate frontend/backend contexts) so they look in the right place.

If the user picked Obsidian in Section A and confirmed `domain_docs_in_vault = true` in step A.6, **skip this section entirely** — the vault layout is single-context by construction, and the location was already decided.

Otherwise, confirm the layout:

- **Single-context** — one `CONTEXT.md` + `docs/adr/` at the repo root. Most repos are this.
- **Multi-context** — `CONTEXT-MAP.md` at the root pointing to per-context `CONTEXT.md` files (typically a monorepo).

### 3. Confirm and edit

Show the user a draft of what's about to be written. The exact set depends on the backend:

- **GitHub / GitLab / local / other**: the `## Agent skills` block for `CLAUDE.md`/`AGENTS.md`, plus the three `docs/agents/*.md` files.
- **Obsidian**: the TOML entry for `~/.config/matt-pocock-skills/repos.toml`, plus (if domain docs go in the vault) initial empty `CONTEXT.md` and `_index.md` files inside `<vault>/<repo_root_in_vault>/`. Nothing is written to the repo.

Let the user edit before writing.

### 4. Write

#### 4a. Backend = GitHub / GitLab / local / other

**Pick the file to edit:**

- If `CLAUDE.md` exists, edit it.
- Else if `AGENTS.md` exists, edit it.
- If neither exists, ask the user which one to create — don't pick for them.

Never create `AGENTS.md` when `CLAUDE.md` already exists (or vice versa) — always edit the one that's already there.

If an `## Agent skills` block already exists in the chosen file, update its contents in-place rather than appending a duplicate. Don't overwrite user edits to the surrounding sections.

The block:

```markdown
## Agent skills

### Issue tracker

[one-line summary of where issues are tracked]. See `docs/agents/issue-tracker.md`.

### Triage labels

[one-line summary of the label vocabulary]. See `docs/agents/triage-labels.md`.

### Domain docs

[one-line summary of layout — "single-context" or "multi-context"]. See `docs/agents/domain.md`.
```

Then write the three docs files using the seed templates in this skill folder as a starting point:

- [issue-tracker-github.md](./issue-tracker-github.md) — GitHub issue tracker
- [issue-tracker-gitlab.md](./issue-tracker-gitlab.md) — GitLab issue tracker
- [issue-tracker-local.md](./issue-tracker-local.md) — local-markdown issue tracker
- [triage-labels.md](./triage-labels.md) — label mapping
- [domain.md](./domain.md) — domain doc consumer rules + layout

For "other" issue trackers, write `docs/agents/issue-tracker.md` from scratch using the user's description.

#### 4b. Backend = Obsidian

**Do not touch the repo.** No `## Agent skills` block, no `docs/agents/` files. All configuration lives in the user's personal config.

1. Create `~/.config/matt-pocock-skills/` if it doesn't exist.
2. Read `~/.config/matt-pocock-skills/repos.toml` if it exists. If a section already keyed by this repo's absolute root is present, update it in place; otherwise append a new section.
3. The section schema (all keys required unless noted):

   ```toml
   ["<absolute-repo-root>"]
   backend = "obsidian"
   vault = "<absolute-vault-path>"
   repo_root_in_vault = "<subpath_base>/<repo_slug>"
   domain_docs_in_vault = true   # or false if the user kept CONTEXT.md / ADRs in the repo
   triage_labels = { needs-triage = "needs-triage", needs-info = "needs-info", ready-for-agent = "ready-for-agent", ready-for-human = "ready-for-human", wontfix = "wontfix" }
   ```

   If the user overrode any of the five triage roles in Section B, reflect those overrides in `triage_labels`.

4. Create the folders inside the vault: `<vault>/<repo_root_in_vault>/Issues/` and `<vault>/<repo_root_in_vault>/PRDs/`. If `domain_docs_in_vault = true`, also create `<vault>/<repo_root_in_vault>/ADRs/` and seed an empty `<vault>/<repo_root_in_vault>/CONTEXT.md`.
5. Seed `<vault>/<repo_root_in_vault>/_index.md` with Dataview blocks for navigation. Template:

   ```markdown
   # <repo_slug>

   > Issues, PRDs, and domain docs for the `<repo_slug>` repository.

   ## Needs triage

   ```dataview
   TABLE WITHOUT ID file.link AS Issue, status, file.ctime AS Created
   FROM "<repo_root_in_vault>/Issues"
   WHERE status = "needs-triage"
   SORT file.ctime DESC
   ```

   ## Ready for an agent

   ```dataview
   TABLE WITHOUT ID file.link AS Issue, prd
   FROM "<repo_root_in_vault>/Issues"
   WHERE status = "ready-for-agent"
   SORT file.ctime DESC
   ```

   ## All open issues

   ```dataview
   TABLE WITHOUT ID file.link AS Issue, status, prd
   FROM "<repo_root_in_vault>/Issues"
   WHERE status != "wontfix"
   SORT id DESC
   ```

   ## PRDs

   ```dataview
   LIST FROM "<repo_root_in_vault>/PRDs"
   ```
   ```

   (Substitute `<repo_root_in_vault>` and `<repo_slug>` with the real values.)

6. Reference: [issue-tracker-obsidian.md](./issue-tracker-obsidian.md) describes the conventions the consumer skills follow when reading and writing in this layout. Do not copy it anywhere — its content is the operational doc for the consumer skills via the TOML pointer.

### 5. Done

Tell the user the setup is complete and which engineering skills will now read from this configuration.

- **GitHub / GitLab / local / other**: they can edit `docs/agents/*.md` directly later. Re-running this skill is only necessary to switch backends.
- **Obsidian**: they can edit `~/.config/matt-pocock-skills/repos.toml` directly later. Re-running this skill is only necessary to switch backends or repoint the vault.

Mention that if they installed Dataview just now, they need to reload Obsidian for the queries in `_index.md` to render.
