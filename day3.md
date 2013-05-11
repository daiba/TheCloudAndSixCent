# sheepdogをインストールする

## はじめに

kvmを使ってゲストOSを作るには，それ用のファイルシステム領域を確保する
必要があります．普通はホストOSのローカルファイルシステム上に用意する
のですが，クラウドと言うくらい複数のホストOSがあると，ゲストOSを
こちらのホストからあちらのホストへとマイグレーションしたいケースも
でてくるので，ファイルシステムをホスト間で共有できると便利です．
sheepdogはまさにそのような目的のために使う技術です．が，この章では
まず1台のOSにインストールところまでを扱います．

## sheepdog用の領域作り

sheepdogを使うにはファイルシステムを作ってそれにuser_xattr属性をつけて
mountする必要があります．Fusionのデフォルト設定で作ったファイルシステムを
変更してsheepdog用の領域を作り出します．まずは現状の確認をしておきます．

    $ mount
    /dev/mapper/VolGroup-lv_root on / type ext4 (rw)

このext4のLVMで使用されている領域を縮小して，そこに新しい
ファイルシステムを作ります．ファイルシステムを縮小するには
CentOSをインストールするときに使ったDVDから修復インストールを
行います．fusionの設定で，DVDが接続した状態にあるかどうか確認し，
さらに起動メディアをDVDに変更し再起動します．つまり，OSをshutdown
した後に以下の設定を行い

* [tahiti: 設定     起動ディスク]
    * 仮想マシンの起動に使用するデバイスを選択してください
        * CD/DVD

Rescue installとしてDVDから起動します．

* [Welcome to CentOS 6.4!]
    * Rescue installed system
* [Choose a Language]
    * English
* [Keyboard Type]
    * jp106
* [Installation Method]
    * Local CD/DVD
* [Setup Networking]
    * No
* [Rescue]
    * Continue
        * shell Start shell


// mount 情報
/dev/mapper/VolGroup-lv_root /mnt/sysimage
/dev/sda1 /mnt/sysimage/boot
/dev /mnt/sysimage/dev
/dev/devpts /mnt/sysimage/dev/pts
/dev/tmpfs /mnt/sysimage/dev/shm
/dev/proc /mnt/sysimage/proc
/dev/sysfs /mnt/sysimage/sys
/selinux /mnt/sysimage/selinux

# umount /mnt/sysimage/selinux
# umount /mnt/sysimage/sys
# umount /mnt/sysimage/proc
# umount /mnt/sysimage/dev/shm
# umount /mnt/sysimage/dev/pts
# umount /mnt/sysimage/dev
# umount /mnt/sysimage/boot
# umount /mnt/sysimage

# e2fsck -f /dev/mapper/VolGroup-lv_root
# resize2fs /dev/mapper/VolGroup-lv_root 10G
# mount /dev/mapper/VolGroup-lv_root /mnt/sysimage
# df -T -h
/dev/mapper/VolGroup-lv_root:
     Type: ext4
     Size: 9.9G
     Used: 1.2G
     Avail: 8.2G
     Use%: 13%
     Mounted on: /mnt/sysimage
# umount /mnt/sysimage


# pvscan
  PV /dev/sda2  VG VolGroup  lvm2 [ 19.51 GiB / 0 free]
  Total: 1 [19.51 GiB] / in use:  1 [19.51 GiB] / in no VG: 0 [0 ]

# lvreduce -L 10G /dev/mapper/VolGroup-lv_root


# pvscan
  PV /dev/sda2  VG VolGroup  lvm2 [ 19.51 GiB / 5.57 GiB free]
  Total: 1 [19.51 GiB] / in use:  1 [19.51 GiB] / in no VG: 0 [0 ]


// 残りが少ないからもう一回

# e2fsck -f /dev/mapper/VolGroup-lv_root
# resize2fs /dev/mapper/VolGroup-lv_root 6G
# mount /dev/mapper/VolGroup-lv_root /mnt/sysimage
# df -T -h
/dev/mapper/VolGroup-lv_root:
     Type: ext4
     Size: 6.0G
     Used: 1.2G
     Avail: 4.5G
     Use%: 21%
     Mounted on: /mnt/sysimage
# umount /mnt/sysimage
# lvreduce -L 6G /dev/mapper/VolGroup-lv_root
# pvscan

# pvscan
  PV /dev/sda2  VG VolGroup  lvm2 [ 19.51 GiB / 9.57 GiB free]
  Total: 1 [19.51 GiB] / in use:  1 [19.51 GiB] / in no VG: 0 [0 ]


# lvscan
  ACTIVE '/dev/VolGroup/lv_root' [6.00 GiB] inherit
  ACTIVE '/dev/VolGroup/lv_swap' [1.97 GiB] inherit

# lvcreate -L 9.57G -n lv_data VoGroup
  Rounding up size to full physical extent 9.57 GiB
  Logical volume "lv_data" created


# pvscan
  PV /dev/sda2  VG VolGroup lvm2 [19.51 GiB / 0 free]
  Total: 1 [19.51 GiB] / in use: 1 [19.51 GiB] / in no VG: 0 [0 ]

# lvscan
  ACTIVE '/dev/VolGroup/lv_root' [6.00 GiB] inherit
  ACTIVE '/dev/VolGroup/lv_swap' [3.94 GiB] inherit
  ACTIVE '/dev/VolGroup/lv_data' [9.57 GiB] inherit

# mkfs.ext4 /dev/VolGroup/lv_data
# exit

[reboot Reboot]

vmware でシャットダウン

[起動ディスク]
仮想マシンの起動に使用するデバイスを選択してください
     ハードディスク(SCSI)

$ sudo mkdir /data

$ diff .fstab fstab
9a10
> /dev/mapper/VolGroup-lv_data /data                   ext4    defaults,user_xattr 1 2

