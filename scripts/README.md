# Build Scripts Reference

This directory contains the core compilation and configuration scripts used to assemble the custom bootable live ISOs and ready-to-use disk images:

- **`build.sh`** — builds a vanilla **Ubuntu** ISO (Calamares config in `calamares/`).
- **`build-popos.sh`** — builds a **Pop!_OS** ISO from the Pop!_OS repositories (Calamares config in `calamares-popos/`).
- **`build-img.sh`** / **`build-popos-img.sh`** — build a ready-to-deploy **cloud disk image** (raw `.img` with cloud-init) for Ubuntu / Pop!_OS.
- **`build-vm.sh`** / **`build-popos-vm.sh`** — build a ready-to-use **VM disk image** (raw `.img` plus QCOW2/VDI/VMDK/VHDX exports via `qemu-img`) for Ubuntu / Pop!_OS.
- **`build-removable.sh`** / **`build-popos-removable.sh`** — build a ready-to-flash **removable-media disk image** (raw `.img` for USB sticks, SD cards, CompactFlash, and similar) for Ubuntu / Pop!_OS. Hybrid firmware (BIOS + UEFI) by default, dynamic swap sizing, NoCloud cloud-init for growpart/resize, same `--user-mode=build|deploy` user flow as the cloud/VM images.

All eight share the same command syntax, stages, options, and hooks. The repository-root `start-here.sh` asks which one to run (or pass `--distro=ubuntu|popos` and `--output=iso|img|vm|removable`).

The image builders (`*-img.sh`, `*-vm.sh`, `*-removable.sh`) replace the ISO stage with a `build_disk_image` stage and add these choices: firmware `--firmware=uefi|hybrid` (UEFI-only: GPT + 512 MB ESP; Hybrid: GPT + 1 MiB BIOS boot partition + the same ESP, GRUB installed for both BIOS and UEFI targets), size `--disk-size=8|16|32|64|<GB>` (cloud/VM: minimum 10, layout ESP 512 MB + swap 4 GB + root = rest; removable: minimum 8, swap 2 GB if size < 16 else 4 GB, root = rest; on first boot growpart resizes root to fill the actual media), allocation tool `--alloc-tool=truncate|fallocate|dd` (sparse, preallocated, or dd zero-written), profile `--profile=desktop|cli` (desktop-ready or CLI/TTY-only), network stack `--network=networkd|network-manager` (CLI images; desktop images always use NetworkManager), and user account `--user-mode=build|deploy` (bake `--username`/`--password` in at build time, or create the user after deployment — cloud-init on cloud images, a first-boot console wizard on VM and removable images). VM builders also take `--formats=qcow2,vdi,vmdk,vhdx|all|none`. Each script keeps its own workspace (`workspace-img/`, `workspace-vm/`, `workspace-removable/`, plus the matching `workspace-popos-*`) and config file (`build-img.cfg`, `build-vm.cfg`, `build-removable.cfg`, plus the matching `build-popos-*.cfg`), so they never collide with the ISO builders. See the repository-root README sections “Building Cloud & VM Disk Images” and “Building Removable Media Disk Images” for details and examples.

---

## The build.sh Script

The primary script `build.sh` orchestrates the entire image generation process. It is designed to run in two distinct environments:
1. **On the Host System**: Prepares directories, downloads the base files via `debootstrap`, enters the chroot environment to trigger customization, compresses the system using SquashFS, and packages the result into a bootable ISO.
2. **Inside the Chroot Environment**: Configures packages, blocks Snap packages via APT pinning, sets up browser repositories, and installs desktop environments. This mode is invoked internally with the `--chroot-internal` flag. Do not invoke this flag manually; the `run_chroot` host function handles this automatically.

### Supported Target Releases
Only the following targets are supported:
- **jammy** (Ubuntu 22.04 LTS) — supports the `calamares` and `ubiquity` installers.
- **noble** (Ubuntu 24.04 LTS) — supports the `calamares` installer.
- **resolute** (Ubuntu 26.04 LTS) — supports the `calamares` installer.

