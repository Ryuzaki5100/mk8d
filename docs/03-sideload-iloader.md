# 03 — Sideloading with iloader

The script produces an **unsigned** IPA with a placeholder team ID (`0000000000`). iloader (or
AltStore/Sideloadly) signs it with your Apple certificate and installs it.

## 1. Install iloader

```bash
open ~/Downloads/iloader-darwin-universal.dmg     # drag to /Applications
```

Connect the iPhone by USB-C, unlock it, and tap **Trust** on the phone when asked.

## 2. Choose the signing identity

| | Free Apple ID | Paid Apple Developer |
|---|---|---|
| Certificate lifetime | **7 days**, then re-sign | 1 year |
| App limit | 3 active apps / 10 per week | 100 devices, many apps |
| `increased-memory-limit` entitlement | **not grantable** (restricted) | can be requested |
| Bundle ID | must be unique-ish, iloader can rewrite | full control |

MK8DX needs ~2.4 GB resident + ~1.3 GB of Metal allocations. On an 8 GB phone iOS grants roughly
3.5 GB to a normal app and ~5–6 GB with the memory entitlement, so **it should run on a free
certificate**; if iOS kills it at launch, the entitlement is the thing to check first.

## 3. Sign and install

In iloader: **Install** → pick `MarioKart8Deluxe.ipa` → choose the certificate → install. Two things
iloader may ask about:

- **Bundle ID**: the IPA carries `org.nxnative.0100152000022000`. With a free account, change it to
  something unique (e.g. `com.yourname.mk8d`) if iloader reports a conflict. nxnative does not care
  about the bundle ID — only the files inside the app matter.
- **Entitlements**: leave the memory entitlement unchecked on a free certificate; the app runs
  without it.

The IPA is **10.6 GB**, so give the transfer time. Keep ~15 GB free on the phone: iOS extracts the
archive on install.

## 4. Trust the developer

Settings → General → VPN & Device Management → *your Apple ID* → **Trust**. Then launch the app.

## 5. Re-signing

A free certificate expires after 7 days and the app stops launching. Re-sign the same IPA with
iloader; **saves are kept** in the app's container, so a re-sign does not lose progress unless iloader
removes the app first. For a stable install, use a paid account.

## 6. If the install fails

| Symptom | Cause | Fix |
|---|---|---|
| `Unable to install … integrity could not be verified` | certificate/entitlement mismatch, or the app was signed with an entitlement the profile cannot carry | re-sign without `increased-memory-limit` |
| Install stalls at "Installing" for very long | 12 GB IPA extraction on device | wait; USB-C transfer is 480 Mb/s on the 17e |
| App launches and exits immediately | jetsam kill at startup (memory) | check the on-device log (`Documents/log.txt` after a successful launch, or Console.app), try the memory entitlement with a paid account |
| "Untrusted Developer" | profile not trusted | Settings → General → VPN & Device Management → Trust |
| Controller not detected | paired after launch | connect the pad first, then launch the game |

## Uninstalling cleanly

Delete the app from the Home Screen. Saves live inside the app container and go with it; copy anything
you want to keep out of **Files → On My iPhone → Mario Kart 8 Deluxe** first.
