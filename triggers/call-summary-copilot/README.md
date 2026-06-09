# Call Summary - Copilot

Launches [GitHub Copilot CLI](https://docs.github.com/copilot/how-tos/use-copilot-agents/use-copilot-cli) when Tuple transcription completes and asks it to summarize the call.

When `call-transcription-complete` fires, this trigger opens your preferred terminal running `copilot` inside the call's transcription directory. Copilot reads the whole transcript and writes a tight summary — outcome, decisions, action items, open questions — then stays open for follow-up questions.

## What it does

- **Reads the full transcript** deterministically via Tuple's bundled watcher (`tuple-call-watcher.py --catchup`), which prints every record across all of the call's session directories (including mid-call restarts).
- **Writes a structured summary** first, without preamble: Summary, Decisions, Action items, Open questions, Notable context (sections with nothing to say are omitted).
- **Answers follow-ups.** The prompt is passed via `-i` (interactive), so after the summary you can keep asking questions and Copilot answers from the transcript.

## Permissions

This trigger does **not** launch Copilot with a blanket auto-approve (`--allow-all-tools`). It pre-authorizes **only the bundled watcher** so its one `--catchup` read runs unattended; everything else falls back to Copilot's normal per-tool approval prompts.

Copilot matches `--allow-tool='shell(<stem>:*)'` on a command's leading token (the `:*` suffix means "this stem followed by a space"), so the launcher passes the relative and absolute forms of the watcher path — all pointing at the same script — permitting `./tuple-call-watcher.py --catchup` and nothing else. On first launch Copilot asks once whether you **trust this directory**; accept it to proceed.

## Account and org policy

Copilot CLI must be enabled for your account and **not restricted by your organization's Copilot policy**. If you see `Access denied by policy settings`, an org/enterprise admin needs to enable the **Copilot CLI** policy. See [your Copilot settings](https://github.com/settings/copilot).

## Options

| Variable | Effect |
| --- | --- |
| `CALL_SUMMARY_COPILOT_EXTRA_ARGS` | Any other `copilot` flags, shell-quoted (e.g. `--effort high`, `--model gpt-5.3-codex`). |
| `PREFERRED_TERM` | Force a terminal: `ghostty` \| `iterm` \| `alacritty` \| `terminal`. |
| `CALL_SUMMARY_COPILOT_DRY_RUN=1` | Ship the watcher and write the launcher without opening a terminal (for testing). |

## Choosing your terminal

By default the trigger opens the first installed of **Ghostty → iTerm → Alacritty → Terminal**. To force one, set `PREFERRED_TERM` at the top of `call-transcription-complete` (or in the environment):

```bash
PREFERRED_TERM="iterm"   # ghostty | iterm | alacritty | terminal
```

The terminal runs `launch-call-summary-copilot.command`, whose `#!/bin/zsh -li` shebang sources your `~/.zprofile` and `~/.zshrc`, so `copilot` resolves from the same PATH and environment you get in a normal terminal.

## Prerequisites

- macOS
- [GitHub Copilot CLI](https://docs.github.com/copilot/how-tos/use-copilot-agents/use-copilot-cli) installed (`npm i -g @github/copilot`) so `copilot` works in a new terminal. Run it once first to sign in (and to clear the one-time directory-trust prompt).
- A Copilot subscription that includes the CLI, not blocked by org policy (see above).
- `python3` (the bundled watcher needs it; install with `xcode-select --install`)
- Tuple transcription enabled for the call

## Installation

Drop this directory into your Tuple triggers folder:

`~/.tuple/triggers/call-summary-copilot/`

The trigger fires the next time call transcription completes.

## How it works

When `call-transcription-complete` fires, Tuple provides `TUPLE_TRIGGER_CALL_ARTIFACTS_DIRECTORY`, the directory containing the completed call's transcription artifacts. This trigger:

1. Copies the fixed `tuple-call-watcher.py` and writes `call-summary-copilot-prompt.md` into that directory.
2. Writes an executable `launch-call-summary-copilot.command` wrapper into that directory.
3. Opens it in your preferred terminal via `open` (LaunchServices). No AppleScript, so it triggers no macOS accessibility prompt.
4. The wrapper starts a login-interactive zsh shell, changes to the transcription directory, and runs `copilot -i` with the prompt and the scoped watcher allowlist (no `--allow-all-tools` — see Permissions); Copilot reads the transcript and writes the summary.

A PID file (`call-summary-copilot.pid`) keeps a duplicate summary from launching for the same call.

For local script testing without opening a terminal, set `CALL_SUMMARY_COPILOT_DRY_RUN=1`. It still ships the watcher and writes the launcher so you can inspect them.
