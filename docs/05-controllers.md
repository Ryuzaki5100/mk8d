# 05 — Controllers on iPhone

Controller support is built into the app and enabled, not something to bolt on afterwards.

## What the app declares

`dm/ios/project.yml` sets, in the generated `Info.plist`:

```yaml
GCSupportsControllerUserInteraction: true
GCSupportedGameControllers:
  - ProfileName: ExtendedGamepad
CADisableMinimumFrameDurationOnPhone: true     # allow >60 Hz frames
LSSupportsGameMode: true                       # iOS Game Mode
```

`ExtendedGamepad` is the profile that covers Xbox, PlayStation (DualSense/DualShock), Switch Pro,
Backbone/Razer Kishi-style pads and MFi controllers. Because the app supports controller user
interaction, iOS also routes the system **controller applet** (Home button hold, player LEDs) normally.

Verify after building:

```bash
IPA=build/ipa/MarioKart8Deluxe.ipa
unzip -p "$IPA" 'Payload/*.app/Info.plist' > /tmp/mk8d-info.plist
plutil -extract GCSupportsControllerUserInteraction raw /tmp/mk8d-info.plist   # true
plutil -extract GCSupportedGameControllers json -o - /tmp/mk8d-info.plist      # ExtendedGamepad
```

## Pairing and players

1. Pair the controller **before** launching the game: iOS Settings → Bluetooth, or just connect a
   wired/Lightning-USB-C pad (Backbone, Kishi, DualSense over USB-C).
2. The first controller connected is **player 1**; the next is player 2, up to **8 players**.
3. A controller that is removed frees its player slot and gets it back when it reconnects. The app log
   names each one:

   ```
   controller 'Xbox Wireless Controller' connected: player 2 (npad 1)
   ```

4. Player 1 is the *Handheld* npad in handheld mode and *No1* docked; player n is *No n*. A
   single-player game merges every controller into player 1; MK8DX supports up to 8.
5. **The on-screen touch gamepad hides itself while a physical controller is connected** — the whole
   screen becomes the game.

## Two players in Mario Kart 8 Deluxe

MK8DX asks through the Switch's controller applet, so the order matters (from the MK8DX profile's own
notes, exercised on the Mac and on iPhone):

> from a save past the Mii pick: **L + R at the title, Down, A (2P), A (Grand Prix), A (50cc)**

Leave 1.5–4 s between presses — the menu fades drop inputs that come too fast. Two players need a
second controller; a single controller can answer only as player 1.

## Rumble and haptics

| Behaviour | Where |
|---|---|
| Game rumble on controllers | on by default (`DM_RUMBLE=0` disables) |
| Strength | launcher setting / `DM_RUMBLE_GAIN=0..3` (default 1) |
| Quiet-rumble filter | `DM_RUMBLE_MIN=0..0.9` — silences the constant low levels some pads turn into noise |
| Touch feedback (phone itself) | launcher setting **Touch feedback**, `DM_TOUCH_HAPTICS`, iOS only |
| Log | `DM_RUMBLE_LOG=1` in `Documents/env.txt` |

MK8DX holds a low rumble band through every race; on pads with rotating motors that reads as constant
buzz. `DM_RUMBLE_MIN=0.45` keeps crashes and drift but drops the drone. Put overrides in the app's
`Documents/env.txt` (Files app → On My iPhone → Mario Kart 8 Deluxe).

## Motion

MK8DX does not need motion, but the phone's gyroscope drives player 1's six-axis sensor when no
controller supplies fresh motion, and shaking the phone becomes a controller shake
(`DM_SHAKE_G`, default 1.5 G). Turn it off with `DM_MOTION=0` (or build with `--no-motion`); the
launcher exposes it as a setting on games that use it.

## USB-C / external display

Your 17e's USB-C can drive a TV or display: the app has an `ExternalDisplayNonInteractive` scene, so the
game goes to the TV and the phone stays the controller/pad. AirPlay works the same way. Combine that
with a controller for a docked-style setup.

## Known limits

- **8 players max**, one per controller; MK8DX's own limit applies (it supports more, but iOS gives you
  one slot per connected pad).
- No amiibo support in nxnative.
- A pad whose gyro reports zeros uses the passive HID fallback on the Mac only; on iOS the phone's own
  motion covers player 1.
- Haptics need a controller with a haptics engine; a plain rumble-only pad gets on/off-style effects.
