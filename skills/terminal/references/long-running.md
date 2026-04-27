# Long-running and observable processes

Dev servers, test watchers, log tails, build watchers, tunnels, and supervised agents all share the same shape: they start, keep running, emit output continuously, and need to be checked on without being killed. tmux is the right harness for all of them.

## Dev servers (next, vite, rails, django, flask, uvicorn, ...)

1. Start in a named window so the user can find it:
   ```
   tmux new-window -d -t <session> -n dev
   tmux send-keys -t <session>:dev 'npm run dev 2>&1 | tee /tmp/dev.log' C-m
   ```
   The `tee` matters — dev servers can print megabytes of HMR noise that overflows tmux scrollback in minutes.

2. Wait for readiness before reporting "running". Don't just `sleep 5` and hope — poll:
   ```
   for i in {1..30}; do
     grep -qE 'Local:|Listening on|Ready in|started server' /tmp/dev.log && break
     sleep 1
   done
   tmux capture-pane -t <session>:dev -p -S -50
   ```
   Then verify the port actually accepts connections: `curl -sf http://localhost:3000 >/dev/null && echo up`.

3. Restarting: send `C-c`, wait for the prompt to return, then re-run:
   ```
   tmux send-keys -t <session>:dev C-c
   sleep 1
   tmux send-keys -t <session>:dev 'npm run dev' C-m
   ```
   Don't `kill-window` and recreate — you lose scrollback and the user loses their attached pane.

4. Reading errors after a code change: capture the *tail* of the log file, not the pane (HMR resets the pane often):
   ```
   tail -n 200 /tmp/dev.log
   ```

## Test watchers (jest --watch, vitest, pytest-watch)

- Watchers expect single-key input (`f` for failed-only, `p` for filter, `q` to quit). Send them as bare keys, not as a string + Enter:
  ```
  tmux send-keys -t <session>:tests f
  ```
- After triggering a run, wait for the "Tests:" summary line before capturing:
  ```
  for i in {1..30}; do
    tmux capture-pane -t <session>:tests -p -S -100 | grep -qE 'Tests:|passed|failed' && break
    sleep 1
  done
  tmux capture-pane -t <session>:tests -p -S -200
  ```
- Don't conflate "watcher idle" with "tests passed". Parse the actual summary line.

## Tailing logs (tail -f, kubectl logs -f, docker logs -f, journalctl -f)

- These are read-only consumers. Open in their own window, leave them alone, and grep when you need something:
  ```
  tmux new-window -d -t <session> -n logs
  tmux send-keys -t <session>:logs 'kubectl logs -f deploy/api -n prod | tee /tmp/api.log' C-m
  ```
  Then: `grep -i error /tmp/api.log | tail -n 50` instead of fighting `capture-pane`.
- For multi-source tailing, one window per source. Don't multiplex with `&` — you lose the ability to stop one without the other.

## Long builds (webpack, turbo, bazel, cargo build --release, docker build)

- Builds are long but finite. Run in a window, then poll for the prompt to return:
  ```
  tmux send-keys -t <session>:build 'pnpm build 2>&1 | tee /tmp/build.log; echo BUILD_DONE_$?' C-m
  until tmux capture-pane -t <session>:build -p -S -20 | grep -q BUILD_DONE_; do sleep 5; done
  tmux capture-pane -t <session>:build -p -S -200 | tail -n 50
  ```
  The sentinel `BUILD_DONE_$?` makes "done" detectable and carries the exit code.
- For builds longer than ~5 min, tell the user it's running and the window name; don't block the conversation polling.

## Tunnels and port forwards (ngrok, cloudflared, ssh -L, kubectl port-forward)

- Always in their own window. They print their URL/port once at startup, then go silent — capture immediately after launch:
  ```
  tmux send-keys -t <session>:tunnel 'ngrok http 3000' C-m
  sleep 3
  tmux capture-pane -t <session>:tunnel -p -S -50 | grep -E 'https://.*ngrok'
  ```
- If they die silently (network blip, auth expiry), the window stays open with a dead process. Re-check before assuming the URL still works.

## Supervised agents and CLIs (claude, aider, cursor-agent, custom scripts)

- Treat like a REPL: own window, one logical input at a time, capture between sends. See `repls.md`.
- For agents that produce streaming output you want to watch live, also `tee` to a log file so you can grep history:
  ```
  tmux send-keys -t <session>:agent 'claude --print "do X" 2>&1 | tee /tmp/agent.log' C-m
  ```
- Never run two supervised agents in the same window. They'll fight over stdin.

## SSH and remote shells

- `ssh user@host` in its own window. Treat the remote prompt as a REPL — send commands with `send-keys`, capture output the same way.
- For long-lived remote work, prefer running `tmux` *on the remote* and attaching, so the work survives connection drops. Open the local ssh window, then `tmux new-session -s work` on the remote.

## docker compose / docker-compose up

- `docker compose up` (no `-d`) is just a long-running process with multiplexed logs — handle it like a dev server. Window named `compose`, `tee` to `/tmp/compose.log`, poll for "started" lines per service.
- Prefer `up` over `up -d` when you want the user to see crashes live. Use `-d` only if the user explicitly wants detached mode.

## Common shape

For every case above the pattern is the same:

1. Named window per process.
2. `tee` to `/tmp/<name>.log` if output is voluminous or you'll need to grep history.
3. Poll for a *readiness signal* (port open, "Ready" line, sentinel echo) — don't guess with `sleep`.
4. Read from the log file for history, from `capture-pane` for current pane state.
5. Leave the window alive; the user is your second pair of eyes.
