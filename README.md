# Arch Linux + Hyprland Install Guide

Blank drive to a working Arch + Hyprland desktop. No prior Arch experience needed, just patience.

> Coming from Ubuntu or Fedora? Arch skips the graphical installer entirely — no "click Next 5 times." You're typing commands one at a time and reading what they spit back at you. Sounds intimidating, isn't really. This guide tells you what each command actually does, not just what to paste.

## Reading this guide

- Command blocks — type these exactly, swapping placeholders like `sdX` for your real drive.
- Tips — optional but worth doing.
- Warnings — skip these at your own risk.
- Each step has a short explanation of *why* you're doing it. Read those even if you're just copy-pasting.

## Before you start

- USB drive (4GB+) flashed with the Arch ISO ([Rufus](https://rufus.ie) works fine)
- Internet connection
- About an hour, and enough patience to read an error message instead of panicking at it

## Vocabulary you'll run into

| Term | Meaning |
|---|---|
| **Live USB** | The temporary Arch environment running off your USB, before anything's installed |
| **Partition** | A section of your drive — think dividing one drawer into compartments |
| **Mount** | Pointing a folder path at a partition so you can read/write to it |
| **Chroot** | Jumping from the live USB into the system you're installing, as if you'd already booted into it |
| **Bootloader** | The thing that runs first on power-on and hands off to the kernel (GRUB, here) |
| **Compositor** | Draws your windows and runs your desktop — that's Hyprland |
| **AUR** | Arch User Repository — community build scripts for stuff not in the official repos |

Everything from here assumes you've booted the Arch ISO and you're looking at `root@archiso ~ #`.

## Contents

**Part 1 — Installing Arch** *(bare drive → bootable desktop)*

1. [Connect to the Internet](#1-connect-to-the-internet)
2. [Find Your Drive](#2-find-your-drive)
3. [Partition the Drive](#3-partition-the-drive)
4. [Format the Partitions](#4-format-the-partitions)
5. [Mount the Partitions](#5-mount-the-partitions)
6. [Install the Base System](#6-install-the-base-system)
7. [Install Essential Packages](#7-install-essential-packages)
8. [Generate fstab](#8-generate-fstab)
9. [Chroot In](#9-chroot-in)
10. [Set Your Timezone](#10-set-your-timezone)
11. [Configure Your Locale](#11-configure-your-locale)
12. [Set Your Hostname](#12-set-your-hostname)
13. [Configure Hosts](#13-configure-hosts)
14. [Set the Root Password](#14-set-the-root-password)
15. [Create Your User](#15-create-your-user)
16. [Enable sudo](#16-enable-sudo)
17. [Install GRUB](#17-install-grub)
18. [Install Hyprland + Desktop Essentials](#18-install-hyprland--desktop-essentials)
19. [Enable Services](#19-enable-services)
20. [Wrap Up](#20-wrap-up)
21. [First Boot](#21-first-boot)

**Part 2 — Everything Else**

22. [Run the Dotfiles Installer](#22-run-the-dotfiles-installer)

---

# Part 1 — Installing Arch

## 1. Connect to the Internet

Check if you're already online:

```bash
ping archlinux.org
```

Getting replies back? Ctrl+C and skip to Step 2. Nothing coming through on Wi-Fi? Connect manually with `iwctl`:

```bash
iwctl
device list
station wlan0 scan
station wlan0 get-networks
station wlan0 connect "YOUR_WIFI_NAME"
exit
```

`device list` shows your Wi-Fi adapter's name — usually `wlan0`, but check before you proceed.

Nothing in this installer is pre-downloaded. Every package gets pulled from Arch's servers live, so without a connection you're stuck here.

---

## 2. Find Your Drive

```bash
lsblk
```

You'll get something like:

```text
sda        238.5G  ← external or secondary drive
nvme0n1    476.9G  ← usually your main SSD
```

Get this wrong and you're wiping the wrong drive. Match the size against what you actually know about your hardware, and unplug anything you don't want touched if you're not sure.

The installer needs to know exactly which physical drive to target — it's not going to guess.

---

## 3. Partition the Drive

```bash
cfdisk /dev/sdX
```

Swap in your drive from Step 2 (`/dev/sda`, `/dev/nvme0n1`, whatever it was).

Create two partitions:

| Partition | Size | Type |
|---|---:|---|
| EFI System | 512 MB | EFI System |
| Linux Filesystem | rest of the drive | Linux filesystem |

Arrow keys to move, `[New]` to create, set the first one's type to EFI System. `[Write]` → `yes` → `[Quit]` when done.

The EFI partition holds the files your firmware needs to find and boot Linux. The bigger partition is where Arch and all your actual files live.

---

## 4. Format the Partitions

```bash
mkfs.fat -F32 /dev/sdX1
mkfs.ext4 /dev/sdX2
```

`sdX1` = EFI partition, `sdX2` = Linux partition (match whatever numbers `cfdisk` showed you).

Partitioning just draws the lines — formatting lays down an actual filesystem inside them. FAT32's required for EFI by the UEFI spec; ext4's just Linux's reliable go-to.

---

## 5. Mount the Partitions

```bash
mount /dev/sdX2 /mnt
mkdir /mnt/boot
mount /dev/sdX1 /mnt/boot
```

This tells the live environment "writes to `/mnt` go to this partition." `pacstrap` (next step) only knows how to install to `/mnt`, so this is what actually connects it to your real drive.

---

## 6. Install the Base System

```bash
pacstrap /mnt base linux linux-firmware
```

| Package | What it's for |
|---|---|
| `base` | Core command-line tools every Arch box needs |
| `linux` | The kernel |
| `linux-firmware` | Firmware blobs for Wi-Fi cards, GPUs, etc. |

This is the actual install — everything before was just prep. One command, and you've got a minimal bootable Linux system on disk.

---

## 7. Install Essential Packages

```bash
pacstrap /mnt networkmanager grub efibootmgr sudo nvim zsh
```

| Package | What it's for |
|---|---|
| `networkmanager` | Wi-Fi/Ethernet once you're off the live USB |
| `grub` | The bootloader |
| `efibootmgr` | Registers GRUB with UEFI |
| `sudo` | Admin commands without full root |
| `nvim` | You'll be editing configs a lot |
| `zsh` | Installed now so it's ready when you set it as your login shell in Step 15 |

The base system is deliberately bare — it can't connect to Wi-Fi or even boot yet. This fills the gaps.

---

## 8. Generate fstab

```bash
genfstab -U /mnt >> /mnt/etc/fstab
```

`fstab` tells Linux which partition mounts where on every boot. `genfstab` writes it automatically from how you mounted things in Step 5, instead of you doing it by hand.

---

## 9. Chroot In

```bash
arch-chroot /mnt
```

Up to now you've been working *from* the live USB, treating `/mnt` as a subfolder. This jumps you inside the system you just installed — commands from here run as if you'd actually booted into it.

---

## 10. Set Your Timezone

```bash
ln -sf /usr/share/zoneinfo/Asia/Jakarta /etc/localtime
hwclock --systohc
```

Swap `Asia/Jakarta` for your own region (`ls /usr/share/zoneinfo` to browse).

Sets what timezone your clock displays, and syncs the hardware clock so time survives a reboot.

---

## 11. Configure Your Locale

```bash
nvim /etc/locale.gen
```

Find `#en_US.UTF-8 UTF-8`, strip the `#`, save and quit (`:wq`).

```bash
locale-gen
echo "LANG=en_US.UTF-8" > /etc/locale.conf
```

Locale controls language, encoding, and formatting system-wide. Uncommenting in `locale.gen` tells Arch to actually build that language pack; `locale.conf` sets it as default.

---

## 12. Set Your Hostname

```bash
echo "arch" > /etc/hostname
```

Your machine's name on a network — mostly useful once you've got more than one box around.

---

## 13. Configure Hosts

```bash
nvim /etc/hosts
```

```text
127.0.0.1 localhost
::1 localhost
127.0.1.1 arch.localdomain arch
```

Lets your machine resolve its own hostname locally without a DNS server — some programs expect this even offline.

---

## 14. Set the Root Password

```bash
passwd
```

Type it twice, nothing shows on screen (that's normal, not broken).

Right now root has no password at all — this closes that hole.

---

## 15. Create Your User

```bash
useradd -m -G wheel -s /bin/zsh excello
passwd excello
```

Swap `excello` for whatever username you want.

Don't daily-drive as root — one typo and you've wrecked something. `-m` makes a home folder, `-G wheel` adds you to the group that'll get sudo access next step, `-s /bin/zsh` sets your shell since it's already installed.

---

## 16. Enable sudo

```bash
EDITOR=nvim visudo
```

Find `# %wheel ALL=(ALL:ALL) ALL`, uncomment it, save and quit.

Always use `visudo` here, never a plain editor — it validates syntax before saving so a typo can't lock you out of sudo entirely.

This is what lets `wheel` group members run sudo instead of switching to root every time.

---

## 17. Install GRUB

```bash
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB
grub-mkconfig -o /boot/grub/grub.cfg
```

Installing the package in Step 7 only put GRUB on disk — this actually registers it with your firmware and builds the boot menu. Skip it and your machine has no idea how to start Linux.

---

## 18. Install Hyprland + Desktop Essentials

```bash
pacman -S hyprland hyprpaper hyprpolkitagent xdg-desktop-portal-hyprland sddm pipewire pipewire-pulse wireplumber kitty zsh-syntax-highlighting zsh-autosuggestions starship waybar swaync wofi firefox ttf-jetbrains-mono-nerd grim wl-clipboard nvim yazi fzf bat zoxide eza fastfetch
```

**Desktop**
| Package | What it's for |
|---|---|
| `hyprland` | The compositor — this is the desktop |
| `hyprpaper` | Wallpaper |
| `hyprpolkitagent` | Password prompts from GUI apps |
| `xdg-desktop-portal-hyprland` | Screen-sharing and native file pickers on Wayland; pulls in `xdg-desktop-portal` |

**Login**
| Package | What it's for |
|---|---|
| `sddm` | Graphical login screen |

**Audio**
| Package | What it's for |
|---|---|
| `pipewire` | Audio server |
| `pipewire-pulse` | PulseAudio compatibility |
| `wireplumber` | Device management and routing |

**Terminal & shell**
| Package | What it's for |
|---|---|
| `kitty` | GPU-accelerated terminal |
| `zsh-syntax-highlighting` | Live command validity coloring |
| `zsh-autosuggestions` | Ghost-text history suggestions |
| `starship` | Shell prompt |

**Bar**
| Package | What it's for |
|---|---|
| `waybar` | Status bar |

**Notifications & launcher**
| Package | What it's for |
|---|---|
| `swaync` | Notification daemon + center |
| `wofi` | App launcher / menu |

**Browser**
| Package | What it's for |
|---|---|
| `firefox` | Nothing web-related ships by default, this is how you get online |

**Fonts**
| Package | What it's for |
|---|---|
| `ttf-jetbrains-mono-nerd` | Icon font for waybar, kitty, eza |

**Screenshots & clipboard**
| Package | What it's for |
|---|---|
| `grim` | Command-line screenshots |
| `wl-clipboard` | Wayland clipboard access |

**CLI tools**
| Package | What it's for |
|---|---|
| `nvim` | Editor |
| `yazi` | Terminal file manager |
| `fzf` | Fuzzy finder |
| `bat` | `cat` with syntax highlighting |
| `zoxide` | Smarter `cd` |
| `eza` | Modern `ls` |
| `fastfetch` | System-info banner |

One command installs everything. If a mirror times out partway through, just rerun it — `pacman` skips what's already there.

---

## 19. Enable Services

```bash
systemctl enable NetworkManager
systemctl enable sddm
```

Installing a package doesn't start it on boot — this schedules both to launch every time you power on. Skip it, no networking and no login screen after reboot.

---

## 20. Wrap Up

```bash
exit
umount -R /mnt
poweroff
```

Pull the USB once it powers off.

`exit` leaves the chroot, `umount -R /mnt` detaches the partitions cleanly (skip it and risk corruption), `poweroff` shuts down so you can boot the real drive next.

---

## 21. First Boot

```text
Power Button → UEFI → GRUB → SDDM → Hyprland → Desktop
```

Log in with the user from Step 15. You'll land on a mostly-empty Hyprland desktop — that's expected, that's the point of Part 2.

---

# Part 2 — Everything Else

## 22. Run the Dotfiles Installer

These are my own dotfiles, and I'd genuinely recommend them for the rest of the setup — wallpaper, waybar, wofi, kitty theme, keybinds, shell config, AUR helper, all of it. Fire up a terminal on your fresh desktop and run:

```bash
bash <(curl -fsSL https://raw.githubusercontent.com/excellogalihs/friedrice/main/install.sh)
```

---

That's it — a booted, working Arch + Hyprland install, ready for the installer script above to finish the job.
