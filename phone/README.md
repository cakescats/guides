# CakesCats Phone: a Google Pixel with GrapheneOS, do it yourself

[Русская версия](README.ru.md)

With this guide you can turn an ordinary Google Pixel into the same phone we sell ready-made — for free — and switch on every feature we advertise:

| Feature | Section |
|---|---|
| GrapheneOS without Google services | [3. Install](#3-install-grapheneos) |
| Hardware security, verified boot | [4. Verify](#4-verify-everything-is-genuine) |
| Emergency wipe with a duress PIN | [6. Duress PIN](#6-emergency-wipe-duress-pin) |
| Isolated profiles: work, personal, crypto | [7. Profiles](#7-isolated-profiles) |
| Private communication and trusted apps | [8. Apps](#8-apps) |
| Per-app network control | [9. Network and VPN](#9-network-and-vpn) |

Time: about an hour. You need a computer with Chrome, Edge or Brave and a USB-C cable.

> Want a ready, configured phone with support? That's [phone.cakescats.com](https://phone.cakescats.com/). Buying one supports our non-profit project.

---

## 1. Which phone

- **Google Pixel 8 or newer** (Pixel 8/8a/9/9a/10, Pro, Fold) — the longest security-update support. Current list: [grapheneos.org/faq#supported-devices](https://grapheneos.org/faq#supported-devices).
- The phone must be **carrier-unlocked**. Carrier-locked models (e.g. some US ones) don't allow unlocking the bootloader.
- Back up what you need: installation **erases all data**.

## 2. Preparation

On the phone (stock Android):

1. Connect to the internet and install updates.
2. **Settings → About phone** → tap **Build number** 7 times to enable developer options.
3. **Settings → System → Developer options** → turn on **OEM unlocking**.
4. Power the phone off.

On the computer:

- **Windows:** install the [Google USB Driver](https://developer.android.com/studio/run/win-usb).
- **Linux:** install `android-sdk-platform-tools-common` (udev rules) and stop `fwupd` during installation.
- **macOS:** nothing needed.

## 3. Install GrapheneOS

Use the official web installer: **[grapheneos.org/install/web](https://grapheneos.org/install/web)**. Never download GrapheneOS from anywhere else.

1. Hold **Power + Volume down** to boot into the bootloader (fastboot). Connect the cable.
2. On the installer page click **Unlock bootloader**. On the phone, select *Unlock the bootloader* with the volume keys and confirm with Power.
3. **Download release** — wait for it.
4. **Flash release** — don't touch the phone or cable until it finishes.
5. **Lock bootloader** — confirm on the phone the same way. **This is required:** verified boot protection doesn't work with an unlocked bootloader.
6. Press Power on *Start* — GrapheneOS boots.

## 4. Verify everything is genuine

- On every boot the phone shows a yellow notice *“Your device is loading a different operating system”* and a **key fingerprint**. Compare it with the list on the [installer page](https://grapheneos.org/install/web#verifying-installation). If it matches, the OS is genuine.
- After setup, turn **OEM unlocking** off again: **Settings → System → Developer options** (enable them the same way, 7 taps on the build number).
- Optional: the built-in **Auditor** app lets you regularly verify the phone from a second device using hardware attestation — see [attestation.app/tutorial](https://attestation.app/tutorial).

## 5. First security settings

Go through setup **without signing into any accounts**. Then:

| What | Where | Setting |
|---|---|---|
| Screen lock | Settings → Security & privacy → Device unlock | **6+ digit PIN** or a password; don't use a pattern |
| Scrambled PIN | Same place → screen lock settings | **Scramble PIN input layout** — on (smudges don't reveal the PIN) |
| Auto reboot | Settings → Security & privacy → Exploit protection → **Auto reboot** | Default 18 h; 8–12 h is a good choice. After a reboot data is fully encrypted at rest again |
| USB-C port | Same place → **USB-C port** | **Charging-only when locked**; stricter: **Charging-only** |
| Wi-Fi and Bluetooth | Settings → Network → Internet / Bluetooth | Turn on automatic switch-off when nothing is connected |
| Updates | Settings → System → System update | Keep automatic; reboot when prompted |

## 6. Emergency wipe (duress PIN)

A separate PIN that **instantly and irreversibly wipes the phone**, including all profiles and eSIMs.

1. **Settings → Security & privacy → Device unlock → Duress Password**.
2. Set a **duress PIN** (for the PIN screen) and a **duress password** (if you use a password).
3. Make it different from your main PIN and easy to remember.

It works anywhere the OS asks for your credentials — the lock screen and any confirmation prompt. The wipe doesn't need a reboot and cannot be interrupted.

> ⚠️ **Don't test the duress PIN on a phone with data** — it really erases everything. Keep backups (section 10).

## 7. Isolated profiles

Each profile is a separate “phone inside the phone” with its own apps, data and encryption keys.

1. **Settings → System → Multiple users** → on → **Add user**.
2. Create, for example:
   - **Work** — work messengers and mail;
   - **Crypto** — only the wallet (see below);
   - the main (Owner) profile — personal.
3. Give each profile its own PIN.
4. Switch via quick settings (user icon). When you're done with a profile, tap **End session** — its keys are evicted from memory and its data stays fully encrypted until you enter its PIN again.
5. To get notifications from other profiles: **Settings → System → Multiple users** → **Send notifications to current user**.

**The Crypto profile:**
- Install only the wallet from its official source (developer site or GitHub via Obtainium), or your hardware wallet's companion app.
- Remove the **Network** permission from every other app in this profile.
- **Never store your seed phrase on the phone** — paper or metal only, kept separately.

## 8. Apps

GrapheneOS has its own **App Store** with system apps: the Vanadium browser, Camera, PDF Viewer, Auditor. You can also install **Accrescent** from it — an app store with signature verification.

Recommended apps (install from official sources: Accrescent, or [Obtainium](https://github.com/ImranR98/Obtainium), which downloads directly from the developer's site or GitHub):

| Task | App |
|---|---|
| Messages and calls | Signal or Molly |
| Passwords | KeePassDX or Bitwarden |
| 2FA codes | Aegis Authenticator |
| Mail | Proton Mail or Tuta |
| Maps without tracking | Organic Maps |
| Browser | Vanadium (built in) |

**If an app doesn't work without Google services** (banking, taxi): App Store → **Sandboxed Google Play**. Install it **in a separate profile**: there Google Play runs as a regular sandboxed app with no special privileges and sees nothing outside that profile.

Two more GrapheneOS features, offered when granting permissions:
- **Storage Scopes** — an app sees only the files you give it;
- **Contact Scopes** — an app sees only the contacts you choose.

## 9. Network and VPN

The **Network permission** exists only in GrapheneOS. Remove it and the app can't reach the internet at all:
- when installing an app — the **Network** toggle;
- for an installed app: long-press the icon → **App info → Permissions → Network**.

The **Sensors** permission (accelerometer and others) can be turned off there too.

**VPN for the whole phone:**
1. Install your VPN client: WireGuard for WireGuard; for a V2Ray/Xray server, such as our VPN Router's, a VLESS-capable client (e.g. v2rayNG via Obtainium).
2. **Settings → Network & internet → VPN** → ⚙ next to the VPN → turn on **Always-on VPN** and **Block connections without VPN**. If the VPN drops, the phone won't go online directly.

## 10. Backups

- **Settings → System → Backup** (Seedvault): an encrypted backup to a USB drive or cloud. Write the **12-word recovery code** on paper.
- Keep passwords and 2FA in KeePassDX or Bitwarden with a separate backup.
- Seed phrases — offline only.

## Quick checklist

- [ ] GrapheneOS installed with the official web installer, bootloader locked
- [ ] Boot key fingerprint matches
- [ ] OEM unlocking turned off
- [ ] 6+ digit PIN, scrambled layout on
- [ ] Auto reboot set, USB-C — *Charging-only when locked*
- [ ] Duress PIN set
- [ ] Profiles created, each with its own PIN
- [ ] Network permission removed from apps that don't need it
- [ ] Always-on VPN with blocking of connections without VPN
- [ ] Seedvault backup, recovery code on paper

Questions — Telegram [@cakescats](https://t.me/cakescats) or [mail@cakescats.com](mailto:mail@cakescats.com). Official documentation: [grapheneos.org/usage](https://grapheneos.org/usage).
