# Minimal Arch Linux + Hyprland + SDDM Installation Guide

A minimal Arch Linux installation with **Hyprland**, **SDDM**, and essential desktop utilities.

## 📋 Goal

After completing this guide, you will have a ready-to-rice Arch Linux + Hyprland setup.

> No KDE Plasma.
>
> No GNOME.
>
> No giant dotfile installers.

---

# 1. Connect to the Internet

Test your connection:

```bash
ping archlinux.org
```

If using Wi-Fi:

```bash
iwctl
device list
station wlan0 scan
station wlan0 get-networks
station wlan0 connect "YOUR_WIFI_NAME"
exit
```

**Why?**

Arch downloads packages directly from online repositories.

---

# 2. Find Your Drive

```bash
lsblk
```

Example:

```text
sda
nvme0n1
```

**Why?**

Shows all storage devices connected to your computer. Make sure you choose the correct drive.

---

# 3. Partition the Drive

```bash
cfdisk /dev/sdX
```

Create:

| Partition | Size |
|---|---:|
| EFI System | 512 MB |
| Linux Filesystem | Remaining Space |

Select **Write → Yes → Quit**.

**Why?**

EFI stores boot files. Linux Filesystem stores Arch Linux and your files.

---

# 4. Format Partitions

```bash
mkfs.fat -F32 /dev/sdX1
mkfs.ext4 /dev/sdX2
```

**Why?**

Formats the EFI and Linux partitions.

---

# 5. Mount Partitions

```bash
mount /dev/sdX2 /mnt
mkdir /mnt/boot
mount /dev/sdX1 /mnt/boot
```

**Why?**

Tells the installer where Arch Linux should be installed.

---

# 6. Install the Base System

```bash
pacstrap /mnt base linux linux-firmware
```

| Package | Purpose |
|---|---|
| base | Essential Linux utilities |
| linux | Linux kernel |
| linux-firmware | Firmware for hardware devices |

---

# 7. Install Essential Packages

```bash
pacstrap /mnt networkmanager grub efibootmgr sudo nvim
```

| Package | Purpose |
|---|---|
| networkmanager | Networking and Wi-Fi |
| grub | Bootloader |
| efibootmgr | Registers GRUB with UEFI |
| sudo | Administrative privileges |
| nvim | Modern text editor |
| zsh | Interactive shell |

---

# 8. Generate fstab

```bash
genfstab -U /mnt >> /mnt/etc/fstab
```

**Why?** Generates the filesystem table.

---

# 9. Enter the Installed System

```bash
arch-chroot /mnt
```

---

# 10. Set Timezone

```bash
ln -sf /usr/share/zoneinfo/Asia/Jakarta /etc/localtime
hwclock --systohc
```

---

# 11. Configure Locale

```bash
nvim /etc/locale.gen
```

Uncomment:

```text
en_US.UTF-8 UTF-8
```

Then:

```bash
locale-gen
echo "LANG=en_US.UTF-8" > /etc/locale.conf
```

---

# 12. Set Hostname

```bash
echo "arch" > /etc/hostname
```

---

# 13. Configure Hosts

```bash
nvim /etc/hosts
```

Add:

```text
127.0.0.1 localhost
::1 localhost
127.0.1.1 arch.localdomain arch
```

---

# 14. Set Root Password

```bash
passwd
```

# 15. Create a User

```bash
useradd -m -G wheel -s /bin/zsh excello
passwd excello
```

# 16. Enable sudo

```bash
EDITOR=nvim visudo
```

Uncomment `%wheel ALL=(ALL:ALL) ALL`.

# 17. Install GRUB

```bash
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB
grub-mkconfig -o /boot/grub/grub.cfg
```

# 18. Install Hyprland Desktop & CLI Goodies

```bash
pacman -S sddm hyprland hyprpaper kitty waybar yazi network-manager-applet pipewire pipewire-pulse wireplumber ttf-jetbrains-mono-nerd hyprpolkitagent nvim zsh zsh-autosuggestions zsh-syntax-highlighting fzf starship fastfetch
```

| Package | Purpose |
|---|---|
| sddm | Graphical login manager |
| hyprland | Wayland compositor |
| hyprpaper | Wallpaper manager |
| kitty | Terminal emulator |
| waybar | Status bar |
| yazi | TUI-based file manager |
| network-manager-applet | Wi-Fi tray applet |
| pipewire | Audio server |
| pipewire-pulse | PulseAudio compatibility |
| wireplumber | Audio session manager |
| ttf-jetbrains-mono-nerd | Fonts and icons |
| hyprpolkitagent | PolicyKit agent |
| nvim | Modern text editor |
| zsh | Interactive shell |
| zsh-autosuggestions | Autosuggestions |
| zsh-syntax-highlighting | Syntax highlighting |
| fzf | Finding files |
| starship | Cross-shell prompt |
| fastfetch | System information tool |

# 19. Enable Services

```bash
systemctl enable NetworkManager
systemctl enable sddm
```

# 20. Finish

```bash
exit
umount -R /mnt
poweroff
```

Remove the USB drive.

# 21. First Boot

```text
Power Button
↓
UEFI
↓
GRUB
↓
SDDM
↓
Hyprland
↓
Desktop
```
