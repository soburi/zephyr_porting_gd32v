Longan Nanoに対応させる
=================================

サンプルコードを大事にする
-----------------------

本稿ではSipeedのLongan NanoをZephyrに対応させる。
ephyrへの移植の作業を行うにあたって、

- 動いているサンプルコードをベースにする

方針で進めていく。
これは、移植作業を行うにあたって、個人の開発ではハード側からの
トラブル究明の手段が限られることなどから、地道でも着実に進む方法を取る必要がある。
このため、スクラッチで作り上げるよりも「動いている既存のサンプルコードを、
Zephyrの構成に嵌め込む」ようにして移植の作業を進めていく。



LチカサンプルをZephyrでコンパイルする
---------------

最もベーシックな動作確認のサンプルとしてLチカのコード
(https://github.com/sipeed/platform-gd32v/tree/master/examples/longan-nano-blink)[https://github.com/sipeed/platform-gd32v/tree/master/examples/longan-nano-blink]
(PlatformIOでコンパイルできるLチカのサンプル)
を使用する。

まずは、Longan Nano用に提供されているサンプルをコンパイル、実行できるようにする。

大まかなながれとして、

- 既存のボードをベースにしてボードの定義を行う
- コンパイルオプション、リンカオプションをサンプルと同じにする。
- サンプルのソースをコンパイルして実行する。

の手順で進めていく。


### SoC, ボードの定義

今回移植作業を行うボードのSipeedのLongan Nanoは、
SoCのGigaDevice GD32VF103CBT8を搭載しており、このCPUのコアはRISC-V(rv32imac)となっている。

ここでは`soc`の下にGD32VF103CBT8、`board`の下にLongan Nanoの定義を追加する。
これもゼロから作るのではなく、既存の類似した構成から流用して作成するのが良い。
ここでは、SoCについては、同じriscvの配下にある`sifive-freedom`の構成を、
ボードについては`stm32_min_dev`を基に作る。(GD32VがSTM32のクローンで構成が類似しているため)
RISC-Vとしてのの共通部分は`arch/riscv`の配下に実装があるので、
`arch`配下にはファイルの追加は不要である。

SoCのフォルダには、少なくとも以下のファイルが必要となる。

- CMakeLists.txt

コンパイル対象のファイルを記載する

- Kconfig.defconfig.series

SoCのシリーズで共通のオプションのデフォルト値を設定するKconfig

- Kconfig.series

SoCのシリーズ名を定義するKconfig

- Kconfig.soc

SoCの型番を定義するKconfig

- linker.ld

リンカスクリプト。

- soc.h

SoC固有の定義を行うためのヘッダファイル。

ここでは、`soc/riscv/riscv-privilege` のディレクトリの配下に`gigadevice-gd32vf103`のフォルダを作って格納する。

`boards` についても同様に以下のような構成でファイルを作成する。


- CMakeLists.txt

コンパイル対象のファイルを記載する

- Kconfig.board

ボード固有のKconfigオプションを定義する

- Kconfig.defconfig


- sipeed_longan_nano_defconfig

[ボード名]_defconfigのファイルで、Kconfigで定義されたオプションの値を設定する。

- sipeed_longan_nano.dts

[ボード名].dtsのファイルで、devicetreeのハード構成を定義する。


基本方針としては流用元のファイルのシンボル名を新しい定義にgrep置換して
定義を作成する。コードの対応を進める過程で矛盾が出てくるので、その時に適宜修正する。

### ビルドコマンドの修正

はじめに、Longan NanoのサンプルのLチカと同じコマンドでビルド、リンクが行われるよう設定する。


構成が出来上がったら空実装のサンプル`samples/basic/minimal`を
以下のコマンドでコンパイルする。

```
west -v build -b sipeed_longan_nano samples/basic/minimal
```

gccに渡されるオプションをサンプルのものと比較して、差異があるところを修正する。
コンパイルオプションの主だったところは`cmake/compiler/gcc.cmake`で定義されているので、
これを適宜修正する。ビルド環境が固まってきたら正しいものに直していくが、ここでは
「サンプルで実績のあるコマンド」でコンパイルするようにする。

差異があるのはほとんどがwarningの指定だが、コード生成の指定に関連する
`-march` と `-mcmodel`の指定などの差異もある。

`march`のrv32に連なる文字は使用するriscvの命令セットを示しており、
Longan Nanoのサンプルで指定されている`imac`だと、

- i 整数命令
- m 乗除算命令
- a アトミック命令
- c 圧縮命令

が有効になる。(つまり、Zephyrで指定しているrv32imaの指定では圧縮命令は使わないのでフットプリントは大きくなるが動作に支障はないはず、となる。が、ここではまずサンプルに合わせる。)

`-mcmodel`ではメモリ配置を指定する。`medlow`は小～中規模のコードに対する指定で、デフォルトの指定。


コンパイルオプションを揃えたら、Longan Nanoの提供コードを組み込む。
ここで必要になるのは初期化のコードなので、依存関係を満たすよう
CMakeLists.txtに追加する。

`RISCV/stubs/`配下のものは標準Cライブラリや標準UNIXのAPIを提供するものであるが、
ここでは使わないようにしたい。
そのため、`RISCV/enbv_Eclipse`と`RISCV/drivers/`からstubsへの参照となっているコードをコメントアウトする。

### リンカスクリプトの作成

コンパイルを通すために、リンカスクリプトを作成する。
Zephyrでは、`boards`、`soc`の構成ファイルの中にある`linker.ld`のファイルを
リンカスクリプトとして使用する。
ここでは、サンプルのリンカスクリプトをそのまま`soc/riscv/riscv-privilege/gigadevice-gd32vf103/linker.ld` としてコピーする。

ここまで行えば`samples/basic/minimal`のコンパイルが通るようになる。


### Lチカサンプルのプロジェクトを作成する

ビルドができるようになったので、LチカサンプルをZephyrのプロジェクトの
形でにしてコンパイルする。
Zephyrのお約束の通り、`CMakeLists.txt`, `prj.conf`を作成して、
サンプルにある、`main.c`, `systick.c`, `systick.h`を格納する。
minimalがコンパイル出来ているのであれば問題なく動作するはず。


OpenOCDの設定
-------------------------

Lチカが出来たら、ボードへの書き込み、デバッグを行うためのOpenOCDの設定を作成する。

Zephyrでは補助コマンドの`west`で
`west flash`, `west debug` のように指定すると、書き込み、デバッガの起動が
行える。

この際に、 `boards`配下の構成ディレクトリにある`board.cmake` が参照される。
ボード、SoCの詳細な実行コマンドはこのファイルで指定する。

longan nanoの場合はOpenOCDを使っていて、実際にopenocdを実行しているのは、
`scripts/west_commands/runners/openocd.py`のスクリプトである。
このスクリプトでは、`--cmd-pre-init`, `--cmd-pre-load`, `--cmd-load`, `--cmd-verify`, `--cmd-post-verify`のオプションを受けつけており、
内部で生成されるopenocdの設定スクリプトに反映される。
すなわち、ここに必要なコマンドを設定すればよいのだが、
複数行にわたるコマンドをデリミタで区切るとpython内部の変数解析で上手く
処理出来ない場合がある。

このスクリプトでは、`boards`配下の構成ディレクトリの下に置かれる`support/openocd.cfg`をopenocdの設定スクリプトとして読み込むので、ここで関数定義を行って、その関数を呼び出す形にすると問題が少ない。


ここまでで出来たこと
------------------------

ここまでの作業で、

- Zephyrのディレクトリ構成に従ったボード、SoCの構成の作成
- ビルド、書き込み、デバッグの動作

ができた。これでビルドの設定に関する部分の作業は完了となる。
upstreamへ適用を考えた場合には、アドホックに共通設定を書き換えている
部分もあるので、必要に応じて問題のない形に修正する必要はある。