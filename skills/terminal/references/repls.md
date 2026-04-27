# Driving interactive REPLs through tmux

Interactive REPLs (Python, Node, psql, ipython, sqlite3, redis-cli, etc.) are stateful. Each line you send may produce output before you can send the next one. Piping a multi-line script into them via stdin loses the prompts, mangles indentation, and exits before you see anything.

## When to use a REPL vs. a script

Default to a script. A REPL is only the right tool when:
- The user explicitly asked for an interactive session.
- You need to inspect intermediate state between steps (variables, query results) and the next step depends on what you saw.
- The environment can only be entered interactively (e.g., a `kubectl exec` shell).

For everything else — calculations, batch transforms, "test this function with these inputs" — write a `.py`/`.js`/`.sql` file and run it. You get one tidy output instead of N round trips.

## REPL workflow inside tmux

1. Open the REPL in a named window:
   ```
   tmux send-keys -t <session>:repl 'python3' C-m
   sleep 1 && tmux capture-pane -t <session>:repl -p -S -50
   ```
   Confirm you see the prompt (`>>>`, `>`, `psql=#`) before sending input.

2. Send one logical statement at a time. For multi-line constructs (function defs, `if`/`for` blocks in Python), send the whole block as a single `send-keys` call with embedded newlines, then a blank line to terminate:
   ```
   tmux send-keys -t <session>:repl 'def is_prime(n):' Enter \
     '    if n < 2: return False' Enter \
     '    for i in range(2, int(n**0.5)+1):' Enter \
     '        if n % i == 0: return False' Enter \
     '    return True' Enter \
     '' Enter
   sleep 1 && tmux capture-pane -t <session>:repl -p -S -50
   ```
   Each `Enter` token sends a real newline. The trailing empty `Enter` closes the block in Python.

3. After each send, capture and verify:
   - Did the prompt return (`>>>`)?
   - Did you get a traceback or `IndentationError`? If yes, recover with `Ctrl-C` (`tmux send-keys -t ... C-c`) and try again.

4. Exit cleanly: `tmux send-keys -t <session>:repl 'exit()' C-m` (or `\q` for psql, `.exit` for sqlite3, `Ctrl-D` via `C-d`).

## Indentation gotchas

- **Spaces, not tabs.** REPLs handle both, but mixing them across `send-keys` calls produces `IndentationError`.
- **Heredocs and stdin pipes lose the prompt.** `python3 -i` will print results but the prompts and your inputs interleave unpredictably. Avoid for anything beyond a one-liner.
- **Bracketed paste mode.** Some terminals enable it; tmux usually strips it. If you see literal `^[[200~` in the pane, disable bracketed paste in the REPL (`%set bracketed_paste off` in IPython) or send via a here-file instead.

## Bail-out signal

If you've sent the same block twice and gotten an error both times, stop driving the REPL. Write the code to a file and run it:

```
tmux send-keys -t <session>:repl 'exit()' C-m
# write /tmp/work.py with the Write tool, then:
tmux send-keys -t <session>:repl 'python3 /tmp/work.py' C-m && sleep 1 && tmux capture-pane -t <session>:repl -p -S -200
```

You'll get the result faster and the user can re-run it themselves.
