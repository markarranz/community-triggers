# Sidekick - Copilot

Launches [GitHub Copilot CLI](https://docs.github.com/copilot/how-tos/use-copilot-agents/use-copilot-cli) as a live companion on your Tuple call when transcription starts.

When `call-transcription-started` fires, this trigger opens your preferred terminal running `copilot` inside the call's transcription directory. Copilot catches up on everything said so far, then follows the live transcript and acts as a sharp third pair on the call.

## What it does

- **Logs the call live.** On every batch of new transcript, Copilot leaves a one-line `·` play-by-play so you can follow along at a glance.
- **Chimes in when it matters.** It escalates from a log line to a real interjection for a bug it can see, an ambiguous decision or action item, a correction, or a direct question.
- **Answers when addressed.** Say "Copilot, ..." (or type into the terminal) and it responds to that turn, then keeps following the call.
- **Summarizes.** It writes a checkpoint summary when recording stops and a final summary (decisions, action items, open threads) when the call ends.

Copilot follows the call with **Tuple's bundled watcher** (`tuple-call-watcher.py`), shipped with this trigger and run verbatim: a fixed, deterministic script rather than a watch loop the model re-authors each session. This is the same watcher the Codex, Cursor, Claude, and OpenCode sidekicks use. Since Copilot CLI has no event-driven wake (no equivalent of Claude Code's `Monitor`), it runs the watcher once with `--catchup`, then repeatedly in `--exit-on-batch` mode (each run blocks until the next batch, prints it, and exits). Copilot runs shell commands synchronously, so a quiet stretch simply parks in the watcher until the next words land.

The prompt is passed via `-i` (interactive), so Copilot runs it automatically and then stays interactive — you can type to it during the call.

## Permissions

This trigger does **not** launch Copilot with a blanket auto-approve (`--allow-all-tools`). Instead it pre-authorizes **only the bundled watcher** and leaves everything else to Copilot's normal per-tool approval prompts.

It does this with scoped `--allow-tool` flags. Copilot matches `shell(<stem>:*)` on a command's leading token — the `:*` suffix means "this stem followed by a space" — so the launcher passes:

```text
--allow-tool=shell(./<session-dir>/tuple-call-watcher.py:*)
--allow-tool=shell(<session-dir>/tuple-call-watcher.py:*)
--allow-tool=shell(<abs-path>/tuple-call-watcher.py:*)
```

All three forms point at the **same** script (relative with `./`, bare relative, and absolute), so the rule matches however Copilot normalizes the command's first token — and nothing but the watcher is ever auto-approved. The agent always invokes the watcher at this one path; mid-call transcription restarts add new session directories that the watcher *reads*, not new commands it's *invoked* as. Any other command the agent attempts (a file read of a past call, say) goes through your normal Copilot approval flow.

On first launch Copilot asks once whether you **trust this directory** — accept it to proceed (this is separate from command approval). If your setup still prompts for the watcher, approve it once with "always allow" and the loop runs unattended from there.

## Account and org policy

Copilot CLI must be enabled for your account, and **not restricted by your organization's Copilot policy**. If you see `Access denied by policy settings` or `Third-party MCP servers are disabled by your organization's Copilot policy`, an org/enterprise admin needs to enable the **Copilot CLI** policy (and, for non-built-in MCP servers, set the **MCP servers** policy to "Allow all" or add the server to the org registry). See [your Copilot settings](https://github.com/settings/copilot).

## MCP servers

Copilot CLI loads MCP servers automatically from `~/.copilot/mcp-config.json` — no flag needed; the sidekick inherits them. The built-in GitHub MCP server is available out of the box. To add servers for just this session, pass `--additional-mcp-config @/path/to/servers.json` via `SIDEKICK_COPILOT_EXTRA_ARGS` (below). Note that third-party MCP servers can be gated by org policy (see above).

## Options

Optional passthrough knobs (export in your shell profile; all off by default):

| Variable | Effect |
| --- | --- |
| `SIDEKICK_COPILOT_EXTRA_ARGS` | Any other `copilot` flags, shell-quoted — e.g. `--effort high`, `--model gpt-5.3-codex`, or `--additional-mcp-config @/path/to/servers.json`. |
| `PREFERRED_TERM` | Force a terminal: `ghostty` \| `iterm` \| `alacritty` \| `terminal`. |
| `SIDEKICK_COPILOT_DRY_RUN=1` | Ship the watcher and write the launcher without opening a terminal (for testing). |

Example:

```bash
export SIDEKICK_COPILOT_EXTRA_ARGS="--effort high"
```

## Choosing your terminal

By default the trigger opens the first installed of **Ghostty → iTerm → Alacritty → Terminal**. To force one, set `PREFERRED_TERM` at the top of `call-transcription-started` (or in the environment):

```bash
PREFERRED_TERM="iterm"   # ghostty | iterm | alacritty | terminal
```

The terminal runs `launch-sidekick-copilot.command`, whose `#!/bin/zsh -li` shebang sources your `~/.zprofile` and `~/.zshrc`, so `copilot` resolves from the same PATH and environment you get in a normal terminal.

## Prerequisites

- macOS
- [GitHub Copilot CLI](https://docs.github.com/copilot/how-tos/use-copilot-agents/use-copilot-cli) installed (`npm i -g @github/copilot`) so `copilot` works in a new terminal. Run it once first to sign in (and to clear the one-time directory-trust prompt).
- A Copilot subscription that includes the CLI, not blocked by org policy (see above).
- `python3` (the bundled watcher needs it; install with `xcode-select --install`)
- Tuple transcription enabled for the call

## Installation

Drop this directory into your Tuple triggers folder:

`~/.tuple/triggers/sidekick-copilot/`

The trigger fires the next time call transcription starts.

## How it works

When `call-transcription-started` fires, Tuple provides `TUPLE_TRIGGER_CALL_ARTIFACTS_DIRECTORY`, the directory containing the current call's transcription artifacts. This trigger:

1. Copies the fixed `tuple-call-watcher.py` and writes `sidekick-copilot-prompt.md` into that directory.
2. Writes an executable `launch-sidekick-copilot.command` wrapper into that directory.
3. Opens it in your preferred terminal via `open` (LaunchServices). No AppleScript and no direct binary launch, so it triggers no macOS accessibility prompt and no stray windows.
4. The wrapper starts a login-interactive zsh shell, changes to the transcripts root, and runs `copilot -i` with the prompt and the scoped watcher allowlist (no `--allow-all-tools` — see Permissions); Copilot runs the bundled watcher to catch up and follow the call.

A PID file (`sidekick-copilot.pid`) keeps a second transcription start from launching a duplicate sidekick for the same call.

For local script testing without opening a terminal, set `SIDEKICK_COPILOT_DRY_RUN=1`. It still ships the watcher and writes the launcher so you can inspect them.
