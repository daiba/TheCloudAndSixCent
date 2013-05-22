# ホストを3台構成で動かす

## はじめに

ここまではVMWare Fusion上の1台のサーバ上でゲストOSを動かして
きました．ここではsheepdogの標準構成である3台のサーバを使って
ゲストOSを動かすところまで説明します．

## バージョン

qemuからsheepdogを使うにはqemuが1.4.1以上でないとゲストOS終了時に
キャッシュのフラッシュをしないバグがあったり，sheepdogがバージョンと
供に機能追加されていることを知ったので使うrpmのバージョンをあげて
みました．使ったのは

* openvswitch-1.10.0-1.x86_64.rpm
* qemu-1.4.1-2013.05.17.x86_64.rpm
* sheepdog-0.5.5_290_g682ebbc-1.el6.x86_64.rpm

と，これらに関連するrpm群で自前で作成しました．ちなみに使用した
CentOSのkernelは2.6.32-358.6.2.el6.x86_64です． 

## インストール手順

基本的にはここまでの手順を踏襲すればいいのですが，rpmの依存関係が
変わっているので事前にインストールしなければならないrpmが変わります．

    $ sudo rpm -ivh kmod-openvswitch-1.10.0-1.el6.x86_64.rpm
    $ sudo yum -y install libpcap
    $ sudo rpm -ivh openvswitch-1.10.0-1.x86_64.rpm
    $ sudo yum -y install libaio alsa-lib bluez-libs \
      esound-libs gnutls pixman libjpeg-turbo pulseaudio-libs
    $ sudo rpm -ivh qemu-1.4.1-2013.05.17.x86_64.rpm
    $ sudo rpm -ivh qemu-img-1.4.1-2013.05.17.x86_64.rpm
    $ sudo yum -y install fuse-libs
    $ sudo rpm -ivh sheepdog-0.5.5_290_g682ebbc-1.el6.x86_64.rpm
    $ sudo yum -y install avahi-libs dbus dmidecode \
      dnsmasq ebtables iscsi-initiator-utils libcgroup \
      lzop nfs-utils numad parted polkit radvd cyrus-sasl-md5 \
      gettext gnutls-utils nc pm-utils libnl numactl yajl netcf
    $ sudo rpm -ivh libvirt-daemon-1.0.3-1.el6.x86_64.rpm \
      libvirt-daemon-config-network-1.0.3-1.el6.x86_64.rpm \
      libvirt-daemon-config-nwfilter-1.0.3-1.el6.x86_64.rpm \
      libvirt-daemon-driver-lxc-1.0.3-1.el6.x86_64.rpm \
      libvirt-daemon-driver-qemu-1.0.3-1.el6.x86_64.rpm \
      libvirt-daemon-driver-interface-1.0.3-1.el6.x86_64.rpm \
      libvirt-daemon-driver-secret-1.0.3-1.el6.x86_64.rpm \
      libvirt-daemon-driver-storage-1.0.3-1.el6.x86_64.rpm \
      libvirt-daemon-driver-network-1.0.3-1.el6.x86_64.rpm \
      libvirt-daemon-driver-nodedev-1.0.3-1.el6.x86_64.rpm \
      libvirt-daemon-driver-nwfilter-1.0.3-1.el6.x86_64.rpm \
      libvirt-client-1.0.3-1.el6.x86_64.rpm \
      libvirt-1.0.3-1.el6.x86_64.rpm

これで必要なrpmは終わりです．この作業を3台のホストzebra, yellow,
xerxesで実行しました．openvswitchを使うためにifcfg-eth0, ifcfg-ovsbr0
libvirtからovsbr0を使うためのovsbr0.xmlは前と同じ物を使います．

## 設定

まずcorosyncの設定を作り

    $ cd /etc/corosync
    $ sudo cp corosync.conf.example corosync.conf
    $ diff corosync.conf.example corosync.conf
    10c10
    <           bindnetaddr: 192.168.1.1
    ---
    >           bindnetaddr: 172.16.3.0

次にcorosyncが自動起動するよう設定します．

    $ sudo chkconfig corosync on

さらに，sheepdog用のTCP:7000向けの通信とcorosync用の226.94.1.1への
TCP:5405向けの通信を通すよう設定します．

    $ sudo diff .iptables iptables
    10a11,12
    > -A INPUT -m state --state NEW -m tcp -p tcp --dport 7000 -j ACCEPT
    > -A IP-INPUT -s 0.0.0.0 -d 226.94.1.1 -m pkt-type multicast -m state --state NEW -m tcp -p tcp --dport 5405 -j ACCEPT

後はsheepdogの起動パラメータを指定します．前回は/以外のファイル
システムを使ったのでmount時のパラメータにuser_xattrを指定していましたが
/ ファイルシステム内のディレクトリであればオプションの指定は必要
ありません．wは最近入ったオプションでwriteback cacheを使う際の
キャッシュサイズをMB単位で指定します．

    $ sudo diff .sheepdog sheepdog
    80c80
    <           $prog -p 7000 /var/lib/sheepdog > /dev/null 2>&1
    ---
    >           $prog -p 7000 -w size=256 /var/lib/sheepdog > /dev/null 2>&1

