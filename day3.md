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

Rescur installで起動すると，ディスク起動時のファイルシステムが
/mnt/sysimage配下にmountされます．

    /dev/mapper/VolGroup-lv_root /mnt/sysimage
    /dev/sda1 /mnt/sysimage/boot
    /dev /mnt/sysimage/dev
    /dev/devpts /mnt/sysimage/dev/pts
    /dev/tmpfs /mnt/sysimage/dev/shm
    /dev/proc /mnt/sysimage/proc
    /dev/sysfs /mnt/sysimage/sys
    /selinux /mnt/sysimage/selinux

そこで/mnt/sysimage以下をunmountして

    # umount /mnt/sysimage/selinux
    # umount /mnt/sysimage/sys
    # umount /mnt/sysimage/proc
    # umount /mnt/sysimage/dev/shm
    # umount /mnt/sysimage/dev/pts
    # umount /mnt/sysimage/dev
    # umount /mnt/sysimage/boot
    # umount /mnt/sysimage

VolGroup-lv_rootを10GBに縮小してみましょう．縮小する際には
e2fskeを実行する必要があります．

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

10Gと指定しているのに9.9Gとなっていますがこれは単位系の問題で，
2^10を基本とするのか，それとも1000を基本とするのかの問題だと
思うので気にしないことにします．

それではLVMを縮小してみます．LVMではPV（物理ボリューム）を
パーティション単位で管理し，このPVを組み合わせてVM
（ボリュームグループ）を作ります．VMは従来のディスクに相当する
管理単位になり，この中からLV（論理ボリューム）を切り分けます．

CentOS6の標準インストールではVolGroupというVGを作っていて，

    $ sudo vgscan
      Reading all physical volumes.  This may take a while...
      Found volume group "VolGroup" using metadata type lvm2

その中に物理媒体が/dev/sda2のPVがひとつあります．

    # pvscan
      PV /dev/sda2  VG VolGroup  lvm2 [ 19.51 GiB / 0 free]
      Total: 1 [19.51 GiB] / in use:  1 [19.51 GiB] / in no VG: 0 [0 ]

その中にlv_rootとlv_swapというLVを作っています．

    $ sudo lvscan
      ACTIVE            '/dev/VolGroup/lv_root' [15.57 GiB] inherit
      ACTIVE            '/dev/VolGroup/lv_swap' [3.94 GiB] inherit

lv_rootというPVを10GBに縮めると，

    # lvreduce -L 10G /dev/mapper/VolGroup-lv_root
    # pvscan
      PV /dev/sda2  VG VolGroup  lvm2 [ 19.51 GiB / 5.57 GiB free]
      Total: 1 [19.51 GiB] / in use:  1 [19.51 GiB] / in no VG: 0 [0 ]

5.57GBのfree領域ができました．これではちょっと小さいと思うので
もう少しfree領域を増やしてみましょう．またe2fsckをかけて
resize2fsを実行します．こんどは6Gまで縮めてみます．

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

これぐら開けておけばゲストOSを色々作ってみても大丈夫でしょう．
ファイルシステムを小さくしたので，次はLVを縮小します．

    # lvreduce -L 6G /dev/mapper/VolGroup-lv_root
    # lvscan
      ACTIVE '/dev/VolGroup/lv_root' [6.00 GiB] inherit
      ACTIVE '/dev/VolGroup/lv_swap' [1.97 GiB] inherit
    # pvscan
      PV /dev/sda2  VG VolGroup  lvm2 [ 19.51 GiB / 9.57 GiB free]
      Total: 1 [19.51 GiB] / in use:  1 [19.51 GiB] / in no VG: 0 [0 ]

9.57GBの空き領域ができたのでここに新しくlv_dataというLVを
作成することにします．作成したらext4でmkfsを実行します．
Rescue installではexitすると再起動するのでexitをコマンドします．

    # lvcreate -L 9.57G -n lv_data VolGroup
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

