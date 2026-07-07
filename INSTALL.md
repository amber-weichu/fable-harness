# Installing Fable Harness

This kit doesn't ship a one-click installer. Instead, you clone the repo and then **ask your own Claude Code to install it**, following the steps below. Claude is good at doing careful, checked file operations — that's the whole point of this kit — so it can safely install itself.

安裝會動到你電腦上 Claude Code 的個人設定檔，請務必讓 Claude 先備份、只用「新增」而非「覆蓋」的方式合併，並在裝完後驗證成功。

## Prerequisite

Clone this repo anywhere on your machine, for example:

```bash
git clone <this-repo-url> fable-harness
```

You don't need to put it in any special location — just remember the path, since the global hooks will point back at it.

> ⚠️ **If you ever move this repo after installing**, the three absolute paths written into `~/.claude/settings.json` (or the project's own `.claude/settings.json`) do NOT update themselves. Grep that file for the old path and repoint all three `command` strings to the new location — then trigger each hook once (a session start, a prompt, and a Stop after an unstested code edit) to confirm they still fire. A hook that silently fails to fire looks identical to "nothing to report" — treat a missing/empty hook result as a bug to investigate, not a quiet pass.
>
> ⚠️ **Windows non-UTF-8 console codepages (e.g. Traditional Chinese `cp950`) can silently kill `verify_gate.py`.** The block message contains a `⛔` emoji; if the process's `sys.stdout` encoding can't represent it, `print()` raises `UnicodeEncodeError` — which the gate's own fail-open `except Exception: pass` swallows completely. The result: the Stop hook returns exit 0 with *zero output*, indistinguishable from "nothing to block." This was caught by actually running `tests/test_verify_gate.py::test_t1_edit_py_without_test_blocks` on a Windows/cp950 machine (it failed with a `JSONDecodeError` on empty stdout) rather than assuming the hook worked from previous UTF-8-default (macOS/Linux) test runs. Fixed by reconfiguring stdout to UTF-8 at the top of `verify_gate.py`. If you install on Windows, re-run the test suite there — don't trust results from a different OS.

## Install

Open Claude Code inside (or pointed at) the cloned repo, and say:

> Install Fable Harness by following INSTALL.md.

Claude should then carry out the following steps, in order. If you're Claude reading this: treat every step as a safety-critical file operation. Do not skip the backup or verification steps, and stop and ask the user if anything looks ambiguous (existing hooks you don't recognize, a settings file that won't parse, a symlinked directory, etc).

1. **Find the repo's own absolute path.** Run `pwd` (or equivalent) inside the cloned repo. You'll need this absolute path later — the hooks are referenced by path, not copied, so this repo must stay where it is after install.

2. **Find the user's global Claude config directory.**
   - macOS / Linux: `$HOME/.claude`
   - Windows: `%USERPROFILE%\.claude` (e.g. `C:\Users\<name>\.claude`)

3. **Read and validate `~/.claude/settings.json` before touching anything.** If it doesn't exist, is empty, or fails to parse as JSON, **stop and tell the user** — do not create a fresh one and do not guess defaults. Installing on top of a broken or missing settings file risks silently discarding the user's existing configuration.

4. **Back up `settings.json`** to a timestamped copy (e.g. `~/.claude/backups/settings.json.bak_<timestamp>`) before making any change. Confirm the backup file actually exists on disk before proceeding.

5. **Merge in three hooks — additively, never destructively.** Add these three entries to the `hooks` section, pointing at the scripts inside *this cloned repo* (use the absolute path from step 1, not a copy):

   | Event | Script | What it does |
   |---|---|---|
   | `SessionStart` | `<repo>/.claude/hooks/inject_protocol.sh` | Injects the behavior protocol at session start |
   | `UserPromptSubmit` | `<repo>/.claude/hooks/prompt_nudge.sh` | Injects a one-line reminder on every user turn |
   | `Stop` | `<repo>/.claude/hooks/verify_gate.py` | Blocks ending a turn where code changed but no test ran |

   Rules for this step:
   - Only **append** these three. Never remove, reorder, or rewrite any hook, model setting, theme, or other key the user already has.
   - If any of these three already exist (installed before), skip re-adding it — this step must be idempotent (safe to run twice).
   - After merging, verify every pre-existing top-level key and every pre-existing hook entry is still present and unchanged. If anything is missing, abort and restore from the step-4 backup.
   - Write via a temp file + atomic rename, not a direct in-place overwrite, so a crash mid-write can't corrupt the user's settings.

6. **Watch out for symlinks.** Some users manage `~/.claude/hooks`, `~/.claude/skills`, `~/.claude/agents`, or `~/.claude/CLAUDE.md` as symlinks (or Windows junctions) into a separate dotfiles/config repo. Before writing into any of these directories, check whether the target — or its parent — resolves (via realpath, not just `is_symlink`, since Windows junctions can hide from that check) to somewhere unexpected. If it does, **don't write through it silently** — tell the user what you found and ask where they'd like the files to actually land.

7. **Command strings must match exactly if they exist in more than one place.** If this project's own `.claude/settings.json` *also* defines these same three hooks (e.g. because you were developing inside this repo), the command string in the project settings and in the global settings must be **character-for-character identical**. Claude Code's native de-duplication relies on this — a stray difference (like a missing `|| exit 0` fallback) will cause the hook to fire twice per event instead of once.

8. **Copy the skill and the three agents.**
   - Copy `<repo>/.claude/skills/adversarial-review/` to `~/.claude/skills/adversarial-review/`.
   - Copy `<repo>/.claude/agents/skeptic.md`, `red-team.md`, and `simplifier.md` to `~/.claude/agents/`.
   - If any destination already exists, don't overwrite it — stop and ask the user first.

9. **Verify the install worked.** Start a brand-new Claude Code session and ask: *"What's your protocol codename?"* It should answer `FABLE-PROTOCOL-V1-CANARY`. If it doesn't, something in steps 5–8 didn't take effect — check the settings file and the copied files before declaring success.


## Optional: tidy up your global CLAUDE.md

If you already have a `~/.claude/CLAUDE.md` with your own development philosophy notes, some of it may now overlap with what the injected protocol already covers (the OODA loop, the definition of done, etc). You can ask Claude to read your global `CLAUDE.md` and shrink any rules that duplicate the injected `FABLE-PROTOCOL` down to a short pointer — while keeping anything that's uniquely yours. This step is optional and purely about reducing duplication; nothing in this kit requires it.

## Uninstall

To remove Fable Harness:

1. Open `~/.claude/settings.json` and delete the three hook entries listed in step 5 above (`SessionStart` → `inject_protocol.sh`, `UserPromptSubmit` → `prompt_nudge.sh`, `Stop` → `verify_gate.py`). Leave everything else untouched.
2. Delete `~/.claude/skills/adversarial-review/` and the three files in `~/.claude/agents/` (`skeptic.md`, `red-team.md`, `simplifier.md`).
3. If you edited your global `CLAUDE.md` in the optional step above and want the old wording back, restore it from your own version history or notes.

You can always ask Claude to do this for you too — the same care applies: back up `settings.json` first, then remove only the entries that belong to this kit.
