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

## Rebuilds

Extraction, rewrite, shader split and iOS shader libraries are reused when their inputs (game files,
tools, profile, Xcode, in-repo spirv-cross) are unchanged — a repeat run is roughly **20 seconds** plus
packing. The `logs/` folder of the previous run is kept.
