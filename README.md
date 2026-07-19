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
pacman -S hyprland hyprpaper hyprpolkitagent xdg-desktop-portal-hyprland sddm pipewire pipewire-pulse wireplumber kitty zsh-autosuggestions zsh-syntax-highlighting starship waybar network-manager-applet ttf-jetbrains-mono-nerd nvim yazi fzf bat fastfetch swaync wofi grim wl-clipboard
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
| `zsh-autosuggestions` | Ghost-text command suggestions as you type |
| `zsh-syntax-highlighting` | Colors commands as valid/invalid while typing |
| `starship` | A customizable, informative shell prompt |

**Bar & tray**
| Package | Purpose |
|---|---|
| `waybar` | The status bar along the top/bottom of your screen |
| `network-manager-applet` | A Wi-Fi icon/menu in your tray |

**Notifications & launcher**
| Package | Purpose |
|---|---|
| `swaync` | Notification daemon and notification center — pops up notifications and gives you a panel to review/clear them |
| `wofi` | An application launcher and general-purpose menu (for launching apps, and for menus other tools can pipe into) |

**Fonts**
| Package | Purpose |
|---|---|
| `ttf-jetbrains-mono-nerd` | A coding font patched with icons (needed for waybar/kitty icons to render) |

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
| `fastfetch` | Prints a system-info summary with ASCII art |

> 💡 **Tip:** This one command installs everything at once. If it fails partway through (e.g. a mirror timing out), just re-run the same command — `pacman` skips anything already installed.

**Why this order?** Each layer depends on the one above it conceptually: the compositor needs to exist before a login manager can launch it, audio needs to exist before a terminal cares about it, and fonts need to be present before the bar/terminal can render icons correctly. Installing in this order isn't strictly required by `pacman`, but it mirrors the order you'll actually configure things in Part 2.

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

If you land on the SDDM login screen, log in with the user you created in Step 15. You should land on a mostly-empty Hyprland desktop — that's expected. Part 2 turns it from "working" into "yours."

---

# Part 2 — Post-Installation Configuration

You're on the desktop now. Everything below is about making things look and behave the way you want, one piece at a time.

> 💡 **A recurring pattern:** most of these programs read their settings from a plain text file under `~/.config/`. You'll open that file in `nvim`, change something, save, and often the change applies immediately (or after a quick reload command) — no restarting the whole desktop needed. That's why this part starts with `nvim` basics — you're about to live inside it.

---

## 22. Neovim Basics

`nvim` has a few different "modes" — this trips up a lot of first-timers coming from Notepad-style editors, so here's the short version:

| Mode | How to enter it | What it's for |
|---|---|---|
| **Normal** | Press `Esc` | The default mode — moving around and running commands, *not* typing |
| **Insert** | Press `i` | Actually typing text |
| **Visual** | Press `v` | Selecting text |
| **Command** | Press `:` | Typing commands like save or quit |

Basic movement (while in Normal mode): `h` `j` `k` `l` move left/down/up/right, `w` jumps forward a word, `gg` jumps to the top of the file, `G` jumps to the bottom.

The commands you'll use constantly:

```vim
:w        " save the file
:q        " quit (fails if you have unsaved changes)
:wq       " save and quit together
:q!       " quit and throw away unsaved changes
/searchterm    " search forward for text, press n for next match
```

> 💡 **Tip:** If you ever get stuck and don't know what mode you're in, just press `Esc` a couple of times — that always gets you back to Normal mode safely.

Its own config lives at `~/.config/nvim/init.lua`. You'll be using `nvim` for nearly every config file in the rest of this guide — including Hyprland's own config, `~/.config/hypr/hyprland.lua` (up next), which happens to be Lua too, so these same editing skills carry straight over.

---

## 23. Get to Know `hyprland.lua`

Hyprland's config file lives at `~/.config/hypr/hyprland.lua` and is written in **Lua** (a small, readable scripting language), not a special Hyprland-only format.

Open it:

```bash
nvim ~/.config/hypr/hyprland.lua
```

At the very top you'll see a line like this:

```lua
hl.config({ autogenerated = true })
```

You can delete that line of code — it's basically useless, not something functional.

