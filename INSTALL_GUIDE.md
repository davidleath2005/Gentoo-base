# Gentoo Base Installation Guide

> **Target:** AMD64 (x86_64) · EFI Stub boot (no GRUB) · OpenRC · NVMe disk · Distribution kernel · UUID-based fstab

This guide follows the [official Gentoo AMD64 Handbook](https://wiki.gentoo.org/wiki/Handbook:AMD64) and is written for copy-paste usability. Every command block is ready to run exactly as shown unless a placeholder (e.g. `YOUR-ROOT-UUID-HERE`) is present.

> **⚠️ Init system: OpenRC** — systemd is never used, enabled, or mentioned as an option. All service management uses `rc-update` and `rc-service`.

---

## Table of Contents

1. [Preparation](#1-preparation)
2. [Disk Partitioning (NVMe)](#2-disk-partitioning-nvme)
3. [Stage 3 Tarball](#3-stage-3-tarball)
4. [Configure Portage (make.conf)](#4-configure-portage-makeconf)
5. [Chroot](#5-chroot)
6. [Portage Sync & Profile](#6-portage-sync--profile)
7. [Timezone, Locale & Hostname](#7-timezone-locale--hostname)
8. [Kernel Installation (Distribution Kernel + EFI Stub)](#8-kernel-installation-distribution-kernel--efi-stub)
9. [fstab Configuration (UUID-based)](#9-fstab-configuration-uuid-based)
10. [Networking & OpenRC Services](#10-networking--openrc-services)
11. [System Tools & Final Steps](#11-system-tools--final-steps)
12. [Reboot](#12-reboot)

---

## 1. Preparation

### 1.1 Download the Gentoo Minimal Install ISO

Go to the official Gentoo mirrors and download the AMD64 minimal install ISO:

```
https://distfiles.gentoo.org/releases/amd64/autobuilds/current-install-amd64-minimal/
```

Download the file named something like `install-amd64-minimal-<date>.iso`.

### 1.2 Write the ISO to a USB Drive

Replace `/dev/sdX` with your USB device (check with `lsblk` first — **do not** write to your target NVMe):

```bash
dd if=install-amd64-minimal-<date>.iso of=/dev/sdX bs=4M status=progress oflag=sync
```

### 1.3 Boot into the Live Environment

1. Insert the USB drive, reboot, and enter your firmware (BIOS/UEFI) boot menu (usually `F12`, `F2`, `Del`, or `Esc`).
2. Select the Gentoo USB drive.
3. At the boot prompt, press **Enter** to boot the default live environment.

### 1.4 Verify UEFI Mode

EFI Stub boot requires UEFI. Confirm the live environment booted in UEFI mode:

```bash
ls /sys/firmware/efi
```

If the directory exists and contains files (e.g. `efivars`), you are in UEFI mode. If the command returns "No such file or directory", you booted in legacy BIOS mode — re-check your firmware settings and ensure **Secure Boot** is either disabled or set to allow unsigned images.

### 1.5 Set Up Networking

#### Wired (DHCP — most common)

The live environment usually starts DHCP automatically. Verify connectivity:

```bash
ping -c 3 gentoo.org
```

If that fails, manually bring up the interface:

```bash
# Find your interface name
ip link

# Bring it up and get a DHCP lease (replace enp1s0 with your interface)
ip link set enp1s0 up
dhcpcd enp1s0
```

#### Wireless

```bash
# List wireless interfaces
iwconfig

# Scan for networks (replace wlan0 with your interface)
iwlist wlan0 scan | grep ESSID

# Connect using wpa_supplicant
wpa_passphrase "YourSSID" "YourPassphrase" > /tmp/wpa.conf
wpa_supplicant -B -i wlan0 -c /tmp/wpa.conf
dhcpcd wlan0
```

### 1.6 Sync Time

An accurate system clock is important for package verification. The live environment may include `chronyd` or `ntpdate`:

```bash
# Option A — chronyd (preferred if available)
chronyd -q 'server pool.ntp.org iburst'

# Option B — ntpdate (fallback)
ntpdate pool.ntp.org
```

Verify the time:

```bash
date
```

---

## 2. Disk Partitioning (NVMe)

> **Target disk:** `/dev/nvme0n1`
>
> If your system has additional drives, confirm you are targeting the correct one:
> ```bash
> lsblk
> ```

### 2.1 Partition Layout

| Partition | Size | Type | Filesystem | Mount Point |
|---|---|---|---|---|
| `/dev/nvme0n1p1` | 512 MiB | EFI System | FAT32 | `/efi` |
| `/dev/nvme0n1p2` | 8 GiB | Linux swap | swap | swap |
| `/dev/nvme0n1p3` | Remaining | Linux filesystem | ext4 | `/` |

### 2.2 Create the Partition Table with fdisk

```bash
fdisk /dev/nvme0n1
```

Inside `fdisk`, enter the following commands (lines beginning with `#` are explanations, not commands):

```
# Create a new GPT partition table (wipes existing data)
g

# Create partition 1 — EFI System Partition
n
1
[Enter — accept default first sector]
+512M

# Set type to EFI System (code 1)
t
1

# Create partition 2 — Swap
n
2
[Enter — accept default first sector]
+8G

# Set type to Linux swap (code 19)
t
2
19

# Create partition 3 — Root
n
3
[Enter — accept default first sector]
[Enter — accept default last sector, uses remaining space]

# Verify the layout
p

# Write and exit
w
```

### 2.3 Format the Partitions

```bash
# EFI System Partition — must be FAT32
mkfs.fat -F 32 /dev/nvme0n1p1

# Swap
mkswap /dev/nvme0n1p2

# Root filesystem
mkfs.ext4 /dev/nvme0n1p3
```

### 2.4 Activate Swap and Mount Filesystems

```bash
# Activate swap
swapon /dev/nvme0n1p2

# Mount root
mount /dev/nvme0n1p3 /mnt/gentoo

# Mount the EFI partition (will also be mounted inside chroot at /efi)
mkdir -p /mnt/gentoo/efi
mount /dev/nvme0n1p1 /mnt/gentoo/efi
```

---

## 3. Stage 3 Tarball

### 3.1 Navigate to the Mount Point

```bash
cd /mnt/gentoo
```

### 3.2 Download the Stage 3 (OpenRC)

Browse the current autobuilds directory to find the latest OpenRC stage3:

```
https://distfiles.gentoo.org/releases/amd64/autobuilds/current-stage3-amd64-openrc/
```

> **⚠️ Critical:** Download the **OpenRC** stage3. The filename contains `openrc` (e.g. `stage3-amd64-openrc-<date>T<time>Z.tar.xz`). Do **NOT** download the `systemd` variant.

```bash
# Use wget or curl to download (replace the filename with the current one from the URL above)
wget https://distfiles.gentoo.org/releases/amd64/autobuilds/current-stage3-amd64-openrc/stage3-amd64-openrc-<date>T<time>Z.tar.xz
```

Alternatively, use `links` (available in the live environment) to browse and download interactively:

```bash
links https://distfiles.gentoo.org/releases/amd64/autobuilds/current-stage3-amd64-openrc/
```

### 3.3 (Optional) Verify the Download

```bash
# Download the digests file alongside the tarball
wget https://distfiles.gentoo.org/releases/amd64/autobuilds/current-stage3-amd64-openrc/stage3-amd64-openrc-<date>T<time>Z.tar.xz.DIGESTS

# Verify SHA512
sha512sum --check --ignore-missing stage3-amd64-openrc-<date>T<time>Z.tar.xz.DIGESTS
```

### 3.4 Extract the Stage 3

```bash
tar xpvf stage3-*.tar.xz --xattrs-include='*.*' --numeric-owner
```

> The flags mean:
> - `x` — extract
> - `p` — preserve permissions
> - `v` — verbose
> - `f` — file
> - `--xattrs-include='*.*'` — preserve extended attributes (required for SELinux/capabilities)
> - `--numeric-owner` — preserve numeric UIDs/GIDs exactly

---

## 4. Configure Portage (`make.conf`)

Edit the Portage build configuration:

```bash
nano /mnt/gentoo/etc/portage/make.conf
```

Replace the contents with the following (adjust `VIDEO_CARDS` for your hardware):

```bash
# These settings were set by the catalyst build script that automatically
# built this stage.
# Please consult /usr/share/portage/config/make.conf.example for a more
# detailed example.
COMMON_FLAGS="-march=native -O2 -pipe"
CFLAGS="${COMMON_FLAGS}"
CXXFLAGS="${COMMON_FLAGS}"
FCFLAGS="${COMMON_FLAGS}"
FFLAGS="${COMMON_FLAGS}"

# Number of parallel make jobs — use all CPU threads
MAKEOPTS="-j$(nproc)"

# Accept all licenses
ACCEPT_LICENSE="*"

# Mirrors — will be set by mirrorselect (see below)
# GENTOO_MIRRORS=""

# CPU architecture flags (auto-detected via -march=native, but explicit is fine)
# CPU_FLAGS_X86="aes avx avx2 f16c fma3 mmx mmxext pclmul popcnt rdrand sse sse2 sse3 sse4_1 sse4_2 ssse3"

# Video card drivers — set for your GPU:
#   Intel iGPU:           intel
#   AMD/Radeon:           radeonsi (or amdgpu)
#   NVIDIA (open source): nouveau
#   NVIDIA (proprietary): nvidia  (requires separate setup)
#   Virtual/none:         -* (or leave empty)
VIDEO_CARDS="intel"

# Example USE flags — customize as needed after base install
# USE="-bluetooth -nls"
```

> **Note on `-march=native`:** This compiles packages optimized for your specific CPU. It is set inside the chroot after rebooting into the installed system. During the initial build from the live environment it still works because the live environment runs on the same hardware.

### 4.1 Select Mirrors (Optional but Recommended)

Inside the live environment (before chroot), or after entering chroot:

```bash
# Install mirrorselect if not already available
emerge --ask app-portage/mirrorselect

# Interactively select geographically close mirrors
mirrorselect -i -o >> /mnt/gentoo/etc/portage/make.conf
```

---

## 5. Chroot

### 5.1 Copy DNS Configuration

```bash
cp --dereference /etc/resolv.conf /mnt/gentoo/etc/
```

### 5.2 Mount Necessary Filesystems

```bash
mount --types proc /proc /mnt/gentoo/proc
mount --rbind /sys /mnt/gentoo/sys
mount --make-rslave /mnt/gentoo/sys
mount --rbind /dev /mnt/gentoo/dev
mount --make-rslave /mnt/gentoo/dev
mount --bind /run /mnt/gentoo/run
mount --make-slave /mnt/gentoo/run
```

### 5.3 Enter the Chroot

```bash
chroot /mnt/gentoo /bin/bash
source /etc/profile
export PS1="(chroot) ${PS1}"
```

Your prompt should now show `(chroot)` to confirm you are inside the new system.

### 5.4 Mount the EFI Partition Inside the Chroot

```bash
mount /dev/nvme0n1p1 /efi
```

> The EFI partition was already formatted and mounted at `/mnt/gentoo/efi` from outside the chroot. Inside the chroot it is accessible as `/efi`. The `mount` command above re-registers it inside the chroot's mount namespace so that tools like `installkernel` and `efibootmgr` can see and write to it correctly.

---

## 6. Portage Sync & Profile

### 6.1 Sync Portage

```bash
# Initial sync from a web snapshot (faster for first run)
emerge-webrsync

# Then sync from the official rsync tree to get the latest updates
emerge --sync
```

If you see news items, read them with:

```bash
eselect news list
eselect news read
```

### 6.2 Select a Profile

List available profiles:

```bash
eselect profile list
```

You will see output similar to:

```
  [1]   default/linux/amd64/23.0 (stable)
  [2]   default/linux/amd64/23.0/systemd (stable)
  ...
```

> **⚠️ OpenRC only:** Select a profile that does **not** contain `systemd` in the name. The plain `default/linux/amd64/23.0` profile uses OpenRC.

```bash
# Select the default OpenRC amd64 profile (adjust the number to match your list)
eselect profile set 1
```

Verify the selection:

```bash
eselect profile show
```

### 6.3 Update the @world Set

This rebuilds any packages whose USE flags changed due to the new profile:

```bash
emerge --ask --verbose --update --deep --newuse @world
```

This may take a while depending on your CPU speed. Go grab a coffee.

---

## 7. Timezone, Locale & Hostname

### 7.1 Set the Timezone

```bash
echo "America/New_York" > /etc/timezone
emerge --config sys-libs/timezone-data
```

> Replace `America/New_York` with your timezone. A full list is available at `/usr/share/zoneinfo/` — e.g. `Europe/London`, `Asia/Tokyo`, `Australia/Sydney`.

### 7.2 Configure Locales

Edit `/etc/locale.gen` and uncomment the locales you need:

```bash
nano /etc/locale.gen
```

Uncomment (remove the `#`) from lines like:

```
en_US ISO-8859-1
en_US.UTF-8 UTF-8
```

Generate the locales:

```bash
locale-gen
```

List available locales and set the system default:

```bash
eselect locale list
# Example output:
#   [1]   C
#   [2]   C.utf8
#   [3]   en_US
#   [4]   en_US.iso88591
#   [5]   en_US.utf8 *   (the * marks the current selection)
#   [6]   POSIX

# Set UTF-8 locale (adjust number to match your list)
eselect locale set 5
```

Set the locale in `/etc/env.d/02locale` for extra robustness:

```bash
cat > /etc/env.d/02locale << 'EOF'
LANG="en_US.UTF-8"
LC_COLLATE="C.UTF-8"
EOF
```

Reload the environment:

```bash
env-update && source /etc/profile && export PS1="(chroot) ${PS1}"
```

### 7.3 Set the Hostname

```bash
nano /etc/conf.d/hostname
```

Change the `hostname` variable to your desired name:

```bash
hostname="gentoo"
```

---

## 8. Kernel Installation (Distribution Kernel + EFI Stub)

This section installs the **Gentoo distribution kernel** (`sys-kernel/gentoo-kernel`), which compiles from source using Gentoo's maintained kernel config (no manual `menuconfig` needed), and configures **EFI Stub** boot so the kernel is booted directly by UEFI firmware — no bootloader required.

> **OpenRC only — no systemd USE flag.** The `installkernel` package handles placing the kernel on the EFI partition. We enable only the `efistub` USE flag, never the `systemd` USE flag.

### 8.1 Configure `installkernel` USE Flags

```bash
mkdir -p /etc/portage/package.use
echo "sys-kernel/installkernel efistub" > /etc/portage/package.use/installkernel
```

### 8.2 Emerge `installkernel`

```bash
emerge --ask sys-kernel/installkernel
```

### 8.3 Install Firmware

```bash
emerge --ask sys-kernel/linux-firmware
```

> `linux-firmware` provides binary blobs for hardware that requires them (Wi-Fi adapters, GPUs, NVMe controllers, etc.). It is large but safe to install broadly.

### 8.4 Install the Distribution Kernel

```bash
emerge --ask sys-kernel/gentoo-kernel
```

This step:
1. Downloads the kernel sources.
2. Configures the kernel using Gentoo's maintained distribution config.
3. Compiles the kernel (this takes several minutes — more on slower hardware).
4. Calls `installkernel` which, with the `efistub` USE flag, copies the compiled kernel image to the EFI partition.

### 8.5 Install `efibootmgr`

```bash
emerge --ask sys-boot/efibootmgr
```

### 8.6 Locate the Kernel on the EFI Partition

After the kernel is installed, find where `installkernel` placed it:

```bash
ls /efi/EFI/
find /efi -name "*.efi" -o -name "vmlinuz*" -o -name "bzImage*"
```

Common locations:
- `/efi/EFI/Gentoo/bzImage.efi` — typical location used by `installkernel` with `efistub`
- `/efi/EFI/Linux/gentoo-*.efi` — used when unified kernel images (UKI) are enabled
- `/efi/EFI/BOOT/BOOTX64.EFI` — UEFI fallback path (boots automatically on most firmware)

### 8.7 Get the Root Partition UUID

```bash
blkid /dev/nvme0n1p3
```

Example output:
```
/dev/nvme0n1p3: UUID="a1b2c3d4-e5f6-7890-abcd-ef1234567890" BLOCK_SIZE="4096" TYPE="ext4" PARTUUID="..."
```

Copy the `UUID=` value (not `PARTUUID`).

### 8.8 Create the UEFI Boot Entry

Replace `YOUR-ROOT-UUID-HERE` with the UUID from `blkid`, and adjust `--loader` to match the actual path found in the previous step:

```bash
efibootmgr --create --disk /dev/nvme0n1 --part 1 \
  --label "Gentoo" \
  --loader '\EFI\Gentoo\bzImage.efi' \
  --unicode 'root=UUID=YOUR-ROOT-UUID-HERE rw'
```

> **Note on the loader path:** Use backslashes (`\`) as directory separators in the `--loader` argument — that is the UEFI convention. The path is relative to the root of the EFI partition (`/efi`). If `find` showed `/efi/EFI/Linux/gentoo-6.x.y-gentoo.efi`, the loader argument would be `'\EFI\Linux\gentoo-6.x.y-gentoo.efi'`.

> **Additional kernel parameters:** Add any extra parameters after `rw` separated by spaces, e.g.:
> ```
> 'root=UUID=YOUR-ROOT-UUID-HERE rw quiet loglevel=3'
> ```

### 8.9 Verify Boot Entries

```bash
efibootmgr -v
```

The Gentoo entry should appear at the top of the boot order or can be reordered with `efibootmgr --bootorder`.

### 8.10 Fallback: UEFI Default Boot Path

Many UEFI firmwares (especially on laptops and some desktops) will automatically boot `/EFI/BOOT/BOOTX64.EFI` without any `efibootmgr` entry — this is the UEFI fallback boot path. If you have trouble with `efibootmgr` entries not persisting (common on some Dell and HP hardware), use this approach:

```bash
mkdir -p /efi/EFI/BOOT

# Copy the kernel — adjust the source path to match what 'find' showed earlier
cp /efi/EFI/Gentoo/bzImage.efi /efi/EFI/BOOT/BOOTX64.EFI
```

> **Important:** This fallback approach embeds the kernel command line **inside** the EFI stub image at build time. With `installkernel` + `efistub`, the kernel parameters are typically passed via the `efibootmgr` entry (Section 8.8). If you rely solely on the fallback path, you may need to configure the kernel command line differently — check the `installkernel` documentation for `--cmdline` options or configure a kernel command line via `CMDLINE` in the kernel config (`CONFIG_CMDLINE`).

---

## 9. fstab Configuration (UUID-based)

Using UUIDs in `/etc/fstab` is critical on systems with multiple drives because device names like `/dev/nvme0n1p3` can change between boots if drive enumeration order changes.

### 9.1 Get UUIDs for All Three Partitions

```bash
blkid
```

Example output (your UUIDs will differ):
```
/dev/nvme0n1p1: UUID="AAAA-BBBB" BLOCK_SIZE="512" TYPE="vfat" PARTLABEL="EFI System Partition" PARTUUID="..."
/dev/nvme0n1p2: UUID="cccccccc-dddd-eeee-ffff-000000000000" TYPE="swap" PARTUUID="..."
/dev/nvme0n1p3: UUID="a1b2c3d4-e5f6-7890-abcd-ef1234567890" BLOCK_SIZE="4096" TYPE="ext4" PARTUUID="..."
```

### 9.2 Write `/etc/fstab`

```bash
nano /etc/fstab
```

Use the following template, replacing the UUID values with the ones from `blkid`:

```
# /etc/fstab — generated with UUIDs for multi-drive stability
# Run 'blkid' to find your UUIDs and replace below
#
# <filesystem>                            <mountpoint>  <type>  <options>           <dump> <pass>

# Root filesystem
UUID=a1b2c3d4-e5f6-7890-abcd-ef1234567890  /             ext4    defaults,noatime    0      1

# EFI System Partition
UUID=AAAA-BBBB                             /efi          vfat    defaults,noatime    0      2

# Swap
UUID=cccccccc-dddd-eeee-ffff-000000000000  none          swap    sw                  0      0
```

> **Field notes:**
> - `noatime` — skips updating access timestamps on every file read; improves SSD/NVMe performance and longevity.
> - `dump` column: `0` = no dump backup, `1` = include in dump.
> - `pass` column: `1` = root (checked first by fsck), `2` = other filesystems (checked after root), `0` = skip fsck.

---

## 10. Networking & OpenRC Services

### 10.1 Configure a Network Interface

#### Wired with DHCP

Find your interface name (it will be something like `enp1s0` or `eth0`):

```bash
ip link
```

Create a symlink to enable the DHCP netifrc script:

```bash
cd /etc/init.d
ln -s net.lo net.enp1s0
```

Configure DHCP in `/etc/conf.d/net`:

```bash
cat >> /etc/conf.d/net << 'EOF'
config_enp1s0="dhcp"
EOF
```

Add the network interface to the default runlevel:

```bash
rc-update add net.enp1s0 default
```

#### Wired with a Static IP

```bash
cat >> /etc/conf.d/net << 'EOF'
config_enp1s0="192.168.1.100/24"
routes_enp1s0="default via 192.168.1.1"
dns_servers_enp1s0="1.1.1.1 8.8.8.8"
EOF
```

### 10.2 Install and Enable `dhcpcd` (Alternative to netifrc)

If you prefer `dhcpcd` as a simpler alternative:

```bash
emerge --ask net-misc/dhcpcd
rc-update add dhcpcd default
```

### 10.3 Set the Root Password

```bash
passwd
```

### 10.4 Install Essential System Tools

```bash
# System logger (choose one — sysklogd is minimal and reliable)
emerge --ask app-admin/sysklogd
rc-update add sysklogd default

# Cron daemon
emerge --ask sys-process/cronie
rc-update add cronie default

# Filesystem tools (e2fsprogs for ext4)
emerge --ask sys-fs/e2fsprogs

# NTP time sync — chrony is modern and accurate
emerge --ask net-misc/chrony
rc-update add chronyd default
```

### 10.5 Configure SSH (Optional)

If you want SSH access:

```bash
rc-update add sshd default
```

---

## 11. System Tools & Final Steps

### 11.1 Add a Regular User

Running as root permanently is dangerous. Create a non-root user:

```bash
useradd -m -G users,wheel,audio,video,usb,portage -s /bin/bash yourusername
passwd yourusername
```

### 11.2 Install `sudo`

```bash
emerge --ask app-admin/sudo
```

Allow members of the `wheel` group to use `sudo`:

```bash
# Use visudo to safely edit sudoers
visudo
```

Uncomment the line:
```
%wheel ALL=(ALL) ALL
```

### 11.3 Install Useful Base Tools

```bash
emerge --ask \
  app-misc/tmux \
  app-editors/nano \
  net-misc/wget \
  app-arch/unzip \
  sys-apps/pciutils \
  sys-apps/usbutils \
  net-wireless/iw \
  net-wireless/wpa_supplicant
```

### 11.4 Rebuild the Kernel Modules (If Needed)

If you installed packages that need kernel modules (e.g. `wpa_supplicant`):

```bash
emerge --ask @module-rebuild
```

### 11.5 Verify the UEFI Boot Setup

Double-check the boot entries and the kernel image is in place:

```bash
efibootmgr -v
find /efi -name "*.efi" -o -name "vmlinuz*"
```

Verify the fstab will be correct on reboot:

```bash
cat /etc/fstab
blkid
```

Confirm the UUIDs in `/etc/fstab` match the output of `blkid`.

---

## 12. Reboot

### 12.1 Exit the Chroot and Unmount

```bash
# Exit chroot
exit

# Unmount in reverse order
umount -l /mnt/gentoo/dev
umount -l /mnt/gentoo/sys
umount /mnt/gentoo/proc
umount /mnt/gentoo/run
umount /mnt/gentoo/efi
umount /mnt/gentoo
```

### 12.2 Reboot

```bash
reboot
```

Remove the USB drive when the machine restarts. The UEFI firmware should boot Gentoo via the EFI stub entry created in Section 8.

### 12.3 First Boot Checklist

After rebooting into your new Gentoo system:

- [ ] Log in as root (or your regular user if SSH is configured)
- [ ] Verify networking: `ping -c 3 gentoo.org`
- [ ] Check running services: `rc-status`
- [ ] Confirm the correct kernel is running: `uname -r`
- [ ] Update the system: `emerge --sync && emerge --ask --update --deep --newuse @world`
- [ ] Check for any Portage news: `eselect news list && eselect news read`

---

## Appendix A: Troubleshooting

### System does not boot after installation

1. Boot back into the Gentoo live USB.
2. Re-mount everything and chroot back in:

   ```bash
   swapon /dev/nvme0n1p2
   mount /dev/nvme0n1p3 /mnt/gentoo
   mount /dev/nvme0n1p1 /mnt/gentoo/efi
   mount --types proc /proc /mnt/gentoo/proc
   mount --rbind /sys /mnt/gentoo/sys && mount --make-rslave /mnt/gentoo/sys
   mount --rbind /dev /mnt/gentoo/dev && mount --make-rslave /mnt/gentoo/dev
   mount --bind /run /mnt/gentoo/run && mount --make-slave /mnt/gentoo/run
   cp --dereference /etc/resolv.conf /mnt/gentoo/etc/
   chroot /mnt/gentoo /bin/bash
   source /etc/profile
   export PS1="(chroot) ${PS1}"
   mount /dev/nvme0n1p1 /efi
   ```

3. Check `efibootmgr -v` for your boot entry.
4. Run `find /efi -name "*.efi"` to verify the kernel file exists.
5. Re-run the `efibootmgr --create` command from Section 8.8 if the entry is missing.

### `efibootmgr` entry disappears after reboot

Some firmware (Dell, HP, Lenovo) deletes boot entries it doesn't recognize. Use the fallback approach from Section 8.10 — copy the kernel to `/efi/EFI/BOOT/BOOTX64.EFI`.

### Kernel panic: unable to mount root filesystem

The root UUID in the kernel command line is wrong. Boot from USB, chroot back in, run `blkid /dev/nvme0n1p3`, and re-run `efibootmgr --create` with the correct UUID.

### No network after first boot

Check your interface name (it may differ from `enp1s0`):

```bash
ip link
ls /etc/init.d/net.*
```

If the symlink does not exist for your interface name, create it:

```bash
cd /etc/init.d
ln -s net.lo net.<your-interface>
rc-update add net.<your-interface> default
rc-service net.<your-interface> start
```

---

## Appendix B: Quick Reference — OpenRC Service Management

| Task | Command |
|---|---|
| Start a service | `rc-service <name> start` |
| Stop a service | `rc-service <name> stop` |
| Restart a service | `rc-service <name> restart` |
| Enable on boot (default runlevel) | `rc-update add <name> default` |
| Disable on boot | `rc-update del <name> default` |
| List all services and status | `rc-status` |
| List services in a runlevel | `rc-update show` |

> There is no `systemctl` in an OpenRC system. All service management is done with `rc-service` and `rc-update`.

---

## Appendix C: Updating the Kernel

When a new `gentoo-kernel` version is available:

```bash
# Sync and upgrade
emerge --sync
emerge --ask --update --deep --newuse @world

# Or upgrade just the kernel
emerge --ask sys-kernel/gentoo-kernel

# Clean up old kernel versions
emerge --ask --depclean

# Update efibootmgr entry if the kernel filename changed
find /efi -name "*.efi"
# Re-run efibootmgr --create with the new filename if needed
```

---

*Guide written for Gentoo AMD64 with OpenRC, EFI Stub boot, and NVMe target disk. For the authoritative reference see the [Gentoo AMD64 Handbook](https://wiki.gentoo.org/wiki/Handbook:AMD64).*
