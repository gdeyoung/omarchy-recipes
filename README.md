# omarchy-recipes

> How to run a heavily customized Omarchy machine as code — the recipe-repo discipline, update-safety model, and plugin hygiene that keep an Arch + Hyprland desktop rebuildable six months after you stopped remembering what you changed.

Sibling of [praxis](https://github.com/gdeyoung/praxis), the operating notes from our self-hosted AI agent fleet. Praxis covers the agent platform; this repo covers the Omarchy desktops that fleet runs on. Same rules, different layer.

## Why this exists

Omarchy makes Arch + Hyprland daily-drivable, and its plugin system makes customization frictionless — which is exactly the trap. Every change lands somewhere different (`~/.config/hypr/`, `~/.config/omarchy/`, `~/.local/bin`, systemd user units, udev rules, compiled `.so` files), and none of it is tracked by the package manager. Six months in, the machine works, but nobody can say what differs from stock or how to rebuild it. Then an update, a refresh, or a new laptop turns archaeology into an outage.

Everything here was **measured on real Omarchy laptops** (Omarchy 4.x, Hyprland 0.56, Quickshell bar). The failure modes documented are ones that actually bit — not folk wisdom.

## Contents

| Doc | What it covers |
|---|---|
| [`recipe-repo.md`](recipe-repo.md) | **The recipe repo** — one private repo per machine: an inventory file, a third-party pin table, and an install script that replays it all. The same-commit rule that keeps it honest |
| [`update-safety.md`](update-safety.md) | **What actually gets overwritten** — `omarchy update` vs `refresh` vs `plugin update`, measured from package file lists and libalpm hooks, and the placement rules that follow |
| [`plugin-hygiene.md`](plugin-hygiene.md) | **Third-party plugins and your own in one bar** — commit pins, namespacing, fork-vs-patch, disable-never-uninstall, build-locked native plugins, the marketplace path |
| [`stock-first.md`](stock-first.md) | **Stock tooling before forks** — inventory what Omarchy already ships before building; the config flag (voxtype remote mode) that replaced a plugin fork, plus the E2E mic-pipeline test pattern |
| [The recipe doctor](https://github.com/gdeyoung/praxis/blob/main/hermes/recipe-doctor.md) | The drift auditor that turns the inventory executable — lives in praxis; the pattern is fleet-generic |

## The plugins that came out of this

Running Omarchy this way produced widgets worth publishing. Each is a standalone repo with its own README, LICENSE, and manifest:

| Repo | What it does |
|---|---|
| [omarchy-appdock](https://github.com/gdeyoung/omarchy-appdock) | KDE-style task manager + minimize engine — dock widget, per-workspace icons, titlebar buttons |
| [omarchy-sysmon](https://github.com/gdeyoung/omarchy-sysmon) | Live RAM, CPU, network rate, and whole-disk stats — four groups, one widget |
| [omarchy-tailfin](https://github.com/gdeyoung/omarchy-tailfin) | Tailscale tabbed panel — health warnings, exit nodes, Mullvad, searchable peers |
| [omarchy-displayplus](https://github.com/gdeyoung/omarchy-displayplus) | Every display setting on one panel — brightness, text size, universal scale, full layout editor |
| [omarchy-powercore](https://github.com/gdeyoung/omarchy-powercore) | Battery and power strategy — per-source profiles, charge protection, clamshell, live watts |

## House rules

Same as praxis:

1. **Numbers or it didn't happen.** Every claim carries how it was verified.
2. **Failures are the content.** A lesson that didn't cost anything is usually wrong.
3. **Attribution upstream.** Forks and adapted patterns name their source.
4. **Nothing private ships.** No hostnames, IPs, user paths, or machine identifiers. The docs describe patterns; our private kit stays private.

## License

MIT — see [LICENSE](LICENSE).