**Why?** Hyprland writes this file out fresh the first time it runs, with the yellow autogenerated warning as a nudge to go customize it. Once you've started editing (which you're doing right now), it no longer serves a purpose.

> 💡 **Get familiar with the layout before you start:** the rest of the file is broken into clearly labeled sections, each marked with a comment header like `---- AUTOSTART ----` or `---- KEYBINDINGS ----`. From here on, whenever this guide says "add this to `hyprland.lua`," it means *inside the matching section* — not just anywhere in the file. The sections you'll touch most in this guide:
> - **AUTOSTART** — the `hl.on("hyprland.start", function() ... end)` block, where programs like `waybar`, `hyprpaper`, `swaync`, and `hyprpolkitagent` get launched on login
> - **KEYBINDINGS** — where `local mainMod = "SUPER"` is defined, followed by a long list of `hl.bind(...)` calls
> - **PERMISSIONS** — a block of commented-out `hl.permission(...)` lines, including ones for `grim` and `xdg-desktop-portal-hyprland` specifically

> 💡 **Tip:** As this file grows, you can split it into smaller files and pull them in with Lua's `require()`:
> ```lua
> require("modules.keybinds")  -- loads ~/.config/hypr/modules/keybinds.lua
> ```
> Each file loads in its own little sandbox, so a mistake in one won't break the rest.

---

## 24. Install an AUR Helper (yay)

Everything you've installed so far came from the **official Arch repositories** via `pacman`. But a huge amount of Linux software — browsers with extra patches, proprietary apps, niche CLI tools, in-development packages — lives instead in the **AUR (Arch User Repository)**: a community-maintained collection of build recipes (`PKGBUILD` files), not pre-built binaries. `pacman` doesn't know how to touch the AUR at all — you need a separate tool called an **AUR helper** to download, build, and install from it as easily as you use `pacman`.

First, install the tools needed to actually *build* software from source:

```bash
sudo pacman -S --needed base-devel git
```

