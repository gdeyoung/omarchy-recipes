# The Copilot key as push-to-talk: reclaiming a firmware chord

> Measured on Omarchy 4.5 / Hyprland 0.56 (Lua config) / voxtype 1.1.1, 2026-10.
> The general pattern: **a "dedicated" laptop key is often a firmware chord,
> and the desktop layer may already own that chord — find the real events and
> the existing bind before adding yours.**

## The story in three sentences

The goal: make the laptop's Copilot key do the same push-to-talk dictation as
the stock F9 hold-bind. The trap: the Copilot key is not a keycode — the EC
firmware emits **LeftWin↓, LeftShift↓, F23↓** (and the mirrored teardown), so
binding plain F23 does nothing. The second trap: Omarchy *already binds that
chord* — `utilities.lua` maps `SUPER + SHIFT + code:201` (= F23, in keycode
notation) to the root menu — so adding a bind on `"SUPER + SHIFT + F23"`
(keysym notation) does not replace it; **both fire** and every press opens the
menu *and* starts recording.

Result after the fix: Copilot key hold → dictation only (state file
`idle → recording → idle`, remote STT round-trip 0.28 s); Super+Space still
opens the root menu; stock F9 / Right-Alt hold / Super+Ctrl+X untouched.

## Step 1 — capture what the key really emits, without root

`evtest` needs root or the `input` group; `wev` may not be installed; with
Hyprland's Lua config `hyprctl keyword` is refused ("non-legacy parsers. Use
eval"). Runtime probe binds via `hyprctl eval` need none of that:

```bash
export HYPRLAND_INSTANCE_SIGNATURE=$(ls -t $XDG_RUNTIME_DIR/hypr/ | head -1)
mkdir -p /tmp/probe && rm -f /tmp/probe/*
hyprctl eval 'for _,k in ipairs({"F23","SUPER + F23","SHIFT + F23",
  "SUPER + SHIFT + F23","F24","SUPER + SHIFT + F24","SUPER + SPACE",
  "SUPER + SHIFT + SPACE","SPACE"}) do
  o.bind(k, nil, "touch /tmp/probe/" .. string.gsub(k, "%W", "-"),
    { non_consuming = true }) end'
```

**Validate the harness before trusting it** — synthesize each candidate with
ydotool and require the matching marker (`ls /tmp/probe`). Evdev code traps:
`SPACE` is **57** (32 is `D`), F23 is 193, LeftMeta 125, LeftShift 42:

```bash
ydotool key 125:1 57:1 57:0 125:0   # super+space → SUPER---SPACE marker
```

Then have the human press the physical key once. One marker appearing
(`SUPER---SHIFT---F23`, no `SPACE` variants) is the ground truth: the chord
is the whole key. A config reload wipes the eval-time probes — cleanup for
free.

## Step 2 — unbind the stock owner of the chord, then bind yours

In `~/.config/hypr/bindings.lua` (the user override file, loaded after the
stock binds):

```lua
-- Copilot key = firmware chord Super+Shift+F23. Stock Omarchy binds the same
-- chord (spelled "SUPER + SHIFT + code:201") to the root menu — unbind first
-- or it opens on every press.
hl.unbind("SUPER + SHIFT + code:201")
o.bind_hold("SUPER + SHIFT + F23", "Start dictation (push-to-talk, Copilot key)",
  "omarchy-dictation start", "Stop dictation (push-to-talk, Copilot key)",
  "omarchy-dictation stop")
```

`o.bind_hold` is Omarchy's helper for press/release pairs: the release half is
transparent, modifier-ignoring and non-consuming, which is exactly what a
firmware chord needs — after F23↑, Shift↑ and Win↑ still teardown, and the
release still fires.

## Step 3 — verify absence of side effects, not just presence of yours

The first iteration "passed" synthetic verification while being broken: the
synthetic chord flipped the dictation state, so it looked done — but the stock
menu bind was firing too. Check what should *not* happen:

```bash
omarchy menu keybindings --print | grep -E 'F23|code:201'   # only your bind
ydotool key 125:1 42:1 193:1                                 # hold…
cat $XDG_RUNTIME_DIR/voxtype/state                           # recording
ydotool key 193:0 42:0 125:0                                 # release → idle
hyprctl layers -j | grep -c omarchy-menu                     # 0 — no menu
ydotool key 125:1 57:1 57:0 125:0                            # Super+Space → 1
```

Then the physical key, pressed by the human, is the final gate. On the fixed
machine the confirmation sentence itself arrived dictated through the key.

## Pitfalls

- **Two notations, one key**: stock spells the bind `code:201` (keycode),
  you'd naturally write `F23` (keysym). They alias — and grepping the stock
  tree for `F23` finds nothing. When a bind "isn't there," grep for the
  keycode form too (`code:<xkb keycode>`, F23 = 201, F24 = 202).
- **Chords arrive in event order** Win↓ Shift↓ F23↓ — mods are already held
  when the real key lands, so chord binds match; nothing needs reordering.
- **A keysym bind does not shadow a keycode bind** on the same chord —
  duplicate binds both dispatch; unbind the exact stock spelling.
- **Reload is your probe cleanup**: `omarchy restart hyprctl` re-reads the
  Lua config and drops all `hyprctl eval` runtime binds.
