# ちょっと複雑なネットワーク

## はじめに

3台のホストOS上にブリッジを作っても連携して動かなければ面白く
ありません．openvswitchではGREを使ってブリッジ同士をつなぐことが
できます．ここではこの機能を利用して3台のホストOS上の仮想
ネットワークを接続してみます．

## ネットワークを作る

まずzebra, yellow, xerxes上にもうひとつずつブリッジを作成します．

    $ cat ovsbr1.xml
    <network>
      <name>ovsbr1</name>
      <bridge name="ovsbr1" />
      <forward mode="bridge" />
      <virtualport type="openvswitch"/>
    </network>
    $ sudo virsh net-define ovsbr1.xml
    $ sudo virsh net-start ovsbr1
    $ sudo virsh net-autostart ovsbr1

このブリッジにもインタフェースを持つようにmonkeyのxml定義にも修正を
加えます．追加するのは以下の定義


    <interface type='bridge'>
      <source bridge='ovsbr1'/>
      <virtualport type='openvswitch'/>
      <model type='virtio'/>
    </interface>

これを

    $ sudo virsh edit monkey

で編集モードに入って既存インタフェースの次に追加します．monkeyを
起動してインタフェースを確認すると新規作成のnicがeth0になっている
と思います．

    vyatta@monkey:~$ show interfaces
    Codes: S - State, L - Link, u - Up, D - Down, A - Admin Down
    Interface        IP Address                        S/L  Description
    ---------        ----------                        ---  -----------
    eth0             -                                 u/u 
    eth1             172.16.3.150/24                   u/u 
    lo               127.0.0.1/8                       u/u 

追加した方の添字が小さいのは気になったので
/opt/vyatta/etc/config/config.bootにある管理ファイルにある
eth0をeth2に直接編集して修正します．以下は修正の状態です．

    interfaces {
        ethernet eth1 {
            address dhcp
            duplex auto
            hw-id 52:54:00:09:53:d5
            smp_affinity auto
            speed auto
        }
        loopback lo
        ethernet eth2 {  <-  元はeth0
            hw-id 52:54:00:74:64:4a
        }
    }
    
これを再起動すればeth2が新しいブリッジに接続した状態で
あがってきます．

vyatta@monkey:~$ show interfaces
Codes: S - State, L - Link, u - Up, D - Down, A - Admin Down
Interface        IP Address                        S/L  Description
---------        ----------                        ---  -----------
eth1             172.16.3.150/24                   u/u 
eth2             -                                 u/u 
lo               127.0.0.1/8                       u/u 
                 ::1/128

eth2はトランクポートです．

    vyatta@monkey:~$ configure
    vyatta@monkey# set interfaces ethernet eth2 vif 10
    vyatta@monkey# set interfaces ethernet eth2 vif 20
    vyatta@monkey# set interfaces ethernet eth2 vif 10 address 192.168.10.1/24
    vyatta@monkey# set interfaces ethernet eth2 vif 20 address 192.168.20.1/24
    vyatta@monkey# set interfaces ethernet eth2 vif 10 mtu 1470
    vyatta@monkey# set interfaces ethernet eth2 vif 20 mtu 1470
    vyatta@monkey# set system gateway-address 172.16.3.2
    vyatta@monkey# commit
    vyatta@monkey# save
    vyatta@monkey# exit

これでmonkey用のネットワークはできあがりでbrige1を通じて
外とつながることができるようになります．次はvlan10とvlan20の
ネットワークを作ります．openvswitchのポート番号ごとにvlanidを
割り振ることも可能ですが，ゲストOSの起動タイミングでポート番号が
付与され終了のタイミングでそれまで使っていたポートをリリース
してしまうので，この方法は管理系がよっぽどしっかりしていないと
使えません．ここではfake bridgeという特定のvlan idだけを通す
ブリッジを使う方法を使います．

まず，openvswitch上でvlan10, vlan20という名前のfake bridgeを
zebra, yellow, xerxesそれぞれのovsbr1上に作成します．

    $ sudo ovs-vsctl add-br vlan10 ovsbr1 10
    $ sudo ovs-vsctl add-br vlan20 ovsbr1 20
    $ sudo ovs-vsctl show
    11471682-2533-4cac-9586-6433ab5bfbd0
        Bridge "ovsbr1"
            Port "vnet1"
                Interface "vnet1"
            Port "vlan20"
                tag: 20
                Interface "vlan20"
                    type: internal
            Port "vlan10"
                tag: 10
                Interface "vlan10"
                    type: internal
            Port "ovsbr1"
                Interface "ovsbr1"
                    type: internal
        Bridge "ovsbr0"
            Port "vnet0"
                Interface "vnet0"
            Port "eth0"
                Interface "eth0"
            Port "ovsbr0"
                Interface "ovsbr0"
                    type: internal
        ovs_version: "1.10.0"

