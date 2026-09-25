# Stock tooling first: the config flag that replaced a plugin fork

> Measured on Omarchy 4.x / voxtype 1.0.1 / Hyprland 0.56, 2026-09. The general
> pattern: **before you fork a plugin, inventory what the stock layer already
> ships — the delta you want is often a config value, not code.**

## The story in three sentences

The wanted feature: Mynah-style voice dictation, but with transcription on a
home-server GPU instead of laptop CPU. The obvious path: fork the dictation
plugin, gut its engine, wire in a custom HTTP client — days of work and a
maintenance fork. The actual path: Omarchy already ships a dictation tool
(voxtype) whose `whisper.mode = "remote"` takes any OpenAI-compatible
transcription endpoint. One config edit pointed it at the LAN whisper server.
Zero new code; upstream updates keep flowing.

Result: ~0.5-0.7 s press-to-text against `large-v3` on a GPU, versus 1.5-4.2 s
for small/base models on a 22-core laptop CPU (the Mynah README's own honest
figures).

## The checklist, generalized

Run this **before** writing a fork, in order:

1. **Grep the stock layer for the capability.** `omarchy commands --all`,
   `/usr/share/omarchy/shell/plugins/` (the bar ships more than the plugin
   marketplace shows), and `pacman -Ql omarchy | grep bin/`. The dictation
   bar indicator existed; nobody advertises it.
2. **Read the stock tool's full config schema.** `voxtype config schema`
   revealed remote mode, endpoint, model, api-key, VAD, on-demand model
   loading — five features that would each have been a fork feature.
3. **Check for a "remote/openai-compatible" escape hatch specifically.**
   Local-first tools increasingly ship one (ollama, LiteLLM proxies,
   OpenAI-compatible endpoints everywhere). If it exists, your custom client
   is a URL and a key, not a daemon.
4. **Only then** decide fork vs configure. Fork when you'd change behavior;
   configure when you'd change a destination.

## Traps that bit (each cost a debug cycle)

- **TOML section placement.** Appending `remote_endpoint = ...` to the end of
  a config file silently puts it in the *last* table (`[status]`), not the one
  you meant (`[whisper]`). The tool then errors "remote_endpoint is required"
  even though it's right there. Use the tool's own `config set section.key`
  command, which writes into the right table.
- **Base-URL shape.** "OpenAI-compatible" is not one shape. This client
  appends the full `/v1/audio/transcriptions` path itself, so the configured
  base must NOT end in `/v1`. Symptom of getting it wrong: a clean 404
  `{"detail":"Not Found"}`. Probe both shapes once, then pin the working one.
- **CLI flag position.** Rust clap tools take global flags *before* the
  subcommand (`tool -c FILE transcribe X`). After it = parse error, and the
  error names the flag, not the position.
- **Secrets don't belong in tool config.** Prefer an env var the tool reads
  (`VOXTYPE_WHISPER_API_KEY`), delivered to a systemd user unit by a drop-in
  `EnvironmentFile=` pointing at a 0600 file outside every repo. Config files
  get copied into kit repos; env files don't.
- **systemd drop-in dirs need the execute bit.** A `drw-------`
  `unit.service.d/` directory silently ignores every drop-in in it. If a
  drop-in "isn't applying," `chmod u+rX` the directory before doubting the
  file.

## The E2E test pattern (no human in the loop)

To prove a mic → transcribe → type pipeline without talking yourself:

1. Sacrificial target: spawn a terminal window running `cat > /tmp/target.txt`
   and focus it (anything with a real text field can have focus stolen mid-test).
2. Synthetic voice: `espeak-ng -v en-us -w /tmp/say.wav "sentence"` played
   through the speakers — the microphone hears the room, exactly like speech.
3. Drive the real path: trigger the tool's control CLI or synthesize the
   hotkey with `ydotool key` (Hyprland 0.56 has no `sendShortcut` dispatcher;
   ydotool evdev codes work), watch the tool's state file
   (`/run/user/1000/<tool>/state`), then read `/tmp/target.txt`.
4. Expect ambient capture: an open mic will transcribe *someone* — a silent
   room test that returns "Thank you." means the pipeline works AND your VAD
  should reject silence-only recordings.

That last point is the giveaway that the whole chain is live: a hallucinated
"Thank you." from room noise is end-to-end proof.