By default, the script installs Hardware Enablement (HWE) kernels (`linux-generic-hwe-XX.04` or `linux-lowlatency-hwe-XX.04`) along with their recommended dependencies (firmware, microcode, etc.) to ensure broad compatibility with modern hardware. You can pin a custom metapackage by setting `TARGET_KERNEL_PACKAGE` in the environment.

---

## The build-popos.sh Script (Pop!_OS)

`build-popos.sh` is a clone of `build.sh` that produces a Pop!_OS ISO. Same stages (`setup_host`, `debootstrap`, `run_chroot`, `build_iso`), same modular syntax, same hooks — with these differences:

- **Repositories**: everything is served from the CDN-backed `https://apt.pop-os.org/` — the `ubuntu` mirror for debootstrap/base packages, plus the `release`, `proprietary`, and `release-ubuntu` suites configured inside the chroot (`apt-origin.pop-os.org` is only a per-suite fallback when the CDN does not publish a suite). All are signed with the Pop!_OS archive keyring (`pop-keyring` takes over after bootstrap). Only the packages the build actually takes from Pop (`base-files`, `pop-*`, the System76 kernel/driver stack) are pinned at priority 1001 (`o=pop-os-release`) — a repo-wide pin (what stock Pop!_OS ships) would force Pop's older `release-ubuntu` rebuilds of the whole GNOME stack and break the Ubuntu desktop metapackages this builder installs. **Staging suites are deliberately excluded.**
- **OS identity**: Pop!_OS's `base-files` is installed explicitly right after the repos are configured, so `/etc/os-release` reports `NAME="Pop!_OS"` / `ID=pop` deterministically (with a warning if the target suite does not ship it yet).
- **Kernel**: `--kernel=system76|generic|lowlatency`. The default `system76` installs `linux-system76` from the Pop!_OS repos (tracks the stable Linux branch); `generic`/`lowlatency` keep the stock Ubuntu HWE kernels.
- **System76 hardware driver (optional)**: `--system76-driver` / `TARGET_SYSTEM76_DRIVER=1` pre-installs `system76-driver` (off by default; only useful on System76 machines — it can always be installed later since the repos stay configured).
- **Bootloader**: GRUB (hybrid BIOS + UEFI) is kept instead of Pop!_OS's systemd-boot, so no >= 1 GiB EFI System Partition is required.
- **Desktops**: same list as `build.sh` (GNOME default). `pop-desktop` and COSMIC are intentionally **not** offered — COSMIC is still too buggy through Calamares. On noble/resolute targets, the end-of-build output explains how users can install COSMIC on the installed system (`sudo apt update && sudo apt install cosmic-session`, then pick the COSMIC session at the login screen).
- **Separate state**: a `workspace-popos/` subdirectory (under the same workspace parent as Ubuntu builds), its own APT package cache, and config file `build-popos.cfg`, so Ubuntu and Pop!_OS builds never collide.
- **Output naming**: ISOs are named `popos-<version>-<desktop>-amd64-<timestamp>.iso` and land in the same output directory as Ubuntu builds (your home directory by default).

### Syntax and Modular Execution
Advanced users can execute individual segments of the build pipeline instead of building the entire ISO in one run:

```bash
./build.sh [options] [start_cmd] [-] [end_cmd]
```

- **Run all stages (default)**: `./build.sh -`
- **Single stage**: Run from `start_cmd` to the end of that command. For example, `./build.sh debootstrap` builds only the base system.
- **Stage range**: Run from `start_cmd` through `end_cmd`. For example, `./build.sh setup_host - run_chroot` runs host preparation, debootstrap, and chroot customization, but stops before compressing the final ISO.

