Reference: https://jpedmedia.com/tutorials/installations/void_install/index.html

Backup in [install.md](install.md) in case the site goes down.

Step 0: Boot from a USB flash drive

## Make text easier to read 
```bash
/bin/bash
setfont -d # Pretty large
setfont /usr/share/kbd/consolefonts/lat5-16.psfu.gz # Not tiny, happy medium
```

## Connect to a network
### Wireless
Find the name of your wifi device
```bash
ip link
```

Sample device names: ETHERNET = enp1s0; WLAN = wlp2s0

Set the device to up and log in to the network
```bash
sudo ip link set wlan0 up
wpa_passphrase "your_SSID" "your_password" | sudo tee /etc/wpa_supplicant/wpa_supplicant.conf
sudo wpa_supplicant -B -i wlp2s0 -c /etc/wpa_supplicant/wpa_supplicant.conf
```

### Wired
Shouldn't need to start this service by default, but just in case:

```bash
dhcpcd
```

## Correct Drive Setup Examples
My drive:
```
/dev/nvme0n1
```

My partitions:
```
/dev/nvme0n1p1 (EFI)
/dev/nvme0n1p2 (filesystem)
/dev/mapper/cryptvoid
```

3 subvolumes created:
```
/mnt/@ - root
/mnt/@home
/mnt/@snapshots 
```

```bash
EFI_UUID=$(blkid -s UUID -o value /dev/nvme0n1p1)
ROOT_UUID=$(blkid -s UUID -o value /dev/mapper/cryptvoid)
LUKS_UUID=$(blkid -s UUID -o value /dev/nvme0n1p2)
```

## Desktop Environment
### Install KDE Plasma
```bash
sudo xbps-install kde5 kde5-baseapps xorg
# the display manager ssdm comes with the kde5 package
sudo ln -s /etc/sv/sddm /var/service/
```

### Uninstall KDE Plasma (Recommended!)
```bash
sudo unlink /var/service/sddm
sudo ln -s /etc/sv/lightdm /var/service
```

### Install Cinnamon
```bash
sudo xbps-install -S cinnamon lightdm lightdm-gtk-greeter xorg
sudo ln -s /etc/sv/lightdm /var/service/
sudo vim /etc/lightdm/lightdm.conf
```

Under `[Seat:*]:` uncomment or add the following lines:
```
user-session=cinnamon
greeter-session=lightdm-gtk-greeter
```

## Quick xbps Reference
```bash
# Install package
xbps-install -S package1 package2
# -S: synchronize package index from repos
# -y: assume/answer yes to prompts
# -u: update installed packages
```

```bash
# Remove packages (2, including dependencies)
xbps-remove -R packagename packagename2
xbps-remove -R -Rvd packagename packagename2
# -R: recursive
# -v: verbose
# -d: dry run; show what would be done without making changes
# -O: clean orphaned packages/files
```

```bash
# Search for packages
xbps-query -Rs searchterm
# -s: search
```

```bash
# List installed packages / repos
xbps-query -lL
xbps-query -L
# -l: installed packages
# -L: configured repositories
```

## Useful packages

Terminal
- tilix

OS Backups
- timeshift

Task Manager
- mate-system-monitor
- glance
- htop

Text Editors
- gedit # notepad-like
- abiword # wordpad-like (buggy)

Multimedia Player
- vlc

Audio Player
- {TBD}

Image Editor
- GIMP

Image Viewer
- gwenview

Screenshots
- flameshot

Dev (everyone)
- git
- python3-pip
- python3-pipx

Emoji Keyboard
- rofi-emoji

Clipboard Manager
- clipit

Dev (elopers)
- vscode (restricted package)
- gcc
- gdb
- make
- binutils
- valgrind

Keyboard Shortcuts (as needed)
- dconf-editor

// https://www.reddit.com/r/voidlinux/comments/1dgi4ot/cant_install_livecd_doesnt_boot/?share_id=UPUHdBLZYQghynqLn61HO&utm_content=1&utm_medium=ios_app&utm_name=ioscss&utm_source=share&utm_term=1

## Emoji Support
Noto fonts are probably the easiest to get set up.
```bash
sudo xbps-install -S noto-fonts-cjk noto-fonts-emoji noto-fonts-ttf noto-fonts-ttf-extra
```

## Pipewire setup
TODO run through setup process / cleanup ; need config files from pulseaudio

```bash
xbps-install -S pipewire
mkdir -p /etc/pipewire/pipewire.conf.d
ln -s /usr/share/examples/wireplumber/10-wireplumber.conf /etc/pipewire/pipewire.conf.d
ln -s /usr/share/examples/pipewire/20-pipewire-pulse.conf /etc/pipewire/pipewire.conf.d

sudo mkdir -p /etc/sv/pipewire
sudo mkdir -p /etc/sv/wireplumber

sudo tee /etc/sv/pipewire/run <<EOF
#!/bin/sh
exec /usr/bin/pipewire
EOF

sudo tee /etc/sv/wireplumber/run <<EOF
#!/bin/sh
exec usr/bin/wireplumber
EOF

sudo chmod +x /etc/sv/pipewire/run
sudo chmod +x /etc/sv/wireplumber/run

sudo ln -s /etc/sv/pipewire /var/service/
sudo ln -s /etc/sv/wireplumber /var/service/
```