$ sudo reboot

$ cd /etc/corosync
$ sudo cp corosync.conf.example corosync.conf
$ diff corosync.conf.example corosync.conf
10c10
<           bindnetaddr: 192.168.1.1
---
>           bindnetaddr: 172.16.3.0

$ sudo chkconfig --list | grep corosync
corosync            0:off     1:off     2:off     3:off     4:off     5:off     6:off
$ sudo chkconfig corosync on
$ sudo chkconfig --list | grep corosync
corosync            0:off     1:off     2:on     3:on     4:on     5:on     6:off

$ sudo /etc/sysconfig
$ sudo diff .iptables iptables
10a11
> -A INPUT -m state --state NEW -m tcp -p tcp --dport 7000 -j ACCEPT

$ sudo /etc/init.d/sheepdog start
Starting Sheepdog QEMU/KVM Block Storage (sheep):          [  OK  ]

$ sudo reboot

$ collie node list
failed to connect to localhost:7000, Connection refused
failed to connect to localhost:7000, Connection refused
failed to get node list
$ sudo /etc/init.d/sheepdog start
Starting Sheepdog QEMU/KVM Block Storage (sheep):          [  OK  ]
$ collie node list
   Idx - Host:Port              Number of vnodes
------------------------------------------------
*    0 - 172.16.3.140:7000        64

$ cd /etc/init.d
$ diff .sheepdog sheepdog
80c80
<           $prog -p 7000 /var/lib/sheepdog > /dev/null 2>&1
---
>           $prog -p 7000 /data > /dev/null 2>&1

$ cd /etc/rc.d
$ sudo mv rc3.d/K79sheepdog rc3.d/S79sheepdog
$ sudo mv rc5.d/K79sheepdog rc5.d/S79sheepdog

$ collie cluster info
Waiting for a format operation

Ctime                Epoch Nodes
$ collie node info
Id     Size     Used     Use%

cannot get information from any nodes
$ collie vdi list
  name        id    size    used  shared    creation time   vdi id
------------------------------------------------------------------

$ collie cluster format --copies=1
$ collie cluster info
running

Ctime                Epoch Nodes
2013-05-07 18:30:30      1 [172.16.3.140:7000]
$ collie node info
Id     Size     Used     Use%
 0     9.3 GB     0.0 MB       0%

Total     9.3 GB     0.0 MB       0%, total virtual VDI Size     0.0 MB
$ collie vdi list
  name        id    size    used  shared    creation time   vdi id
------------------------------------------------------------------

$ sudo reboot

$ qemu-img create sheepdog:papeete 1G
Formatting 'sheepdog:papeete', fmt=raw size=1073741824

$ qemu-system-x86_64 -boot d -drive file=sheepdog:papeete,cache=writeback -cdrom /home/kei/isos/vyatta-livecd_VC6.5R1_amd64.iso -nographic
Could not open option rom 'sgabios.bin': No such file or directory

ISOLINUX 4.02 debian-20101014  Copyright (C) 1994-2010 H. Peter Anvin et al
Press control and F then 1 for help, or ENTER to
boot:
[    2.579331] ioremap error for 0x7fff000-0x8000000, requested 0x10, got 0x0

Welcome to Vyatta - vyatta ttyS0

vyatta login: vyatta
Password:
Linux vyatta 3.3.8-1-amd64-vyatta #1 SMP Mon Nov 12 12:04:26 PST 2012 x86_64
Welcome to Vyatta.
This system is open-source software. The exact distribution terms for
each module comprising the full system are described in the individual
files in /usr/share/doc/*/copyright.
vyatta@vyatta:~$ install system
Welcome to the Vyatta install program.  This script
will walk you through the process of installing the
Vyatta image to a local hard drive.

Would you like to continue? (Yes/No) [Yes]:
Probing drives: OK
Looking for pre-existing RAID groups...none found.
The Vyatta image will require a minimum 1000MB root.
Would you like me to try to partition a drive automatically
or would you rather partition it manually with parted?  If
you have already setup your partitions, you may skip this step.

Partition (Auto/Union/Parted/Skip) [Auto]:

I found the following drives on your system:
 sda     1073MB


Install the image on? [sda]:

This will destroy all data on /dev/sda.
Continue? (Yes/No) [No]: yes

How big of a root partition should I create? (1000MB - 1073MB) [1073]MB:

Creating a new disklabel on sda
parted /dev/sda mklabel msdos
Creating filesystem on /dev/sda1: OK
Mounting /dev/sda1
Copying system files to /dev/sda1:
 97% [=================================================> ]
OK
I found the following configuration files
/opt/vyatta/etc/config/config.boot
Which one should I copy to sda? [/opt/vyatta/etc/config/config.boot]:

Enter password for administrator account
Enter password for user 'vyatta':
Retype password for user 'vyatta':
I need to install the GRUB boot loader.
I found the following drives on your system:
 sda     1073MB


Which drive should GRUB modify the boot partition on? [sda]:

Setting up grub: OK
Done!
vyatta@vyatta:~$ sudo poweroff

$ qemu-system-x86_64 -drive file=sheepdog:pateete,cache=writeback -nographic

Welcome to Vyatta - vyatta ttyS0

vyatta login: vyatta
Password:
Linux vyatta 3.3.8-1-amd64-vyatta #1 SMP Mon Nov 12 12:04:26 PST 2012 x86_64
Welcome to Vyatta.
This system is open-source software. The exact distribution terms for
each module comprising the full system are described in the individual
files in /usr/share/doc/*/copyright.
vyatta@vyatta:~$ sudo power off

$ sudo yum install sgabios
$ sudo ln -s /usr/share/sgabios/sgabios.bin /usr/share/qemu
 
