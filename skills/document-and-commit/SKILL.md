---
name: document-and-commit
description: Review currently staged changes, write a clear commit message describing what changed and why, and create the commit. Use when the user asks to "commit", "document and commit", "commit the staged changes", or "write a commit message and commit". Only operates on already-staged changes — does not stage files itself.
---

# document-and-commit

Document the currently staged changes and create a single commit. This skill assumes the user has already staged what they want committed (`git add`). It does **not** stage additional files, does not modify the working tree, and does not push.

## Steps

1. **Inspect the staging area.** Run these in parallel:
   - `git status` (no `-uall`) — confirm what is staged vs. unstaged vs. untracked.
   - `git diff --cached` — the actual staged hunks. This is what the commit message must describe.
   - `git log -n 10 --oneline` — match the repo's existing commit message style (subject length, prefix conventions like `fix(scope):`, etc.).

2. **If nothing is staged**, stop and tell the user. Do not auto-stage. Do not create an empty commit.

3. **If unstaged or untracked changes exist alongside staged ones**, mention them briefly so the user knows they will be left out — but proceed with the commit on what is staged unless the user objects.

4. **Draft the commit message.**
   - Subject: short (≤72 chars), imperative mood, matching repo style. Use `add` for genuinely new things, `update`/`change` for modifying existing behavior, `fix` for bug fixes, `refactor` for no-behavior-change restructuring, `docs`/`test`/`chore` as appropriate.
   - Body (optional, only when the diff isn't self-explanatory): wrap at ~72 cols, focus on **why** rather than restating the diff. Note non-obvious tradeoffs, linked issue/PR numbers if visible in recent log, and anything a future reader would want to know.
   - Do not invent motivation. If the "why" isn't clear from the diff or conversation context, keep the message to a factual subject line.

5. **Check for accidentally-staged secrets.** Scan the diff for things that look like credentials: `.env`, `*.pem`, `*.key`, `credentials.json`, hardcoded API tokens (long random strings under keys named `token`/`secret`/`password`/`api_key`). If found, **stop and warn the user** — do not commit until they confirm.

6. **Create the commit** using a heredoc to preserve formatting:

   ```bash
   git commit -m "$(cat <<'EOF'
   <subject line>

   <optional body>

   Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
   EOF
   )"
   ```

7. **Verify** with `git status` and `git log -1 --stat`. Report the new commit's subject and short hash to the user.

## Constraints

- Never amend an existing commit — always create a new one. If a pre-commit hook fails, fix the underlying issue, re-stage the fix, and create a **new** commit (not `--amend`).
- Never pass `--no-verify`, `--no-gpg-sign`, or `-c commit.gpgsign=false` unless the user explicitly asks.
- Never `git add -A` / `git add .` as part of this skill — the user controls what gets staged.
- Never push. If the user wants to push, they will ask separately.
- Do not edit, reformat, or "clean up" files while preparing the commit. Document what is staged; nothing more.
