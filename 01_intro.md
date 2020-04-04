はじめに
============

本稿は中国製RISC-VマイコンのGigaDevice GD32Vを搭載する
Sipeed Longan NanoをオープンソースのRTOSのZephyr RTOSに
対応させた手順、手法を解説する。


GigaDevice GD32Vについて
---------------------------

GD32VはRISCVの命令セットを採用した中国GigaDevice社製のマイコン(SoC)である。
同社ではGD32というARMコアを採用したSoCを製造しており、この
CPUコアをRISCVに変更したものである。GD32の大きな特徴として、米ST Microelectronics社の
STM32F103シリーズとピン互換で、ほぼ同一の機能を実装したクローンだということが挙げられる。
ペリフェラル類も同等、レジスタも含めて類似しておりSTM32のリプレースとして使うことができる。
(STM32F103が廉価で非常に良く利用されているマイコンである。)

GD32VのRISCV実装は中国 芯来科技(Nuclei System Technology)のN200を利用している。
N200ではRISCV標準の命令を拡張、追加しており、多重割り込みのハンドリングの高速化のための
追加機能が実装されている。

同社ではHummingbird E203というRISCV実装をオープンソースで公開している。


Longan Nanoについて
-----------------------------

Longan NanoはGD32Vを搭載した中国Sipeed社製のコンパクトなマイコンボードである。
160x80の小型のカラー液晶を搭載している点以外は、ほぼ単純にGD32Vの端子を
基板上に引き出したブレイクアウトボードである。実売価格が800円前後と
手頃で、小型ながらも液晶がついているため

Zephyr について
-------------------------------

ZephyrはLinux Foundationが開発を推進するRealtime OSである。
Linuxで実績のあるデバイスドライバの構成管理システムを移植しており、
多種多様なボード、デバイスがサポートされている。
開発も活発で、オープンソースのIoTプラットフォームとしても注目されている。

本稿で作成したソースのサンプルは如何に格納する。

[https://github.com/soburi/zephyr/tree/longan_nano_tutorial](https://github.com/soburi/zephyr/tree/longan_nano_tutorial)