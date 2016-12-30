cd ~/Builds
wget <package snapshot>
tar -xzvf <package>.tar.gz
cd <new package directory>
makepkg -s
sudo pacman -U <package>.pkg.tar.xz

edit /etc/default/pacman under Options: ILoveCandy
visudo
uncomment wheel, sudo group
su <regular user>
cd ~
pacman -S xdg-usr-dirs
xdg-user-dirs-update

PACAUR:
pacman -S wget expect
mv Public ./Builds
cd Builds
go to AUR and get cower snapshot URL
get PGP key from pinned comment
gpg recv-keys <url>
wget <cower.tar.gz>
tar -xzvf cower.tar.gz
cd cower
chmod +x PKGBUILD
makepkg -i
pacman -U cower
cd ..
wget <pacaur snapshot>
tar -xzvf pacaur.tar.gz
cd pacaur
chmod +x PKGBUILD
makepkg

GNOME:
pacman -S xf86-video-intel synaptics
pacman -S gnome (will also install xorg-server and gdm)
pacman -S xorg-xinit
cp /etc/X11/xinit/xinitrc ~/.xinitrc
edit and add  exec gnome-session

SYSTEM M

