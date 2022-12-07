ArchLinux Install 
-----------------
date: 	2022-12-5<br>
auther: lesen & huang<br>
description: 
	Record the whole process of installing archiso.

## Install Opertion Manual

### Connetion WIFI
```terminal
default zsh
root@archiso ~ # bash
[root@archiso ~]# zsh
root@archiso ~ # iwctl
iwctl> device list
iwctl> station wlan0 get-networks
iwctl> station wlan0 connect "CMCC-z75t-5G"
iwctl> passphrase:******  #PASSWORD
iwctl> station wlan0 scan
iwctl> station wlan0 show
iwctl> q #Ctrl+d
```
### Three Kinds Diskpart
1. GPT Partition encrypt
| Device | Size | Type |
|:--------:|:--:|:---------:|
|/dev/sda1 |512M|EFI System|
|/dev/sda2 |512M|Linux filesystem #boot|
|/dev/sda3 |100%free|Linux lvm #encrypt|

2. GPT Partition discrypt
| Device | Size | Type |
|:------:|:----:|:----:|
|/dev/sda1 |512M|EFI System|
|/dev/sda2 |100%free|Linux lvm|

3. MBR Partition
| Device | Size | Type |
|:------:|:----:|:----:|
|/dev/sda1 |100%free|Linux lvm|

### Disk Dispatch

1. Partition
```terminal
root@archiso ~ # fdisk /dev/sd[a-z]
fdisk> gpt dos#MBR
fdisk> p #print disk info
fdisk> n #new partition
fdisk> t #partiton type
fdisk> w #store
```
2. Encrypt Partition sda3

```terminal
root@archiso ~ # cryptsetup luksFormat /dev/sda3
...
Are you sure? :YES
Enter passphrase for /dev/sda3: 
Verfiy passphrase:
...
root@archiso ~ # cryptsetup open --type luks /dev/sda3 lvm
Enter passphrase for /dev/sda3: 
root@archiso ~ # pvcreate --dataalignment 1m /dev/mapper/lvm
root@archiso ~ # vgcreate volgroup0 /dev/mapper/lvm
```

### Create LVM
```terminal
root@archiso ~ # partprobe 
root@archiso ~ # pvcreate --dataalignment 1m /dev/sda[1-2]
root@archiso ~ # vgcreate volgroup0 /dev/sda[1-2]
root@archiso ~ # lvcreate -L 60G volgroup0 -n lv_root
root@archiso ~ # lvcreate -l 100%free volgroup0 -n lv_home
root@archiso ~ # modprobe dm_mod #device_mapper
root@archiso ~ # vgscan
root@archiso ~ # vgchange -ay
```

1. Format Partition And LVM
2. Mount Device to Directory
3. Write Mount info to /mnt/etc/fstab 

```terminal
root@archiso ~ # mkfs.fat -F32 /dev/sda1
root@archiso ~ # mkfs.xfs /dev/sda2
root@archiso ~ # mkfs.xfs /dev/volgroup0/lv_root
root@archiso ~ # mkfs.xfs /dev/volgroup0/lv_home
root@archiso ~ # mount /dev/volgroup0/lv_root /mnt
root@archiso ~ # mkdir /mnt/home
root@archiso ~ # mkdir /mnt/boot
root@archiso ~ # mkdir /mnt/etc
root@archiso ~ # mount /dev/volgroup0/lv_home /mnt/home
root@archiso ~ # mount /dev/sda2 /mnt/boot
root@archiso ~ # genfstab -U -p /mnt >> /mnt/etc/fstab
```

### To Archlinux / Directory down Install base and Switch Archlinux.
```terminal
root@archiso ~ # pacman -S archlinux-keyring #if oldiso neet update key
root@archiso ~ # pacstrap -i /mnt base #only install base, can arch-chroot.
root@archiso ~ # arch-chroot /mnt
```

### Install Linux Kernel.
```terminal
[root@archiso /]# pacman -S vim emacs nano
[root@archiso /]# pacman -S base-devel
[root@archiso /]# pacman -S linux linux-lts linux-headers linux-lts-headers
```

### If have lvm2 encrypt Module, Add to /etc/mkinitcpio.conf Configure File.
```terminal
[root@archiso /]# pacman -S lvm2
[root@archiso /]# vim /etc/mkinitcpio.conf
...
#		usr, fsck and shutdown hooks.
HOOKS=(base udev autodetect modconf block lvm2 encrypt filesystems keyboard fsck)
#	COMPRESSION
...
:wq
```

### Use XFileSystem neet Install xfsdump, then Make.
```terminal
[root@archiso /]# pacman -S xfsdump
[root@archiso /]# mkinitcpio -p linux #Or linux-lts
```

### Install Network Tools and Remote SSH, Set and Create User, Password.
```terminal
[root@archiso /]# pacman -S openssh networkmanager dialog
[root@archiso /]# systemctl enable NetworkManager sshd
[root@archiso /]# pacman -S wpa_supplicant wireless_tools netctl
[root@archiso /]# vim /etc/locale.gen #set encoded.
[root@archiso /]# locale-gen
[root@archiso /]# useradd -m -g users -G wheel lesen
[root@archiso /]# echo lesen | passwd --stdin lesen #root
[root@archiso /]# pacman -S sudo
[root@archiso /]# visudo
...
## Uncomment to allow members of group wheel to execute any command
%wheel ALL=(ALL) ALL
...
:wq
```

### Install Boot
```terminal
[root@archiso /]# pacman -S grub dosfstools os-prober mtools efibootmgr
[root@archiso /]# mkdir /boot/EFI
[root@archiso /]# mount /etc/sda1 /boot/EFI
[root@archiso /]# grub-install --target=i386-pc --recheck /dev/sda 
Installing for i386-pc platform.
[root@archiso /]# grub-install --target=x86_64-efi --bootloader-id=grub_uefi --recheck
Installing for x86_64-efi platform.
Installing finished. No error reported.
[root@archiso /]# cp /usr/share/local/en\@quot/LC_MESSAGES/grub.mo /boot/grub/local/en.mo
```
### Tell Grub Which a Device is Encrypt.
```terminal
[root@archiso /]# vim /etc/default/grub
...
GRUB_CMDLINE_LINUX_DEFAULT="cryptdevice=/dev/sda3:volgroup0:allow-discards loglevel=3 quiet"
GRUB_ENABLE_CRYPTODISK=y
...
:wq
```
### Continue build Grub Configure file.
```terminal
[root@archiso /]# grub-mkconfig -o /boot/grub/grub.cfg
...
done
[root@archiso /]# exit
exit
root@archiso ~ # umount -a
...
32 root@archiso ~ # reboot
```

### Configure SWAP and Timesync
```terminal
[root@archlinux ~]# dd if=/dev/zero of=/swap bs=1M count=2048 status=progress
[root@archlinux ~]# chmod 600 /swap
[root@archlinux ~]# mkswap /swap
[root@archlinux ~]# echo "/swap none swap sw 0 0" | tee -a /etc/fstab
[root@archlinux ~]# swapon -a
[root@archlinux ~]# free -h
[root@archlinux ~]#
[root@archlinux ~]# timedatectl list-timezones
[root@archlinux ~]# timedatectl set-timezone Asia/Shanghai
[root@archlinux ~]# systemctl restart systemd-timesyncd
[root@archlinux ~]# timedatectl set-ntp true
```