Supported commands:
- `setup_host`: Installs the required host build tools (`debootstrap`, `squashfs-tools`, `xorriso`) and prepares the workspace directory. GRUB packages are installed inside the chroot, not on the host.
- `debootstrap`: Pulls the minimal Ubuntu base structure from the mirror into the chroot directory.
- `run_chroot`: Enters the rootfs to disable snapd, set up security policies, and install the chosen desktop profile, browsers, and packages.
- `build_iso`: Compresses the chroot workspace using SquashFS, copies the Casper kernel/initrd boot files, and builds the final hybrid UEFI/BIOS bootable ISO image with GRUB.

---

## Customizing the Live Installer

The Calamares installer configuration files are stored inside the `calamares/` subdirectory for Ubuntu builds and `calamares-popos/` for Pop!_OS builds (same layout, Pop!_OS branding under `branding/pop/`). During the `run_chroot` stage, the builder installs `settings.conf`, `modules/`, and `branding/` into `/etc/calamares/` inside the live system, and the curated `i18n/SUPPORTED` locale list into `/usr/share/i18n/` (the stock file is backed up first):

- **settings.conf**: Defines the order of Calamares modules (welcome, locale, keyboard, partition, users, summary, then the exec phase, and finished) and selects the branding component.
- **modules/**: Contains configuration YAML files for individual installer steps (such as `partition.conf`, `packages.conf`, `locale.conf`, etc.). Modify these files to change installer workflows or pre-configure default options.
- **branding/**: Holds installer branding (`branding.desc`, rendered with the release version at build time), the logo/icon images, and the install slideshow (`show.qml`).
- **i18n/SUPPORTED**: Curated locale list that keeps the installer's language step responsive.

## The CLI Installer (`cli-installer/`)

For machines where Calamares cannot run (no display, a minimal/server ISO, or
a TTY-only VM console), this project also ships a Calamares-equivalent shell
installer: [`cli-installer/install-system`](cli-installer/install-system). It
walks the user through the same `welcome → locale → keyboard → partition →
users → summary → mount → unpackfs → … → finished` sequence from a terminal
and uses the exact same YAML configs under `calamares/` and `calamares-popos/`
as its source of truth. See the main [README](../../README.md#installing-from-the-live-iso-cli-installer)
or [cli-installer/README.md](cli-installer/README.md) for the install
command, UI mode, and feature limits.

---

## Workspace Lifecycle and Cleanups

During execution, the builder creates a `workspace/` directory (`workspace-popos/` for Pop!_OS builds) to store temporary assets:
- **workspace/chroot/**: Contains the live system rootfs during build.
- **workspace/image/**: Stores bootloader files, kernels, and metadata files destined for the ISO filesystem.

Where the workspace lives depends on the build mode:
- **Basic mode**: `/var/cache/ubuntu-vanilla-build/` — a root-owned system directory regular users cannot touch, always on a Linux-native filesystem (so WSL `/mnt/c` unpack issues can't occur).
- **Advanced mode**: you are asked interactively for the workspace directory (default: `~/uvb-workspace`); non-interactive runs use the default silently.
- **Override (any mode)**: `UBUNTU_VANILLA_WORKSPACE=/some/path` sets the parent directory explicitly and skips the prompt. Paths on Windows-backed mounts (`/mnt/...`, `/media/...`, 9p/DrvFs) are relocated automatically to the system directory.

The finished ISO and its `.sha1`/`.sha256` checksums are written to your home directory (`/home/$USER`) by default. In advanced mode you are asked for the output directory (default: your home directory), and `UVB_OUTPUT_DIR=/some/path` overrides it in any mode.

On successful compilation, the script automatically deletes the workspace directory to reclaim disk space. In basic mode, a failed or interrupted build also unmounts the chroot bind mounts and removes the workspace automatically. In advanced mode (`--advanced`), the workspace is preserved on failure so you can inspect it or resume from a specific stage (e.g. `./build.sh --advanced run_chroot`).
