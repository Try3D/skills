---
name: terminal
description: Run background, long-lived, or interactive terminal commands inside tmux so the user can observe and intervene. Use this skill whenever you are about to start a dev server, watcher, REPL (python/node/psql/etc.), tail logs, or any process that runs longer than a single shell call — even if the user did not say "tmux".
metadata:
  author: Saran
---

# Terminal & background tasks

Run anything long-lived, backgrounded, or interactive inside tmux so the user has observability and can attach.

## Core rules

- **Reuse before creating.** Check `$TMUX` and `tmux ls` first. Only make a new session if nothing suitable already exists.
- **Inside tmux (`$TMUX` set): open a new window in the *current* session, detached** (`tmux new-window -d ...`). Never start a nested session. Do not switch the user's active window unless asked — `-d` is what keeps you from hijacking their focus.
- **Outside tmux: start one detached session and tell the user its name** so they can `tmux attach -t <name>`.
- **Launch and read output in the same turn.** Chain `send-keys` with `capture-pane` in one shell call. Splitting "run" and "check" across turns wastes a round trip and hides failures.
- **Only kill what you created.** Never `kill-session`/`kill-window` on something you didn't start without confirming.

## Step by step

1. Detect tmux:
   ```
   echo "$TMUX"; tmux ls
   ```

2. Pick the session:
   - If `$TMUX` is set, use the current session: `tmux display-message -p '#S'`.
   - Else create one: `tmux new-session -d -s <session>`.

3. Open a new window for the task **in the background** (`-d` is critical — without it tmux switches the user away from their current window):
   ```
   tmux new-window -d -t <session> -n <name>
   ```
   Name the window after the task (`dev`, `tests`, `repl`) so the user can find it. To start a command in the same call, append it: `tmux new-window -d -t <session> -n <name> '<cmd>'`.

4. Send the command and capture output in one call:
   ```
   tmux send-keys -t <session>:<name> '<cmd>' C-m && sleep 1 && tmux capture-pane -t <session>:<name> -p -S -200
   ```

5. For follow-up input (REPL prompts, password prompts, interactive menus), repeat step 4 one logical line at a time. **Do not pipe multi-line stdin into interactive programs** (`printf '...\n...\n' | python3 -i`) — indentation and prompt state get mangled. If the work is non-interactive, write a script file and run it instead of driving a REPL line by line.

6. Re-read output later:
   ```
   tmux capture-pane -t <session>:<name> -p -S -500
   ```

7. Clean up when done (only sessions/windows you created):
   ```
   tmux kill-window -t <session>:<name>
   tmux kill-session -t <session>
   ```

## Universal pattern for long-running processes

Whether it's a dev server, build, watcher, log tail, tunnel, or supervised agent, the shape is the same:

1. One named window per process.
2. Pipe output through `tee /tmp/<name>.log` so history survives scrollback limits and `grep` works later.
3. Wait for a *readiness signal* — a log line, an open port, a sentinel echo — not a fixed `sleep`.
4. Read recent state from `capture-pane`, historical state from the log file.
5. Leave the window alive so the user can attach and watch.

## Deeper guides (load on demand)

- `references/long-running.md` — dev servers (next/vite/rails/django), test watchers (jest/vitest/pytest-watch), log tailing (kubectl/docker/journalctl), long builds, tunnels (ngrok/ssh -L/port-forward), docker compose, supervised agents, ssh sessions. Read this whenever you're about to start any of those.
- `references/repls.md` — interactive REPLs (Python, Node, psql, ipython, sqlite3). Indentation handling, multi-line blocks, when to bail out and write a script.
- `references/troubleshooting.md` — capture returns nothing/stale, send-keys lands wrong, nested sessions, truncated output, hung prompts, ANSI noise.
