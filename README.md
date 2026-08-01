# Arch Linux + Hyprland Installation Guide

This guide walks you from a blank drive to a fully working **Arch Linux + Hyprland** desktop — no prior Arch experience needed.

> **New to Arch?** Unlike Ubuntu or Fedora, Arch doesn't hold your hand — there's no graphical installer, no desktop pre-installed, and no "click Next 5 times" setup. You install it piece by piece from a terminal, which sounds scary but is actually just typing one command at a time and reading what it tells you. This guide explains **what each command does and why**, not just what to type.

## How to read this guide

- 🔧 **Command blocks** are things you type into the terminal, exactly as shown (swap out placeholders like `sdX` for your actual drive name).
- 💡 **Tip** boxes give optional but useful advice.
- ⚠️ **Warning** boxes flag things that can break your system if rushed.
- Every step ends with a **Why?** explanation in plain English — read these even if you just copy-paste the commands.

## What you'll need

- A USB drive (4GB+) with the Arch Linux ISO already flashed onto it (using a tool like [Rufus](https://rufus.ie))
- A working internet connection (Wi-Fi or Ethernet)
- About an hour, and a willingness to read error messages instead of panicking at them

## A few words you'll see a lot

| Term | What it means |
|---|---|
| **Live USB / Live environment** | The temporary Arch system running off your USB drive, before anything is installed to the actual computer |
| **Partition** | A section of your hard drive, like dividing one drawer into two compartments |
| **Mount** | Making a partition accessible at a specific folder path, so you can read/write to it |
| **Chroot** | "Change root" — jumping from the temporary live USB environment *into* the system you're installing, as if you'd already booted into it |
| **Bootloader** | The program that runs first when you power on, and hands control to the Linux kernel (we're using GRUB) |
| **Compositor** | The program that draws windows on screen and manages your desktop (Hyprland, in our case) |
| **AUR** | The Arch User Repository — a community-run collection of build scripts for software that isn't in the official Arch repos |

Everything below assumes you've already booted from the Arch ISO and you're staring at a terminal prompt that looks like `root@archiso ~ #`.

## Table of Contents

**Part 1 — Installing Arch Linux** *(base system → bootable desktop)*

1. [Connect to the Internet](#1-connect-to-the-internet)
2. [Find Your Drive](#2-find-your-drive)
3. [Partition the Drive](#3-partition-the-drive)
4. [Format the Partitions](#4-format-the-partitions)
5. [Mount the Partitions](#5-mount-the-partitions)
6. [Install the Base System](#6-install-the-base-system)
7. [Install Essential Packages](#7-install-essential-packages)
8. [Generate the Filesystem Table (fstab)](#8-generate-the-filesystem-table-fstab)
9. [Enter Your New System (chroot)](#9-enter-your-new-system-chroot)
10. [Set Your Timezone](#10-set-your-timezone)
11. [Configure Your Locale](#11-configure-your-locale)
12. [Set Your Hostname](#12-set-your-hostname)
13. [Configure Hosts File](#13-configure-hosts-file)
14. [Set the Root Password](#14-set-the-root-password)
15. [Create Your Own User Account](#15-create-your-own-user-account)
16. [Enable `sudo` for Your User](#16-enable-sudo-for-your-user)
17. [Install the Bootloader (GRUB)](#17-install-the-bootloader-grub)
18. [Install Hyprland and Desktop Essentials](#18-install-hyprland-and-desktop-essentials)
19. [Enable Background Services](#19-enable-background-services)
20. [Finish Up](#20-finish-up)
21. [First Boot](#21-first-boot)

**Part 2 — Post-Installation Setup** *(bare desktop → your desktop, automated)*

22. [Run the Dotfiles Installer](#22-run-the-dotfiles-installer)

---

# Part 1 — Installing Arch Linux

## 1. Connect to the Internet

First, check whether you're already online:

```bash
ping archlinux.org
```

If you see replies coming back (lines with `bytes from`), you're connected — press `Ctrl+C` to stop the ping and skip to Step 2.

If you're on Wi-Fi and see nothing, connect manually with `iwctl` (Arch's built-in Wi-Fi tool):

```bash
iwctl
device list
station wlan0 scan
station wlan0 get-networks
station wlan0 connect "YOUR_WIFI_NAME"
exit
```

> 💡 **Tip:** `device list` shows your Wi-Fi adapter's name — it's usually `wlan0`, but double-check it matches what's listed before continuing.

**Why?** The installer doesn't come with anything pre-downloaded — every package (the base system, the kernel, your desktop) is fetched live from Arch's servers. No internet, no install.

---

## 2. Find Your Drive

```bash
lsblk
```

This lists every storage device attached to your computer, something like:

```text
sda        238.5G  ← an external or secondary drive
nvme0n1    476.9G  ← usually your main internal SSD
```

> ⚠️ **Warning:** Getting this wrong means installing Arch over the wrong drive and losing its data. Check the size (`238.5G`, `476.9G`, etc.) against what you know about your hardware before moving on. If unsure, unplug any drive you don't want touched.

**Why?** You need to tell the installer exactly which physical drive to use — it won't guess for you.

---

## 3. Partition the Drive

```bash
cfdisk /dev/sdX
```

Replace `sdX` with your actual drive name from Step 2 (e.g. `/dev/sda` or `/dev/nvme0n1`).

Inside `cfdisk`, create two partitions:

| Partition | Size | Type |
|---|---:|---|
| EFI System | 512 MB | EFI System |
| Linux Filesystem | Remaining space | Linux filesystem |

Use the arrow keys to navigate, `[New]` to create a partition, and set the **EFI System** type on the first one. When done, select **[Write]** → type `yes` to confirm → **[Quit]**.

**Why?** The EFI partition stores the small files your motherboard's firmware needs to find and start Linux. The second, much larger partition is where Arch itself and all your files will actually live.

---

## 4. Format the Partitions

```bash
mkfs.fat -F32 /dev/sdX1
mkfs.ext4 /dev/sdX2
```

`sdX1` is your EFI partition, `sdX2` is your Linux filesystem partition (match the numbers `cfdisk` showed you).

**Why?** Creating partitions just draws the boundaries — formatting is what actually lays down a filesystem (a way of organizing files) inside each one. FAT32 is required for the EFI partition by the UEFI standard; ext4 is Linux's reliable, general-purpose filesystem.

---

## 5. Mount the Partitions

```bash
mount /dev/sdX2 /mnt
mkdir /mnt/boot
mount /dev/sdX1 /mnt/boot
```

**Why?** "Mounting" tells the live environment: *"when I write to `/mnt`, actually write to this partition."* The installer (`pacstrap`, next step) only knows how to install to `/mnt` — mounting is what connects that folder to your real drive.

---

## 6. Install the Base System

```bash
pacstrap /mnt base linux linux-firmware
```

| Package | Purpose |
|---|---|
| `base` | The essential command-line tools every Arch system needs |
| `linux` | The Linux kernel — the core that talks to your hardware |
| `linux-firmware` | Extra firmware files many devices (Wi-Fi cards, GPUs) need to function |

**Why?** This is the actual "installation" step — everything before this was just preparing empty space. This one command downloads and copies a minimal, bootable Linux system onto your drive.

---

## 7. Install Essential Packages

```bash
pacstrap /mnt networkmanager grub efibootmgr sudo nvim zsh
```

| Package | Purpose |
|---|---|
| `networkmanager` | Handles Wi-Fi and Ethernet once you're no longer on the live USB |
| `grub` | The bootloader — what actually boots into your new system |
| `efibootmgr` | Registers GRUB with your motherboard's UEFI firmware |
| `sudo` | Lets your regular user run administrator-level commands when needed |
| `nvim` | A text editor for editing config files (you'll use this constantly) |
| `zsh` | The shell itself, installed now so it's already on disk by the time you create your user account and set it as their login shell in Step 15 |

**Why?** The base system from Step 6 is deliberately minimal — it can't even connect to Wi-Fi or boot on its own yet. These packages fill in those gaps.

---

## 8. Generate the Filesystem Table (fstab)

```bash
genfstab -U /mnt >> /mnt/etc/fstab
```

**Why?** `fstab` is a file that tells Linux which partition to mount where, every time it boots. `genfstab` writes this automatically based on how you mounted things in Step 5 — you'd otherwise have to write it by hand.

---

## 9. Enter Your New System (chroot)

```bash
arch-chroot /mnt
```

**Why?** So far you've been working *from* the live USB, treating `/mnt` as a subfolder. `arch-chroot` jumps you *inside* the system you just installed, so from here on, commands run as if you'd actually booted into your new Arch install — because for all practical purposes, you now have.

---

## 10. Set Your Timezone

```bash
ln -sf /usr/share/zoneinfo/Asia/Jakarta /etc/localtime
hwclock --systohc
```

Replace `Asia/Jakarta` with your own region — run `ls /usr/share/zoneinfo` to browse available options.

**Why?** This tells Linux what time zone to display clocks in, and syncs your hardware clock so the time stays correct after reboot.

---

## 11. Configure Your Locale

```bash
nvim /etc/locale.gen
```

Find the line `#en_US.UTF-8 UTF-8` and remove the `#` at the start to uncomment it. Save and quit (`:wq`).

Then:

```bash
locale-gen
echo "LANG=en_US.UTF-8" > /etc/locale.conf
```

**Why?** "Locale" controls language, character encoding, and formatting (dates, currency, etc.) system-wide. Uncommenting a locale in `locale.gen` tells Arch to actually generate that language pack; `locale.conf` tells the system which one to use by default.

---

## 12. Set Your Hostname

```bash
echo "arch" > /etc/hostname
```

**Why?** The hostname is your machine's name on a network — useful for identifying it, especially if you have more than one computer around.

---

## 13. Configure Hosts File

```bash
nvim /etc/hosts
```

Add these three lines:

```text
127.0.0.1 localhost
::1 localhost
127.0.1.1 arch.localdomain arch
```

**Why?** This lets your own machine resolve its own hostname locally, without needing to ask a DNS server — some programs expect this to work even with no internet.

---

## 14. Set the Root Password

```bash
passwd
```

You'll be prompted to type a password twice (it won't show on screen — that's normal).

**Why?** `root` is Arch's built-in administrator account. Right now it has no password at all, which is a security hole you're closing here.

---

## 15. Create Your Own User Account

```bash
useradd -m -G wheel -s /bin/zsh excello
passwd excello
```

Replace `excello` with whatever username you want.

**Why?** You shouldn't do your daily computing as `root` — one typo could wreck your whole system. `-m` creates a home folder for this user, `-G wheel` puts them in the `wheel` group (needed for admin permissions, next step), and `-s /bin/zsh` sets zsh as their shell right away, since Step 7 already installed it.

---

## 16. Enable `sudo` for Your User

```bash
EDITOR=nvim visudo
```

Find the line `# %wheel ALL=(ALL:ALL) ALL` and remove the `#`. Save and quit.

> ⚠️ **Warning:** Always edit this file with `visudo`, never a plain text editor — it checks your syntax before saving, so a typo can't lock you out of `sudo` entirely.

**Why?** This tells Linux "anyone in the `wheel` group can use `sudo`." Since your user is in `wheel` (Step 15), this is what lets you run admin commands like `sudo pacman -Syu` instead of switching to `root` every time.

---

## 17. Install the Bootloader (GRUB)

```bash
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB
grub-mkconfig -o /boot/grub/grub.cfg
```

**Why?** Installing the `grub` package (Step 7) only put the program on disk — these two commands actually register it with your UEFI firmware and generate its boot menu configuration. Without this, your computer has no idea how to start Linux at all.

---

## 18. Install Hyprland and Desktop Essentials

```bash
pacman -S hyprland hyprpaper hyprpolkitagent xdg-desktop-portal-hyprland sddm pipewire pipewire-pulse wireplumber kitty zsh-syntax-highlighting zsh-autosuggestions starship waybar swaync wofi firefox ttf-jetbrains-mono-nerd grim wl-clipboard nvim yazi fzf bat zoxide eza fastfetch
```

Grouped by what each thing does:

**Desktop itself**
| Package | Purpose |
|---|---|
| `hyprland` | The Wayland compositor — this *is* your desktop environment |
| `hyprpaper` | Sets your wallpaper |
| `hyprpolkitagent` | Handles password prompts for admin actions triggered from GUI apps |
| `xdg-desktop-portal-hyprland` | Lets Wayland-aware screen-sharing (Discord, OBS) and native file-picker dialogs work correctly; pulls in the base `xdg-desktop-portal` framework as a dependency |

**Login screen**
| Package | Purpose |
|---|---|
| `sddm` | The graphical login screen you'll see on every boot |

**Audio**
| Package | Purpose |
|---|---|
| `pipewire` | Modern Linux audio server |
| `pipewire-pulse` | Compatibility layer so apps expecting PulseAudio still work |
| `wireplumber` | Manages audio devices and routing behind the scenes |

**Terminal & shell**
| Package | Purpose |
|---|---|
| `kitty` | A fast, GPU-accelerated terminal emulator |
| `zsh-syntax-highlighting` | Colors commands as valid/invalid while typing |
| `zsh-autosuggestions` | Ghost-text command suggestions as you type |
| `starship` | A customizable, informative shell prompt |

**Bar**
| Package | Purpose |
|---|---|
| `waybar` | The status bar along the top/bottom of your screen |

**Notifications & launcher**
| Package | Purpose |
|---|---|
| `swaync` | Notification daemon and notification center — pops up notifications and gives you a panel to review/clear them |
| `wofi` | An application launcher and general-purpose menu (for launching apps, and for menus other tools can pipe into) |

**Browser**
| Package | Purpose |
|---|---|
| `firefox` | A full-featured web browser — nothing web-related is installed by default, so this is what you'll actually use to get online once you're past the terminal |

**Fonts**
| Package | Purpose |
|---|---|
| `ttf-jetbrains-mono-nerd` | A coding font patched with icons (needed for waybar/kitty icons to render, and for `eza`'s icons) |

**Screenshots & clipboard**
| Package | Purpose |
|---|---|
| `grim` | Takes screenshots from the command line — the Wayland equivalent of a screenshot tool |
| `wl-clipboard` | Command-line clipboard access (`wl-copy` / `wl-paste`) — Wayland's equivalent of `xclip` |

**CLI tools**
| Package | Purpose |
|---|---|
| `nvim` | Text editor |
| `yazi` | Terminal file manager |
| `fzf` | Fuzzy finder for files, history, and more |
| `bat` | A nicer `cat`, with syntax highlighting |
| `zoxide` | A smarter `cd` that learns your most-used directories and jumps to them by typing just part of the name |
| `eza` | A modern replacement for `ls`, with colors, icons, and a built-in tree view |
| `fastfetch` | Prints a system-info summary with ASCII art |

> 💡 **Tip:** This one command installs everything at once. If it fails partway through (e.g. a mirror timing out), just re-run the same command — `pacman` skips anything already installed.

**Why this order?** Each layer depends on the one above it conceptually: the compositor needs to exist before a login manager can launch it, audio needs to exist before a terminal cares about it, and fonts need to be present before the bar/terminal can render icons correctly.

---

## 19. Enable Background Services

```bash
systemctl enable NetworkManager
systemctl enable sddm
```

**Why?** Installing a package doesn't automatically make it start on boot — `systemctl enable` is what schedules a service to launch every time you power on. Without this, you'd have no networking and no login screen after rebooting.

---

## 20. Finish Up

```bash
exit
umount -R /mnt
poweroff
```

Once it powers off, physically remove your USB drive.

**Why?** `exit` leaves the chroot environment, `umount -R /mnt` safely detaches your new system's partitions (skipping this can corrupt files), and `poweroff` shuts the machine down cleanly so you can boot from the actual drive next.

---

## 21. First Boot

```text
Power Button → UEFI → GRUB → SDDM → Hyprland → Desktop
```

If you land on the SDDM login screen, log in with the user you created in Step 15. You should land on a mostly-empty Hyprland desktop — that's expected.

---

# Part 2 — Post-Installation Setup

## 22. Run the Dotfiles Installer

These are my personal dotfiles, and I'd recommend them for getting the rest of your desktop set up — wallpaper, waybar, wofi, kitty theme, keybinds, shell config, AUR helper, all of it. Open a terminal on your fresh desktop and run:

```bash
bash <(curl -fsSL https://raw.githubusercontent.com/excellogalihs/friedrice/main/install.sh)
```

---

That's the whole setup, start to finish: a booted, working Arch install with Hyprland, ready to be finished off by the installer script above.
