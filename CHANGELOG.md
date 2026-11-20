> This is an upstream changelog from [DarkFlippers/unleashed-firmware](https://github.com/DarkFlippers/unleashed-firmware), not a release announcement from this fork. For official Unleashed releases, see [upstream Releases](https://github.com/DarkFlippers/unleashed-firmware/releases).

## Main changes
- Current API: 88.27
- Apps: Build tag (**16nov2026**) - **Check out more Apps updates and fixes by following** [this link](https://github.com/xMasterX/all-the-plugins/commits/dev)
- SubGHz: **New protocol - BelKorn, 118 bits, rolling code with no encryption, 36 bit serial, with custom buttons** - (Korn, BelGate, Furniteh and other rebrands; the frame carries no manufacturer field, so they all read as `BelKorn`)
- SubGHz: **Add Manually: `BelKorn`**
- SubGHz: **The read list names the system a multi-system protocol decoded, instead of the protocol - CAME, Nice FloR-S, Faac SLH and KeeLoq**
- SubGHz: **DEA Mio sends back the discriminator the original remote sent, instead of always setting the overflow bits**
- SubGHz: **DEA Mio asks for the discriminator in Advanced Add Manually and shows it as `Ds:`**
- SubGHz: **Advanced Add Manually asks for TE where the protocol carries one - Princeton, ZKTeco and FAAC TM**
- SubGHz: **Advanced Add Manually asks for the guard time after TE, where the protocol carries one - Princeton and ZKTeco**
- SubGHz: **Add Manually has one entry per system, with the frequency and modulation picked in the next screen**
- SubGHz: **The band picker lists the system's own frequencies first and every other one below, so an odd installation can still be crafted**
## Other changes
- OFW: fix: resolve PVS-Studio warnings reported on dev
- NFC: **Fix third-party NFC apps reading the wrong NTAG/Ultralight pages (Weebo "Wrong password" on every amiibo) - the Ultralight AES fields no longer sit in the middle of `MfUltralightData`**
- SubGHz: **Fix the read list beep going missing with `Delete old signals` on and memory full**
- SubGHz: **A signal turned away by a full read list now buzzes with `Delete old signals` on too**
- Remote Analyzer: **BelGate dropped from the app, it is a firmware Sub-GHz protocol now**
- Remote Analyzer: **`How to use` lists the protocols the app decodes on its own**
- SubGHz: **Advanced Add Manually key field is now as long as the protocol's bits, instead of padding it with `00` to 8 bytes**
- SubGHz: **Fix Add Manually failing on FAAC TM - only Princeton carries a `Guard_time`, so writing one into every TE protocol broke it**
- SubGHz: **`subghz rx`, `tx` and `tx_from_file` over the CLI work on the internal radio again, instead of stopping at `Device 0 failed to start`** (by @GodKimba | PR #1216 | Fixes #1215)
<br><br>

----

[-> How to install firmware](https://github.com/DarkFlippers/unleashed-firmware/blob/dev/documentation/HowToInstall.md)

[-> Unleashed FW Web Installer](https://web.unleashedflip.com)

## **Recommended update option - Web Updater**

### What `e`, ` `, `c` means? What I need to download if I don't want to use Web updater?

What build I should download and what this name means - `flipper-z-f7-update-(version)(e / c).tgz` ? <br>
`flipper-z` = for Flipper Zero device<br>
`f7` = Hardware version - same for all flipper zero devices<br>
`update` = Update package, contains updater, all assets (plugins, IR libs, etc.), and firmware itself<br>
`(version)` = Firmware version<br>

| Designation | [Base Apps](https://github.com/xMasterX/all-the-plugins#default-pack) | [Extra Apps](https://github.com/xMasterX/all-the-plugins#extra-pack) |
|-----|:---:|:---:|
| ` ` | ✅ |  |
| `c` |  |  |
| `e` | ✅ | ✅ |

**To enable RGB Backlight support go into LCD & Notifications settings**

⚠️RGB backlight [hardware mod](https://github.com/quen0n/flipperzero-firmware-rgb#readme), works only on modded flippers! do not enable on non modded device!

Firmware Self-update package (update from microSD) - `flipper-z-f7-update-(version).tgz` for mobile app / qFlipper / web<br>
Archive of `scripts` folder (contains scripts for FW/plugins development) - `flipper-z-any-scripts-(version).tgz`<br>
SDK files for plugins development and uFBT - `flipper-z-f7-sdk-(version).zip`
