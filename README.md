# Minimal Arch Linux + Hyprland + SDDM Installation Guide

## Goal

After completing this guide, you will have a clean Arch Linux setup with:

```
✓ Arch Linux
✓ Hyprland
✓ SDDM login screen
✓ Wi-Fi support
✓ Audio support
✓ Waybar
✓ Wofi
✓ Kitty
✓ Dolphin
✓ Wallpapers
✓ Nerd Fonts
```

No KDE Plasma.
No GNOME.
No giant dotfile installers.

---

## 1. Connect to the Internet

Test your connection:

```bash
ping archlinux.org
```

If using Wi-Fi:

```bash
iwctl
```

Inside `iwctl`:

```bash
device list
station wlan0 scan
station wlan0 get-networks
station wlan0 connect "YOUR_WIFI_NAME"
exit
```

### Why?

Arch downloads packages directly from online repositories.

---

## 2. Find Your Drive

```bash
lsblk
```

Example output:

```
sda
nvme0n1
```

### Why?

Shows all storage devices connected to your computer.

Make sure you choose the correct drive.

---

## 3. Partition the Drive

```bash
cfdisk /dev/sdX
```

Replace `sdX` with your drive.

For UEFI systems:

| Partition         | Size             |
|-------------------|------------------|
| EFI System        | 512M             |
| Linux Filesystem  | Remaining Space  |

Select:

```
Write → Yes → Quit
```

### Why?

**EFI System Partition**
Stores boot files required by UEFI.

**Linux Filesystem Partition**
Stores Arch Linux and your files.

---

## 4. Format Partitions

Example:

```bash
mkfs.fat -F32 /dev/sdX1
mkfs.ext4 /dev/sdX2
```

### Why?

**FAT32**
Required for UEFI booting.

**EXT4**
Stores your Linux filesystem.

---

## 5. Mount Partitions

```bash
mount /dev/sdX2 /mnt

mkdir /mnt/boot
mount /dev/sdX1 /mnt/boot
```

### Why?

Tells the installer where Arch Linux should be installed.

---

## 6. Install the Base System

```bash
pacstrap /mnt base linux linux-firmware
```

### Packages

| Package          | Purpose                      |
|-------------------|------------------------------|
| `base`            | Essential Linux utilities    |
| `linux`           | Linux kernel                 |
| `linux-firmware`  | Firmware for hardware devices|

---

## 7. Install Essential Packages

```bash
pacstrap /mnt networkmanager grub efibootmgr sudo nano
```

### Packages

| Package           | Purpose                     |
|--------------------|------------------------------|
| `networkmanager`  | Networking and Wi-Fi         |
| `grub`            | Bootloader                   |
| `efibootmgr`      | Registers GRUB with UEFI     |
| `sudo`            | Administrative privileges    |
| `nano`            | Simple text editor           |

---

## 8. Generate fstab

```bash
genfstab -U /mnt >> /mnt/etc/fstab
```

### Why?

Defines which partitions should automatically mount during boot.

---

## 9. Enter the Installed System

```bash
arch-chroot /mnt
```

### Why?

Switches into your newly installed Arch environment.

---

## 10. Set Timezone

For Indonesia:

```bash
ln -sf /usr/share/zoneinfo/Asia/Jakarta /etc/localtime

hwclock --systohc
```

### Why?

Ensures your system displays the correct time.

---

## 11. Configure Locale

Edit:

```bash
nano /etc/locale.gen
```

Uncomment:

```
en_US.UTF-8 UTF-8
```

Generate locales:

```bash
locale-gen
```

Set the default locale:

```bash
echo "LANG=en_US.UTF-8" > /etc/locale.conf
```

### Why?

Provides UTF-8 support and prevents locale-related warnings.

---

## 12. Set Hostname

```bash
echo "arch" > /etc/hostname
```

### Why?

Gives your computer a network name.

Example:

```
excello@arch
```

---

## 13. Configure Hosts File

Edit:

```bash
nano /etc/hosts
```

Add:

```
127.0.0.1   localhost
::1         localhost
127.0.1.1   arch.localdomain   arch
```

### Why?

Allows the system to resolve its own hostname correctly.

---

## 14. Set Root Password

```bash
passwd
```

### Why?

Protects the administrator account.

---

## 15. Create a User

Replace `excello` with your username:

```bash
useradd -m -G wheel excello

passwd excello
```

### Why?

A regular user account is safer for daily use.

---

## 16. Enable sudo

```bash
EDITOR=nano visudo
```

Uncomment:

```
%wheel ALL=(ALL:ALL) ALL
```

### Why?

Allows users in the `wheel` group to use `sudo`.

---

## 17. Install GRUB

```bash
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB

grub-mkconfig -o /boot/grub/grub.cfg
```

### Why?

GRUB allows your computer to boot Arch Linux.

---

## 18. Install Hyprland Desktop

```bash
pacman -S sddm hyprland hyprpaper kitty waybar wofi dolphin network-manager-applet pipewire pipewire-pulse wireplumber ttf-jetbrains-mono-nerd polkit-kde-agent
```

### Packages

| Package                    | Purpose                     |
|-----------------------------|------------------------------|
| `sddm`                     | Graphical login manager      |
| `hyprland`                 | Wayland compositor            |
| `hyprpaper`                | Wallpaper manager             |
| `kitty`                    | Terminal emulator             |
| `waybar`                   | Status bar                    |
| `wofi`                     | Application launcher          |
| `dolphin`                  | File manager                  |
| `network-manager-applet`   | Wi-Fi tray applet              |
| `pipewire`                 | Audio server                  |
| `pipewire-pulse`           | PulseAudio compatibility       |
| `wireplumber`              | Audio session manager           |
| `ttf-jetbrains-mono-nerd`  | Fonts and icons                |
| `polkit-kde-agent`         | Authentication dialogs          |

---

## 19. Enable Services

```bash
systemctl enable NetworkManager

systemctl enable sddm
```

### Why?

**NetworkManager**
Automatically manages network connections.

**SDDM**
Starts the graphical login screen during boot.

---

## 20. Finish the Installation

```bash
exit

umount -R /mnt

poweroff
```

Remove the USB drive after shutdown.

---

## 21. First Boot

Boot sequence:

```
Power Button
      ↓
     UEFI
      ↓
     GRUB
      ↓
     SDDM
      ↓
Select Hyprland
      ↓
     Login
      ↓
    Desktop
```

No need to type:

```
Hyprland
```

on every startup.

SDDM launches Hyprland automatically.

---

## Final Package List

```
base
linux
linux-firmware

networkmanager
grub
efibootmgr
sudo
nano

hyprland
hyprpaper
kitty
waybar
wofi
dolphin

pipewire
pipewire-pulse
wireplumber

network-manager-applet
polkit-kde-agent

ttf-jetbrains-mono-nerd

sddm
```

This setup gives you a clean, lightweight Arch + Hyprland desktop with a graphical login screen while remaining significantly smaller than KDE Plasma or GNOME.
