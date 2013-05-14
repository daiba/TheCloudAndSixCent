# libvirtの使い方

## はじめに

qemuによる仮想サーバ，openvswitchによる仮想ネットワーク，
sheepdogによる仮想サーバ向け分散ファイルシステム．これらはそれぞれ
独自のコマンドを使って管理するのが前提ですがlibvirtを使うとこれらを
まとめて一元管理することができます．

## インストール

libvirtは標準ではsheepdogをサポートしていないので，設定を書き直した
上でrpmを作る必要があります．rpmの作り方については項を改めて説明するので，
開発環境で必要なrpmが出来上がっているという前提で話を進めます．
開発環境からrpmを移して

    $ scp libvirt* tahiti:~/rpms

libvirtが依存しているパッケージをインストールします．

    $ sudo yum -y install avahi-libs
    $ sudo yum -y install dbus
    $ sudo yum -y install dmidecode
    $ sudo yum -y install dnsmasq
    $ sudo yum -y install ebtables
    $ sudo yum -y install iscsi-initiator-utils
    $ sudo yum -y install libcgroup
    $ sudo yum -y install lzop
    $ sudo yum -y install nfs-utils
    $ sudo yum -y install numad
    $ sudo yum -y install parted
    $ sudo yum -y install polkit
    $ sudo yum -y install radvd
    $ sudo yum -y install cyrus-sasl-md5
    $ sudo yum -y install gettext
    $ sudo yum -y install gnutls-utils
    $ sudo yum -y install nc
    $ sudo yum -y install pm-utils
    $ sudo yum -y install libnl
    $ sudo yum -y install numactl
    $ sudo yum -y install yajl
    $ sudo yum -y install netcf
    $ sudo yum -y install libpcap

ここまでインスールすればlibvirt関係のrpmをインストールすることが
できます．依存関係があるので丸ごと1発で実行して，反映させるために
リブートしておきます．

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

    $ sudo reboot

これでlibvirtを使う用意ができました．

##libvirtを使う

libvirtを使うにはopenvswitchで作成するネットワークやqemuで作成
する仮想サーバをxmlフォーマットで定義しておく必要があります．
仮想ネットワークの定義はnetwork，仮想サーバの定義はdomainという
タグを持ちます．/etc/sysconfig/network-scriptsで

    $ cat ifcfg-ovsbr0
    DEVICE=ovsbr0
    DEVICETYPE=ovs
    HOTPLUG=no
    NM_CONTROLLED=no
    ONBOOT=yes
    OVSBOOTPROTO=dhcp
    OVSDHCPINTERFACES=eth0
    TYPE=OVSBridge

と記述しているovsbr0に対するxmlは以下のようになります．

    $ cd ~/xmls
    $ cat ovsbr0.xml
    <network>
      <name>ovsbr0</name>
      <bridge name="obsbr0"/>
      <forward mode="bridge"/>
      <virtualport type="openvswitch"/>
    </network>

openvswitchを使う際にはforward modeやvirtualport typeは固定値に
なるので，基本的には名前を指定してopenvswitchを使っていることを
示しているだけです．この定義を記述したら，net-defineコマンドを
使ってovsbr0を登録し，ネットワーク機能がホスト起動時に自動設定
するように設定します．

    $ sudo virsh net-define ovsbr0.xml
    $ sudo virsh net-start ovsbr0
    $ sudo virsh net-autostart ovsbr0
    $ sudo virsh net-list
     名前               状態     自動起動  永続
    ----------------------------------------------------------
     default              動作中  はい (yes)  はい (yes)
     ovsbr0               動作中  はい (yes)  はい (yes)

これでopenvswitch経由で仮想ネットワークの管理ができる
ようになりました．libvritは定義ファイルを元に必要な
パラメータを自動生成して使用します．

    $ sudo virsh net-dumpxml ovsbr0
    <network>
      <name>ovsbr0</name>
      <uuid>75cfd54f-360a-a6d5-ee45-b9a2d624f062</uuid>
      <forward mode='bridge'/>
      <bridge name='obsbr0' />
      <virtualport type='openvswitch'/>
    </network>

次はサーバ定義ですが，現状問題があります．
kvmは最近qemuに吸収されました．そのため，libvirt側の設定にも
変更が必要なのですが，まだドキュメントが揃っていないようです．
おまけにqemu自体の動きも不安定です．以下のxml定義は試行錯誤で
動かしたものですが，あるべき記述とは異なっているかもしれません．

    $ cat papeete.xml
    <domain type='kvm'>
      <name>papeete</name>
      <title>qemu, openvswitch and sheepdog ver.</title>
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
          <driver name='qemu' type='raw' io='threads'/>
          <source protocol='sheepdog' name='papeete'>
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

いくつかのキーポイントを説明します．

    <domain type='kvm'>

typeをqemuにするとqemuによるエミュレーションとなりkvmと記述すると
kvmによるエミュレーションとなります．qemuと比較するとkvmは爆速です．

      <memory unit='GB'>1</memory>
      <currentMemory unit='GB'>1</currentMemory>

