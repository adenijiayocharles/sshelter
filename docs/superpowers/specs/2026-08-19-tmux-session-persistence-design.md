# Tmux Session Persistence — Design

## Problem

naden's terminal sessions are entirely in-memory (`sessions: Arc<Mutex<HashMap<String, ActiveSession>>>` in `src-tauri/src/local_terminal.rs:25`, same pattern for the SSH path). A restart of the naden process kills every open shell — there is no way to reopen an app and find your remote work where you left it.

A live shell can't literally survive a process restart on any platform (the shell is a child of the SSH connection, which is a child of the process). The standard way to get around this is to run the remote shell inside `tmux` on the server: the shell then belongs to `tmux`, not to naden's SSH connection, so it survives naden closing, crashing, or restarting. Reconnecting later just means attaching to the same `tmux` session again.

## Goals

- Opt-in, per-server persistence of SSH terminal sessions via `tmux` on the remote host.
- Closing a naden tab detaches the session (leaves it running); an explicit action ends it.
- Reconnecting (manually or via existing auto-reconnect-on-drop) reattaches to the exact same remote session rather than creating a new one.
- Graceful, non-blocking fallback when `tmux` isn't available on the remote host.

## Non-goals

- Local shell sessions (`kind: "local"`) — no remote host to wrap.
- Auto-reopening tabs on naden launch — this design only makes *reconnecting* attach to existing state; the user still initiates the reconnect.
- Automatic cleanup/expiry of orphaned remote sessions.
- Retroactively cleaning up remote sessions when persistence is turned off for a server.

## Architecture / flow

Gated by a new per-server boolean, `persist_session`. When enabled, `run_session` (`src-tauri/src/ssh/connection.rs:365`) replaces its current connect → open PTY channel → `.shell()` sequence with:

1. After auth (`connection.rs:410`), open a short-lived exec channel and run `command -v tmux`, bounded by a timeout (~5s).
   - Missing or timed out → fall back to today's plain `.shell()` path unchanged. Emit `terminal:persistence-unavailable:{session_id}` once (frontend shows a one-time dismissible banner, not a blocking prompt).
2. Present → open another short exec channel, run `tmux list-sessions -F '#{session_name}:#{session_created}'`, filter to names prefixed `naden-{server_id}-`.
3. No matches → generate a new session name (see Naming) and proceed directly to step 5.
4. One or more matches → emit `ssh:session-picker-prompt:{session_id}` with the list (name, created-at) and block on the same `mpsc` `rx` pattern already used for host-key and hook-confirm prompts (`connection.rs`), waiting for the frontend's choice: resume a specific name, or start new.
5. Open the interactive PTY channel as before (`request_pty`), but `exec("tmux new-session -A -s '<name>'")` in place of `.shell()`. `-A` (attach-or-create) makes this race-safe even if the chosen session vanished between steps 2/4 and 5.
6. `initial_dir` / env-var injection (`connection.rs:428-448`) is unchanged — still typed into the pty after attach.
7. Once the name is settled (new or resumed), emit `ssh:session-name:{session_id}` so the frontend can remember it for future reconnects.

Closing a tab requires no new backend logic: `close_terminal_session` (`ssh_commands.rs:588`) already just closes the channel via `session_manager.close_session`, never sending `exit`. tmux's default behavior on client hangup is to detach, not kill — persistence on close falls out of the existing path for free.

## Backend

- **Migration**: new file `0019_server_persist_session.sql` (existing migrations are never edited) adding `persist_session BOOLEAN NOT NULL DEFAULT 0` to `servers`. Plain settings field, no vault/encryption involvement — threaded through `src/lib/commands/server.ts` and the server form like any other setting.
- **Naming**: `naden-{server_id}-{suffix}`, where `server_id` is our own UUID and `suffix` is an app-generated random hex string (enough entropy, e.g. 8 hex chars, to make local collisions a non-concern). No user input reaches the shell string, but it is still single-quote-escaped the same way `initial_dir` and env vars already are (`connection.rs:431,445`), for consistency.
- **New Tauri commands**:
  - `confirm_session_choice(session_id, choice: "resume:<name>" | "new")` — mirrors `confirm_host_key` / `confirm_hooks`; resolves the blocked `rx` in `run_session`.
  - `end_persistent_session(server_id, tmux_session_name)` — opens its own short-lived exec channel (independent of any live tab) and runs `tmux kill-session -t '<name>'`.
