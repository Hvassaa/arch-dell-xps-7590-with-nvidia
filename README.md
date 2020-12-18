# Installation of Arch Linux (With Gnome) on dell xps 7590 (With Gtx 1650)

## Do the normal UEFI installation

## User setup

* Install "sudo"
* Add user to the wheel group
* Allow users in wheel to sudo (via "visudo")

## Configure pacman 

* Add The color feature
* Add multilib
* Add ILoveCandy feature

## Install Nvida and setup

Install nvidia related packages

* nvidia
* lib32-nvidia-utils
* nvidia-settings
* prime-run
* maybe mesa-demos for glxinfo/glxgears

Now, run `prime-run program` to run a program under the Nvidia card.

## Make GDM always use Xorg instead of Wayland

https://wiki.archlinux.org/index.php/GDM#Use_Xorg_backend

Uncomment `#WaylandEnable=false` in `/etc/gdm/custom.conf`

## Fix GDM's complaints about entropy

See https://wiki.archlinux.org/index.php/GDM#GDM_does_not_start_until_input_is_provided

The processor supports the RDRAND instruction, so just add the following `random.trust_cpu=on` to the kernel parameters. Then regenerate grub.cfg:

    # grub-mkconfig -o /boot/grub/grub.cfg

## Fix GDM hanging on boot

See https://wiki.archlinux.org/index.php/GDM#Black_screen_on_AMD_or_Intel_GPUs_when_an_NVidia_(e)GPU_is_present

Add `intel_agp i915` modules to the array modules in `/etc/mkinitcpio.conf`. Then regenerate regenerate initramfs:

    # mkinitcpio -P

## Optional, install powertop and tlp

### Powertop

Install `powertop`, to monitor power usage.

### tlp

See https://wiki.archlinux.org/index.php/TLP

Install `tlp` and `tlp-rdw`, for power saving features. (Also, use NetworkManager for this). Then:

* Enable `NetworkManager-dispatcher.service`
* Mask `systemd-rfkill.service` and `systemd-rfkill.socket`


## Optional, use optimus-manager to power card completely down

### (Maybe install `yay`)

See https://github.com/Jguer/yay

### Install `gdm-prime` and `optimus-manager` from aur

See https://github.com/Askannz/optimus-manager

* Install `gdm-prime` from the aur. Remove the repo supplied `gdm` before. Make sure the wayland is disabled, as above.
* Install `optimus-manager` from the aur, and reboot. 

### Usage and gotchas

I have a bug, where I can't switch graphics, unless I first run `prime-offload`.

Then you can switch with

    optimus-manager --switch intel # for intel graphics
    optimus-manager --switch nvidia # for nvidia graphics
    optimus-manager --switch hybrid # for hybrid graphics

### Enable runtime PM for nvidia card

See https://linrunner.de/tlp/faq/powercon.html#faq-powercon-hybrid-graphics

It seems my nvidia card does not power down in intel mode. This can be fixed with the following:

* First, check if runtime pm is "bad" for nvidia card under the tunables section in `powertop`.
* If it is, it might help to set `RUNTIME_PM_DRIVER_BLACKLIST="mei_me"` under `/etc/tlp.conf`.

This makes my system draw ~2W when idling.