これらvlan10, vlan20のlibvirt用のxml定義はそれぞれ
以下のようになります．

    $ cat vlan10.xml
    <network>
      <name>vlan10</name>
      <bridge name="vlan10"/>
      <forward mode="bridge"/>
      <virtualport type="openvswitch"/>
    </network>
    $ vat vlan20.xml
    <network>
      <name>vlan20</name>
      <bridge name="vlan20"/>
      <forward mode="bridge"/>
      <virtualport type="openvswitch"/>
    </network>

これをzebra, yellow, xerxesにそれぞれ登録します．

    $ sudo virsh net-define vlan10.xml
    $ sudo virsh net-define vlan20.xml
    $ sudo virsh net-start vlan10
    $ sudo virsh net-start vlan20
    $ sudo virsh net-autostart vlan10
    $ sudo virsh net-autostart vlan20

これでzebra, yellow, xerxesにvlan10, vlan20が流れるブリッジが
用意できたので，GREを使って接続します．接続するにはzebra上の
ovsbr1にgre1, gre2というポートを新しく作成し，それぞれ
yellow上ovsbr1に新規作成するgre1，xerxes上ovsbr1に新規作成する
gre2と接続します．

    // zebra
    $ sudo ovs-vsctl add-port ovsbr1 gre1
    $ sudo ovs-vsctl set interface gre1 type=gre options:remote_ip=172.16.3.148
    $ sudo ovs-vsctl add-port ovsbr1 gre2
    $ sudo ovs-vsctl set interface gre2 type=gre options:remote_ip=172.16.3.149
    
    // yellow
    $ sudo ovs-vsctl add-port ovsbr1 gre1
    $ sudo ovs-vsctl set interface gre1 type=gre options:remote_ip=172.16.3.147
    
    // xerxes
    $ sudo ovs-vsctl add-port ovsbr1 gre2
    $ sudo ovs-vsctl set interface gre2 type=gre options:remote_ip=172.16.3.147
    
これでopenvswitchの用意はできました．次はyellow上にvlan10に接続する
londonという名前のゲストOSを作ります．londonのlibvirt用のxmlは
以下のようになります．

    <domain type='kvm'>
      <name>london</name>
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
          <source protocol='sheepdog' name='london'>
            <host name='127.0.0.1' port='7000'/>
          </source>
          <target dev='vda' bus='virtio'/>
        </disk>
        <interface type='bridge'>
          <source bridge='vlan10'/>
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

interface定義中のsource bridgeがvlan10になっているのがポイントで他は
今までと違いありません．xerxesでもvlan20に接続したkingというゲストOS
用のxmを用意して，libvirtにそれぞれ登録します．

londonでvyattaの設定をして

    $ configure
    # set system host-name london
    # set system time-zone Asia/Tokyo
    # set interfaces ethernet eth1 address 192.168.10.2/24
    # set system gateway-address 192.168.10.1
    # set system name-server 172.16.3.2
    # set interfaces ethernet eth1 mtu 1470
    # commit
    # save
    # exit

同様にkingでもvyatta設定を行います．

    $ configure
    # set system host-name king
    # set system time-zone Asia/Tokyo
    # set interfaces ethernet eth1 address 192.168.20.2/24
    # set system gateway-address 192.168.20.1
    # set system name-server 172.16.3.2
    # set interfaces ethernet eth1 mtu 1470
    # commit
    # save
    # exit

これでlondonとkingの間でpingが通るようになりました．

    vyatta@king:~$ ping 192.168.10.2
    PING 192.168.10.2 (192.168.10.2) 56(84) bytes of data.
    64 bytes from 192.168.10.2: icmp_req=1 ttl=63 time=6.68 ms
    64 bytes from 192.168.10.2: icmp_req=2 ttl=63 time=2.24 ms
    ^C
    --- 192.168.10.2 ping statistics ---
    2 packets transmitted, 2 received, 0% packet loss, time 1003ms
    rtt min/avg/max/mdev = 2.243/4.465/6.687/2.222 ms

あとはmonkey上でnat設定をいれればlondonやkingから外の世界に
つながることができるようになります．

    $ configure
    # set nat source rule 10 outbound-interface eth1
    # set nat source rule 10 source address 192.168.0.0/16
    # set nat source rule 10 translation address masquerade
    # commit
    # save
    # exit

## まとめ

VMWare Fusion上に作ったzebra, yellow, xerxesというcentos上に
monkey, london, kingというゲストOSを作成しゲストOSから外の世界に
つなげるところまで試しました．zebra, yellow, xerxesのゲストOS用
ブリッジを接続しているので，monkey, london, kingはどのホスト上でも
動かすことができます．動かす際には

    $ sudo virsh dumpxml monkey > monkey_dump.xml

のようなファイルを作って別のホストにコピーし，libvirt上に定義
してやる必要があります．
