# Install Arch Linux with LVM on LUKS Btrfs root
Drawing from [this](https://fogelholk.io/installing-arch-with-lvm-on-luks-and-btrfs/) nice guide.
When in doubt, always consult the [wiki](https://wiki.archlinux.org/index.php/Installation_Guide).

#### Download the image from https://www.archlinux.org/ and format a USB drive:

##### on Linux:
lsblk -f              # make sure you choose the right path!
dd if=archlinux.img of=/dev/sdX bs=16M && sync disk

##### on macOS:
```bash
`diskutil list`
`sudo umount /dev/disk2`                         # again, choose the right path!
`dd if=archlinux.img of=/dev/rdisk2 bs=1m`       # notice the path is /dev/**r**disk2 - this will transfer more quickly
# you'll get a notification from macOS like "the disk format isn't recognized" - no worries: simply remove the drive and proceed
```

#### Boot from the USB. 
If the USB fails to boot, make sure that secure boot is disabled in the BIOS configuration.

##### Set keymap:
By default, the US keymap is loaded. Otherwise, consult the wiki to find your desired keyboard layout:
`loadkeys sv-latin1`

##### Connect to WiFi:
`wifi-menu`

##### Create partitions:
This will be the partition scheme:

```bash
1 512MB EFI partition   # Hex code ef00
2 250MB Boot partition  # Hex code 8300
3 -rest Root		# Hex code 8E00
```

`gdisk /dev/sdX` 

- Zap the drive

`x`
`z`

- Write a new GPT table

`o`

- Partitioning

```bash
n (Add a new partition)
Partition number 1
First sector 2048 (default)
Last sector +512M
Hex code EF00
```

```bash
n (Add a new partition)
Partition number 2
First sector 1050624 (default)
Last sector +512M
Hex code 8300 
```

```bash
n (Add a new partition)
Partition number 3
First sector 2099200 (default)
Last sector (hit Enter to use remaining disk space)
Hex code 8E00
```

- hit `p` to view the partitioning scheme, then confirm with:

`w`
`Y`

##### Encryption and LVM:

- LUKS formatting:

```bash
cryptsetup luksFormat /dev/sda3
Are you sure? YES
Enter passphrase (twice)
```

- Open the container:

`lvm` is the name of the decrypted device - use whichever you prefer:
`cryptsetup luksOpen /dev/sdX3 lvm`

- Create LVM:

`pvcreate /dev/mapper/lvm`
`vgcreate lvmvg /dev/mapper/lvm`
`lvcreate -L 8G lvmvg -n swapvol`
`lvcreate -l 100%FREE lvmvg -n rootvol`

##### Format the partitions:

```bash
mkfs.vfat -F32 /dev/sda1
mkfs.ext4 /dev/sda2
mkfs.btrfs -L btrfs /dev/mapper/lvmvg-rootvol
mkswap /dev/mapper/lvmvg-swapvol
swapon /dev/mapper/lvmvg-swapvol
```

##### Create and mount `btrfs` subvolumes:

```bash
mount /dev/mapper/lvmvg-rootvol /mnt
btrfs subvolume create /mnt/root
btrfs subvolume create /mnt/home
umount /mnt
mount -o subvol=root,ssd,compress=lzo /dev/mapper/lvmvg-rootvol /mnt
mkdir /mnt/{boot,home}
mount -o subvol=home,ssd,compress=lzo /dev/mapper/lvmvg-rootvol /mnt/home
```

##### Mount remaining partitions:

```bash
mount /dev/sda2 /mnt/boot
mkdir /mnt/boot/efi
mount /dev/sda1 /mnt/boot/efi
```

##### Edit pacman.conf:

`vim /etc/pacman.conf`

- uncomment `Misc. Options` entries and add:

`ILoveCandy`

##### Install the system and include stuff needed for connecting to WiFi when booting into the newly installed system:
`pacstrap /mnt base base-devel btrfs-progs grub grub-efi-x86_64 sudo zsh zsh-completions vim git efibootmgr dialog wpa_supplicant grml-zsh-config xdg-user-dirs wget`

##### Generate and edit `fstab`:

`genfstab -pU /mnt >> /mnt/etc/fstab`

- Make sure to include -p**U**!

`vim /mnt/etc/fstab`

- Change `relatime` on all non-boot partitions to `noatime` (reduces wear if using an SSD)

##### Make /tmp a ramdisk (add the following line to /mnt/etc/fstab):

`tmpfs	/tmp	tmpfs	defaults,noatime,mode=1777	0	0`

##### Change root into the new system:

`arch-chroot /mnt /bin/bash`

##### Setup system clock:

```bash
ln -s /usr/share/zoneinfo/US/Central /etc/localtime
hwclock --systohc --utc
```

##### Set the hostname:

`echo bikini-bottom > /etc/hostname`

##### Update locale:

```bash
echo LANG=en_US.UTF-8 >> /etc/locale.conf
echo LANGUAGE=en_US >> /etc/locale.conf
echo LC_ALL=C >> /etc/locale.conf
```

##### Set password for root:

`passwd`

##### Add normal user (remove -s flag if you don't want to use zsh):

```bash
useradd -m -g users -G wheel -s /bin/zsh MYUSERNAME
passwd MYUSERNAME
```

##### Configure mkinitcpio with modules needed for the initrd image:

- Consult the wiki for any necessary modules you might need:

`vim /etc/mkinitcpio.conf`

```bash
...
MODULES="i915 crc32c-intel ext4"
...
HOOKS="base udev autodetect modconf block encrypt lvm2 btrfs resume filesystems keyboard fsck"
...
```

##### Regenerate initrd image:

`mkinitcpio -p linux`

##### Install and configure GRUB:

`grub-install`

- Find persistent UUID pathname for root partition:

```bash
blkid
(find your partition's uuid)
echo /dev/disk/by-uuid/<UUID> >> /etc/default/grub
```

- In `/etc/default/grub` edit the kernel parameters:

```bash
GRUB_CMDLINE_LINUX_DEFAULT="quiet transparent_hugepage=never"
GRUB_CMDLINE_LINUX="cryptdevice=/dev/disk/by-uuid/<insert encrypted root UUID here>:lvm:allow-discards root=/dev/mapper/lvmvg-rootvol rootflags=subvol=root resume=/dev/mapper/lvmvg-swapvol"

# save, exit and:

grub-mkconfig -o /boot/grub/grub.cfg
```

- Ignore any `lvmetad` warnings, they won't interfere with the `conf` setup

##### Drop out of the system:

`exit`

##### Unmount all partitions:

`umount -R /mnt`
`swapoff -a`

##### Reboot into the new system - make sure to remove the USB:
`reboot`

##### Rejoice and / or troubleshoot :)

##### System setup:

- Login as `root`

- Connect to WiFi with `wifi-menu`

- Add normal user to sudoers:

`visudo`
Uncomment `username ALL=(ALL) ALL`

- Populate `home` directory with familiar directories and prepare Builds directory:

```bash
cd ~
xdg-user-dirs-update
mv Public ./Builds
```

- Install Arch Build System:

```bash
cd ~/Builds
pacman -S abs
sudo abs
```

- Install Pacaur and dependencies:

```bash
pacman -S expect links
links aur.archlinux.org

# Navigate to 'aur' and look for falconindy's pinned comment:
# Note: gpg occasionally gets messed up from recent updates, in which case you might find a patched version in [Testing].
gpg --recv-keys --keyserver hkp://pgp.mit.edu 1EB2638FF56C0C53

# Hit 'Download Snapshot' and exit when the download finishes, then:
tar -xzvf cower.tar.gz
cd cower
chmod +x PKGBUILD
makepkg -i
pacman -U cower
cd ..

# Now go grab pacaur from the AUR and do the same thing (make sure to gpg --recv-keys here as well)
tar -xzvf pacaur.tar.gz
cd pacaur
chmod +x PKGBUILD
makepkg -i
```

- Install and configure Xorg and Gnome:

```bash
pacman -S xf86-video-intel xorg-server xorg-xinit gnome 
# (will also install gdm)


# maintain authenticated session:
vim ~/.xserverrc
# add (note that the first line is not one of my comments, include it WITH the #):

#!/bin/sh
exec /usr/bin/Xorg -nolisten tcp "$@" vt$XDG_VTNR

# Exit and:
cp /etc/X11/xinit/xinitrc ~/.xinitrc
vim ~/.xinitrc

# Edit and add:

if [ -d /etc/X11/xinit/xinitrc.d ] ; then
    for f in /etc/X11/xinit/xinitrc.d/?*.sh ; do
        [ -x "$f" ] && . "$f"
    done
    unset f
fi

# add this to the bottom of the file:
exec gnome-session


# start X at login:
vim ~/.zprofile (or ~/.bash_profile if you prefer)
# add:
if [ -z "$DISPLAY" ] && [ -n "$XDG_VTNR" ] && [ "$XDG_VTNR" -eq 1 ]; then
  exec startx
fi
```

##### Deep breath and reboot again.

#### btrfs Backups to external USB with `snapper` and `snap-sync`

##### Install snapper and create snapshot configs
```bash
pacman -S snapper
snapper -c root create-config /
snapper -c home create-config /home
# You'll need to delete the automatically-generated subvolumes,
# as they are inherently flawed and shouldn't be relied upon (See [1])
btrfs subvolume delete /.snapshots
btrfs subvolume delete /home/.snapshots
# Now, create new subvolumes and mount them at the mountpoints snapper expects:
mkdir /snapshots
mkdir /.snapshots
mkdir /home/.snapshots
btrfs subvolume create /snapshots/root_snaps
btrfs subvolume create /snapshots/home_snaps
# Lookup subvolume ID for root_snaps and home_snaps,
# and mount them to their respective snapshot mountpoints
btrfs subvolume list -a /
mount -o compress=lzo,subvolid=<ROOT_SNAPS_ID> /dev/mapper/lvmvg-root /.snapshots
mount -o compress=lzo,subvolid=<HOME_SNAPS_ID> /dev/mapper/lvmvg-root /home/.snapshots
# Enable automatic snapshots
systemctl start snapper-timeline.timer snapper-cleanup.timer
systemctl enable snapper-timeline.timer snapper-cleanup.timer
```
[1](https://bbs.archlinux.org/viewtopic.php?id=194491)

##### Install `snap-sync` and backup to btrfs-formatted USB volume

```bash
mkfs.btrfs -L Backups /dev/sdbX
mount -o compress=lzo /dev/sdbX /mnt
pacaur -S snap-sync
sudo snap-sync
# Follow the directions to backup to the USB.
# It's possible you might have to create a 'Backups' directory at /mnt
# (I did for some reason, but can't remember why -.-')
```

#### Add pacman / pacaur alias to `.zshrc`
`echo 'alias -g pac="pacaur"' >> ~/.zshrc`

#### Install neovim
```bash
pacman -S neovim
echo 'alias -g vim="neovim"' >> ~/.zshrc
# copy to system clipboard by putting this in init.vim:
        noremap <C-y> "+y
```


#### Slightly hacky method to make Powerline fonts work:
##### Add to your `.zshrc`:
export LC_ALL=en_US.UTF-8
