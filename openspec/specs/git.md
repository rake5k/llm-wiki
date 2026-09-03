# Spec: Git Commit Safety (Concurrent Writers)

## Description

The wiki repository has more than one writer. Besides the `/llm-wiki` command in the current
session, a wiki graph is commonly written by: another Claude Code session running `/llm-wiki`
in a different project, a Logseq/Obsidian git plugin that auto-commits on a timer, and
the user editing pages in the app right now. At any moment the working tree can therefore
hold uncommitted edits that this run did not make, and `HEAD` can have moved since this
run last looked.

Two consequences, both of which have caused real data loss:

1. A whole-tree stage (`git add -A`, `git add .`, `git commit -a`) sweeps another writer's
   in-progress page edits into this run's commit — an unreviewed, unrelated change
   attributed to `/llm-wiki`.
2. A history rewrite (`git reset`, `git commit --amend`, force-push) drops any commit
   another writer created on top of this run's commit — recoverable only via the reflog.

Every `/llm-wiki` command that commits (`ingest`, `prune`, `lint --fix`, `import`) and
setup (`setup.sh` and `scripts/init-wiki.sh`, which performs the initial commit) follow the
requirements below.

---

## Requirements

### Staging

- REQ-900: A command that commits SHALL stage only the paths it wrote itself, by explicit
  pathspec: `git add -- <path> ...`.
- REQ-901: A command MUST NOT use `git add -A`, `git add .`, `git add :/`, or
  `git commit -a` in the wiki repository.
- REQ-902: The commit SHALL carry an explicit pathspec (`git commit -m "<msg>" -- <path> ...`)
  so that content another writer left staged in the index is not included.
- REQ-903: The set of committed paths SHALL equal the set of pages reported to the user in
  that run's summary (plus the Access-Log when its pending appends ride along, per
  specs/query.md REQ-451).

### Working-Tree Awareness

- REQ-910: Before committing, a command SHOULD run `git status --porcelain` and treat any
  modified path it did not write as another writer's work in progress: it SHALL leave that
  path unstaged and SHALL mention it in the report.
- REQ-911: A command MUST NOT require a clean working tree, MUST NOT stash, and MUST NOT
  revert foreign changes to obtain one.

### History

- REQ-920: A command MUST NOT rewrite wiki history: no `git reset` (any mode), no
  `git commit --amend`, no rebase, no force-push.
- REQ-921: A commit that turns out to be wrong SHALL be corrected by a NEW commit
  (`git revert <sha>` when it must be undone).
- REQ-922: A command MUST NOT delete or expire the reflog, which is the only recovery path
  for a commit lost to a rewrite.

### Setup

- REQ-930: `scripts/init-wiki.sh` SHALL record every file it creates (pages, `.gitignore`,
  `llm-wiki.yml`) in a manifest and stage the initial commit from that manifest alone.
- REQ-931: A file setup skipped because it already existed MUST NOT be staged — it may
  hold another writer's uncommitted edits.
- REQ-932: When the manifest is empty (nothing created), setup SHALL make no commit.

---

## Scenarios

### Scenario 1: Ingest commits alongside a concurrent editor

```
GIVEN /llm-wiki ingest wrote Wiki/Tech/Strapi and Wiki/Tech (hub)
AND the user has an uncommitted edit in Wiki/People/Client-X from the Logseq app
WHEN the command commits
THEN it runs `git add -- <Strapi page> <Tech hub>`
AND `git commit -m "wiki: ..." -- <Strapi page> <Tech hub>`
AND Wiki/People/Client-X stays modified and unstaged
AND the report notes the foreign uncommitted path
```

### Scenario 2: Another session commits on top

```
GIVEN this run created commit A
AND another session then created commit B on top of A
WHEN this run notices its commit message was wrong
THEN it MUST NOT run `git reset --soft HEAD~1` or `git commit --amend`
   (either would drop B)
AND it creates a new commit (or `git revert A`) instead
```

### Scenario 3: Setup over an existing graph

```
GIVEN the wiki already had Wiki/Schema.md with local uncommitted changes
AND setup skips it and creates only the hub pages and llm-wiki.yml
WHEN the initial commit runs
THEN only the created hub pages and llm-wiki.yml are staged and committed
AND Wiki/Schema.md keeps its uncommitted changes
```

---

## Acceptance Criteria

- [ ] No `git add -A` / `git add .` / `git commit -a` anywhere in the skill or the setup scripts
- [ ] Every commit carries an explicit pathspec on both `add` and `commit`
- [ ] Committed paths match the run's reported page list
- [ ] Foreign uncommitted paths survive a `/llm-wiki` commit and are reported
- [ ] No `git reset` / `--amend` / force-push in any workflow
- [ ] Setup commits from its created-file manifest; skipped files are not staged
- [ ] Setup makes no commit when it created nothing

---

## Dependencies

- specs/ingest.md REQ-052, specs/prune.md REQ-621, specs/lint.md REQ-203, and
  specs/setup.md REQ-810 are the commit points these requirements constrain
- specs/query.md REQ-451 defines the non-structural Access-Log append that rides along
  with the next commit
