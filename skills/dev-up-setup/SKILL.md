---
name: dev-up-setup
description: One-time setup that gets `dev-up` working on a machine — connect the claude-in-chrome browser, name the device, and verify. Run with `/dev-up-setup`.
disable-model-invocation: true
---

# dev-up-setup

Run this **once per machine/browser**, before the first [`dev-up`](../dev-up/SKILL.md). It gets the
`claude-in-chrome` extension connected and — the part that matters when you have several browsers
or computers — gives this browser a **name**, so `dev-up` can pick the right one without guessing.

You only need this when `dev-up` reports no connected browser, when the browser shows up unnamed,
or when you've added a new machine to the mix.

## Steps

### 1. Make sure the extension is installed and signed in

`dev-up` drives the browser through the **claude-in-chrome** extension (and its MCP server). If
it's already installed and signed into the same Claude account, skip to step 2.

If not, install "Claude in Chrome" from the official source and sign in with this account — follow
Anthropic's current install docs rather than guessing the steps, since the flow changes. The
extension must run in the browser **on the machine where you'll run the dev server** (`localhost`
only resolves there).

### 2. Connect and name this browser

Run `switch_browser`. It broadcasts a connection request to every browser with the extension and
waits (up to 2 min) for you to click **Connect** in the one you want — and on that screen you can
**name it**. Give it something that tells your machines apart at a glance: `"Otávio Brave"`,
`"work-laptop Chrome"`, `"desktop Firefox"`.

Naming is the whole point of doing this now: with two browsers or two computers connected, `dev-up`
asks which to use and lists them by name. An unnamed device is just a deviceId — easy to pick wrong.

### 3. Verify

Run `list_connected_browsers`. Confirm this browser appears **with the name you gave it** and is
marked as on this computer. If the name didn't stick, repeat step 2.

That's it — `dev-up` will now find and pin tabs in this browser. To rename later, just run
`/dev-up-setup` again and reconnect with a new name.