libvirtはSI接頭辞と二進接頭辞を区別して使っています．つまり，
1GiB = 2^30 = 1,073,741,824 B で，1GB = 10^9 = 1,000,000,000 Bを
意味します．ちなみにGiBはギビバイトと発音するようです．

        <type arch='x86_64' machine='pc'>hvm</type>

従来machineの引数は沢山あったような気がするのですが
現状で使えるのはpcです．ちなみに以下をサポートしているようです．

    $ qemu -M -?
    Supported machines are:
    pc         Standard PC (alias of pc-0.14)
    pc-0.14    Standard PC (default)
    pc-0.13    Standard PC
    pc-0.12    Standard PC
    pc-0.11    Standard PC, qemu 0.11
    pc-0.10    Standard PC, qemu 0.10
    isapc      ISA-only PC

0.14というのが何のバージョンを意味しているのか理解
できていません．今使っているlibvirtは1.0.3です．

      <features>
        <acpi/>
      </features>

標準ではacpi以外にapic, paeという機能を付けます．ゲストOS
起動時にやけに理由がかかるので，その問題点切り分けのために
外しています．acpiを外すとゲストOSをshutdownした際にホストOSに
制御が切り替わらなくなるので，これは外せません．

      <clock offset='localtime'/>

サーバの時計が日本時間であったほうが見やすいのでこのように
しています．

        <emulator>/usr/bin/qemu-system-x86_64</emulator>

emulatorはフルパスで指定しないとlibvirtが理解しません．

        <disk type='network' device='disk'>
          <driver name='qemu' type='raw' io='threads'/>
          <source protocol='sheepdog' name='papeete'>
            <host name='127.0.0.1' port='7000'/>
          </source>
          <target dev='vda' bus='virtio'/>
        </disk>

最新のsheepdog rpmを使う際にはdriverのオプションとして
cache='writeback'が使えますが，今利用しているバージョンでは
使えません．host name='127.0.0.1'でsheepdogを利用しています．

        <interface type='bridge'>
          <source bridge='ovsbr0'/>
          <virtualport type='openvswitch'/>
          <model type='virtio'/>
        </interface>

libvirtからopenvswitchを使う際の一般的な記述方法です．

        <serial type='pty'>
          <target port='0'/>
        </serial>
        <console type='pty'>
          <target type='serial' port='0'/>
        </console>

textモードでゲストOSを扱う際に必要な設定です．このファイルから
定義をlibvirtに読み込むと必要なパラメータを自動設定します．

    $ sudo virsh define papeete.xml
    $ sudo virsh dumpxml papeete
    <domain type='kvm'>
      <name>papeete</name>
      <uuid>efe5441a-45e8-015b-dfa0-f68d386ee87e</uuid>
      <title>qemu, openvswitch and sheepdog ver.</title>
      <memory unit='KiB'>976563</memory>
      <currentMemory unit='KiB'>976563</currentMemory>
      <vcpu placement='static'>1</vcpu>
      <os>
        <type arch='x86_64' machine='pc-0.14'>hvm</type>
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
          <source protocol='sheepdog' name='papeete'>
            <host name='127.0.0.1' port='7000'/>
          </source>
          <target dev='vda' bus='virtio'/>
          <address type='pci' domain='0x0000' bus='0x00' slot='0x04' function='0x0'/>
        </disk>
        <controller type='usb' index='0'>
          <address type='pci' domain='0x0000' bus='0x00' slot='0x01' function='0x2'/>
        </controller>
        <interface type='bridge'>
          <mac address='52:54:00:17:48:81'/>
          <source bridge='ovsbr0'/>
          <virtualport type='openvswitch'>
            <parameters interfaceid='ab117811-d0c6-2a23-f7fb-e13825d681a4'/>
          </virtualport>
          <model type='virtio'/>
          <address type='pci' domain='0x0000' bus='0x00' slot='0x03' function='0x0'/>
        </interface>
        <serial type='pty'>
          <target port='0'/>
        </serial>
        <console type='pty'>
          <target type='serial' port='0'/>
        </console>
        <memballoon model='virtio'>
          <address type='pci' domain='0x0000' bus='0x00' slot='0x05' function='0x0'/>
        </memballoon>
      </devices>
    </domain>

uuidやmacアドレス，pci設定が追加されています．これでゲストOSの
準備はできました．

    $ sudo virsh start papeete
    $ sudo virsh console papeete

でゲストOSを起動し，そのコンソール画面をターミナル上に出力させる
ことができます．consoleから抜ける際にはC-]（コントロールキーを
押しながら ] ）を押してください．再接続するにはもう一度

    $ sudo virsh console papeete

をコマンドします．ゲストOSの起動状態の確認は

    $ sudo virsh list

異常な状態になったゲストOSを強制収容させるためには

    $ sudo virsh destroy

とコマンドします．これでlibvirtによる基本的な管理ができるように
なりました．
