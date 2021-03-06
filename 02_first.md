Zephyrへのプラットフォーム追加
======================

何をやるのか
--------------

新しいボードやCPUをZephyrで使うためには、

- 開発環境の作成
- CPUやボード特有の部分のOS対応
- 独自のデバイスに対するドライバの作成

の対応を行う必要がある。

まず、ビルドやボードへの書き込み、デバッグを行うための開発環境の作成を行う必要がある。
これらの作業はZephyr対応の作業自体でも常用するものなので、最初に必要となる対応である。

次にブート周りをメインとする、CPU、ボード固有コードの作成を行う。
この対応を行うと、最低限のOSを動作ができるようになる。
Zephyrのソースコードは、CPUコア、SoC、ボードの3層で階層化しており、
それぞれのレイヤーごとに再利用が可能である。
ZephyrのCPU/SoC依存のコードは半導体ベンダー自らによって実装が行われている場合も多く、
CPUやSoCレベルの対応を行うケースは少ない。(SoCのレベルのソースを読むことは多いはず)
起動の固有部分以降の処理は、OSがハード差異を抽象化、隠ぺいして実装されているので、
個別の対応はほとんど発生しない。

OSの基本動作の次は、実用的な機能を利用するためにSoCのペリフェラル類や、
ボードに接続されたデバイスそれぞれ固有のドライバを作成する。
Zephyrでは、統一的なドライバモデルと、デバイス種別ごとの共通APIでハードウェア抽象化レイヤーを
作っている。つまりドライバ開発のいくつかお決まりのルールを守って、共通APIを実装すれば、
既存のソースコード資産を利用することができる。

概ね、上記のような順で対応していくことでZephyrに新しいボードやCPUを対応させることができる。




Zephyrの主なフォルダ構成
---------------------

- arch, board, soc

  この3つのディレクトリはCPUコア、ボード、SoCに依存したコードが含まれている。

- drivers

  各種ドライバのソース

- dts

  devicetreeで使われる定義など

- kernel, ext, lib, subsys

  ZephyrのOS本体となるソース。

- cmake, script

  CMakeによるビルドスクリプトや補助コマンド`west`が参照するスクリプト類

- doc

  ドキュメント類


新しいCPU/ボードに対応するには、主に`arch`, `boards`, `soc`のディレクトリにコードを作成することになる。
ドライバを作成するときには、ドライバの実装コードを格納する`drivers`と、
ドライバに必要なdevicetreeの定義を格納するため`dts`に対してファイルの追加を行う。


Kconfig, devicetree, CMake
------------------------------

Zephyrのビルドシステムはその中核にLinuxで実績のあるKconfig, devicetreeのシステムが
置かれている。Linuxも多くのデバイスドライバのソースを収録し、それを
柔軟に組み替えて多種多様なボードの対応を実現している。(Linuxにおいても、
この仕組みはカーネルがサポートするARM搭載ボードの爆発的な増加に対応するため
導入された経緯がある)
また、ビルドのプラットフォーム差異を吸収できるCMakeもビルドシステムに組み込まれている。
Zephyrのソースを改変するにあたって、これらのツールの基本的な用法については、
ある程度把握している必要がある。