# VirtualCatsOS: a private workspace on your Mac (Apple silicon)

[Русская версия](README.ru.md)

This is the step-by-step guide to build, for free, the same setup we install for our clients. You will get:

- **an Ubuntu (ARM64) virtual machine in UTM** — whatever you do inside is separated from macOS;
- **all network traffic of the VM goes through Tor**, and if Tor is not running the VM simply has no internet (nothing can leak around it);
- **the VM is stored in an encrypted VeraCrypt container with a hidden volume** (“false bottom”): an outer volume with ordinary files and a hidden volume whose existence cannot be shown without the second password.

Time: 1.5–2 hours. You need a Mac with Apple M1 or newer, ideally 16 GB of memory, and about 80 GB of free space.

> Don't want to do it yourself? We'll set it up and support it for you — see [os.cakescats.com](https://os.cakescats.com/). That is also how you support our non-profit project.

---

## Contents

1. [Know this first](#1-know-this-first)
2. [Downloads](#2-downloads)
3. [VeraCrypt container with a hidden volume](#3-veracrypt-container-with-a-hidden-volume)
4. [The virtual machine in UTM](#4-the-virtual-machine-in-utm)
5. [Installing Ubuntu](#5-installing-ubuntu)
6. [Ubuntu privacy settings](#6-ubuntu-privacy-settings)
7. [All traffic through Tor](#7-all-traffic-through-tor)
8. [Browser](#8-browser)
9. [Everyday use](#9-everyday-use)
10. [Troubleshooting](#10-troubleshooting)

---

## 1. Know this first

- **It is not a magic button.** The workspace protects against macOS telemetry, against leaking your IP address, and against someone who gets your Mac seeing what's inside the VM. It won't help if you log into your personal accounts inside the workspace, or if the Mac itself has spyware recording the screen and keyboard.
- **The hidden volume only works if you follow the rules** in section 9 — otherwise you can damage or reveal it.
- **Passwords cannot be recovered.** Lose them and the data is gone. Write them down on paper and keep them away from the Mac.
- Follow the laws of your country.

## 2. Downloads

| What | Where | Why |
|---|---|---|
| **VeraCrypt 1.26.29 or newer**, “FUSE-T” build for macOS | [veracrypt.io/en/Downloads.html](https://veracrypt.io/en/Downloads.html) | Encrypted container. **Do not use** 1.26.6–1.26.28: a bug there weakens hidden volumes in file containers |
| **FUSE-T** | [fuse-t.org](https://www.fuse-t.org/) | Needed by VeraCrypt on Mac, no kernel extension required |
| **UTM** | [mac.getutm.app](https://mac.getutm.app/) (free) | Virtual machine |
| **Ubuntu Server LTS for ARM64** | [ubuntu.com/download/server/arm](https://ubuntu.com/download/server/arm) | The operating system inside |

**Verify the Ubuntu image** in Terminal:

```bash
shasum -a 256 ~/Downloads/ubuntu-*-live-server-arm64.iso
```

Compare the result with the `SHA256SUMS` file on the download page. If it doesn't match, download again.

## 3. VeraCrypt container with a hidden volume

Install FUSE-T, then VeraCrypt. On first launch macOS will ask for permission in System Settings → Privacy & Security — allow it.

### 3.1. Create the container

1. VeraCrypt → **Create Volume** → **Create an encrypted file container** → **Hidden VeraCrypt volume** → **Normal mode**.
2. Location: e.g. `~/Documents/archive.hc`. A boring file name is better.
3. **Outer volume:** AES, size **120 GB** (the VM needs ~60 GB in the hidden volume, plus room for ordinary files in the outer one).
4. Outer volume password — **password #1** (the one you can reveal). Long, and different from the hidden one.
5. Outer volume filesystem: **exFAT**.
6. Move the mouse inside the window until the bar fills up, then **Format**.
7. VeraCrypt opens the outer volume. **Copy plausible but unimportant files there** — documents, photos, PDFs. This is the “first bottom”.
8. **Hidden volume:** size **65 GB** (VeraCrypt shows the maximum), filesystem **Mac OS Extended**, password — **password #2** (the main one, different from #1).
9. Done. Click **Dismount All**.

### 3.2. How it works

- **Password #1** opens the outer volume with ordinary files.
- **Password #2** opens the hidden volume with the virtual machine.
- From the outside `archive.hc` looks like random data; it is not possible to prove there is a second volume inside.

### 3.3. Remove traces in macOS

Mount the hidden volume (password #2). Say it appears as `/Volumes/NO NAME`:

```bash
sudo mdutil -i off "/Volumes/NO NAME"             # Spotlight won't index it
sudo tmutil addexclusion ~/Documents/archive.hc    # Time Machine won't back it up
```

## 4. The virtual machine in UTM

1. UTM → **+** → **Virtualize** → **Linux**.
2. Memory: **8 GB** (4 GB if your Mac has 8 GB). CPU cores: **4**.
3. Boot ISO Image: select the Ubuntu image.
4. Disk: **48 GB**. Shared directory: **none**.
5. Name: something neutral, e.g. `work`. **Save**.
6. **Move the VM into the hidden volume.** Right-click the VM → **Show in Finder**. Quit UTM, drag `work.utm` into the mounted hidden volume and delete the original. Open UTM → **File → Open** → choose `work.utm` in the hidden volume.
7. VM settings (right-click → **Edit**):
   - **Sharing → Clipboard Sharing: off** (the Mac and VM clipboards don't mix);
   - **Network → Network Mode: Shared Network**.

## 5. Installing Ubuntu

Start the VM and go through the installer:

1. Language and keyboard — your choice.
2. Installation type — the default is fine.
3. Network — leave as is (DHCP).
4. Storage — **Use an entire disk**. Keep **Set up this disk as an LVM group**; **Encrypt the LVM group with LUKS** is optional (a second layer of encryption).
5. User name and computer name — **not your real ones** (e.g. `user` and `workstation`).
6. **Install OpenSSH server** — no. **Import SSH identity** — no. Extra snaps — none.
7. After installation: **Reboot Now**. If it boots into the installer again, shut it down, clear the CD/DVD drive in UTM settings and start it again.

Log in and install the desktop:

```bash
sudo apt update && sudo apt full-upgrade -y
sudo apt install -y ubuntu-desktop-minimal spice-vdagent
sudo reboot
```

## 6. Ubuntu privacy settings

Open Terminal in Ubuntu:

```bash
# Remove reporting and statistics
sudo apt purge -y ubuntu-report popularity-contest apport whoopsie 2>/dev/null
sudo apt autoremove -y

# Disable login "news"
sudo sed -i 's/^ENABLED=1/ENABLED=0/' /etc/default/motd-news 2>/dev/null
sudo systemctl disable --now motd-news.timer 2>/dev/null

# Firewall: deny incoming connections
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw enable

# Automatic security updates
sudo apt install -y unattended-upgrades
sudo dpkg-reconfigure -plow unattended-upgrades
```

In **Ubuntu Settings**:

- **Privacy → Location Services**: off.
- **Privacy → Diagnostics**: Never.
- **Privacy → File History**: off, and clear it.
- **Power → Screen Blank**: 5 minutes; **Screen Lock**: on.

## 7. All traffic through Tor

Install Tor and set up “transparent” mode: every program in the VM reaches the internet only through Tor. If Tor is stopped, there is no internet.

```bash
sudo apt install -y tor nftables
```

Add the Tor settings:

```bash
sudo tee -a /etc/tor/torrc >/dev/null <<'EOF'

# --- VirtualCatsOS: transparent mode ---
VirtualAddrNetworkIPv4 10.192.0.0/10
AutomapHostsOnResolve 1
TransPort 127.0.0.1:9040
DNSPort 127.0.0.1:5353
EOF
sudo systemctl restart tor
```

Network rules (all TCP and DNS go to Tor, everything else is blocked, IPv6 is blocked):

```bash
sudo tee /etc/nftables.conf >/dev/null <<'EOF'
#!/usr/sbin/nft -f
flush ruleset

table ip torify {
  chain nat_out {
    type nat hook output priority -100; policy accept;
    meta skuid "debian-tor" return
    ip daddr 127.0.0.0/8 return
    udp dport 53 redirect to :5353
    tcp flags & (fin|syn|rst|ack) == syn redirect to :9040
  }
  chain filter_out {
    type filter hook output priority 0; policy drop;
    oifname "lo" accept
    meta skuid "debian-tor" accept
    ct state established,related accept
    udp dport 67 accept
  }
}

table ip6 noipv6 {
  chain filter_out {
    type filter hook output priority 0; policy drop;
    oifname "lo" accept
  }
}
EOF
sudo systemctl enable --now nftables
```

Point the VM's DNS to Tor:

```bash
sudo mkdir -p /etc/systemd/resolved.conf.d
printf '[Resolve]\nDNS=127.0.0.1:5353\nDNSStubListener=yes\nDomains=~.\n' | sudo tee /etc/systemd/resolved.conf.d/tor.conf
sudo systemctl restart systemd-resolved
```

**Check:**

```bash
curl -s https://check.torproject.org/api/ip
```

You should see `"IsTor":true`. To test the “kill switch”, stop Tor (`sudo systemctl stop tor`) and run the command again — there should be no answer. Then `sudo systemctl start tor`.

> Internet time sync doesn't work in this mode; the VM takes its time from the Mac, which is enough. If Tor complains about the clock, reboot the VM.

## 8. Browser

There is no official stable Tor Browser for ARM64 Linux yet (only nightly builds). So:

- **Firefox** (already installed) — all its traffic already goes through Tor at the system level. Settings → **Privacy & Security** → **Strict**, **Delete cookies and site data when Firefox is closed** — on, Firefox data collection — off. In `about:config` set `privacy.resistFingerprinting = true`.
- **Don't log into personal accounts** (mail, social networks) you also use outside this workspace — that defeats the purpose.
- When the Tor Project releases a stable Tor Browser for ARM64 Linux, install it.

## 9. Everyday use

**Start working:**
1. VeraCrypt → select `archive.hc` → **Mount** → **password #2** (hidden volume).
2. UTM → start the `work` VM.

**Finish working:**
1. Shut down Ubuntu (menu → Power Off).
2. Quit UTM.
3. VeraCrypt → **Dismount All**.

**Rules for the false bottom:**
- Now and then mount the **outer** volume (password #1) and change files there so it looks alive.
- **Before writing to the outer volume**, enable hidden volume protection: VeraCrypt → **Mount** → **Mount Options** → **Protect hidden volume against damage** → enter **password #2**. Without it, writing to the outer volume can overwrite the hidden one.
- Never keep a copy of `work.utm` outside the hidden volume.
- Don't keep several backup versions of the container: comparing two copies can reveal that “free” space is changing.

## 10. Troubleshooting

| Problem | Fix |
|---|---|
| No internet in the VM | `sudo systemctl status tor` — Tor must be `active`. Wait a minute after boot. |
| Need to turn Tor mode off temporarily | `sudo systemctl stop nftables` (traffic goes direct!). Back: `sudo systemctl start nftables`. |
| VeraCrypt won't mount | Check FUSE-T is installed and allowed in Privacy & Security. |
| UTM doesn't show the VM after moving it | UTM → File → Open → choose `work.utm` inside the mounted hidden volume. |
| Hidden volume is full | Create a bigger container and move `work.utm`. An existing hidden volume can't be enlarged. |

Questions — Telegram [@cakescats](https://t.me/cakescats) or [mail@cakescats.com](mailto:mail@cakescats.com).