起動ディスクをDVDのままにしておくとまたDVDから起動してしまうので
デバイス選択をハードディスクに戻してから再起動させます．

* [起動ディスク]
    * 仮想マシンの起動に使用するデバイスを選択してください
        * ハードディスク(SCSI)

起動したら新しく作成したLV用のマウントポイントを作成し，/etc/fstabを
修正します．sheepdog用にuser_xattrを付けるのを忘れずに．
第5列の数字はファイルシステムをdumpするかどうかのフラグです．sheepdog
用のログなどもこのファイルシステムに入るのでここは1を指定しました．
第6列の数字は起動時にfsckを行うかどうかのフラグで，ここではルートファイル
システム以外でfsckを行うことを意味する2を指定しています．
設定したら再起動して，うまく反映されているかを確認しましょう．

   $ sudo mkdir /data
   $ diff .fstab fstab
   9a10
   > /dev/mapper/VolGroup-lv_data /data  ext4    defaults,user_xattr 1 2
   $ sudo reboot

ディスクの用意ができたらcorosyncを設定して動かしてみましょう．といっても
まだ1台しかないので最低限の機能だけ動かします．必要な設定は，使用する
ネットワークアドレス設定と

    $ cd /etc/corosync
    $ sudo cp corosync.conf.example corosync.conf
    $ diff corosync.conf.example corosync.conf
    10c10
    <           bindnetaddr: 192.168.1.1
    ---
    >           bindnetaddr: 172.16.3.0

OS起動時のオートスタートです．設定できたら再起動して
動作確認します．

    $ sudo chkconfig corosync on
    $ sudo reboot

ここからはsheepdog用の設定です．sheepdogはローカルホストのTCP:7000
にアクセスするのでiptablesに設定を追加します．

    $ sudo /etc/sysconfig
    $ sudo diff .iptables iptables
    10a11
    > -A INPUT -m state --state NEW -m tcp -p tcp --dport 7000 -j ACCEPT
    
起動スクリプトが/var/lib/sheepdogをsheepdog用のマウントポイントとして
いるので，これを今回設定した/dataに変更し，/etc/rc3.d, rc5.dのK79sheepdog
をS79sheepdogに名称変更します．

    $ cd /etc/init.d
    $ diff .sheepdog sheepdog
    80c80
    <           $prog -p 7000 /var/lib/sheepdog > /dev/null 2>&1
    ---
    >           $prog -p 7000 /data > /dev/null 2>&1
    
    $ cd /etc/rc.d
    $ sudo mv rc3.d/K79sheepdog rc3.d/S79sheepdog
    $ sudo mv rc5.d/K79sheepdog rc5.d/S79sheepdog
    $ sudo /etc/init.d/sheepdog start
    Starting Sheepdog QEMU/KVM Block Storage (sheep):          [  OK  ]
    $ sudo reboot

sheepdogが正しく動いてるかどうかは以下のコマンドで確認できます．

    $ collie node list
       Idx - Host:Port              Number of vnodes
    ------------------------------------------------
    *    0 - 172.16.3.140:7000        64

format コマンドでディスクを何重にコピーするかを指定すればsheepdogの
準備は終了です．ここではマシンが1台しかないのでcopies=1を指定しています．

    $ collie cluster format --copies=1
    $ collie cluster info
    running
    
    Ctime                Epoch Nodes
    2013-05-07 18:30:30      1 [172.16.3.140:7000]
    $ collie node info
    Id     Size     Used     Use%
     0     9.3 GB     0.0 MB       0%
    
    Total     9.3 GB     0.0 MB       0%, total virtual VDI Size     0.0 MB

sheepdogの準備ができたのでゲストOSを作ってみましょう．kvm-qemuは
qemuに統合されているので，従来kvmで行っていたやり方とコマンド体系が
違っているのに気をつけてください．例えば，qemu-system-x86_64という
コマンドを使います．

    $ qemu-img create sheepdog:papeete 1G
    Formatting 'sheepdog:papeete', fmt=raw size=1073741824
    
