# rpmパッケージを作る

## はじめに

今回の話を進めるにあたっていくつかのrpmパッケージを自分で作成する
必要に迫られました．知っているようでしらないrpmの使い方と作り方に
ついてまとめてみます．

## rpmの使い方

知っていると便利なrpmコマンドオプションをいくつか並べてみます．

* インストール

    \# rpm -i package.rpm

* 複数パッケージのインストール

    \# rpm -i package_a.rpm package_b.rpm

* 詳細情報付きインストール

    \# rpm -ivh package.rpm

* 進行状況表示付きインストール

    \# rpm -i --percent package.rpm

* パッケージのアップデート

    \# rpm -U package.rpm

* 古いバージョンのパッケージをインストール

    \# rpm -i --oldpackage package.rpm

* インストール済みの他のパッケージに含まれるファイルを
置き換えてでもインストール

    \# rpm -i --replacefiles package.rpm

* インストール済みパッケージを再インストール

    \# rpm -i --replacepkgs package.rpm

* インストールせずテストのみ実行

    \# rpm -i --test package.rpm
 
* インストール済みパッケージ確認

    $ rpm -q package

* インストール済み全パッケージ

    $ rpm -qa

* あるファイルがどのパッケージに含まれているか

    $ rpm -qf /usr/local/bin/commandfile

* あるコマンドの機能を提供しているのはどのパッケージか

    $ rpm -q --whatporvides /usr/bin/strip

* あるパッケージが依存するファイルを確認

    $ rpm -qR package

* 未インストールのパッケージに帯する問い合わせ

    $ rpm -qpR /path/to/package.rpm

* パッケージが含んでいるファイルを表示

    $ rpm -ql xfce

## rpmの作り方

rpmコマンドの使い方がわかったところで，今度は作り方についてまとめて
みます．ディストリビューションやrpmを公開しているグループごとに
やりかたの違いがあるので，これは自分流の簡単なまとめです．

まず覚えておいてほしいのはルートで作業してはいけない，ということです．
必ず一般ユーザ権限で作業を行ってください．一番簡単なrpm作成方法は
以下のようになります．

1. 構築用ディレクトリ作成

    $ mkdir -p ~/rpm/{BUILD,RPMS,SOURCES,SPECS,SRPMS}

2. .rpmmacros 設定

    $ echo "%_topdir /home/WHOAMI/rpm" > ~/.rpmmacros

3. ソースファイル配置

    $ mv example-1.0.2.tar.gz ~/rpm/SOURCES

4. SPECファイル配置

    $ mv example.spec ~/rpm/SPECS/

5. rpmを作る

    $ cd ~/rpm/SPECS
    $ rpmbuild -bb exsample.spec

これで~/rpm/RPMS/ARCH/配下にrpmファイルが出来上がります．
ARCHはアーキテクチャタイプでi386, x86_64, noarchがあります．
rpmbuildコマンドには以下のオプションがあります．

    -ba バイナリRPM & ソースRPM作成
    -bb バイナリRPM作成
    -bc %buildセクションまで実行
    -bp %prepセクションまで実行
    -bi バイナリRPM作成 & %installセクションまで実行
    -bl 依存ファイル検証
    -bs ソースRPM作成
    -showrc 実行する際に使用するマクロ種別とその内容の表示

## specfileの構造

詳しいことは日本語で書いてあるspecfileの[解説資料][5-1]を
読んでいただくとして，概略は以下のようになります．まず
specファイルは複数のセクションからなります．

* id セクション
* %description セクション
* %prep セクション
* %build セクション
* %install セクション
* %clean セクション
* %pre セクション
* %post セクション
* %preun セクション
* %postun セクション
* %files セクション

idセクションではrpmの一意性を保証するためのパラメータを指定します．
例えば以下のパラメータです．idセクションにはセクションを指定する
%id といった行はありません．

    Name: examples
    Version: 1.0.2
    Source: examples-1.0.2.tar.gz

Sourceには~/rpm/SOURCES/配下においてあるファル名を指定します．