Then clone and build `yay` itself (yay is an AUR package too, so it's installed "by hand" this one time):

```bash
git clone https://aur.archlinux.org/yay.git
cd yay
makepkg -si
cd ..
rm -rf yay
```

| Package/Command | Purpose |
|---|---|
| `base-devel` | A package group bundling `gcc`, `make`, `patch`, and other tools needed to compile source code |
| `git` | Needed to download (`clone`) both `yay`'s source and, later, any AUR package's source |
| `makepkg -si` | Builds the package from its `PKGBUILD` (`-s` also grabs and installs any missing dependencies first) and `-i` installs the resulting package once it's built |

Once installed, using `yay` feels just like `pacman`, except it searches both the official repos *and* the AUR:

```bash
yay -S somepackage     # install a package (official or AUR)
yay -Syu                # update everything, official repos + AUR, in one go
yay -Ss keyword          # search by name/description
```

> ⚠️ **Warning:** AUR packages are user-submitted build scripts, not vetted by Arch the way official packages are. `makepkg`/`yay` will always show you the `PKGBUILD` before building — actually skim it, especially for less popular packages, since it's just a script that runs with your permissions.

### `yay` vs `paru` — which one should you use?

Both are AUR helpers that do the same core job (search, download, build, install, and keep AUR + official packages updated together), and both were originally written by the same author, so day-to-day usage feels nearly identical. The differences are mostly under the hood:

| | `yay` | `paru` |
|---|---|---|
| **Written in** | Go | Rust |
| **Age / ecosystem** | Older, the long-time default in most Arch guides and tutorials | Newer "spiritual successor," written after the author moved on from `yay` |
| **PKGBUILD review** | Shows diffs on update; review step is easy to skim past | Defaults to a more insistent interactive review before building, which some find safer, others find naggy |
| **Config style** | Command-line flags, a `~/.config/yay/config.json` | Similar flags, plus a `~/.config/paru/paru.conf` with a few extra knobs (e.g. batch installs, dedicated cleanup commands) |
| **Maintenance** | Stable, feature-complete, in wide use | Actively developed, tends to pick up newer conveniences first |

**Why does this matter?** Neither is objectively "better" — they solve the exact same problem and can even be swapped later without side effects, since both just wrap `makepkg`. This guide uses `yay` because it's what most beginner tutorials and community dotfiles reference, so troubleshooting help is easier to find. If you'd rather use `paru` instead, the install steps are identical, just swap the repository URL:

```bash
git clone https://aur.archlinux.org/paru.git
cd paru
makepkg -si
cd ..
rm -rf paru
```

---

## 25. Turn On `hyprpolkitagent`

This one has no config file — it just needs to be *running*. It's what lets password prompts actually appear when a graphical app asks for admin permission (installing something with a GUI, mounting a drive, etc).

Find the **AUTOSTART** section — look for the comment header `---- AUTOSTART ----` followed by a block like this:

```lua
hl.on("hyprland.start", function()
	hl.exec_cmd("systemctl --user start hyprpolkitagent")
end)
```

If it's not there already, add it. This is the block that runs once, right when Hyprland starts — anything you want launched automatically on login goes inside this `function() ... end`, as a line calling `hl.exec_cmd(...)`.

Check it's running:

```bash
pgrep -a hyprpolkitagent
```

If that prints a line with a process ID, it's working.

---

## 26. Turn On `swaync` (Notifications)

`swaync` ("sway notification center") is what actually shows notification pop-ups on screen and gives you a panel to review or clear them — without it, apps can *try* to send notifications, but nothing will ever appear.

Like `hyprpolkitagent`, it has no config file you're required to touch — it just needs to be running. Go back to the same **AUTOSTART** block from Step 25 and add a second `hl.exec_cmd(...)` line for `swaync`:

```lua
hl.on("hyprland.start", function()
	hl.exec_cmd("systemctl --user start hyprpolkitagent")
	hl.exec_cmd("swaync")
end)
```

You'll keep adding to that second line as you go — `hyprpaper`, `waybar`, and `nm-applet` all join it in the next few steps, the same way `&` chains them into one shell command that fires everything off together.

Check it's running:

```bash
pgrep -a swaync
```

If that prints a line with a process ID, it's working — try triggering a test notification:

```bash
notify-send "Hello" "This is a test notification"
```

> 💡 **Tip:** If you want to customize how notifications and the panel look (colors, size, position), `swaync` reads from `~/.config/swaync/style.css` and `~/.config/swaync/config.json` — copy the defaults first so you have something to start from:
> ```bash
> mkdir -p ~/.config/swaync
> cp /etc/xdg/swaync/config.json ~/.config/swaync/config.json
> cp /etc/xdg/swaync/style.css ~/.config/swaync/style.css
> ```

**Why turn this on before waybar?** Step 28 (waybar) adds a bell-icon module that clicks through to `swaync` — that module is just a thin wrapper around `swaync-client` and does nothing until `swaync` itself is actually running. Getting it turned on now means it's ready by the time you wire it into the bar.

---

## 27. Set a Wallpaper with Hyprpaper

Hyprpaper's config lives at `~/.config/hypr/hyprpaper.conf`.

```bash
nvim ~/.config/hypr/hyprpaper.conf
```

Add:

```ini
wallpaper {
    monitor = DP-3
    path = ~/myFile.jxl
    fit_mode = cover
}
```

| Field | Meaning |
|---|---|
| `monitor` | Which screen this applies to. Run `hyprctl monitors` in a terminal to see your monitor's real name (e.g. `eDP-1` for a laptop screen). Leave it blank for a fallback that covers any monitor without a specific entry. |
| `path` | The image file to use. |
| `fit_mode` | How the image fills the screen: `cover` (fill and crop), `contain` (fit without cropping), `tile`, or `fill` (stretch). Defaults to `cover`. |

Got two monitors? Just add a second block:

```ini
wallpaper {
    monitor = DP-3
    path = ~/Pictures/wall1.jxl
    fit_mode = cover
}

wallpaper {
    monitor = DP-2
    path = ~/Pictures/wall2.jxl
    fit_mode = cover
}
```

To make hyprpaper start automatically every login, go back to the **AUTOSTART** block (Step 25/26) and add `hyprpaper` into the chained `hl.exec_cmd(...)` line:

```lua
hl.on("hyprland.start", function()
	hl.exec_cmd("systemctl --user start hyprpolkitagent")
	hl.exec_cmd("hyprpaper & swaync")
end)
```

If you change the wallpaper config later without restarting anything:

```bash
hyprctl hyprpaper reload
```

---

## 28. A Minimal Waybar Setup That Actually Works

Waybar reads two files: `~/.config/waybar/config.jsonc` (what to show) and `~/.config/waybar/style.css` (how it looks).

```bash
mkdir -p ~/.config/waybar
nvim ~/.config/waybar/config.jsonc
```

```jsonc
{
    "layer": "top",
    "modules-left": ["hyprland/workspaces"],
    "modules-center": ["clock"],
    "modules-right": ["custom/notification", "pulseaudio", "network", "cpu", "memory", "tray"],
    "battery": {
        "format": "{capacity}% {icon}",
        "format-icons": ["", "", "", "", ""]
    },
    "clock": {
        "format": "󰥔 {:%a, %d. %b  %H:%M}"
    },
    "cpu": {
        "format": " {}%"
    },
    "hyprland/workspaces": {
        "persistent-workspaces": {
            "*": [1, 2, 3, 4, 5]
        }
    },
    "memory": {
        "format": " {}%"
    },
    "network": {
        "format": "  {essid}",
        "tooltip": false
    },
    "custom/notification": {
        "tooltip": false,
        "format": "{icon}",
        "format-icons": {
            "notification": "<span foreground='red'><sup></sup></span>",
            "none": "",
            "dnd-notification": "<span foreground='red'><sup></sup></span>",
            "dnd-none": ""
        },
        "return-type": "json",
        "exec-if": "which swaync-client",
        "exec": "swaync-client -swb",
        "on-click": "swaync-client -t -sw",
        "on-click-right": "swaync-client -d -sw",
        "escape": true
    },
    "pulseaudio": {
        "format": "{icon} {volume}%",
        "format-icons": {
            "default": ["", " ", " "]
        },
        "scroll-step": 2
    }
}
```

Then the styling:

```bash
nvim ~/.config/waybar/style.css
```

```css
* {
    font-family: "JetBrainsMono Nerd Font";
    font-size: 13px;
    min-height: 0;
}

window#waybar {
    background-color: rgba(20, 20, 30, 0.85);
    color: #cdd6f4;
}

#workspaces button {
    padding: 0 8px;
    color: #cdd6f4;
}

#workspaces button.active {
    background-color: #89b4fa;
    color: #1e1e2e;
    border-radius: 6px;
}

#clock, #network, #pulseaudio, #battery, #tray, #custom-notification {
    padding: 0 10px;
}
```

Make it launch automatically by adding `waybar` into the same chained line in the **AUTOSTART** block:

```lua
hl.on("hyprland.start", function()
	hl.exec_cmd("systemctl --user start hyprpolkitagent")
	hl.exec_cmd("waybar & hyprpaper & swaync")
end)
```

**Why laid out this way?** `modules-left/center/right` are the bar's three zones. Workspaces go on the left (near your window list), the clock sits center (glanceable at a distance), and system status — notifications, network, volume, battery, tray icons — groups on the right, matching the layout most desktop bars use. The `custom/notification` module is just a clickable bell icon that talks to `swaync` (already turned on in Step 26) via `swaync-client` — waybar itself has no idea what a notification even is, it's only forwarding clicks. The `tray` module is included in `modules-right` because Step 33 hooks the Wi-Fi tray icon into it.

---

## 29. Style Wofi (App Launcher)

`wofi` is your app launcher — press a keybind, start typing an app's name, hit `Enter` to open it. Out of the box it works but looks like a plain grey box; this step gives it the same dark/accent look as your waybar.

First, make sure it's actually bound to a key. Find the **KEYBINDINGS** section — look for the comment header `---- KEYBINDINGS ----`, right after which you'll see `local mainMod = "SUPER"` followed by a long list of `hl.bind(...)` calls. Check whether this line is already there (Hyprland's default template ships with a launcher bind as an example):

```lua
hl.bind(mainMod .. " + D", hl.dsp.exec_cmd("wofi --show drun"))
```

If it's missing, add it into that same list, below `local mainMod = "SUPER"`.

`--show drun` tells wofi to act specifically as an application launcher (reading `.desktop` files), rather than one of its other modes (like a run-command prompt or window switcher).

Now style it. Wofi reads from two files, same pattern as waybar:

```bash
mkdir -p ~/.config/wofi
nvim ~/.config/wofi/config
```

```ini
width=600
height=400
location=center
show=drun
prompt=Search...
filter_rate=100
allow_markup=true
no_actions=true
halign=fill
orientation=vertical
content_halign=fill
insensitive=true
allow_images=true
image_size=32
gtk_dark=true
```

Then the CSS:

```bash
nvim ~/.config/wofi/style.css
```

```css
window {
    font-family: "JetBrainsMono Nerd Font";
    font-size: 14px;
    background-color: rgba(20, 20, 30, 0.95);
    border-radius: 12px;
    border: 2px solid #89b4fa;
}

#input {
    margin: 10px;
    padding: 8px;
    border-radius: 8px;
    background-color: #1e1e2e;
    color: #cdd6f4;
    border: none;
}

#inner-box {
    margin: 5px;
    padding: 5px;
    background-color: transparent;
}

#outer-box {
    margin: 5px;
    padding: 10px;
    background-color: transparent;
}

#entry {
    padding: 8px;
    border-radius: 8px;
}

#entry:selected {
    background-color: #89b4fa;
}

#entry:selected #text {
    color: #1e1e2e;
}

#text {
    color: #cdd6f4;
}
```

| Setting | What it does |
|---|---|
| `width` / `height` | The size of the launcher box in pixels |
| `location=center` | Pops up centered on screen instead of a corner |
| `show=drun` | Makes app-launcher mode the default if you ever run bare `wofi` without `--show drun` |
| `gtk_dark=true` | Asks GTK to render its bits (like scrollbars) in dark mode too, so nothing looks out of place |
| `allow_images=true` / `image_size` | Shows each app's icon next to its name, sized to fit |

**Why match the waybar colors?** `#89b4fa` (the blue accent) and `#1e1e2e`/`#cdd6f4` (background/text) are the same values used in Step 28's `waybar/style.css` — reusing them here just keeps the bar, launcher, and (if you style it later) swaync panel feeling like one consistent theme instead of three unrelated GTK apps bolted together.

After saving, reopen wofi with your new keybind (`SUPER+D`) to see the changes — no reload command needed, it re-reads its config fresh every time it launches.

---

## 30. Screenshots with `grim` + `wl-clipboard`

`grim` takes the screenshot; `wl-clipboard` (specifically its `wl-copy` half) puts the result straight onto your clipboard, so you can paste it directly into Discord, a chat, an editor — anywhere — with no intermediate file.

Head back to the **KEYBINDINGS** section (same place as the wofi bind in Step 29). The default template actually already ships with this exact bind as an example:

```lua
hl.bind(mainMod .. " + SHIFT + S", hl.dsp.exec_cmd("grim - | wl-copy"))
```

If you don't see it, add it below `local mainMod = "SUPER"` alongside your other binds. Feel free to remap the key combo — `SUPER+SHIFT+S` is just what ships by default.

**Why?** `grim -` tells `grim` to write the screenshot to stdout instead of saving a file, and the pipe (`|`) feeds those image bytes straight into `wl-copy`, which places them on the clipboard. Press the keybind, and a full-screen screenshot is silently sitting on your clipboard, ready to paste — no popup, no file to clean up later.

> ⚠️ **If screenshots silently do nothing:** check the **PERMISSIONS** section — look for the comment header `---- PERMISSIONS ----`. Newer Hyprland versions gate screen capture behind a permission system, and the template ships with the exact lines you need already written, just commented out:
> ```lua
> -- hl.permission("/usr/(bin|local/bin)/grim", "screencopy", "allow")
> -- hl.permission("/usr/(lib|libexec|lib64)/xdg-desktop-portal-hyprland", "screencopy", "allow")
> ```
> If you ever turn on `enforce_permissions = true` (also in that section, commented out above those two lines) and screenshots or screen-sharing suddenly stop working, come back here and uncomment both lines — the first covers `grim` directly, the second covers screen-sharing/recording through `xdg-desktop-portal-hyprland`. A Hyprland restart is required for permission changes to take effect.

> 💡 **Tip:** If you'd rather save a file *and* copy it, use `tee` to split the output:
> ```lua
> hl.bind(mainMod .. " + SHIFT + S", hl.dsp.exec_cmd("grim - | tee ~/Pictures/screenshot-$(date +%s).png | wl-copy"))
> ```
> This writes a timestamped PNG to `~/Pictures/` and still copies it to the clipboard in the same keypress.

> 💡 **Tip:** This setup always grabs the *whole* screen. If you want to drag-select a specific region instead, pair `grim` with `slurp` (`sudo pacman -S slurp`) and swap the command for `grim -g "$(slurp)" - | wl-copy` — `slurp` lets you click-and-drag an area, and its output tells `grim` exactly which pixels to capture.

---

## 31. Pick a Kitty Theme and Font

Kitty (your terminal) has a built-in theme browser — no config file editing required:

```bash
kitten themes
```

Use the arrow keys or type to search, preview a theme live, and press `Enter` to apply it.

**Why does this "just work"?** Behind the scenes it saves the theme to `~/.config/kitty/current-theme.conf` and quietly links it into your `kitty.conf` for you.

After picking one, either restart kitty or reload it live with `Ctrl+Shift+F5`.

Pick a font the same way:

```bash
kitten choose-fonts
```

**Why?** A Nerd Font (like the `ttf-jetbrains-mono-nerd` you already installed) makes the terminal render icons correctly and just feels a lot more comfortable to look at all day.

After selecting a font, restart kitty to see it applied.

---

## 32. Yazi Basics

Yazi is a terminal-based file manager — think of it as a fast, keyboard-driven alternative to a Files/Finder window.

```bash
yazi
```

| Key | Action |
|---|---|
| `h` `j` `k` `l` | Move left / down / up / right (same as nvim) |
| `Enter` | Open a file, or step into a folder |
| `Space` | Select/deselect a file |
| `y` then `p` | Copy, then paste |
| `d` | Cut (move) |
| `a` | Create a new file or folder |
| `r` | Rename |
| `Tab` | Toggle the preview pane |
| `q` | Quit |

---

## 33. Connect to Wi-Fi Day-to-Day with `nmtui`

```bash
nmtui
```

A simple text menu opens:

1. Choose **Activate a connection**
2. Arrow down to your Wi-Fi network, press `Enter`
3. Type the password when asked
4. It'll show as connected in the list

**Why not `iwctl` again?** `iwctl` (from Step 1) only works inside the temporary live USB environment. Now that `networkmanager` is installed and running, `nmtui` is its friendly menu — and unlike `iwctl`, it remembers networks and reconnects automatically every time you boot.

### Add `nm-applet` to autostart

You already installed `network-manager-applet` back in Step 18, but it won't run on its own — it needs to be launched when Hyprland starts, just like `hyprpaper` and `waybar`. Go back to the **AUTOSTART** block one more time and add `nm-applet` into the chained line — it should now look like the full default:

```lua
hl.on("hyprland.start", function()
	hl.exec_cmd("systemctl --user start hyprpolkitagent")
	hl.exec_cmd("waybar & hyprpaper & nm-applet & swaync")
end)
```

**Why?** `nm-applet` is the little Wi-Fi icon that sits in your tray (next to the volume/battery icons) once `waybar`'s tray module picks it up. Without this line, NetworkManager itself still auto-connects to known networks fine on boot — but you'd have no visual indicator or quick-click menu to switch networks, check signal strength, or forget a network without dropping back into `nmtui` every time.

> 💡 **Tip:** If you don't see the tray icon appear after adding this, double check `"tray"` is included in your `modules-right` list in `~/.config/waybar/config.jsonc` (already added in Step 28) — the applet runs invisibly without a tray to dock into.

To just check your connection status without the menu:

```bash
nmcli device status
```

---

## 34. Finish Your Shell: Plugins, Prompt, and `.zshrc`

This last step brings together everything your shell needs: the two zsh plugins from Step 18, a Starship prompt, and fzf's keyboard shortcuts. Rather than editing `.zshrc` piece by piece, you'll write it once, in full.

**What each plugin actually does:**
- **zsh-autosuggestions** — as you type, it shows a faint, greyed-out guess of the rest of the command based on your history. Press the `→` right-arrow key to accept it.
- **zsh-syntax-highlighting** — colors what you're typing green if it's a valid, runnable command, and red if it isn't — so you catch typos before hitting Enter.

**Pick a Starship prompt preset.** Starship comes with several ready-made prompt styles so you don't have to design one from scratch. See what's available:

```bash
starship preset --list
```

Save one as your active config — for example, the Nerd Font icons preset:

```bash
starship preset nerd-font-symbols -o ~/.config/starship.toml
```

`starship preset <name> -o <file>` writes a complete, working `starship.toml` for you. You can open that file later in `nvim` and tweak individual pieces once you know what you like.

**Get to know fzf's shortcuts.** Once fzf is sourced into your shell, it adds keyboard shortcuts that work anywhere on the command line:

| Shortcut | What happens |
|---|---|
| `Ctrl+R` | Fuzzy-search your command **history** — type a couple of letters instead of pressing ↑ repeatedly |
| `Ctrl+T` | Fuzzy-**find a file or folder** and drop its path at your cursor |
| `Alt+C` | Fuzzy-`cd` straight into any subfolder |
| `something **<Tab>` | Fuzzy-completes an argument — e.g. `nvim **<Tab>` |

You can also chain it with other commands: `git checkout $(git branch | fzf)` lets you fuzzy-pick a branch to switch to.

**Now write the full `.zshrc`:**

```bash
nvim ~/.zshrc
```

```bash
eval "$(starship init zsh)"
export EDITOR='nvim'
source <(fzf --zsh)
source /usr/share/zsh/plugins/zsh-syntax-highlighting/zsh-syntax-highlighting.zsh
source /usr/share/zsh/plugins/zsh-autosuggestions/zsh-autosuggestions.zsh
HISTFILE=~/.zsh_history
HISTSIZE=10000
SAVEHIST=10000
setopt SHARE_HISTORY
setopt HIST_IGNORE_ALL_DUPS
alias update='sudo pacman -Syu'
alias yayu='yay -Syu'
alias search='nvim $(fzf --preview="bat --color=always {}")'
```

| Line | Plain-English explanation |
|---|---|
| `eval "$(starship init zsh)"` | Turns on the Starship prompt you just configured. Keeping it near the top avoids conflicts with anything loaded after it. |
| `export EDITOR='nvim'` | Tells other programs (like `git commit` or `sudo -e`) to open `nvim` instead of defaulting to `vi`. |
| `source <(fzf --zsh)` | Loads fzf's keyboard shortcuts (`Ctrl+R`, `Ctrl+T`, `Alt+C`) straight from your installed fzf binary. |
| `source .../zsh-syntax-highlighting.zsh` | Turns on the red/green live syntax highlighting. |
| `source .../zsh-autosuggestions.zsh` | Turns on the ghost-text history suggestions. |
| `HISTFILE=~/.zsh_history` | The file where your command history gets saved permanently, instead of vanishing when you close the terminal. |
| `HISTSIZE=10000` | How many past commands zsh keeps in memory while you're using it. |
| `SAVEHIST=10000` | How many of those get actually written to `HISTFILE` on disk. |
| `setopt SHARE_HISTORY` | Every open terminal window shares one live history — run something in one tab, and pressing ↑ in another tab sees it instantly. |
| `setopt HIST_IGNORE_ALL_DUPS` | Skips saving a command to history if it's an exact repeat of the one right before it. |
| `alias update='sudo pacman -Syu'` | A shortcut — type `update` instead of the full pacman upgrade command. |
| `alias update yay='yay -Syu'` | Same idea, but updates official packages and AUR packages together via `yay`. |
| `alias search='nvim $(fzf --preview="bat --color=always {}")'` | Fuzzy-find a file with a preview, then open it directly in `nvim` — two steps in one word. |

> ⚠️ **Order matters:** keep `zsh-syntax-highlighting` sourced *before* `zsh-autosuggestions` — it hooks deeply into how zsh handles what you type, and sourcing other plugins after it can quietly break the highlighting.

Save and reload your shell (`source ~/.zshrc`, or just open a new terminal) — your prompt, plugins, and shortcuts are all live from here on.

---

That's the whole setup, start to finish: a booted, working Arch install with Hyprland, a themed bar and launcher, notifications, screenshots, an AUR helper, and a shell that's actually pleasant to type in. From here, everything is just iteration — tweak colors, add keybinds, install more from the AUR — at your own pace.
