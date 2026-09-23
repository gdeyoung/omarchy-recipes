# Plugin hygiene — third-party plugins and your own, in one bar

Omarchy's plugin system makes accumulation frictionless: `omarchy plugin add <repo>` and the widget appears. Every failure mode below arrived later, on machines that were working fine.

## Pin what you validated

Third-party plugin trees change under you — authors push, you run `omarchy plugin update`, behavior shifts. A **pins table** in the recipe repo records, per plugin: id, source repo, exact commit, role, and retirement notes. Moving off a pin is a deliberate act: update, validate, re-pin.

Retirement notes matter as much as pins. "Disabled: no backend for the VPN we actually run" is a decision recorded where the next person — or agent — will find it, instead of in someone's memory of a Tuesday.

## Namespace your own widgets

Widgets you build ship under your own plugin id (reverse-DNS, e.g. `gdeyoung.sysmon`) with their own public repo, README, LICENSE, and manifest, installed by their own `install.sh`. Two payoffs: `omarchy plugin update` never re-extracts your tree (no upstream can clobber it), and the widget is publishable without exhuming it from a private kit.

## Fork vs. marked patch

Sooner or later an upstream plugin needs one line changed. Both options are real:

- **Fork it** when the change is structural. Our display-settings panel is a fork of crmne's hyprmoncfg, published standalone as [omarchy-displayplus](https://github.com/gdeyoung/omarchy-displayplus): our text-size slider and universal-scale control added, module/ipc re-id'd so fork and upstream can coexist, upstream's daemon kept installed as the backend.
- **Mark the patch in place** when it's cosmetic — but know the clock: `omarchy plugin update` re-extracts the tree and silently removes it. Keep such patches one-line, marked with a dated comment, documented in the pins table with the exact re-apply instruction. **A patch you can't afford to lose is a fork you haven't made yet.**

The bite mark: a one-line "open on the watchlist page, not the hub" patch to a stock-ticker plugin survived three weeks of daily use — until a routine plugin update quietly reverted it. No error, no notification; the panel just opened on the wrong page one morning.

## Disable, never uninstall stock widgets

When one of your widgets replaces a stock one (our calendar widget replaced the stock clock), **disable the stock widget — never uninstall it**. It keeps receiving updates, you keep a reference for diffing, and returning to stock is a toggle instead of an archaeology dig. The inventory records each: `omarchy.clock — disabled, replaced by <ours>, format inherited`.

## Native `.so` plugins are build-locked

Plugins like hyprbars (titlebar buttons) compile against the exact Hyprland build. After a Hyprland upgrade they fail to load — which presents as *config errors referencing a missing plugin*, not as "you need to rebuild." Keep the build recipe in the kit, rebuild after every Hyprland bump, and design for graceful degradation: our dock's minimize engine and our stats widget don't touch native plugin APIs, so they keep working through a broken-`.so` window.

## The bar is config, not vibes

`omarchy bar put/move <id> --section <l|c|r>` decides placement and order; the resulting `shell.json` is snapshotted into the recipe repo like any other config. Explicit placement beats whatever the default order was, and the snapshot survives a refresh.

## Publishing path (marketplace)

Every widget we've published followed the same route: standalone repo → validate locally (`omarchy plugin add` from the repo URL, then days of real use) → submit to the marketplace. One expectation to set: marketplace review has real security standards. We removed an in-plugin AUR install path — a plugin shell-script executing package installs as the user — to pass review, replacing it with a pointer to manual instructions. Design for that from the start: a plugin should install nothing outside its own tree unless the operator runs the command themselves.
