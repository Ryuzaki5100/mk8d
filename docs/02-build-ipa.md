# 02 — Building the IPA

One command drives the whole chain: identify the game → extract → rewrite → split shaders → build iOS
shader libraries → generate the Xcode project → build → pack the IPA.

## The command

```bash
cd ~/projects/mk8d/nxnative
../.venv/bin/python scripts/build_ipa_from_nsp.py \
  "/Users/ryuzaki/emulation/switch-emulation/games/Mario Kart 8" \
  -k ~/.config/nxnative/prod.keys \
  --work-dir /Users/ryuzaki/projects/mk8d/build \
  --progress lines
```

Useful options:

| Option | Purpose |
|---|---|
| `-p v1507328` | pin the version (default: the newest NSP in the folder) |
| `--platform ios` | the default; `mac` / `mac-app` build for the Mac instead |
| `--romfs separate` | keep the RomFS out of the IPA (small IPA, push files separately) |
| `--allow-other-metal` | build with a Metal compiler that differs from `dmsh_chain.lock` (untested) |
| `--no-motion`, `--show-fps`, `--env-file` | bake switches into the app |
| `--prepare-xcode` | stop after generating the Xcode project (sign/run from Xcode) |
| `-o out.ipa`, `--app-name`, `--bundle-id` | override the output and identity |

Everything intermediate goes under `--work-dir`:

```
build/
├── extract/            the game's files (ExeFS, RomFS, DLC RomFS)
├── patched_v1507328/   the rewritten code modules + rewriter tables
├── shader/             shader package, caches, rejects.tsv
├── profile/            the per-version profile derived from the repo's
├── aoc/                DLC RomFS images
├── logs/               each stage's full output
└── ipa/                the finished IPA
```

## Stages

The script prints one line per stage with progress and ETA; the tools' own output goes to `build/logs/`.

| Stage | What happens |
|---|---|
| 1 | identify the game from the NSPs (title ID → `dm/games/<id>/`) and derive the version profile |
| 2 | extract the base game (`nx_extract` with your `prod.keys`) |
| 3 | extract the DLC, if present |
| 4 | rewrite `main`, `subsdk0`, `sdk` so their system calls land in the host |
| 5 | split the game's shaders out of Nintendo's containers into the shader package |
| 6 | build the **iOS** shader libraries (`IOSOUT`), i.e. `iphoneos` metallibs |
| 7 | generate the Xcode project (`xcodegen`) and build the app |
| 8 | pack the unsigned IPA (`pack_ipa.py`) |

### Measured on this machine (MK8DX 4.0.1, Xcode 27.2, 12 workers)

| Stage | Time | Output |
|---|---|---|
| 1 identify + `nx_extract` build | 0:01 | — |
| 2 extract v1507328 | **0:12** | 6,602 files, 11.3 GB |
| 3 extract DLC | 0:00 | 1 DLC (the 124 KB catalogue ticket) |
| 4 rewrite | **0:04** | `main`, `subsdk0`, `sdk` |
| 5 shader split | **1:12** | **29,651 shaders** |
| 6 iOS shader libraries | **25:47** | 29,651 of 29,651, 0 from cache |
| 7 Xcode app build | **0:16** | 259 files compiled |
| 8 IPA pack | **4:00** | `MarioKart8Deluxe.ipa`, 10.6 GB |
| **Total** | **31:36** | 104,938 archive entries |

The iOS shader step dominates: it is ~30k `iphoneos` metallib compiles. It is cached — a repeat build
reuses them.

## What to verify afterwards

```bash
IPA=build/ipa/MarioKart8Deluxe.ipa
ls -lh "$IPA"

# identity and iOS target
unzip -p "$IPA" 'Payload/*.app/Info.plist' > /tmp/info.plist
plutil -extract CFBundleIdentifier raw /tmp/info.plist          # org.nxnative.0100152000022000
plutil -extract CFBundleDisplayName raw /tmp/info.plist         # Mario Kart 8 Deluxe
plutil -extract MinimumOSVersion raw /tmp/info.plist            # 26.0
plutil -extract GCSupportsControllerUserInteraction raw /tmp/info.plist   # 1

# what is inside
unzip -l "$IPA" | awk '{print $4}' | cut -d/ -f1-3 | sort -u | head
# Payload/MarioKart8Deluxe.app/{msl,romfs,aoc,dm}
```

The IPA is **unsigned** (ad-hoc signed executable carrying the entitlements), with bundle ID
`org.nxnative.0100152000022000`; the sideloader replaces the signature.

### Verified output of this build

| Property | Value |
|---|---|
| File | `~/projects/mk8d/MarioKart8Deluxe.ipa`, **10.6 GB** (11 GB on disk) |
| Bundle (inside `Payload/`) | `DmIOS.app` (display name comes from Info.plist) |
| `CFBundleIdentifier` | `org.nxnative.0100152000022000` |
| `CFBundleDisplayName` | **Mario Kart 8 Deluxe** |
| `CFBundleShortVersionString` | `4.0.1` |
| `MinimumOSVersion` | `26.0` |
| `GCSupportsControllerUserInteraction` | `true` |
| `GCSupportedGameControllers` | `[{"ProfileName":"ExtendedGamepad"}]` |
| Executable arch | `Mach-O 64-bit executable arm64` (44 MB) |
| Entitlements (ad-hoc) | `com.apple.developer.kernel.increased-memory-limit = true` |
| Contents | `Payload/DmIOS.app/{DmIOS, romfs.img, msl/, aoc/, dm/}` |

### One build-time warning, and why it is acceptable

```
[RED] this IPA uses v1507328 (4.0.1), which game profile 0100152000022000 was not brought up on
      (v1441792 4.0.0, v1507328 4.0.1): its settings, with this version's own shaders;
      every module is a registered build
```

The game profile records its Mac bring-up from **4.0.0**. Because the folder has only the 4.0.1 update
NSP (the profile's own note says a build from the *registered* profile needs **both** update NSPs), the
script derived a profile for 4.0.1. That exact shape — 4.0.1, derived profile, Booster Course DLC — is
the configuration the profile documents as tested (a full three-lap Grand Prix on an iPhone 15 Pro Max
on iOS 27.0.1). Every module in this build is a registered build ID, so no rewrite fix is missing; only
the combined shader tables are derived, from this version's own shaders.

## Rebuilds

Extraction, rewrite, shader split and iOS shader libraries are reused when their inputs (game files,
tools, profile, Xcode, in-repo spirv-cross) are unchanged — a repeat run is roughly **20 seconds** plus
packing. The `logs/` folder of the previous run is kept.
