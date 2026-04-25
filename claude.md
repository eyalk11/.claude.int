#

#  Guidelines for Claude-code
* in general , you can use  FindFileRg in pwsh to find relevant files in directory. it will do fuzzy search.

- prefer to use pwsh mcp instead of Bash when installing or running programs. You can use bash for grepping etc.
- Notice that Bash doesn't recognize windows paths, and if an absolute path needed use : /c/gitproj/... for example
- Don't over-do tasks
- if you don't know where to start / what file I want you to work on. try the following: 
a. read  c:\temp\active_buffer (which usually contains the file I'm working on) 
b. Try to read the readme.md or claude.md in the current git repo to understand file structure.
c. Try to look for a similar file Bash(find . | rg "text" ) should be good..
d. ask


- There's a file modification bug in Claude Code. The workaround is: always use complete absolute Windows paths when editing 

- if there is a bug that you try to fix and the user tell you to test using whatever command, and after testing the issue presist or there is another similar issue,
continue to fix it, and rerun the test command. As long as the command doesn't cost money or possibly dangerous. 

- Try less to ask the user to run tests manually if you can do it. 


- if user ask to approve command : 
if Claude tries: 
● Update(.claude\settings.json)
  ⎿  Denied by auto mode classifier ∙ /feedback if incorrect 
Then claude shouldn't try it and just load update settings skill.


- if something not work more than 3 times, probably don't try again before asking. unless you are getting some progress.

- NEVER overwrite, delete, or replace untracked files (files not in git) without first asking the user. Before any operation that could destroy untracked files (e.g. `git clean`, `git checkout -f`, `git reset --hard` when untracked files would be clobbered, branch recovery, worktree replacement, copying over a directory), run `git status --porcelain` and `git ls-files --others --exclude-standard` to enumerate them. List the untracked files that would be lost to the user and wait for explicit approval before proceeding. This applies even in auto mode.
- NEVER push commits unless specifically instructed to


### python instructions 

- perfer pytest to unittest 
* prefer to use simple_exception_handling to vanila try except.

### Example 1: Handling Exceptions
```python
from commonpy.simpleexceptioncontext import SimpleExceptionContext

callback = lambda e: self._eng.statusChanges.emit(f'Exception in processing {e}')
with SimpleExceptionContext('exception in processing',callback=callback):
    ret=self.process_internal(ls)
    logging.debug(f"process end {ret}")
    return ret

```
In this example , a callback is called when the exception occurs. 


### Example 2: Exception Handling with Decorators
```python

@simple_exception_handling(err_description='error in get_symbol_history', return_succ=(None, []), never_throw=True)
@excp_handler(polygon.exceptions.BadResponse, handler=excphandler)
def get_symbol_history(sym, startdate, enddate, iscrypto=False):
    # Your code to fetch symbol history here
    pass 
```
note that never_throw means it wouldn't throw , return_succ tells it to return the tuple  in case an exception happened.
### about mcps 

- git-grep is used only for grepping code (i.e. not issues)
- pwsh mcp doesn't save session (i.e. doing cd and running cmd should be done in one mcp call)
- github mcp can be used to list issues


## WSL stuff

- To access wsl file system use \\wsl.localhost\Ubuntu-20.04\home\ekarni for example 
- To execute commands in wsl use  `wsl.exe bash -c "command here"` with Bash

# Behavioral  Guidelines 

Behavioral guidelines to reduce common LLM coding mistakes. Merge with project-specific instructions as needed.

**Tradeoff:** These guidelines bias toward caution over speed. For trivial tasks, use judgment.

## 1. Think Before Coding

**Don't assume. Don't hide confusion. Surface tradeoffs.**

Before implementing:
- State your assumptions explicitly. If uncertain, ask.
- If multiple interpretations exist, present them - don't pick silently.
- If a simpler approach exists, say so. Push back when warranted.
- If something is unclear, stop. Name what's confusing. Ask.

## 2. Simplicity First

**Minimum code that solves the problem. Nothing speculative.**

- No features beyond what was asked.
- No abstractions for single-use code.
- No "flexibility" or "configurability" that wasn't requested.
- No error handling for impossible scenarios.
- If you write 200 lines and it could be 50, rewrite it.

Ask yourself: "Would a senior engineer say this is overcomplicated?" If yes, simplify.

## 3. Surgical Changes

**Touch only what you must. Clean up only your own mess.**

When editing existing code:
- Don't "improve" adjacent code, comments, or formatting.
- Don't refactor things that aren't broken.
- Match existing style, even if you'd do it differently.
- If you notice unrelated dead code, mention it - don't delete it.

When your changes create orphans:
- Remove imports/variables/functions that YOUR changes made unused.
- Don't remove pre-existing dead code unless asked.

The test: Every changed line should trace directly to the user's request.

## 4. Goal-Driven Execution

**Define success criteria. Loop until verified.**

Transform tasks into verifiable goals:
- "Add validation" → "Write tests for invalid inputs, then make them pass"
- "Fix the bug" → "Write a test that reproduces it, then make it pass"
- "Refactor X" → "Ensure tests pass before and after"

For multi-step tasks, state a brief plan:
```
1. [Step] → verify: [check]
2. [Step] → verify: [check]
3. [Step] → verify: [check]
```

Strong success criteria let you loop independently. Weak criteria ("make it work") require constant clarification.

---

**These guidelines are working if:** fewer unnecessary changes in diffs, fewer rewrites due to overcomplication, and clarifying questions come before implementation rather than after mistakes.
