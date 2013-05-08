# はじめに

この資料はVMware Fusion上でクラウドの実験をしてみた
ログをまとめたものです．使った機材は
* MacBook Air OSX 10.8.3
* VMWare Fusion 4.1.4
* CentOS-6.4-x86_64-netinstall.iso

で，作ったCentOS上に以下のバージョンのrpmをインストールし，

* qemu-0.15.0-1.el6.rfx.x86_64
* sheepdog-0.2.3-2.el6.x86_64
* openvswitch-1.7.3-1.x86_64
* libvirt-1.0.3-1.el6.x86_64

最終的にゲストOSとしてembedded linuxのルータであるvyattaを動かしました．

* vyatta-livecd_VC6.5R1_amd64.iso

今回使ったrpmは安定してるとはいいがたく，この資料を公開するまでに
バージョンアップしてしまうかもしれませんし，作り方も変わってしま
うかもしれません．ですが，VMwareを使っているので，追試も検証も容易に
できると思います．

資料の流れは以下のようになります

1. ホストOSをセットアップする
2. openvswitchを作る
3. qemuとopenvswitchをインスールする
4. openvswitch用ネットワーク設定
5. libvirtを作る
6. libvirtをインストールする
7. sheepdog用にファイルシステムを作る
8. sheepdogをインストールする
9. ゲストOSをセットアップする
10. 仮想ネットワークで遊んでみる

「雲と六セント」という題名は「月と六ペンス」のもじりです．
wikipediaを見たところ，題名の「月」は夢を，「六ペンス」は現実を
意味しているのだそうです．CentOS6で（動作確認ができるかどうかの）
ちゃちなクラウドを作る話にはぴったりな名付けだと思いませんか．
