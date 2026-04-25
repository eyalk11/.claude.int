---
name: recover-from-transcripts
description: Recover lost or deleted files by reconstructing them from past Claude Code session transcripts (~/.claude/projects/<project>/*.jsonl). Use when the user asks to "recover from transcripts", "restore deleted files", "rebuild from history", or wants to retrieve content of files Claude wrote in earlier sessions that no longer exist on disk.
---

# recover-from-transcripts

Reconstruct files that Claude Code created or edited in past sessions by replaying the `Write` / `Edit` / `MultiEdit` tool calls recorded in session transcripts.

Session transcripts for the current project live at:
`C:\Users\<user>\.claude\projects\<slugified-cwd>\*.jsonl`

The slug replaces `:` and `\` with `-` (e.g. `C:\autoproj\chess_analyzer` → `C--autoproj-chess-analyzer`).

## Steps

1. **Confirm scope with the user.** Ask which file(s) or which directory to recover if not already specified. Confirm the project's transcript directory path before scanning.

2. **Locate candidate sessions.** Grep the transcript directory for the target filename or path. Each hit may be a Write, an Edit, a Read result, or just a mention in conversation — only Write/Edit/MultiEdit tool_use entries carry recoverable content.

3. **Replay tool calls in order.** For each target file, walk every `*.jsonl` chronologically and reconstruct final content:
   - `Write` → replaces full content with `input.content`.
   - `Edit` → string-replace `old_string` with `new_string` (once, or all if `replace_all: true`).
   - `MultiEdit` → apply each edit in `input.edits` in order.
   - Ignore `Read` tool_result blocks — they may show partial content with `cat -n` line-number prefixes that must NOT be written back.

4. **Use a Python script for the replay.** Bash/grep cannot reliably parse JSON with embedded newlines. A minimal template:

   ```python
   import json, os, glob
   project_dir = r'C:\Users\<user>\.claude\projects\<slug>'
   targets = ['filename1.md', 'filename2.py']  # match against tool_use input.file_path
   target_dir = r'C:\path\to\restore\into'
   os.makedirs(target_dir, exist_ok=True)

   for name in targets:
       state = {'final': None}
       def walk(o):
           if isinstance(o, dict):
               if o.get('type') == 'tool_use' and o.get('name') in ('Write','Edit','MultiEdit'):
                   inp = o.get('input', {}); fp = inp.get('file_path', '')
                   if name in fp:
                       if o['name'] == 'Write':
                           state['final'] = inp.get('content', '')
                       elif o['name'] == 'Edit' and state['final'] is not None:
                           cnt = 1 if not inp.get('replace_all') else -1
                           state['final'] = (state['final'].replace(inp['old_string'], inp['new_string'])
                                             if cnt == -1 else
                                             state['final'].replace(inp['old_string'], inp['new_string'], 1))
                       elif o['name'] == 'MultiEdit' and state['final'] is not None:
                           for e in inp.get('edits', []):
                               cnt = 1 if not e.get('replace_all') else -1
                               state['final'] = (state['final'].replace(e['old_string'], e['new_string'])
                                                 if cnt == -1 else
                                                 state['final'].replace(e['old_string'], e['new_string'], 1))
               for v in o.values(): walk(v)
           elif isinstance(o, list):
               for v in o: walk(v)

       for fn in sorted(glob.glob(os.path.join(project_dir, '*.jsonl')), key=os.path.getmtime):
           with open(fn, encoding='utf-8') as f:
               for line in f:
                   try: walk(json.loads(line))
                   except: continue
       if state['final'] is None:
           print(name, 'NOT RECOVERABLE — only mentioned, never written via Write tool')
       else:
           out = os.path.join(target_dir, name)
           with open(out, 'w', encoding='utf-8', newline='') as f: f.write(state['final'])
           print('wrote', out, len(state['final']), 'bytes')
   ```

5. **Report what was and was NOT recovered.** Files that only appear as filenames in directory listings (e.g. `ls` output) but never as a `Write` `tool_use` are NOT recoverable from transcripts — say so explicitly. Suggest checking other backups (git stash, git fsck dangling objects, OS shadow copies, IDE history) for the rest.

6. **Verify before overwriting.** If the recovery target path already exists with different content, show a diff and ask before overwriting. Never silently clobber a file the user might have edited since.

## Notes

- Older sessions can be archived or rotated out — recovery only works as far back as transcripts still exist locally.
- `Bash` heredoc writes (`cat <<EOF > file`) are recorded as `command` strings, not as `content`; parsing those is best-effort and should be flagged to the user.
- `git fsck --unreachable` won't help for files that were never staged.

## Fallback sources when transcripts don't have the content

If a target file is referenced in transcripts only as a filename (never as a `Write` `tool_use`), it was authored before the earliest surviving transcript. Try these in order:

1. **Vim swap files** at `~/.vim/swap/<basename>.md.sw{p,o,n,m}` — binary, but each data block is 4096 bytes and stores lines as NUL-separated strings near the end of the block. Parse in Python:
   ```python
   import re
   data = open(swp_path,'rb').read()
   for i in range(4096, len(data), 4096):  # skip header block 0
       for p in data[i:i+4096].split(b'\x00'):
           try: s = p.decode('utf-8')
           except: continue
           if re.match(r'^[\t\x20-\x7e]+$', s) and any(c.isalpha() for c in s) and len(s) >= 2:
               print(repr(s))
   ```
   Lines come out in **reverse order** (last line of file first) — re-reverse them. The swap file's first block also contains the original full path of the file (helpful for confirming which file you're looking at). `vim -r <swp> +'wq newfile'` does NOT work in `-es` batch mode on Windows; the manual parse above is more reliable.
2. **Vim undo files** at `~/.vim/undo/` — path-encoded with `%` separators (e.g. `C%%gitproj%foo%bar.md`). Binary, harder to parse than swap, but contains undo history.
3. **OS file backup tools** — on this user's machine, `C:\filebackup\ekarni\<HOSTNAME>\Data\` mirrors parts of `C:\Users\<user>\` and may contain the actual `.md` files or older versions.
4. **IDE local history** — VS Code: `%APPDATA%\Code\User\History`; JetBrains: `<project>/.idea/workspace.xml` history fragments.
