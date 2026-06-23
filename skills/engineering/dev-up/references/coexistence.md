# Coexistence — sharing the MCP tab group

Read this when [`SKILL.md`](../SKILL.md) step 3 shows **more than one connected browser** or
**other `localhost:<port>` tabs in the group**. If you're the only tab in the group, none of this
applies — skip it.

One `claude-in-chrome` MCP tab group is **shared**: several Claude instances live in sibling tabs
inside it, and any instance can act on any tab. Everything here protects that shared space.

## The one rule

**You own exactly one tab: `TARGET_TAB_ID`** (`localhost:PORT`). Every interaction with the app
goes through it.

- Confirm the URL once when connecting, then use the `tab_id` directly; re-validate only if an
  action fails.
- **Never** act on a `localhost:<other-port>` tab — it belongs to another project or instance.
- Tabs that aren't yours aren't your problem: don't touch them, don't block them.
- **Never close or modify the tab group itself.** Operate only on your own `tab_id`.

## Last-tab hazard (shutdown)

Closing your tab is safe *unless it's the last one in the group* — then the browser can collapse
the whole group, taking other instances' tabs with it.

If yours might be the last tab, or you can't tell whether other instances are still attached,
**leave the tab open** (or `navigate` it to `about:blank`) and just stop the server and watcher.
Never risk the group to tidy up a single tab.
