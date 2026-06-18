---
name: dev-up
description: |
  Starts this folder's dev server on a specific port, opens a browser tab pinned to that
  port, and arms an error watcher — without touching other projects' tabs. Use when invoking
  `/dev-up <port>` or when asked to start/view/monitor the server of one copy on a port
  without disturbing the others running in parallel.
---

# dev-up

## Why it exists

Several copies of the same project (branches in sibling folders), each on its own port, with
several Claude Code instances in the same browser, all share **one single MCP tab group**
(`claude-in-chrome`) — so one instance can click on another's tab. This skill pins each session
to **its** port.

## Invocation

`/dev-up <port>`. With no argument, ask for the port. The port (`PORT`) is the source of
truth: the server binds to it, the tab navigates to it, the Monitor watches its log.

Two principles: **inspect before asking** (offer options with real data — apps, browsers —
never in the abstract); **investigate, don't assume** (stack-agnostic: find out how the project
runs by looking at the files).

## Flow

### 1. Is the port already in use?

```bash
ss -ltn "sport = :PORT" | grep -q LISTEN && echo BUSY || echo FREE
```

(`ss` checks who's *listening*; `curl -sf` would give a false "free" if the root responds
404/500.) **Busy** → reuse it, skip to step 3. **Free** → start it.

### 2. Start the server

1. **Find the dev command.** Read the manifest. **Monorepo** (`workspaces`, `turbo.json`,
   `pnpm-workspace.yaml`, `nx.json`): the root `dev` starts *all* apps — list the workspaces
   with a `dev` script and, if there's more than one, ask which. Work in the app's dir.
2. **Override the port:**
   - Command already pins a port (`--port N`, `-p N`, `PORT=N`) → **swap the number** for `PORT`.
   - Doesn't pin one → add the framework's flag (if unsure, `<bin> --help` or `find-docs`;
     fallback `PORT=PORT`).
   - **Force the exact port** when possible (e.g. Vite `--strictPort`) — no auto-increment.
   - **Heads-up:** if the app pins its base URL/port elsewhere (auth callbacks, CORS origins,
     OAuth redirects — grep the env/config for a `localhost:<port>`), running on a *different*
     port can silently break login/CORS. See *Troubleshooting / gotchas*.
3. **Run it in the background** (`run_in_background: true`), with a dedicated log:

   ```bash
   cd apps/<app> && ./node_modules/.bin/<bin> dev --port PORT > /tmp/dev-up-PORT.log 2>&1
   ```

   If you edited env (`.env`/config) this session, **source it in the launch command**
   (`set -a; . ./.env; set +a; <bin> dev ...`): a background child inherits the launching
   shell's env — which a version manager (mise/asdf/direnv) may have populated at boot with
   *stale* values — not your file edit. See *Troubleshooting / gotchas*.

4. **Wait for the TCP bind** in a **blocking** (foreground) Bash — do *not* use
   `run_in_background` here. If the wait runs in the background, you move on and connect the tab
   before the server exists; `navigate` hits a port with no listener. Block until the bind, then
   go to step 3:

   ```bash
   until ss -ltn "sport = :PORT" | grep -q LISTEN; do sleep 0.5; done
   ```

   ~20s without coming up → read the log and report the error. (A first build with no cache can
   take much longer than that — if the log shows "compiling", keep waiting.) If the log shows a
   **missing-module / dependency error** instead (common in a fresh checkout or worktree), the
   deps aren't installed: install with the lockfile's package manager (`package-lock.json`→`npm i`,
   `pnpm-lock.yaml`→`pnpm i`, `yarn.lock`→`yarn`, `bun.lockb`/`bun.lock`→`bun i`) and retry once.

5. **"Server already running" even with the port free.** Some dev servers (e.g. recent Next)
   allow only *one* instance per directory, regardless of port. If the command refuses with
   "another server is already running" even though `ss` says FREE, there's another instance in
   the same dir on another port — offer to reuse it or take it down (with confirmation), don't
   force it.

### 3. Connect and pin the tab