とはいえ，なにかおかしいようでwオプションをいれると挙動が不安定です．
入れない方が得策です．後はsheepdogが自動起動するよう設定を入れて
再起動します．

    $ sudo chkconfig sheepdog on
    $ sudo reboot

うまくsheepdogが起動できない事があるので，起動時にsheepプロセスが
見当たらないときには手動で起動してください．

    $ sudo /etc/init.d/sheepdog start

sheepdogをファイルコピー数を3に指定してフォーマットすれば使える
ようになります．

    $ collie cluster format --copies=3
    using backend farm store
    $ collie cluster info
    Cluster status: running
    
    Cluster created at Tue May 21 14:12:00 2013
    
    Epoch Time           Version
    2013-05-21 14:12:01      1 [172.16.3.147:7000, 172.16.3.148:7000, 172.16.3.149:7000]

ここではzebra, yellow, xerxesの3台のマシンが見えています．

sheepdogの次はqemu用の設定です．kvmを使うためにはkvmとkvm_intel
というカーネルモジュールを予め組み込んでおく必要があり，
さらに，/dev/kvmという特殊ファイルのグループ権限がkvmに
なっている必要があります．そこで/etc/init.d/mykvmというファイルを
作成して，起動時に必ず設定するようにしてみました．

    # cat /etc/init.d/mykvm
    #!/bin/bash
    #
    
    . /etc/init.d/functions
    
    if [ "$1" != "start" ]; then
      exit 0
    fi
    
    modprobe kvm
    modprobe kvm_intel
    chown root:kvm /dev/kvm

このファイルに対して

    $ sudo chown root:root /etc/init.d/mykvm
    $ sudo chmod 755 /etc/init.d/mykvm
    $ sudo chkconfig --add mykvm
    $ sudo chkconfig mykvm on

とコマンドしておけば次回リブート時からqemuからkvmを使うことが
できるようになります．ちなみにFusionでintel-vtの設定がうまく
できていないと，

    $ sudo modprobe kvm_intel
    FATAL: Error inserting kvm_intel (/lib/modules/2.6.32-358.6.2.el6.x86_64/kernel/arch/x86/kvm/kvm-intel.ko): Operation not supported

というエラーが発生します．ここまでできたら，ファイルシステムを作って
ゲストOSをインストールしておきましょう．

    $ qemu-img create sheepdog:monkey 1G
    $ sudo qemu-system-x86_64 -boot d -enable-kvm -drive file=sheepdog:monkey,cache=writeback -cdrom /home/kei/isos/vyatta-livecd_VC6.5R1_amd64.iso -nographic

インストールできたら一旦終了して，libvirtにmonkey_tmp.xmlを登録します．

    $ cat monkey_tmp.xml 
    <domain type='kvm'>
      <name>monkey</name>
      <title>qemu, openvswitch and sheepdogs ver.</title>
      <memory unit='GB'>1</memory>
      <currentMemory unit='GB'>1</currentMemory>
      <vcpu placement='static'>1</vcpu>
      <os>
        <type arch='x86_64' machine='pc'>hvm</type>
        <boot dev='hd'/>
        <bootmenu enable='no'/>
      </os>
      <features>
        <acpi/>
      </features>
      <clock offset='localtime'/>
      <on_poweroff>destroy</on_poweroff>
      <on_reboot>restart</on_reboot>
      <on_crash>destroy</on_crash>
      <devices>
        <emulator>/usr/bin/qemu-system-x86_64</emulator>
        <disk type='network' device='disk'>
          <driver name='qemu' type='raw' cache='writethrough' io='threads'/>
          <source protocol='sheepdog' name='monkey'>
            <host name='127.0.0.1' port='7000'/>
          </source>
          <target dev='vda' bus='virtio'/>
        </disk>
        <interface type='bridge'>
          <source bridge='ovsbr0'/>
          <virtualport type='openvswitch'/>
          <model type='virtio'/>
        </interface>
        <serial type='pty'>
          <target port='0'/>
        </serial>
        <console type='pty'>
          <target type='serial' port='0'/>
        </console>
        <membaloon model='virtio'/>
      </devices>
    </domain>
    $ sudo virsh define monkey_tmp.xml
    $ sudo virsh start monkey
    $ sudo virsh console monkey

vyattaが起動したら基本的な設定をいれておきます．ゲストOSの
名前はmonkeyです．

    $ configure
    # set system host-name monkey
    # set system time-zone Asia/Tokyo
    # set interfaces ethernet eth1 address dhcp
    # commit
    # save
    # exit

これでゲストOSが1台動く環境ができあがりました．

## まとめ

qemu, openvswitch, sheepdog, libvirtを新しい物に入れ替えてcentos6上で
動かしました．rpmの作り方はまた別途説明しますが，READMEなどに記載されて
いた手順では動かなくて手を加えています．init.dを使ってsheepdogがうまく
起動できない点やkvmのアクセス権に関しては本来どのような手順をたどるべき
なのか，調べる必要がありそうです．
