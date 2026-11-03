# mk8d — Mario Kart 8 Deluxe on iPhone (no JIT)

A reproducible build of **Mario Kart 8 Deluxe (Switch, 4.0.1)** as a self-contained iOS app, using
[upstream nxnative](https://github.com/seedds/nxnative) — the project that rewrites a Switch game's
ARM64 code into a native Apple app.

**What this is not:** an emulator, and not a JIT. The game's code is statically recompiled and linked
into the app; its shaders are translated ahead of time to Metal. Nothing is interpreted on the phone
and **no keys are used on the phone** — `prod.keys` is only read on the Mac while unpacking the NSP.

| | |
|---|---|
| Game | Mario Kart 8 Deluxe, title ID `0100152000022000` |
| Packed version | 4.0.1 (`v1507328`, the newest in the folder and a version the profile registers) |
| Output | `MarioKart8Deluxe.ipa`, **10.6 GB**, unsigned (signed at sideload time) |
| Engine | nxnative, upstream commit `ad7af9c` (vendored in [`nxnative/`](nxnative)) |
| Verified device | iPhone 17e — A19, 8 GB RAM, 256 GB, iOS 26+ |
| Runtime cost | profile: ~2.4 GB resident + ~1.3 GB Metal in a race |

## Repository layout

```
mk8d/
├── nxnative/       upstream source snapshot (433 MB, source + game profiles; no game content)
├── docs/           prerequisites, build, sideload, device testing, controllers
├── build/          build tree (git-ignored): extract/, patched_*/, shader/, logs/, ipa/
└── .gitignore      NSP / keys / IPA / build output are never committed
```

`build/` is where the script puts everything: the extracted game, rewritten modules, shader caches,
per-stage logs and the IPA. It is ~20 GB and **never** enters git.

## Rebuild from scratch

```bash
# 1. toolchain (see docs/01-prerequisites.md)
brew install xcodegen
python3 -m venv .venv && .venv/bin/pip install lz4 capstone fonttools zstandard

# 2. keys stay outside the repository
mkdir -p ~/.config/nxnative && unzip -o ProdKeys.NET-*.zip prod.keys title.keys -d ~/.config/nxnative

# 3. build the IPA
cd nxnative
../.venv/bin/python scripts/build_ipa_from_nsp.py \
  "/path/to/Mario Kart 8" \
  -k ~/.config/nxnative/prod.keys \
  --work-dir ../build
```

Then sideload `build/ipa/MarioKart8Deluxe.ipa` (or the copy the script drops beside the NSP folder)
with iloader / AltStore / Sideloadly — see [`docs/03-sideload-iloader.md`](docs/03-sideload-iloader.md).

## Controllers

The app advertises `GCSupportsControllerUserInteraction` and an `ExtendedGamepad` profile, so any
controller iOS recognises works: Xbox, PlayStation (DualSense/DualShock), Switch Pro, Backbone and
similar. Up to **8 players** (one per controller, local co-op), the touch gamepad disappears while a
physical controller is connected, and rumble/haptics are mapped per player. Full detail and the
2-player setup for MK8DX: [`docs/05-controllers.md`](docs/05-controllers.md).

## Documentation

| Page | Contents |
|---|---|
| [01 — Prerequisites](docs/01-prerequisites.md) | Xcode + iOS SDK component, xcodegen, .NET 10, Metal lock, Python, keys |
| [02 — Build the IPA](docs/02-build-ipa.md) | the exact command, every stage, timings, logs, what to verify |
| [03 — Sideload](docs/03-sideload-iloader.md) | iloader signing, free vs paid certificate, entitlements |
| [04 — On device](docs/04-testing-on-device.md) | first launch, launcher settings, memory, known gaps |
| [05 — Controllers](docs/05-controllers.md) | supported pads, players, rumble, motion, TV-out |

## Legal

No game content, no keys and no built app are in this repository. Building requires NSPs of a game
you own; the IPA produced contains that game and must not be shared. nxnative is MIT-licensed (see
[`nxnative/LICENSE`](nxnative/LICENSE)).
