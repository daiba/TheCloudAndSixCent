# ホストOSをセットアップする

## VMwareFusionの準備

まずはFusion上でCentOSを動かしましょう．インストール手順は
すでに理解しているという前提で話を進めます．そうは言っても
作業を追試できないといけないので，windowタイトルとそこで何を
選択するかをまとめてみます．

* [新規仮想マシンの作成]
    * ディスクを使用せずに続行
* [インストールメディア]
    * オペレーティングシステムのインストールディスクまたはイメージを使用
         * CentOS-6.4-x86_64-netinstall.iso
* [オペレーティングシステムの選択]
    * オペレーティングシステム：Linux
    * バージョン: CentOS(64ビット)
* [終了]
    * 仮想マシン概要
         * ゲストOS: CentOS(64ビット)
         * メモリ: 1GB
         * ディスクサイズ（最大）: 20GB
         * ネットワーク: 共有ネットワーク(NAT)
         * CD/DVD: CentOS-6.4-x86_64-netinstall.iso

このあと設定をカスタマイズします．プロセッサを2個以上の値にするのは
必須条件です．名前を付けること，メモリサイズを増やすことは推奨条件です．

* [設定のカスタマイズ]
    * 名前: tahiti
* [プロセッサとメモリ]
    * プロセッサ: 2個のプロセッサコア
    * メモリ: 2048MB

次にFusion上で'Intel VT'をオンにします．これが設定できないとkvmは
動きません．VMwareが動くOSやバージョンによって設定方法が違うようです．
[VMwareのコミュニティサイト][1]にまとめがあったのでこれを参考に
してください．ここではFusion4を使っているので_hostname_.vmxという
ファイルに以下の行を追加することになります．

    vhv.enable = "TRUE"

先ほどVMにtahitiという名前を付けたので，ファイルは

    /Users/kei/Documents/Virtual Machines.localized/tahiti.vmwarevm/tahiti.vmx

にあります．

## CentOSのインストール

CentOSのインストーラはGUIモードが標準ですが，Fusion上で動かしてることも
あるし，メモリを節約するためにtextモードでインストールすることにします．
インストーラ画面でtabを押せば編集モードになるので，以下のようにtextと
追記するだけです．

    vmlinuz initrd=initrd.img xdriver=vesa nomodeset text

これから後の行程は皆さんもおなじみでしょう．さらっと流して行きます．

* [Choose a Language]
    * English
* [Keyboard Type]
    * jp106
* [Installation Method]
    * URL
* [Configure TCP/IP]
    * [*] Enable IPv4 support
    * (*) Dynamic IP configuration (DHCP)
    * [ ] Enabel IPv6 support
* [URL Setup]
    * http://ftp.iij.ad.jp/pub/linux/centos/6.4/os/x86_64/images/install.img
    * [*] Enable HTTP proxy
        * http://proxy.example.com:8080
* [Would you like to use VNC?]
    * Use text mode
* [Warning]
    * Re-initialize all
* [Time Zone Selection]
    * [ ] System clock uses UTC
    * Asia/Tokyo
* [Root Password]
* [Partitioning Type]
    * Use entire drive
    * [*] sda 20480 MB
* [Writing storage configuration to disk]
    * Write changes to disk
* [Complete]
    * Reboot

OSインストールが終わったのでrootでloginして作業しましょう．
まずはhostname設定を

    # cat /etc/sysconfig/network
    NETWORKING=yes
    HOSTNAME=tahiti


使っているネットワークがproxy経由でINTERNETにアクセスする
環境であればyumをproxy対応させる必要があります．

    # tail -1 /etc/yum.conf
    proxy=http://proxy.example.com:8080

いつまでもrootで作業してるのも気持ち悪いので一般ユーザを
作ってsudoを使うようにします．

    # useradd -m kei
    # passwd kei

keiというユーザをwheelグループに所属させて，wheelからsudoを
使うようにします．

    # diff /etc/.sudoers /etc/sudoers
    108c108
    < # %wheel     ALL=(ALL)     NOPASSWD: ALL
    ---
    > %wheel     ALL=(ALL)     NOPASSWD: ALL

    # diff /etc/.group /etc/group
    11c11
    < wheel:x:10:
    ---
    > wheel:x:10:kei
    34a35
    > kei:x:500:

    # exit

一般ユーザでloginしなおして，Fusionを動かしてるmacからsshでログイン
できるように設定します．そのためにはopenssh-clientsをインストール
する必要があります．ついでにmanもインストールしておきます．

    $ mkdir .ssh
    $ chmod 700 .ssh
    $ sudo yum -y install openssh-clients
    $ sudo yum -y install man
    $ ip addr show dev eth0 | grep inet
         172.16.3.140

Fusionを動かしているmac上で鍵を作って，公開鍵をCentOSに送ります．

    $ ssh-keygen -t dsa -f tahiti_dsa
    $ scp .ssh/tahiti_dsa.pub kei@172.16.3.140:~/.ssh/authorized_keys

mac上で.ssh/configファイルを作っておけば

    $ ssh tahiti

とコマンドするだけでloginできるようになります．Fusionで作るVMは
起動するタイミングでIPアドレスが変わることがないようなので，
一旦確保したIPをconfigに記載することができます．

    $ cat .ssh/config
      Host tahiti
      User kei
      Port 22
      Hostname 172.16.3.140
      IdentityFIle ~/.ssh/tahiti_dsa
      TCPKeepAlive yes
      ServerAliveInterval 10

TCPKeepAliveとServerAliveIntervalは普段から使っているおまじないです．
これでFusionのアプリケーションウィンドウを最小化して，macのterminal
からloginすることができるようになりました．

OSインストールの最後の作業はrpmの最新化と必要なyumリポジトリの追加です．
念のためupdateした後にrebootを行っています．

    $ sudo yum -y update
 
    $ sudo reboot
    $ sudo rpm -httpproxy http://proxy.example.co.jp:8080 -ivh http://pkgs.repoforge.org/rpmforge-release/rpmforge-release-0.5.3-1.el6.rf.x86_64.rpm
    $ sudo rpm -httpproxy http://proxy.example.co.jp:8080 -ivh http://ftp.jaist.ac.jp/pub/Linux/Fedora/epel/6/x86_64/epel-release-6-8.noarch.rpm

これでqemuとsheepdogをインストールできるようになりました．ついでに
作業用ディレクトリもここで作成しておきます．

    $ sudo yum --enablerepo rpmforge-extras install qemu
    $ sudo yum --enablerepo epel install sheepdog
    $ mkdir ~/{rpms,isos,xmls}
    
----

[1]: http://communities.vmware.com/docs/DOC-8970  "VMware community site"
