# What Omarchy actually overwrites — measured, not assumed

Before deciding where a custom change is "safe," know which mechanism rewrites what. This model was verified live on Omarchy 4.0.4 (pacman file lists, libalpm hooks in `/usr/share/libalpm/hooks/`, the `omarchy` CLI dispatch source). Re-verify specifics after a major Omarchy bump — **the model is the durable part.**

## The four mechanisms

| Mechanism | Rewrites | Survives |
|---|---|---|
| `omarchy update` (pacman upgrade) | `/usr`, `/etc` only | everything under `~/.config`; enabled user services for installed packages |
| `omarchy refresh` (reset config to defaults) | reseeds `~/.config/omarchy/`, `~/.config/hypr/` | `/usr/local/bin`, `~/.local/bin`, installed packages, `~/.config` outside those two trees |
| `omarchy plugin update` | that plugin's tree under `~/.config/omarchy/plugins/<id>/` | plugin dirs in your own namespace (never re-extracted from upstream) |
| kernel / Hyprland upgrade | module ABI, not config | configs intact, but Hyprland-build-locked `.so` plugins must be rebuilt; sched_ext schedulers may fall back to the stock fair class until their package catches up (degraded, not hung) |

The key fact most folk models get wrong: **pacman tracks zero files under `~/.config`.** Packages ship skeletons via `/etc/skel/.config/`, copied only at first-user creation — so a normal update *cannot* clobber user config. Custom hooks in `~/.config/omarchy/hooks/post-update.d/` *run during* update; they are not modified by it.

## Safe placement, derived from the table

- **Scripts:** `/usr/local/bin` or `~/.local/bin` — no update mechanism touches either.
- **Your plugins:** ship under your own plugin-id namespace; `omarchy plugin update` never re-extracts them.
- **Plain pacman packages:** upgrade-stable. Only kernel-coupled software has version-churn exposure (documented above, self-healing).
- **Theme/wallpaper hooks and agent glue:** `~/.config/omarchy/hooks/<event>.d/` dies only to `omarchy refresh` — keep the files in the recipe repo so `install.sh` re-seeds them, and note that exposure in the inventory.

## Probes (classification, not guesswork)

```bash
pacman -Q <pkg>                              # installed? version
pacman -Ql <pkg> | grep -E 'home|\.config'   # does this package own user config? (no output = no)
ls /usr/share/libalpm/hooks/ | grep -iE 'snap|omarchy'   # what wraps pacman transactions
systemctl is-enabled <unit>                  # survives updates iff the unit file does
```

One Omarchy-specific note: its snapshot wrapper around pacman transactions (`limine-snapper-sync` — locks snapper, exposes bootable snapshots in the bootloader menu) already does the job. Adding `snap-pac` beside it duplicates the work for no benefit.

## The interaction with the recipe repo

Safe-from-overwrite does not exempt a change from the inventory. Every item placed per the table above still gets its [recipe-repo](recipe-repo.md) row — the overwrite model answers *"will it survive?"*, the inventory answers *"does anyone know it's here?"*
