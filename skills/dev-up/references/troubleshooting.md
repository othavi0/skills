# Troubleshooting — startup gotchas

Stack-agnostic edge cases seen in practice. Reach here from [`SKILL.md`](../SKILL.md) step 2 when
startup misbehaves.

## Won't bind / errors on launch

- **Missing-module / dependency error.** Common in a fresh checkout or worktree — the deps aren't
  installed. Install with the lockfile's package manager and retry once:
  `package-lock.json`→`npm i`, `pnpm-lock.yaml`→`pnpm i`, `yarn.lock`→`yarn`,
  `bun.lockb`/`bun.lock`→`bun i`.
- **"Another server is already running" despite a FREE port.** Some dev servers (e.g. recent Next)
  allow only *one* instance per directory, regardless of port — there's another instance in the
  same dir on another port. Offer to reuse it or take it down (with confirmation); don't force it.
- **The port keeps coming back after you kill it.** An external supervisor (a `turbo dev`/`bun
  dev` in a terminal, a process manager) is respawning it. Don't kill blindly — tell the user.

## Runs, but behaves wrong

- **An edited `.env`/config doesn't take effect.** A running server won't pick up env changes, and
  a server you launched in the background inherits the *launching shell's* env — which a version
  manager (mise/asdf/direnv) may have seeded at boot with stale values. Source it in the launch
  command (`set -a; . ./.env; set +a; …`) or relaunch from a fresh shell. Don't trust an edit
  alone.
- **The app hardcodes its base URL/port.** Auth callbacks, CORS origins, OAuth redirect URIs and
  cookie domains are often pinned to a specific `http://localhost:<port>` in env/config. Run on a
  *different* port than they expect and login (often a silent bounce to `/login`), CORS, or cookies
  break. Grep the config/env for the port before switching; if it's pinned, warn the user that
  another port breaks those flows.
- **The background server dies when the launching process exits or pauses** — notably while it's
  off dispatching long-running subagents. If *you* launched it with `run_in_background` (the normal
  path), the harness re-invokes you when that task exits, so the death is signalled for free. Just
  re-check the port (`ss -ltn "sport = :PORT"`) when you come back from long subagent work.

## Watching a server you didn't start (port-poll fallback)

The default watcher (`SKILL.md` step 3) only tails the log; it relies on step 2's
`run_in_background` task to signal the server's death. If you **reused** a server another session
or supervisor started (step 1 was BUSY), there's no such task — its death is invisible. Add a
second Monitor that polls the port and exits when it drops, so you still get the alert:

```bash
miss=0
while true; do
  if ss -ltn "sport = :PORT" | grep -q LISTEN; then miss=0; else miss=$((miss+1)); [ $miss -ge 2 ] && break; fi
  sleep 2
done
echo "SERVER dropped on port PORT — ask me to start it again"
```

`persistent: true`, no `timeout_ms`. Two misses (~4s) = really down, not a restart.
