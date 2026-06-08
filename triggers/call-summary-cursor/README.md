# Call Summary - Cursor

Launches the [Cursor CLI agent](https://cursor.com/cli) when Tuple transcription completes and asks it to summarize the completed transcript.

The trigger writes a `launch-call-summary-cursor.command` wrapper next to the transcript files and opens it in your preferred terminal. The wrapper runs as `#!/bin/zsh -li`, so `cursor-agent` resolves from the same interactive shell environment you get in a new terminal. No install location is hard-coded.

Cursor starts in the transcription directory with a focused summary prompt. It reads the whole call by running Tuple's bundled watcher (`tuple-call-watcher.py --catchup`, shipped with this trigger), produces a concise summary with decisions, action items, and open questions, then stays available for transcript-backed follow-up questions. The prompt is passed as the initial message and no `-p` flag is used, so Cursor stays interactive afterward.

## Permissions

This trigger does **not** launch Cursor with a blanket auto-approve (`--force` / `--yolo`). Instead it pre-authorizes **only the bundled watcher** — the one command the summary needs to run unattended — and leaves everything else to your own Cursor approval settings.

It does this by writing a project-scoped allowlist to `<transcription-dir>/.cursor/cli.json` (Cursor's working directory for the session):

```json
{ "permissions": { "allow": ["Shell(*/tuple-call-watcher.py)"], "deny": [] } }
```

Cursor's `Shell(<base>)` rules match on the command's first token, so this permits `./tuple-call-watcher.py --catchup` — and nothing else. The file is written only if you don't already have a `.cursor/cli.json` there, so a setup you manage yourself is never overwritten. On first launch Cursor also asks once whether you trust the directory; accept it to proceed.

## Prerequisites

- macOS
- [Cursor CLI](https://cursor.com/cli) installed so `cursor-agent` works in a new terminal. Run it once first to sign in (and to clear any one-time workspace-trust prompt).
- `python3` (the bundled watcher needs it; install with `xcode-select --install`)
- Tuple transcription enabled for the call

## Installation

Drop this directory into your Tuple triggers folder:

`~/.tuple/triggers/call-summary-cursor/`

The trigger fires when call transcription completes.

## How it works

When `call-transcription-complete` fires, Tuple provides `TUPLE_TRIGGER_CALL_ARTIFACTS_DIRECTORY`, the directory containing the completed transcription artifacts. This trigger:

1. Copies the fixed `tuple-call-watcher.py` and writes `call-summary-cursor-prompt.md` into that directory.
2. Writes a project-scoped `.cursor/cli.json` allowlist there permitting only the bundled watcher (see Permissions).
3. Writes an executable `launch-call-summary-cursor.command` wrapper into that directory.
4. Opens it in your preferred terminal via `open` (Ghostty → iTerm → Alacritty → Terminal; set `PREFERRED_TERM` to choose). No AppleScript, so it triggers no macOS accessibility prompt.
5. The wrapper starts a login-interactive zsh shell, changes into the transcription directory, and runs `cursor-agent` with the summary prompt as its initial message.

For local script testing without opening a terminal, set `CALL_SUMMARY_CURSOR_DRY_RUN=1`.
