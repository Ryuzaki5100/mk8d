# 04 — On device

## First launch

1. The app opens on a **launcher screen** (not the game): the game starts loading in the background
   while you pick settings, and **Play** applies them.
2. Settings screen sections: **Game**, **Controls**, **Performance**. Rows MK8DX exposes:
   Frame rate (30/60), Dynamic resolution, Console mode (Docked/Handheld), Upscaling, Game language,
   Picture (Fit/Fill), Touch feedback.
   **Reset to tested defaults** puts everything back to the profile's tested values.
3. Press **Play**. The first boot runs the intro and lands on the title screen (~10–20 s on this class
   of device).

## Live overlays and logs

Everything on the phone is tuned through `Documents/env.txt`
(Files app → On My iPhone → **Mario Kart 8 Deluxe** → `env.txt`), one `KEY=VALUE` per line:

| Switch | Effect |
|---|---|
| `DM_SHOW_FPS=1` | frame-rate counter, top left |
| `DM_METAL_HUD=1` | Apple's Metal performance HUD (GPU time, memory) |
| `DM_FULLSCREEN`, `DM_OPMODE=1\|0` | display / docked-vs-handheld without the launcher |
| `DM_LANG=en-US` | game language |
| `DM_RUMBLE=0`, `DM_RUMBLE_MIN=0.45` | silence rumble / drop the constant low band |
| `DM_MOTION=0` | motion controls off |
| `DM_TOUCH_HAPTICS=0` | no touch feedback |
| `DM_NVN_GPUPROF=every:600` | per-pass GPU time in the log |

The game's log is `Documents/log.txt` in the same folder (also visible over USB/iloader's file browser).

## Memory

The MK8DX profile documents **~2.4 GB resident and ~1.26 GB of Metal allocations in a race** on the
Mac. On an iPhone the launcher shows `os_proc_available_memory()` — the headroom iOS is granting you.
On an 8 GB phone expect roughly 3.5 GB without the memory entitlement, ~5–6 GB with it. That is why
this game fits where Mario Kart World (18.8 GB) cannot.

If the app is killed at launch, the cause is almost always memory: check the last lines of
`Documents/log.txt`, and see [`03-sideload-iloader.md`](03-sideload-iloader.md) about the
`increased-memory-limit` entitlement.

## Known gaps for this game

From `dm/games/0100152000022000/README.md` and the project's development notes:

- **Mii heads draw nothing** — the host ships no MiiModel firmware data. Bodies, names and the Mii
  picker itself work.
- The 124 KB "DLC unlocker" is a catalogue ticket; the actual Wave 1–6 courses live in the 4.0.1
  update and are included because the update is packed.
- **amiibo** is not supported by the host at all.
- The Mac build renders some menu/dialogue text blank; on iPhone it is fine (the profile says so).
- 2P needs a second controller and the specific menu sequence in
  [`05-controllers.md`](05-controllers.md#two-players-in-mario-kart-8-deluxe).

## Storage and uninstall

The app is ~12 GB installed (RomFS inside). Deleting the app removes saves with it — copy what you want
out of `Documents/` first.

## Rebuilding after a change

Re-running the same command in [`02-build-ipa.md`](02-build-ipa.md) reuses the extraction, rewrite,
shader split and iOS shader libraries, so a repeat build is roughly **20 seconds** plus packing.
Change one of those inputs (a new update NSP, a different profile) and only the affected stages rerun.
