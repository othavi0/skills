---
name: dev-up
description: |
  Starts this folder's dev server on a chosen port, opens one browser tab pinned to that port,
  and arms an error watcher — then hands control back. Use when invoking `/dev-up <port>`, or
  when asked to start, open, view, run, or monitor the dev server of a project on a specific
  port — especially when other servers or tabs are running in parallel and must not be disturbed.
---

# dev-up

Pins one dev server to one **port**, opens one browser **tab** you own at that port, and arms a
**watcher** on the log — so you can hand control back and walk away. The port is the source of
truth: the server binds it, the tab navigates to it, the watcher follows it.

When several copies of a project run side by side (sibling worktrees, or multiple Claude
instances sharing one `claude-in-chrome` MCP tab group), that one-tab discipline is what stops
you from clicking another session's tab. Those isolation rules live in
[`references/coexistence.md`](references/coexistence.md) — read it the moment step 4 shows **more
than one browser, or other `localhost` tabs in the group**.

First time on this machine (extension not installed, or no browser connected)? Run
[`dev-up-setup`](../dev-up-setup/SKILL.md) once first.

## Invocation

`/dev-up <port>`. With no argument, ask for the port.

Two habits throughout: **inspect before asking** (offer options built from real data — apps,
browsers — never in the abstract), and **investigate, don't assume** (stack-agnostic: learn how
the project runs by reading its files, not by guessing).

## Flow

### 1. Is the port already in use?

```bash
ss -ltn "sport = :PORT" | grep -q LISTEN && echo BUSY || echo FREE
```

`ss` checks who's *listening* — `curl` would report a false "free" when the root answers 404/500.
**Free** → start it (step 2). **Busy** → it's already running; skip step 2's launch, go to step 3
(still arm the watcher — on `/tmp/dev-up-PORT.log` if a prior `dev-up` created it).

### 2. Start the server

1. **Find the dev command.** Read the manifest. In a **monorepo** (`workspaces`, `turbo.json`,
   `pnpm-workspace.yaml`, `nx.json`) the root `dev` starts *every* app — list the workspaces with
   a `dev` script, ask which if there's more than one, and work in that app's dir.
2. **Override the port.** Command already pins one (`--port N`, `-p N`, `PORT=N`) → swap the
   number for `PORT`. Doesn't → add the framework's flag (`<bin> --help` or `find-docs` if unsure;
   fallback `PORT=PORT`). Force the exact port where the framework allows (e.g. Vite
   `--strictPort`) so it can't auto-increment onto a neighbour. **Before settling on a port other
   than the app's usual one**, grep env/config for a hardcoded `localhost:<port>` (auth callbacks,
   CORS, OAuth redirects): if it's pinned, a different port silently breaks login/CORS — see
   [`references/troubleshooting.md`](references/troubleshooting.md).
3. **Run it in the background** (`run_in_background: true`), with a dedicated log:

   ```bash
   cd apps/<app> && ./node_modules/.bin/<bin> dev --port PORT > /tmp/dev-up-PORT.log 2>&1
   ```

   Background is load-bearing twice: it frees the shell, **and** the harness re-invokes you when
   the task exits — so the server dying is caught for free, no port-polling needed. If you edited
   env (`.env`/config) this session, source it in the launch command
   (`set -a; . ./.env; set +a; <bin> dev ...`) — a background child inherits the launching shell's
   env, which a version manager may have seeded with stale values, not your edit.
4. **Wait for the TCP bind in a blocking (foreground) Bash** — never `run_in_background` here, or
   you move on before a listener exists:

   ```bash
   until ss -ltn "sport = :PORT" | grep -q LISTEN; do sleep 0.5; done
   ```

   ~20s with no bind → read the log and report. (A first, uncached build can take far longer — if
   the log says "compiling", keep waiting.) A **missing-module / dependency error**, or "another
   server is already running" despite a FREE port → see
   [`references/troubleshooting.md`](references/troubleshooting.md).

### 3. Arm the watcher — the moment the port binds

Do this **before** the tab. It's the skill's **contract** — what earns its keep while you're away
— and arming it here, attention still on the terminal, is what stops it slipping through the
handback. One Monitor, persistent, filtering the log for trouble:

```bash
tail -n 0 -f /tmp/dev-up-PORT.log | grep -E --line-buffered \
  "[Ee]rror|Exception|Traceback|[Ww]arn|Failed to compile|unhandled|ECONNREFUSED|EADDRINUSE|panic|FATAL"
```

`description: "errors on port PORT"`, `persistent: true`, **no `timeout_ms`** (a timeout would
silently stop watching mid-session). `tail -n 0` = new lines only. **No port-polling here** — the
server's death already re-invokes you via step 2's background task; the Monitor only catches errors
while it's alive. (Reused a server you didn't start? Its death won't auto-signal — re-check the
port when you return, or add the port-poll fallback in
[`references/troubleshooting.md`](references/troubleshooting.md).)

**Triage before you push.** A real error → `PushNotification`. A transient infra hiccup (DNS
`EAI_AGAIN`, an external-service blip, a `DeprecationWarning`) isn't actionable — don't push. Log
flooding with *expected* errors during a schema migration → `TaskStop` the watcher until it
stabilises instead of spamming.

### 4. Connect and pin the tab

1. **Select the browser.** `list_connected_browsers`.
   - **One connected** → use it directly; the round-trip isn't worth a question.
   - **Two or more** (several browsers, or several machines) → `AskUserQuestion` listing **every**
     browser (display name + deviceId) plus the option to *confirm in Chrome*, recommending the
     device marked on this computer — `localhost:PORT` only resolves where the server runs. Then
     `select_browser` with the chosen deviceId. → also read
     [`references/coexistence.md`](references/coexistence.md) now: more than one browser means a
     shared group.
   - **A browser with no friendly name, or unsure which machine?** The *confirm in Chrome* option
     calls `switch_browser`, which lets the user pick *and name* it on connect (e.g. "Otávio
     Brave"), so it's labelled next time.
2. **Find or create the tab.** `tabs_context_mcp` (`createIfEmpty: true`): reuse an existing tab on
   `localhost:PORT`, else `tabs_create_mcp`, then `navigate` to `http://localhost:PORT` (try
   `https://` if it won't load). If the context shows **other `localhost:<port>` tabs**, you're in
   a shared group → [`references/coexistence.md`](references/coexistence.md).
3. **Record the `tab_id` as `TARGET_TAB_ID`** — the one tab you own.
4. **Confirm the final URL — don't trust `navigate`'s return** (it sometimes reports the old
   `chrome://newtab/` before the nav settles). Re-check with `tabs_context_mcp` or `read_page`. If
   it redirected to login (`/login`, `/auth`, `/sign-in`, `/entrar`, `/acesso`, `/conta` …),
   **stop and ask the user to log in** — you never enter credentials; continue once they confirm.
   (URL unchanged but the screen differs → a client-side hydration redirect, not auth; treat as
   loaded.)

No screenshot.

### 5. Hand control back

Gate first: the watcher is the contract, so **don't report until `TaskList` shows the Monitor task
running** — no Monitor, the job isn't done, however good the tab looks. Then report: port, log
path, `TARGET_TAB_ID`, Monitor armed. Stop.

## After setup

- **Driving or debugging the app** through `TARGET_TAB_ID` (clicks, forms, reading the console) →
  [`references/interacting.md`](references/interacting.md).
- **Something's off at startup** (env not taking effect, hardcoded base URL, deps, port respawns)
  → [`references/troubleshooting.md`](references/troubleshooting.md).
- **Coexisting with other tabs/instances** → [`references/coexistence.md`](references/coexistence.md).

## Shutting down

When the task is done, **ask whether to shut down**. If yes, in order:

1. `TaskStop` the server's background task (just killing the PID lets a supervisor respawn it).
2. `TaskStop` the watcher.
3. `fuser -k PORT/tcp` — kills exactly who holds the listening socket (`lsof -ti tcp:PORT | xargs
   kill` would also catch the **browser** attached to the port). If the port keeps coming back,
   an external supervisor is respawning it — tell the user, don't kill blindly.
4. **Close only `TARGET_TAB_ID`** with `tabs_close_mcp` — never the shared group, never a tab you
   didn't open. ⚠️ If yours might be the *last* tab in the group, closing it can collapse the whole
   group; the last-tab hazard and the rest of the shared-group rules are in
   [`references/coexistence.md`](references/coexistence.md). Then remove the log.
