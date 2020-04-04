ドライバを作成する
======================

Zephyrにおけるドライバ
---------------------

SoCが持つ各種ペリフェラルを動作させるためにドライバを作成する。

Zephyrでは`struct device`の構造体で、デバイスを表現しており、
これと関連付けて実行される関数群がドライバと言える。
Zephyrでのデバイスドライバの使われ方として、実体としてのデバイスを持たない
ものもデバイスとして扱って、初期化のシーケンスに組み込むという方法も良く使われている。

LチカではタイマーとGPIOの機能(の一部)を使うので、これを実装して、
ZephyrのLチカサンプルである`sample/basic/blinky`を動作させる。


RISC-VのMachine Timerドライバ
-----------------------------

RISC-VにはCPUに供給されるクロックを使ったタイマー機能がある。
Lチカは待ち時間を計測する必要があるので、これを有効にする。

```
CONFIG_RISCV_MACHINE_TIMER=y
```

このタイマー機能は仕組みとしてはGD32Vでもそのまま使えるはずなのだが、
GD32Vでは省電力化のために、このタイマーのクロックを1/4(4分周)して使っている。
このため、タイマーのカウント数を4倍にする必要がある。

タイマーが動作すると、kernelのスケジューラも動作が可能になるので、
マルチスレッドの実行ができるようになる。
先に無効化していた`CONFIG_MULTITHREADING`も有効に戻す。

GPIOのドライバ
-------------------

LチカではGPIOの出力機能を使って出力端子に接続したLEDを点滅させる。
GPIOは出力の他に、入力、割り込みなどの機能があり、全て実装すると
そこそこ考慮すべき事項がある。ここではLチカに必要な出力機能のみ実装を行う。

GPIOの機能はZephyrで標準的なAPIが定められており、
ドライバは`struct gpio_driver_api`に合わせて各種関数を実装する。
ここでは`config`と`write`を実装すればLチカには十分である。

ZephyrのドライバのTypicalな書き方では、スタティック変数としてデバイス情報
構造体を定義して、dtsから生成された値を設定する。

一部抜粋すると、

```
#define GPIO_DEVICE_INIT(__name, __suffix, __base_addr, __port, __cenr, __bus) \
       static const struct gpio_stm32_config gpio_stm32_cfg_## __suffix = {   \
               .base = (u32_t *)__base_addr,                                  \
       };                                                                     \
       DEVICE_AND_API_INIT(gpio_stm32_## __suffix,                            \
～～～
#define GPIO_DEVICE_INIT_STM32(__suffix, __SUFFIX)                   \
       GPIO_DEVICE_INIT(DT_GPIO_STM32_GPIO##__SUFFIX##_LABEL,        \
                        __suffix,                                    \
                        DT_GPIO_STM32_GPIO##__SUFFIX##_BASE_ADDRESS, \
～～～
```

のような書き方になる。多くのデバイスドライバでは、レジスタアドレスを起点にして
メモリ操作で制御を行うので、dtsにアドレスを記載してBASE_ADDRESSのマクロで参照する
コードが含まれる。



LEDの定義
------------------

dtsでGPIOに接続されているLEDの構成を記述することができる。
`samples/basic/blinky`ではLED0の点滅を行っているので、
LED0としてaliasを定義する。

```
leds {
    compatible = "gpio-leds";
    led_red: led_red {
        gpios = <&gpioc 13 GPIO_INT_ACTIVE_LOW>;
        label = "LED_R";
    };
};

aliases {
    led0 = &led_red;
};
```

ここまでで出来たこと
--------------------

ここまでで、タイマー機能の有効化、GPIOの(一部)実装を行った。
ここまで実装を行うと、`samples/basic/blinky`をコンパイル、実行が出来る。
