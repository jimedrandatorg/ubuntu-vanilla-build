# Ubuntu CLI Installer

A TTY-friendly, Calamares-equivalent installer for the **Ubuntu Vanilla Build**
live ISO. It walks the user through the same stages as
[scripts/calamares/settings.conf](../calamares/settings.conf) (welcome →
locale → keyboard → partition → users → summary → mount → unpackfs →
machineid → fstab → locale → keyboard → users → displaymanager →
networkcfg → hwclock → initramfs → grubcfg → bootloader → packages → umount
→ finished) but entirely from a shell.

## Why

* Boots a desktop ISO into a TTY (or a CLI-only ISO) and installs the system
  from the live environment — useful on machines without a working display
  server or for fully scripted headless installs.
* The Calamares-based ISO already works on desktops; this script is a
  *companion* path for the same ISO that runs entirely from a shell.

## Usage

The script is a single self-contained bash file
([`install-system`](install-system)). To use it on the live system,
**make it executable first**, then **copy it into the system** (e.g.
`/usr/local/bin/`), then **run it**:

```bash
# 1. Make it executable FIRST (a fresh checkout from the repo is not
#    always +x):
chmod +x install-system

# 2. Copy the script into the system — install(1) sets mode 0755 in one step:
sudo install -m 0755 install-system /usr/local/bin/

# 3. Run it as a normal system command:
sudo install-system
```

If you are iterating on the script and only need it on the live system
temporarily, you can skip step 2 and run it in place:

```bash
chmod +x install-system
sudo ./install-system
```

The script is **strictly interactive** by design — every step asks the user
on the TTY.

### UI mode

At the welcome screen the user is asked which interface to use:

1. **TUI (whiptail/dialog boxes)** — recommended, falls back to plain text
   if neither `whiptail` nor `dialog` is installed.
2. **Plain text prompts** — pure `read` based, works in any TTY, zero
   extra dependencies.

### What it covers

| Step | Source of truth in Calamares |
| --- | --- |
| Locale | `calamares/i18n/SUPPORTED` (curated UTF-8 list) |
| Timezone | `calamares/modules/locale.conf` (`useSystemTimezone`, no GeoIP) |
| Disk | `calamares/modules/partition.conf` (GPT, 512 MiB ESP, 4 GiB swap, root fills rest) |
| User | `calamares/modules/users.conf` (sudo group, /bin/bash, minLength 8) |
| Initramfs | `calamares/modules/shellprocess_bug-LP#1829805.conf` |
| Bootloader | `calamares/modules/bootloader.conf` + `grubcfg.conf` (GRUB_ENABLE_CRYPTODISK) |
| Packages | `calamares/modules/packages.conf` (remove `calamares`/`casper`/…, try-install language packs) |

### What it does **not** cover

* **LUKS full-disk encryption** — Calamares offers it; this CLI installer
  intentionally does not (manual cfdisk users can still set it up by hand).
* **Unattended / non-interactive mode** — by design, every step is
  answered on the TTY.
* **GeoIP** — the ISO has none (see `locale.conf`); users pick manually.

## Files

* `install-system` — the installer (one self-contained bash script).
* `my plan to create the installer.txt` — the original design notes.
* `README.md` — this file.
