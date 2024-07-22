# Void Linux install

Backup reference of [this page](https://jpedmedia.com/tutorials/installations/void_install/index.html) with comments. Installing void linux (chroot method) with encryption and btrfs.

`sdX` == the drive to partition. `Xn` == the partition on sdX.

## Create Partitions

```bash
fdisk /dev/sdX
```

```bash
Create GPT partition table
Command (m for help): g

Create EFI System Partition (ESP)
Command (m for help): n
Partition number (1-128, default 1): (enter for default)
First sector (2048-500118158, default 2048): (enter for default)
Last sector, +/-sectors or +/-size{K,M,G,T,P}...): +200M (choose 128M to 1G)
Do you want to remove the signature? [Y]es/[N]o: y
Command (m for help): t
Partition type or alias (type L to list all): 1

Create the root partition
Command (m for help): n
Partition number (2-128, default 2): (enter for default)
First sector (2099200-500118158, default 2099200): (enter for default)
Last sector, +/-sectors or +/-size{K,M,G,T,P}...): (enter for default)

Write changes to the disk
Command (m for help): w
```

## Setup Encryption & Subvolumes

```bash
cryptsetup luksFormat --type luks1 -y /dev/sdX2
cryptsetup open /dev/sdX2 cryptvoid
mkfs.fat -F32 -n EFI /dev/sdX1
mkfs.btrfs -L Void /dev/mapper/cryptvoid
BTRFS_OPTS="rw,noatime,compress=zstd,discard=async"
mount -o $BTRFS_OPTS /dev/mapper/cryptvoid /mnt 
btrfs su cr /mnt/@
btrfs su cr /mnt/@home
btrfs su cr /mnt/@snapshots
umount /mnt
mount -o $BTRFS_OPTS,subvol=@ /dev/mapper/cryptvoid /mnt
mkdir /mnt/{efi,home,.snapshots}
mount -o $BTRFS_OPTS,subvol=@home /dev/mapper/cryptvoid /mnt/home
mount -o $BTRFS_OPTS,subvol=@snapshots /dev/mapper/cryptvoid /mnt/.snapshots
mkdir -p /mnt/var/cache
btrfs su cr /mnt/var/cache/xbps
btrfs su cr /mnt/var/tmp
btrfs su cr /mnt/srv
mount -o rw,noatime /dev/sdX1 /mnt/efi
lsblk
```

## Installation

```bash
REPO=https://mirrors.servercentral.com/voidlinux/current/
ARCH=x86_64
mkdir -p /mnt/var/db/xbps/keys
cp /var/db/xbps/keys/* /mnt/var/db/xbps/keys/
XBPS_ARCH=$ARCH xbps-install -S -R "$REPO" -r /mnt base-system linux-mainline btrfs-progs cryptsetup vim

for dir in dev proc sys run; do mount --rbind /$dir /mnt/$dir; mount --make-rslave /mnt/$dir; done
cp /etc/resolv.conf /mnt/etc/
BTRFS_OPTS=$BTRFS_OPTS PS1='(chroot) # ' chroot /mnt/ /bin/bash
```

### Locale, users

```bash
ls /usr/share/zoneinfo
ln -sf /usr/share/zoneinfo/America/Indianapolis /etc/localtime
vim /etc/default/libc-locales
# Uncomment your locale e.g. en_US.UTF-8, en_US ISO-8859-1

xbps-reconfigure -f glibc-locales
echo "<your chosen hostname>" > /etc/hostname
cat <<EOF > /etc/hosts
#
# /etc/hosts: static lookup table for host names
#
127.0.0.1        localhost
::1              localhost
127.0.1.1        myhostname.localdomain myhostname
EOF
passwd
useradd <USER>
passwd <USER>
usermod -aG wheel,adm,disk,lp,plugdev,audio,video,users,input <USER>
chsh -s /bin/bash root
EDITOR=vim visudo
# Uncomment %wheel ALL=(ALL:ALL) ALL
```

### Repos

```bash
xbps-install -S
xbps-install void-repo-nonfree
xbps-install -S
xbps-install void-repo-multilib
xbps-install -S
```

### fstab

```bash
EFI_UUID=$(blkid -s UUID -o value /dev/sdX1)
ROOT_UUID=$(blkid -s UUID -o value /dev/mapper/cryptvoid)
LUKS_UUID=$(blkid -s UUID -o value /dev/sdX2)

cat <<EOF > /etc/fstab
UUID=$ROOT_UUID / btrfs $BTRFS_OPTS,subvol=@ 0 1
UUID=$ROOT_UUID /home btrfs $BTRFS_OPTS,subvol=@home 0 2
UUID=$ROOT_UUID /.snapshots btrfs $BTRFS_OPTS,subvol=@snapshots 0 2
UUID=$EFI_UUID /efi vfat defaults,noatime 0 2
tmpfs /tmp tmpfs defaults,nosuid,nodev 0 0
EOF
```

### Bootloader

```bash
xbps-install grub-x86_64-efi
echo GRUB_ENABLE_CRYPTODISK=y >> /etc/default/grub
vim /etc/default/grub
# set GRUB_CMDLINE_LINUX_DEFAULT="loglevel=4 rd.auto=1 rd.luks.allow-discards"
grub-install --target=x86_64-efi --efi-directory=/efi --bootloader-id="Void"
```

### QoL

```bash
dd bs=515 count=4 if=/dev/urandom of=/boot/keyfile.bin
cryptsetup -v luksAddKey /dev/sdX2 /boot/keyfile.bin
chmod 000 /boot/keyfile.bin 
chmod -R g-rwx,o-rwx /boot 
cat <<EOF >> /etc/crypttab
cryptvoid UUID=$LUKS_UUID /boot/keyfile.bin luks
EOF
echo 'install_items+=" /boot/keyfile.bin /etc/crypttab "' > /etc/dracut.conf.d/10-crypt.conf
ln -s /etc/sv/dhc /etc/runit/runsvdir/default
```

### Install programs

```bash
xbps-install -S NetworkManager firefox tilix fastfetch dbus git
```

### Link services

```bash
ln -s /etc/sv/dhcpcd-eth0 /var/service
ln -s /etc/sv/dhcpcd /var/service
ln -s /etc/sv/NetworkManager /var/service
ln -s /etc/sv/dbus /var/service
xbps-reconfigure -fa
```

### Finish

```bash
exit
reboot
```