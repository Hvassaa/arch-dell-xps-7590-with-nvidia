# Installation of Arch Linux (With Gnome) on dell xps 7590 (With Gtx 1650)

- [Installation of Arch Linux (With Gnome) on dell xps 7590 (With Gtx 1650)](#installation-of-arch-linux--with-gnome--on-dell-xps-7590--with-gtx-1650-)
  * [Do the normal UEFI installation](#do-the-normal-uefi-installation)
  * [User setup](#user-setup)
  * [Configure pacman](#configure-pacman)
  * [Esential packages](#esential-packages)
  * [Make GDM always use Xorg instead of Wayland](#make-gdm-always-use-xorg-instead-of-wayland)
  * [Fix GDM's complaints about entropy](#fix-gdm-s-complaints-about-entropy)
  * [Fix GDM hanging on boot](#fix-gdm-hanging-on-boot)
  * [Optional, for SSDs](#optional--for-ssds)
  * [Optional, install powertop, tlp and thermald](#optional--install-powertop--tlp-and-thermald)
    + [Powertop](#powertop)
    + [tlp](#tlp)
    + [thermald](#thermald)
  * [Setup hibernation](#setup-hibernation)
    + [Setup kernel parameters in grub](#setup-kernel-parameters-in-grub)
    + [Add hook to mkinitcpio](#add-hook-to-mkinitcpio)
    + [Hibernate](#hibernate)
  * [Desktop setup](#desktop-setup)
    + [Adjust xdg-mime](#adjust-xdg-mime)
  * [Nvida setup](#nvida-setup)
    + [Install these need packages](#install-these-need-packages)
    + [Enable runtime power management for the Nvidia card](#enable-runtime-power-management-for-the-nvidia-card)
    + [Set tlp to always enable runtime power management](#set-tlp-to-always-enable-runtime-power-management)
    + [Launch programs with Nvidia](#launch-programs-with-nvidia)
    + [Check that it worked](#check-that-it-worked)
  * [(OLD VERSION, I think it is better to use the official nvidia prime offload) Optional, use optimus-manager to power card completely down](#-old-version--i-think-it-is-better-to-use-the-official-nvidia-prime-offload--optional--use-optimus-manager-to-power-card-completely-down)
    + [(Maybe install `yay`)](#-maybe-install--yay--)
    + [Install `gdm-prime` and `optimus-manager` from aur](#install--gdm-prime--and--optimus-manager--from-aur)
    + [Usage and gotchas](#usage-and-gotchas)
    + [Enable runtime PM for nvidia card](#enable-runtime-pm-for-nvidia-card)

## Do the normal UEFI installation

I went with

* 3 partitions
  - One for boot (/boot)
  - One for swap
  - One for root (/)
* Ext4 for root (and FAT32 for boot)
* Grub as a bootloader

## User setup

* Install "sudo"
* Create user
* Make sure user is in the wheel group
* Allow users in wheel to sudo (via "visudo")

## Configure pacman 

* Add The color feature
* Add multilib
* Add ILoveCandy feature

## Esential packages

    base base-devel git neovim man-db man-pages

## Make GDM always use Xorg instead of Wayland

https://wiki.archlinux.org/index.php/GDM#Use_Xorg_backend

Uncomment `#WaylandEnable=false` in `/etc/gdm/custom.conf`

## Fix GDM's complaints about entropy

See
https://wiki.archlinux.org/index.php/GDM#GDM_does_not_start_until_input_is_provided

The processor supports the RDRAND instruction, so just add the
following `random.trust_cpu=on` to the kernel parameters. Then
regenerate grub.cfg:

    # grub-mkconfig -o /boot/grub/grub.cfg

## Fix GDM hanging on boot

See
https://wiki.archlinux.org/index.php/GDM#Black_screen_on_AMD_or_Intel_GPUs_when_an_NVidia_(e)GPU_is_present

Add `intel_agp i915` modules to the array modules in
`/etc/mkinitcpio.conf`, so you end up with

    MODULES=(intel_agp i915)

plus whatever else you had before or need.

Then regenerate regenerate initramfs:

    # mkinitcpio -P

## Optional, for SSDs

See https://wiki.archlinux.org/index.php/Solid_state_drive

Essentially, just enable the systemd timer `fstrim.timer`, for
sustained long-term performance and wear-leveling on SSD:

    systemctl enable fstrim.timer

## Optional, install powertop, tlp and thermald

### Powertop

Install `powertop`, to monitor power usage.

### tlp

See https://wiki.archlinux.org/index.php/TLP

Install `tlp` and `tlp-rdw`, for power saving features. (Also, use
NetworkManager for this). Then:

* Enable `NetworkManager-dispatcher.service`
* Mask `systemd-rfkill.service` and `systemd-rfkill.socket`
* (See under the Nvidia section for always enables runtime pm)

### thermald

Install `thermald` and enable it. It helps regulate an Intel cpu's
temperature. 

## Setup hibernation
See 
https://wiki.archlinux.org/index.php/Power_management/Suspend_and_hibernate

### Setup kernel parameters in grub
Add `resume=UUID=uuid-of-my-swap-partition` (check `/etc/fstab`). Regenerate the grub config:
```
# grub-mkconfig -o /boot/grub/grub.cfg
```

### Add hook to mkinitcpio
Add `resume` to hooks under `/etc/mkinitcpio.conf`. The line should look something like
```
HOOKS=(base udev autodetect modconf block filesystems keyboard resume fsck)
```
Regenerate the initramfs:
```
# mkinitcpio -P
```

### Hibernate
Hibernate with `systemctl hibernate`


## Desktop setup

### Adjust xdg-mime
See 
https://wiki.archlinux.org/index.php/Xdg-utils

Install `xdg-utils`, then something like the following should be sufficient:
```
xdg-settings set default-web-browser firefox.desktop
xdg-mime default evince.desktop application/pdf
```

## Nvida setup

The goal here is to have a nvidia prime setup. 
This means the dGPU is powered down, when it is not in use, and is 
only powered on when you explicitly want it.

### Install these need packages
Install nvidia related packages

* nvidia
* lib32-nvidia-utils
* nvidia-prime
* optionally, mesa-demos for glxinfo/glxgears
* optionally, nvidia-settings

### Enable runtime power management for the Nvidia card
Put the following into `/etc/modprobe.d/nvidia.conf`:

```
options nvidia "NVreg_DynamicPowerManagement=0x02"
blacklist nouveau
```
and this into `/lib/udev/rules.d/80-nvidia-pm.rules`
```
# Remove NVIDIA USB xHCI Host Controller devices, if present
ACTION=="add", SUBSYSTEM=="pci", ATTR{vendor}=="0x10de", ATTR{class}=="0x0c0330", ATTR{remove}="1"

# Remove NVIDIA USB Type-C UCSI devices, if present
ACTION=="add", SUBSYSTEM=="pci", ATTR{vendor}=="0x10de", ATTR{class}=="0x0c8000", ATTR{remove}="1"

# Remove NVIDIA Audio devices, if present
ACTION=="add", SUBSYSTEM=="pci", ATTR{vendor}=="0x10de", ATTR{class}=="0x040300", ATTR{remove}="1"

# Enable runtime PM for NVIDIA VGA/3D controller devices on driver bind
ACTION=="bind", SUBSYSTEM=="pci", ATTR{vendor}=="0x10de", ATTR{class}=="0x030000", TEST=="power/control", ATTR{power/control}="auto"
ACTION=="bind", SUBSYSTEM=="pci", ATTR{vendor}=="0x10de", ATTR{class}=="0x030200", TEST=="power/control", ATTR{power/control}="auto"

# Disable runtime PM for NVIDIA VGA/3D controller devices on driver unbind
ACTION=="unbind", SUBSYSTEM=="pci", ATTR{vendor}=="0x10de", ATTR{class}=="0x030000", TEST=="power/control", ATTR{power/control}="on"
ACTION=="unbind", SUBSYSTEM=="pci", ATTR{vendor}=="0x10de", ATTR{class}=="0x030200", TEST=="power/control", ATTR{power/control}="on"
```

### Set tlp to always enable runtime power management
It seems tlp normally disables runtime pm when on AC. 
This can be fixed with these two lines in the config (`/etc/tlp.conf`):
```
RUNTIME_PM_ON_AC=auto
RUNTIME_PM_ON_BAT=auto
```
This might also be needed, haven't tested it through:
```
RUNTIME_PM_DRIVER_BLACKLIST="mei_me"
```

### Launch programs with Nvidia
Now, run `prime-run program` to run a program under the Nvidia card.

### Check that it worked
Check that 
```
cat /sys/bus/pci/devices/0000:01:00.0/power/runtime_status
```
returns `suspended`, when you are not running anything under the dGPU.
```
cat /proc/driver/nvidia/gpus/*/power
```
Should return `Runtime D3 status:          Enabled (fine-grained)`

Finally, check the power draw via `# powertop` while on battery and no peripherals connected. 
It should be ~2W (at least for me).


## (OLD VERSION, I think it is better to use the official nvidia prime offload) Optional, use optimus-manager to power card completely down

### (Maybe install `yay`)

See https://github.com/Jguer/yay

### Install `gdm-prime` and `optimus-manager` from aur

See https://github.com/Askannz/optimus-manager

* Install `gdm-prime` from the aur. Remove the repo supplied `gdm`
  before. Make sure the wayland is disabled, as above.
* Install `optimus-manager` from the aur, and reboot.

### Usage and gotchas

I have a bug, where I can't switch graphics, unless I first run
`prime-offload`. (As of 12th January 2021)

Then you can switch with

    optimus-manager --switch intel # for intel graphics
    optimus-manager --switch nvidia # for nvidia graphics
    optimus-manager --switch hybrid # for hybrid graphics

See https://github.com/Askannz/optimus-manager/issues/356

### Enable runtime PM for nvidia card

See
https://linrunner.de/tlp/faq/powercon.html#faq-powercon-hybrid-graphics

It seems my nvidia card does not power down in intel mode. This can be
fixed with the following:

* First, check if runtime pm is "bad" for nvidia card under the
  tunables section in `powertop`.
* If it is, it might help to set
  `RUNTIME_PM_DRIVER_BLACKLIST="mei_me"` under `/etc/tlp.conf`.

This makes my system draw ~2W when idling.