- **Reconnect must skip discovery and reattach directly.** `run_session` gains an optional `tmux_session_name` param that, when present, skips steps 1–4 entirely and goes straight to `exec("tmux new-session -A -s '<name>'")`. `openTerminalSession` (frontend command) and its Rust command handler gain an optional `tmuxSessionName` passthrough for this.

## Frontend / store

- **`TerminalSession` type** (`src/store/terminalStore.ts:20`) gains:
  - `tmuxSessionName?: string` — set from `ssh:session-name:{session_id}`.
  - `persistenceNotice?: string` — set from `terminal:persistence-unavailable:{session_id}`, rendered as a small dismissible banner in the tab header (not a blocking overlay).
  - `sessionPickerPrompt?: { sessions: { name: string; createdAt: number }[] }` — set from `ssh:session-picker-prompt:{session_id}`, wired the same way as the existing `hostKeyPrompt`/`hookConfirmPrompt` listeners (`terminalStore.ts:98-119`, `242-259`).
- **New store actions**:
  - `confirmSessionChoice(sessionId, choice)` — clears `sessionPickerPrompt`, calls the new IPC command.
  - `endPersistentSession(serverId, tmuxSessionName)` — callable from the picker overlay (no live tab required) or a tab's context menu; when called from an active tab, closes it locally afterward the same way `closeSession` does.
- **`openSession`** gains an optional `tmuxSessionName` param, passed through to `terminalCommands.openTerminalSession`. Regular "Connect" clicks from the sidebar omit it (full discovery/picker flow runs). `scheduleReconnect` and `reconnectSession` (`terminalStore.ts:125-146, 363-379`) capture `s.tmuxSessionName` before `teardownResources` and pass it through on the follow-up `openSession` call, so a network-drop auto-reconnect silently reattaches to the same tmux session instead of re-triggering the picker or minting a new one.
- **UI**:
  - Server form: a "Persist session (tmux)" checkbox, wired through `commands/server.ts`.
  - `TerminalOverlays.tsx`: new picker overlay, sibling to the existing host-key overlay — lists sessions with relative "started Xh ago" timestamps, a **Resume** button per row, a small **end** action per row, and a **Start new session** button.
  - Tab context menu: new "End persistent session" item, shown only when `tmuxSessionName` is set.

## Edge cases

- Discovery exec calls (`command -v tmux`, `tmux list-sessions`) are timeout-bounded (~5s); timeout is treated the same as "tmux missing" rather than blocking connect indefinitely — matters for restricted/non-POSIX remote shells.
- Closing a still-connecting tab while the picker prompt is outstanding must abort `run_session` cleanly, same mechanism already used to cancel a pending host-key/hook-confirm prompt.
- A listed session can vanish between discovery and attach (killed from another tab in the meantime); `new-session -A` silently creates a fresh session under that name rather than erroring.
- `tmux kill-session` force-kills even if something is currently attached elsewhere (another naden tab, or a manual `ssh` + `tmux attach`); the "End" action's UI copy should say so.
- Orphan accumulation is intentional for v1 (detach-on-close, no auto-expiry) — a known limitation, not a bug.
- Turning `persist_session` off for a server only affects future connects; it does not clean up or retroactively manage existing remote sessions, and the picker (the only UI path to reach them) stops appearing.
- Local shell sessions and jump-host chains are unaffected/out of scope, as noted above.

## Testing

- **Rust**: unit tests for session-name generation/escaping and for parsing `tmux list-sessions -F` output, alongside existing escaping tests near `connection.rs`. Using the mock-SSH-server harness already exercised by `ssh_commands_tests.rs`: persistence-off still calls `.shell()` (regression guard); persistence-on with no tmux falls back and emits the notice event exactly once; persistence-on with tmux present and no existing sessions execs `tmux new-session -A -s <generated-name>`.
- **Frontend (vitest)**: `terminalStore` tests asserting the `ssh:session-picker-prompt` listener populates `sessionPickerPrompt` correctly, `confirmSessionChoice` calls the IPC command with the exact chosen name (not just call count), and `scheduleReconnect`/`reconnectSession` pass an existing `tmuxSessionName` through on reconnect rather than omitting it.
- **Manual**: end-to-end verification against a real SSH+tmux target (or local Docker box) during implementation — kill naden mid-session, relaunch, confirm reattach shows correct scrollback/running state.
