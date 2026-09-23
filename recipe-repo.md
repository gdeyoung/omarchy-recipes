# The recipe repo — managing an Omarchy machine as code

> One private git repo per machine holds every deliberate non-stock change: config snapshots, scripts, unit files, plugin sources. The inventory file is the audit; the install script is the replay. A machine managed this way can be rebuilt from nothing — and, the part that actually pays, can answer "what's different about this box, and why" a year later.

## The problem

Omarchy customization lands in a dozen places the package manager never sees:

| Change | Where it lives |
|---|---|
| Keybinds, window rules, monitors | `~/.config/hypr/*.lua` |
| Bar layout, shell plugins | `~/.config/omarchy/` (`shell.json`, `plugins/`) |
| Helper scripts | `~/.local/bin/`, `/usr/local/bin/` |
| Services, udev rules | `~/.config/systemd/user/`, `/etc/udev/rules.d/` |
| Native plugins | compiled `.so` files, locked to the Hyprland build |
| Theme/wallpaper hooks | `~/.config/omarchy/hooks/` |

None of this is owned by pacman. Six months of frictionless customization later, the machine works — but a `refresh`, a disk failure, or a new laptop turns "what did we change?" into an outage. (What each update mechanism actually rewrites is measured in [update-safety.md](update-safety.md); this doc is the discipline that makes the question answerable.)

## The three artifacts

1. **The kit** — a private git repo, one per machine. Holds every deliberate non-stock change: config snapshots, scripts, unit files, udev rules, and the full source of plugins that are ours. Private on purpose — it names real hosts and real paths.
2. **The inventory** (`RECIPE.md`) — a state table of everything non-stock: each path marked stock / modified / added / disabled, and **which file in the repo accounts for it**. Plus a pins table for third-party plugins (see [plugin-hygiene.md](plugin-hygiene.md)) and a "machine facts" section stating what the recipe assumes (filesystem layout, GPU, display scale) — so a rebuild on different hardware fails at the right place instead of subtly.
3. **The replay** (`install.sh`) — an ordered script that turns a fresh Omarchy install into this machine: packages → third-party plugins → our plugins → config snapshots → services. Perfection isn't required; honesty about order is.

## The rule

> Any non-stock change gets an inventory row **and** a home in the repo **in the same commit** — or it gets reverted.

That one sentence is the whole discipline. Without it the kit decays into a snapshot of the machine's first week. With it, the inventory is a live map an auditor can walk (see [the recipe doctor](https://github.com/gdeyoung/praxis/blob/main/hermes/recipe-doctor.md)).

## Why the inventory, when you have the script?

The install script is the *replay*; the inventory is the *audit*. You run `install.sh` once per machine and trust it on faith thereafter. You read `RECIPE.md` constantly — to answer "what owns this behavior," to decide whether an update can break something, and to review a change someone (including past-you) made without telling anyone. A script tells you what it does; only an inventory tells you what the machine *is*, including everything the script didn't install.

## Snapshots, and their failure mode

Config snapshots are copies **into** the repo of live files. The failure mode runs the other direction: live config evolves during tuning sessions while the repo copy sits still — a rebuild then resurrects last month's tuning with no error anywhere. The doctor's `diff -q` per snapshot catches this; until you have one, the habit is: change live → refresh snapshot → commit, before moving on.

## Structure

```
kit/
  RECIPE.md      inventory + pins + machine facts
  install.sh     ordered replay
  hypr/          config snapshots (~/.config/hypr, shell.json)
  own/           our plugin sources (each also published standalone)
  system/        udev rules, systemd user units, root scripts
  ai/            agent-stack notes (sanitized snapshots, no secrets)
```

## Pitfalls

- **Stale snapshots** (above) are the #1 finding the first time a drift audit runs — budget for finding several.
- **One source of truth for pins.** If a checker (or a second doc) hand-copies pin SHAs, a transposed digit "proves" a healthy install is drift. The recipe file is the only place pins are written; everything else reads them.
- **Don't track what you didn't change.** A stock config committed "for safety" adds diff noise and implies ownership you don't have. The inventory's state column documents "stock, not tracked" — that's a fact worth recording, not a file worth committing.
- **Install order is dependency order.** Base layers first (e.g. the floating-window plugin other things assume), your plugins next, config snapshots after the things they reference exist, shell restart last — and only after native `.so` plugins have been rebuilt for the running Hyprland.
