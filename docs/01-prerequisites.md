# 01 — Prerequisites

The NSP script checks its own environment before extracting anything
(`check_prereqs`, `check_native_python` in `scripts/build_ipa_from_nsp.py`): Python modules and tools,
the **iOS SDK for an iPhone build** (a separate download in Xcode's Components), the Metal compiler,
and the shader translator (it starts it once with `--source-sha`). A missing piece stops the build
there, not an hour later.

## Verified environment

| Requirement | Verified here | Notes |
|---|---|---|
| macOS + Apple silicon | macOS 27.0.1, arm64 | Xcode 27.2 |
| **Full Xcode** | `/Applications/Xcode.app`, `xcode-select -p` points at it | not Command Line Tools |
| **iOS SDK component** | `-sdk iphoneos27.2` | Xcode → Settings → Components → iOS SDK (a separate download since Xcode 26) |
| **Metal toolchain** | `xcodebuild -downloadComponent MetalToolchain` | `xcrun -sdk iphoneos metal --version` must print a version |
| Metal compiler fingerprint | `Apple metal version 32023.922 (metalfe-32023.922.4)` | this is exactly `dmsh_chain.lock`'s `metal_iphoneos`; a different one needs `--allow-other-metal` |
| **.NET SDK 10** | 10.0.403, runtime 10.0.14 | the shader translator is a .NET apphost |
| `DOTNET_ROOT` | set (Home Manager on this machine) | probe paths: `/usr/local/share/dotnet`, `/opt/homebrew/opt/dotnet/libexec`, `~/.dotnet` |
| **xcodegen** | 2.46.0 (`brew install xcodegen`) | generates the Xcode project |
| Python | 3.12+ (3.14.8 here) | `lz4`, `capstone` 5.x, `fonttools`, `zstandard` |
| `zstd` | on `PATH` | used while unpacking |
| Keys | `prod.keys` (and `title.keys`) | **used on the Mac only**, never shipped |
| Disk | ~20 GB free | extraction + shaders + IPA |

## 1. Xcode and its components

```bash
sudo xcode-select -s /Applications/Xcode.app
sudo xcodebuild -license accept
xcodebuild -downloadComponent MetalToolchain    # iOS *and* macOS Metal compilers
xcodebuild -downloadComponent iOSSDK            # if `-sdk iphoneos` is not listed
xcrun -sdk iphoneos metal --version             # Apple metal version 32023.922 …
xcodebuild -showsdks | grep iphoneos            # -sdk iphoneos27.2
```

The Metal compiler version matters: `dm/gfx/tools/dmsh_chain.lock` pins the compiler the project's
shaders were tested with. A different one is refused unless you pass `--allow-other-metal`, and
GitHub issue #242 records shader corruption with an untested compiler.

## 2. .NET 10

The translator (`dm/gfx/tools/dmsh_xlate_bin/dmsh_xlate`) is a .NET apphost and needs a **10.x
runtime** for arm64. Prove it works rather than trusting `dotnet` on `PATH`:

```bash
cd nxnative
./dm/gfx/tools/dmsh_xlate_bin/dmsh_xlate --source-sha
# must equal dmsh_chain.lock's xlate_source line
```

If you manage the machine with Nix/Home Manager, `pkgs.dotnet-sdk_10` alone is not enough — the Nix
store is not one of the paths the script probes. Export it:

```nix
home.packages = [ pkgs.dotnet-sdk_10 ];
home.sessionVariables.DOTNET_ROOT = "${pkgs.dotnet-sdk_10}/share/dotnet";
```

## 3. Python

```bash
python3 -m venv .venv
.venv/bin/pip install lz4 capstone fonttools zstandard
```

`zstandard` covers Python 3.14; on 3.13 and older the project asks for `backports.zstd`.
An Intel (Rosetta) Python is rejected on purpose: the universal translator would start as x86_64 and
find no x86_64 .NET.

## 4. xcodegen

```bash
brew install xcodegen && xcodegen --version
```

`build_ipa.sh` runs it to generate `Gen/` — the per-game Xcode project with the compile commands taken
from `dm/host/Makefile`.

## 5. Keys

```bash
mkdir -p ~/.config/nxnative
unzip -o ProdKeys.NET-*.zip prod.keys title.keys -d ~/.config/nxnative
```

Keep them **outside this repository** (the `.gitignore` also excludes `*.keys`, but outside is safer).
They are used by `nx_extract` to decrypt the NSPs on the Mac. Nothing is decrypted on the phone and no
key is embedded in the IPA.

## 6. The game files

The script takes a **folder of NSPs** for one game:

```
Mario Kart 8/
├── Mario Kart 8 Deluxe.nsp                                        base
├── Mario Kart 8 Deluxe […][v1507328][US][Up v4.0.1].nsp           update 4.0.1
└── Mario Kart 8 Deluxe [Wave 1-6 DLC Unlocker]….nsp               catalogue ticket (124 KB)
```

It packs the **newest version in the folder** (v1507328 = 4.0.1 here), which the MK8DX profile
registers as a tested build. A `.nup` container or an already-extracted dump folder also work (a dump
needs no keys, but then the app has no icon and the name comes from the profile).
