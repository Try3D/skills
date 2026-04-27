# tmux troubleshooting

## `capture-pane` returns nothing or stale text

- The command hasn't produced output yet. Increase the `sleep` between `send-keys` and `capture-pane` (1s → 3s for slow startups like dev servers).
- You captured before the command printed. Use `-S -200` to scroll back further: `tmux capture-pane -t <target> -p -S -200`.
- Wrong target. Confirm with `tmux list-windows -t <session>` and `tmux list-panes -t <session>:<window>`.

## `send-keys` lands in the wrong window

- You omitted the window in the target. Always use `<session>:<window>`, not just `<session>` — the latter sends to the active window, which may be the user's.
- The window was renamed or killed. Re-list with `tmux list-windows -t <session>`.

## Nested tmux sessions

If `$TMUX` is set and you ran `tmux new-session` anyway, you've nested. Symptoms: `sessions should be nested with care, unset $TMUX to force` warning, or two prefix keys needed to do anything.

Fix: never call `tmux new-session` when `$TMUX` is set. Use `tmux new-window` in the existing session instead.

If you already nested, exit the inner session with `tmux kill-session -t <inner>` from outside, or detach with the inner prefix + `d`.

## Output is truncated

- Pane scrollback default is 2000 lines. For long-running processes, redirect to a log file when you launch:
  ```
  tmux send-keys -t <target> '<cmd> 2>&1 | tee /tmp/<name>.log' C-m
  ```
  Then read with `tail -n 500 /tmp/<name>.log` instead of `capture-pane`.
- Increase scrollback for the session: `tmux set-option -t <session> history-limit 50000`.

## Process keeps dying when window closes

- Killing a window kills the process inside it. If you want the process to outlive the window, you didn't need a window — but you almost always do, since the whole point is observability. Don't reach for `nohup`; just leave the window open.

## Prompt is hung waiting for input

- Send the missing input with `send-keys`. For a literal Enter use `C-m` or `Enter`. For Ctrl-C use `C-c`. For a password prompt, send the password followed by `C-m`.
- If you're not sure what it's waiting for, capture more scrollback: `tmux capture-pane -t <target> -p -S -500`.

## Color codes / ANSI escape junk in capture

`capture-pane -p` strips most escapes by default. If you still see them, add `-e` only when you want them, otherwise omit. For log-style clean output, prefer reading from the `tee`'d file.