1. **Select the browser.** `list_connected_browsers`. **Exactly one connected → use it directly**
   (no question — the round-trip isn't worth it). **Two or more → ask** (`AskUserQuestion`) listing
   all of them (names + deviceIds) + the option to confirm in Chrome, recommending the device
   marked "on this computer": `localhost:PORT` only resolves on the machine where the server
   started.
2. **Find or create the tab.** `tabs_context_mcp` (`createIfEmpty: true`): if there's already a
   tab on `localhost:PORT`, reuse it; otherwise `tabs_create_mcp`. `navigate` to
   `http://localhost:PORT` (if it doesn't load, try `https://`).
3. **Record the `tab_id` as `TARGET_TAB_ID`.**
4. **Check the final URL — but don't trust `navigate`'s return.** It sometimes returns the old
   URL (`chrome://newtab/`) before the navigation settles. Confirm the real URL with a second
   `tabs_context_mcp` (or `read_page`) before deciding. If it redirected to login (`/login`,
   `/auth`, `/sign-in`, `/entrar`, `/acesso`, `/conta` and the like), **stop and ask the user to
   log in** — you never enter credentials. Once they say so, continue. (If the URL *stayed* on
   the requested route but the screen shows something else, it's a client-side redirect from
   hydration, not auth — treat it as loaded.)

No screenshot — hand control back.

### 4. Arm the watcher (Monitor tool)

The watcher is this skill's **contract** — it's what delivers value while you hand control back.
Always arm it, even when `/dev-up` is just one step of a larger job: don't walk off to the task
leaving the server unwatched.

It watches **errors/warnings in the log** and the **port (if it drops)**. Watching the port —
not a PID — survives dev server restarts. Check the listener with `ss` (not `lsof`):
`lsof -ti tcp:PORT` lists *any* process attached to the port, including the connected
**browser** — when it disconnects, it falsely looks like the server went down:

```bash
tail -n 0 -f /tmp/dev-up-PORT.log | grep -E --line-buffered \
  "[Ee]rror|Exception|Traceback|[Ww]arn|Failed to compile|unhandled|ECONNREFUSED|EADDRINUSE|panic|FATAL" &
TPID=$!
miss=0
while true; do
  if ss -ltn "sport = :PORT" | grep -q LISTEN; then miss=0; else miss=$((miss+1)); [ $miss -ge 2 ] && break; fi
  sleep 2
done
kill $TPID 2>/dev/null
echo "SERVER dropped on port PORT — ask me to start it again"
```

`description: "server on port PORT"`, `persistent: true`. **Don't** set a short `timeout_ms` —
`persistent` already keeps the watcher alive; a few-minute timeout leaves the server unwatched in
the middle of a long session, with no warning. `tail -n 0` = only new lines; two misses (~4s
without the port) = really down, not a restart. Server dropped → offer to start it again.
Relevant error → `PushNotification` — but **triage first**: a transient infra error (DNS
`EAI_AGAIN`, a connection to an external service, `DeprecationWarning`) isn't actionable, not
worth a push. If the server is mid schema migration and the log fills with expected
column/table errors, pause the watcher (`TaskStop`) until it stabilizes instead of flooding.

### 5. Hand control back

Report briefly: port, log, `TARGET_TAB_ID`, Monitor armed. Stop.

## Golden rule of isolation

- **You own one tab: `TARGET_TAB_ID`** (`localhost:PORT`). Every interaction with the app goes
  through it.
- **Confirm the URL when connecting**, then use the `tab_id` directly; re-validate only if an
  action fails.
- **Never** act on a `localhost:<other-port>` tab — those belong to other projects/instances.
- Tabs that aren't yours aren't your problem — don't touch them, don't block them.
- **Never close or modify the shared MCP tab group itself** — it's collective, and other Claude
  instances live in sibling tabs inside it. Operate only on your own `tab_id` (see *Shutting down*
  for the last-tab hazard).

## Interacting with and debugging the tab (after setup)

- **Text > screenshot.** To inspect/click, use `find` or `read_page` (`filter=interactive`) —
  cheap and gives refs. Screenshot (~1.5k tok, viewport only, no refs) only for the visual
  (layout/CSS/render). **The first navigation to a route triggers an on-demand compile**
  (Vite/Turbopack/webpack) — while it runs the renderer is busy and `screenshot`/CDP actions
  time out ("renderer busy"). Wait for the route to settle (or retry once); during the build,
  prefer text / `javascript_tool` reads over screenshots.
- **Predictable steps → `browser_batch`.** Chain known actions (`navigate`+`read_page`,
  `form_input` across several refs, click+type+press) in a single call and cut round-trips. It
  **doesn't pass output→input**: if the next step depends on a ref you only discover now, go the
  normal way.
- **The server is watched; the client is on demand.** The Monitor is passive and notifies you on
  its own — but only from the **server log** (stdout). The **client** side (React, `fetch`
  4xx/5xx, a browser exception) lives in the **browser console**, which the shell can't observe →
  **there's no automatic notification**. To see it, run `read_console_messages` (`onlyErrors`)/
  `read_network_requests` on the tab **when** the screen looks broken or an action fails — don't
  count on a spontaneous alert.
- **Storage/cookies.** `javascript_tool` reads `localStorage`/`sessionStorage` and non-HttpOnly
  cookies directly. A **HttpOnly** session cookie is invisible to JS — and seeing it would
  require a CDP/debug port, which would abandon this setup; so it's out of the skill's reach.

## Troubleshooting / gotchas

Stack-agnostic edge cases seen in practice — check these when something's off:

- **An edited `.env`/config doesn't take effect.** A running server won't pick up env changes,
  and a server you launch in the background inherits the env of the *shell that launched it* —
  which a version manager (mise/asdf/direnv) may have populated at boot with stale values. If you
  changed env this session, source it in the launch command (`set -a; . ./.env; set +a; …`) or
  relaunch from a fresh shell. Don't trust an edit alone.
- **The app hardcodes its base URL/port.** Auth callbacks, CORS origins, OAuth redirect URIs and
  cookie domains are often pinned to a specific `http://localhost:<port>` in env/config. Run on a
  *different* port than they expect and login redirects, CORS, or cookies break silently (often a
  bounce to `/login`). Grep the config/env for the port before switching; if it's pinned, warn the
  user that another port breaks those flows.
- **The background server dies when the launching process exits or pauses** — notably while it's
  off dispatching long-running subagents. The watcher's port-drop detection catches this and you
  offer a restart; if you're about to hand off to long subagent work, re-check the port
  (`ss -ltn "sport = :PORT"`) when you come back instead of assuming it's still up.
- **Fresh checkout/worktree → missing dependencies.** A missing-module error on startup means deps
  aren't installed — install with the lockfile's package manager and retry (see step 2).
- **The port keeps coming back after you kill it.** An external supervisor (a `turbo dev`/`bun dev`
  in your terminal, a process manager) is respawning it — don't kill blindly, tell the user (see
  *Shutting down*).

## Shutting down

When the task is done, **ask whether to shut down**. If yes, in this order:

1. `TaskStop` on the **server's background task** (if you started it with `run_in_background`) —
   just killing the PID isn't enough: the harness/supervisor may respawn it and the port "comes
   back".
2. `TaskStop` on the Monitor.
3. Take down the listener with `fuser -k PORT/tcp` (kills exactly who holds the listening socket;
   `lsof -ti tcp:PORT | xargs kill` would catch the **browser** attached to the port, not the
   server). If the port keeps coming back anyway, there's an external supervisor — tell the user,
   don't keep killing blindly.
4. **Close only your own tab — nothing else.** `tabs_close_mcp` on `TARGET_TAB_ID` and only that.
   "Close/finish" means *only the tab(s) you were monitoring*: never close the **shared MCP tab
   group** or any tab you didn't open — other Claude instances are working in sibling tabs in that
   same group, and taking the group down kills their tabs too. ⚠️ **Last-tab hazard:** if yours is
   the *last* tab in the group, closing it can make the browser collapse the whole group. If it
   might be the last, or you're unsure whether others are still attached, **leave the tab open**
   (or navigate it to `about:blank`) and just stop the server + monitor — never risk the group to
   tidy up one tab. Then remove the log.