pactl info
PulseAudio on pipewire ...
pavucontrol

Fix after crashes:
```bash
sudo sv stop pipewire & sudo sv stop wireplumber
sudo sv start pipewire ; sudo sv start wireplumber
```

## Keyring
```bash
xbps-install -S gnome-keyring
```

Keyring stores encrypted secrets/passwords/keys for you. Useful so you don't need to reauthenticate e.g. every time you git push from VSCode.

## Install Restricted Packages (e.g. discord)
### Setup
```bash
echo XBPS_ALLOW_RESTRICTED=yes >> etc/conf # Allow building restricted pkgs
git clone https://github.com/void-linux/void-packages.git
cd void-packages
./xbps-src binary-bootstrap
```

```bash
./xbps-src pkg discord vscode
./xbps-src -h # optional: list available targets/options
sudo xbps-install --repository hostdir/binpkgs/nonfree discord
# Might be in just hostdir/binpkgs depending on the package; discord's in nonfree
```

e.g. to install discord:
```bash
./xbps-src pkg discord
sudo xbps-install --repository hostdir/binpkgs/nonfree discord
# Might be in just hostdir/binpkgs depending on the package; discord's in nonfree
Discord
# or search installed applications from the desktop
```

If you want to update from void-packages:
```bash
git pull
./xbps-src update-sys # all packages
xbps-install <package name>
./xbps-src pkg discord # one specific package
xbps-install discord
```

#### Discord Fix
Discord won't launch if there's an update available (even if it's not yet available through xbps!). To prevent this behavior:

```bash
vim ~/.config/Discord/settings.json
# Add this line:
"SKIP_HOST_UPDATE": true
```

### Install Flatpaks
Flatpaks are supposed to make things easier for developers; all dependencies are sandboxed and consistent across distributions; packages will take up less space the more you have installed. Sometimes limited by their sandbox.

```bash
sudo xpbs-install -S flatpak
flatpak remote-add --if-not-exists flathub https://dl.flathub.org/repo/flathub.flatpakrepo
reboot
flatpak install flathub <package-name>
flatpak run <package-name>
```

Note: the application won't appear in your list of programs TODO: show how to add it

## Basic Customization & Shortcuts (Cinnamon)
### Emoji Keyboard
```bash
sudo xbps-install -S rofi-emoji python3-pipx
mkdir -p ~/.config/rofi
cat <<EOF > ~/.config/rofi/config.rasi
configuration {
    modi: "window,drun,run,emoji";
}
EOF
pipx install rofimoji ensurepath
```

### Startup Applications
e.g. Discord
```bash
mkdir -p ~/.config/autostart
cat <<EOF > ~/.config/autostart/Discord.desktop
[Desktop Entry]
Type=Application
Exec=Discord
Hidden=false
NoDisplay=false
X-GNOME-Autostart-enabled=true
Name=Discord
Comment=Discord app
EOF
```

### Keyboard Shortcuts in cinnamon-settings
Change these:
- Sound icon in tray -> Configure: Show Menu: (None)
- General -> Troubleshooting: Toggle Looking Glass: change to Super+K
- System: Lock Screen: Super+L
- Launchers: Home folder -> None
Add these under Custom commands:
- Super+Shift+S: `flameshot gui`
- Super+I: `cinnamon-settings`
- Super+E: `nemo`
- Ctrl+Shift+Esc: `mate-system-monitor`
- Super+.: `/home/<USER>/.local/bin/rofimoji`
    - `$USER` does not work, not sure why

### Right-click menu actions
TODO: start menu - run as Root
TODO: show how 7zip shortcut implemented

### Theme
- Applications: Material-Palenight-BL-LB
- Icons: Wings-Dark-Icons
- Desktop: Material-Palenight-Bl-LB

### Fonts
What I have set up in `cinnamon-settings`:
- Default: Roboto Regular 10
- Desktop: Roboto Regular 10
- Document: Montserrat Regular 11
- Monospace: Source Code Pro Regular 10
- Window title: Roboto Bold 10

Organize your fonts however you wish in `/usr/share/fonts`

### (Cinnamon) Set Terminal shortcut to open your Terminal app
```bash
dconf-editor /org/cinnamon/desktop/applications/terminal/
```

Change the command to match the one to run your terminal (e.g. tilix for Tilix)

### What to do if you typo your startup passphrase
Power cycle your machine. This ain't bitlocker, pal.

TODO: void mklive 