descriptionセクションはこれを指定する%descriptionから始まる
セクションで以下のようなパラメータを指定します．

    %description
    Buildroot: %{_builddir}/%{name}-%{version}-root
    BuildRequires: gcc, bison, flex
    Requires: libcrypt

buidrootは作業ディレクトリの場所を示していて，今回の例では
~/rpm/BUILDROOT/examples-1.0.2-root 配下に作ることになります．
BuildRequires は開発時に必要なrpmパッケージを，Requiresは
実行時に必要なrpmパッケージを記載します．

prepセクションはビルドディレクトリの初期化とソースの展開を
行います．例えば以下のように記述すると，Source:で指定した
ファイルを~/rpm/BUILD/配下に展開したのち作業ディレクトリを
移動します．

    %prep
    [ -d $RPM_BUILD_ROOT ] && rm -rf $RPM_BUILD_ROOT;
    %setup

buildセクションは構築に必要なコマンドをすべて記述します．
例えばconfigure & makeすればよい場合には

    %build
    %configure
    make

configureでパラメータ指定する場合には例えば以下のようになります．

    %build
    ./configure -prefix=/usr/local --enable-ssl

installセクションでは$RPM_BUILD_ROOTをルートディレクトリに
見立てインストールを実行します．make installを使う場合には

    %install
    make PREFIX=$RPM_BUILD_ROOT/usr install

のようになりますし，installコマンドを使う場合には以下
のようになります．

    %install
    /usr/bin/install -m755 example PREFIX=$RPM_BUILD_ROOT/usr/bin/example
    /usr/bin/install -m 755 example2 PREFIX=$RPM_BUILD_ROOT/usr/bin/example2
    
cleanセクションではクリーンアップ処理を実行します．例えば
以下のようになります．

    %clean
    rm -rf $RPM_BUILD_ROOTd
    
次はpre, post, preun, postunの各セクションについてです．ここでは
起動スクリプト登録などを行います．例えば，

    %post
    svc -u /service/exampledaemon > /dev/null 2>&1

    %preun
    if [ "$1" = 0 ]; then
        svc -dx /service/exampledaemon > /dev/null 2>&1
    fi
    exit 0

    %postun
    if [ "$1" -gt 1 ]; then
        rm -f /service/exampledaemon
    fi
    exit 0

    # $1 = 1；初回インストール
    # $1 = 2以上；アップグレード時
    # $1 = 0；アンインストール時

最後にfiles セクションです．ここではrpmパッケージに含めるファイルと
その属性を指定します．基本的には

* %defattr()マクロでデフォルト権限と所属を指定
* %attr()マクロでファイルごとの権限と所属を指定
* %config()マクロで設定ファイルを指定
* %config(noreplace)アップデート時に上書きしない
* %config(missingok)なくてもよいファイル

というマクロを使って以下のような記述をします．

    %files
    %defattr(-,root,root,-)
    %{_bindir}/example
    %{_bindir}/example2
    %config %attr(0600,root,root,-) %{_sysconfigdir}/example/defaults/*
    %config(noreplace) $_{sysconfdir}/example/hostname.conf

ここで使うのが内蔵マクロです．rpmbuild --showrcでも調べることができますが
知っておくと便利なのは以下のマクロです．

    %{_prefix}: /usr
    %{_bindir}: /usr/bin
    %{_sbindir}: /usr/sbin
    %{_libexecdir}: /usr/libexec
    %{_datadir}: /usr/share
    %{_sysconfdir}: /usr/etc
    %{_sharedstatedir}: /usr/com
    %{_localstatedir}: /usr/var
    %{_libdir}: /usr/lib
    %{_includedir}: /usr/include
    %{_oldincludedir}: /usr/include
    %{_infodir}: /usr/info
    %{_mandir}: /usr/man

## まとめ

specfileは奥が深く一朝一夕には理解できないと思います．rpmを公開している
サイトではspecfileも公開していることが多いので，まずはそれを参考に
どう記述するとどのような挙動をするのか調べてみるのが理解を進める
よい方法だと思います．

----

[5-1]: 
http://vinelinux.org/docs/vine6/making-rpm/make-spec.html "VINE LINUX ONLINE MANUAL"

