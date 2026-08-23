# AGENTS.md

## Repository conventions

- All markdown files use **CRLF** line endings. After creating/editing a `.md`, run `sed -i 's/\r*$/\r/' <file>`.
- Lab reports follow a fixed structure (mirror `Lab 2/Lab 2.md`): front-matter → `## Lab Summary // Objective` → `## Architecture Diagram` (ASCII art) → `## Evidence Folder` table → `## Overview` → session sections with `## Environment Setup` + numbered tasks → `## Verification Command` → `## Short-Answer Questions` → `## Cleanup & Teardown` → checklist → `## Conclusion` → `## Expansion Ideas` + `## References`.
- Every task uses `**Why:**` / `**Result:**` / `Evidence: <div align="left">` blocks; each `Evidence:` block must be a self-closed `<div>` with matching `</div>`. Review checks keep divs and code fences balanced.
- Evidence folders are named `evidence labX` (with a space). Screenshots are named `taskN.png` (one per command block); multiple images of one task count as ONE evidence row in the Evidence Folder table.
- User preference: keep the original folder names (`LAB 0.md`, `LAB 1`, `Lab 2`, `Lab 3`, `Lab 3/evidence lab 3`) even though README's structure table lists different names (`Lab0_Environment_Setup`, etc.). A past rename to README names was reverted; don't repeat it.
- `.freebuff/` and `AGENTS.md` are **local-only** (listed in `.gitignore`). User explicitly asked to keep them out of the public repo — never stage, commit, or push them. Visual explainers for Lab 1/2 live in `.freebuff/`.

## Git & shared checkout

- This is a **shared local checkout**: other agents or the user commit/push independently (e.g., folder-rename commits have appeared from outside the thread). Inspect `git log` and `git status` before any consequential git operation; don't assume HEAD or the index is yours.
- Pushed commits cannot be undone with reset/force-push — undo with a new rename commit instead.
- Git detects renames only when delete + new path are staged together (e.g., adding `.md` to a filename shows `-417` deletion until both sides are staged).
- Windows `core.ignorecase=true`: case-only renames need a temp-name dance (`mv "Lab 1" "LAB1_tmp" && mv "LAB1_tmp" "LAB 1"`) to actually change case on disk.

## Lab-specific gotchas

- Lab 1 Task 7: the service account is `devsofia` (never `dev-user`). `dev-user-binding` is only the RoleBinding *name*. `SA=system:serviceaccount:dev:devsofia` makes the `kubectl auth can-i` checks return `yes`/`no` as documented.
- Lab 3 Task 5 (`datakey.bin` as OpenSSL pass): `-pass file:` reads only the first line of the binary key — a random data key may contain a newline, silently shrinking the effective key. Prefer `-K "$(xxd -p -c 256 datakey.bin)"` if bulletproofing.
- Lab 3 Task 6: `aws kms decrypt --ciphertext-blob fileb://` expects raw binary, but the CLI saves base64 text — `base64 -d datakey.enc > datakey.bin` before using `fileb://`.
- Lab 2 Task 5 (RBAC secret isolation) is a known confusion point for the user — explain Role (WHAT) vs RoleBinding (WHO + WHICH room) separately, note both are namespace-scoped, and explain `--as=` as "ask the question in character". User responds best to plain-words command tables and analogies.