これでゲストOS用のファイル領域が用意できたので，インストールしてみましょう．
まず，インストール用のisoイメージをmacからコピーします．

    $ scp vyatta-livecd_VC6.5R1_amd64.iso tahiti:~/isos/

qemuはローカルファイルをcdromとして認識できるので以下のコマンドで
vyattaのlivecdを実行することができます．

    $ qemu-system-x86_64 -boot d \
    > -drive file=sheepdog:papeete,cache=writeback \
    > -cdrom /home/kei/isos/vyatta-livecd_VC6.5R1_amd64.iso \
    > -nographic

この部分を書いていて気づいたのですが，このように実行するとemulation
がqemuで動くので遅いです．本来は-enable-kvm オプションを付けて
kvmで動かす必要があります．

それから起動時にエラーメッセージに出るかどうか気をつけてみて
ください．どうも使っているqemu rpmにはバグがあるようで，
sgabios.binという倍なりファイルがrpmに梱包されていないようです．
この場合以下のようなエラーメッセージがでます．

    Could not open option rom 'sgabios.bin': No such file or directory

textモードでインストールや操作をするのであれば問題ないはずですが，
念を入れるのであれば

    $ sudo yum install sgabios
    $ sudo ln -s /usr/share/sgabios/sgabios.bin /usr/share/qemu

として，sgabios.binを/usr/share/qemu配下に追加すればエラーはでなく
なります．

ここからはvyattaとqemuに起因する現象との格闘になります．こんなエラーを
吐いたり

    [2.579331] ioremap error for 0x7fff000-0x8000000, requested 0x10, got 0x0

grumメニューを選択してからlinuxカーネルが動き出すまで『ものすごく』時間が
かかることがあります．何か余計なドライバーを読み込もうとしているのだと
思うのですが，条件を突き止めることができていません．以前のkvm-qemuでは
問題なく動いていたので，ここにもqemuのバグがある可能性があります．

さて，ここからはvyattaのインストールになります．

* vyatta login:
    * vyatta  管理権限ユーザ
* Password:
    * vyatta  初期設定パスワード

ディスクへのインストールを実行します．基本的にデフォルト設定のまま
進めばインストールできます．

    vyatta@vyatta:~$ install system

* Would you like to continue? (Yes/No) [Yes]:
    * Yes
* Partition (Auto/Union/Parted/Skip) [Auto]:
    * Auto
* Install the image on? [sda]:
    * sda
* Continue? (Yes/No) [No]:
    * Yes
* How big of a root partition should I create? (1000MB - 1073MB) [1073]MB:
    * 1073
* Which one should I copy to sda? [/opt/vyatta/etc/config/config.boot]:
    * /opt/vyatta/etc/config/config.boot
* Enter password for user 'vyatta':
    * パスワードを設定してください
* Retype password for user 'vyatta':
    * パスワードを再記入してください
* Which drive should GRUB modify the boot partition on? [sda]:
    * sda

ゲストOSを終了するにはpoweroffコマンドを実行します．

   vyatta@vyatta:~$ sudo poweroff

正しくインストールできたかどうか確認するため，もう一度実行して
みます．今回はハードディスクからの起動です．

    $ qemu-system-x86_64 -drive \
    > file=sheepdog:pateete,cache=writeback -nographic
 
Loginメッセージが表示されればインストールは成功です．

## まとめ

Fusionのゲストとして作成したCentOSをホストとしてvyattaをゲストとして
動かすことができました．ただし，本来3台以上でデータを分散管理する
sheepdogを1台で管理するように設定していますし，ネットワーク設定も
行っていません．ネットワーク設定はlibvirt経由で行います．今回
使っているqemuにはいくつかバグがあるようなので，地道に迂回策を
探して使っています．
