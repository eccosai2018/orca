---
name: gstf-update-df
description: >-
  Pull the latest upstream changes from origin/main into the local main
  branch, then merge main into the working development branch — without
  losing local uncommitted work and without letting upstream "Orca" branding
  clobber this fork's "Dark Factory" rebrand. Use when asked to sync/update
  Dark Factory with upstream, pull latest main, or merge main into
  development.
---

# Update Dark Factory from upstream main

This skill repeats the sync workflow used to bring upstream `stablyai/orca`
changes (via `origin/main` on `eccosai2018/orca`) into this fork's
`development` branch, while preserving the Dark Factory rebrand described in
`CLAUDE.md`'s "Fork rebrand" section. Re-read that section before resolving
any conflicts — it is the authority on what should say "Dark Factory" vs.
what upstream constants/paths are deliberately left as "Orca".

## Preconditions

Run from the `dark_factory` repo root. Confirm you're actually looking at
this repo (`git remote -v` should show `eccosai2018/orca`) — this workspace
contains multiple unrelated repos side by side.

## Steps

1. **Check working tree state.** `git status`. Note any uncommitted changes
   on `development` — do not discard them. If any modified file is also
   likely to be touched by upstream (rebrand-adjacent files like `README.md`,
   `mobile/app.json`, or anything under `src/main/`, `src/shared/`,
   `config/electron-builder.config.cjs` that carries "Orca" → "Dark Factory"
   edits), stash just those files before merging:
   `git stash push -m "wip: <description>" -- <file> <file>`. Leave unrelated
   uncommitted work alone unless it also collides.

2. **Fetch and fast-forward local `main`.**
   ```
   git fetch origin
   git fetch origin main:main
   ```
   This updates the local `main` ref to `origin/main` without touching
   whatever branch is currently checked out. If `main` is currently checked
   out with local commits ahead of `origin/main`, stop and ask the user how
   to proceed — do not force-update a branch with unpushed local commits.

3. **Merge `main` into `development`.**
   ```
   git merge main --no-edit
   ```
   Expect conflicts in files that both the rebrand and upstream touched.

4. **Resolve conflicts using the rebrand-survives principle.** For every
   conflicted file:
   - Adopt upstream's (`main`'s) new code, functionality, and refactors —
     don't silently drop real upstream work by blindly taking "ours".
   - Wherever `development`'s side represents a Dark Factory rebrand change
     per `CLAUDE.md` (UI text/i18n strings, `productName` /
     `'DarkFactory'`, `BASE_APP_NAME` usages, `TERM_PROGRAM` env value, git
     attribution trailers/footers, `MAIN_RELEASE_REPO`, the Keychain service
     name, `package.json` homepage), that rename must survive — reapply it
     inside upstream's restructured code if upstream refactored the same
     area.
   - Leave alone the items `CLAUDE.md` explicitly marks as **not** renamed
     (native bundle paths `Orca Computer Use.app` / `Orca.exe`, the
     `X-Orca-Agent-Hook-Token` protocol header, the `Orca.MobilePairing`
     firewall rule name, internal identifiers like `OrcaRuntimeService` /
     `orca-profiles/` / `ORCA_AGENT_HOOK_TOKEN`) — keep whichever side is
     functionally correct there without forcing a rename.
   - For `src/renderer/src/i18n/locales/en.json` specifically: take the
     union of keys from both sides (don't drop new upstream strings). Rename
     "Orca" → "Dark Factory" in display-string values, including in newly
     added upstream strings, since UI text is explicitly safe to keep
     extending per `CLAUDE.md`. Don't rename URLs/proper nouns pointing at
     infrastructure that's deliberately still upstream (e.g.
     `stablyai/orca` GitHub links, `onorca.dev` links not yet migrated).
   - For a modify/delete conflict (upstream deleted a file this fork
     modified for the rebrand): check whether upstream relocated/renamed the
     logic (`git log main --oneline -- <old-path>`,
     `git log main --diff-filter=A --name-only -- '*<keyword>*'`). If
     relocated, port the Dark Factory-specific values into the new location
     and delete the old file. If genuinely removed upstream, keep this
     fork's version rather than losing the feature.
   - Remove all conflict markers. Verify none remain:
     `grep -rn '^<<<<<<<\|^=======$\|^>>>>>>>' --include='*.ts' --include='*.tsx' --include='*.cjs' --include='*.json' src config`.
   - `git add` each file as it's resolved.

5. **Verify before committing.** `git status` should show no unmerged paths.
   Run the typecheck pass matching the areas touched (`pnpm tc:node`,
   `tc:cli`, and/or `tc:web` — see `CLAUDE.md`) and fix any new errors
   introduced by the conflict resolution (not pre-existing unrelated ones).
   If `src/renderer/src/i18n/locales/en.json` was touched, validate it
   parses: `node -e "JSON.parse(require('fs').readFileSync('src/renderer/src/i18n/locales/en.json','utf8'))"`.

6. **Commit the merge.** `git commit --no-edit`.

7. **Restore any stashed work.** If step 1 stashed files, `git stash pop`
   and resolve any remaining textual conflicts between the stash and the
   freshly merged content by hand (should be rare if the stashed files'
   local edits didn't overlap upstream's changed lines — check with
   `git stash show -p` beforehand if unsure).

8. **Report a concise summary**: what was pulled in (rough commit count /
   notable changes), which files had conflicts and how each was resolved,
   any typecheck follow-ups still needed, and confirmation the merge commit
   was created (short hash). Do not push — leave that to the user.
