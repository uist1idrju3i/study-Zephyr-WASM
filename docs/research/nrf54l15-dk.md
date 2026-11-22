# nRF54L15-DK の調査

## SoC とボードの仕様

nRF54L15 DK は nRF54L15 SoC を搭載する開発キットです。

- CPU: Arm Cortex-M33F @ 128 MHz (TrustZone・FPU・DSP 命令あり)
- RAM: 262,048 B (約 256 KB) が Zephyr から利用可能
- NVM: 1,524 KiB の RRAM (Resistive RAM)。従来の NOR フラッシュと違い消去単位や erase-before-write の制約が小さいメモリです (コントローラは `nordic,rram-controller`)
- 無線: Bluetooth Low Energy (Channel Sounding 含む Bluetooth 6.0)、802.15.4 (Thread/Zigbee/Matter)、独自 2.4 GHz、NFC
- RISC-V の補助コア (VPR) も搭載されますが小規模で、WAMR の実行対象は Cortex-M33 側です
- DK の付属品: SEGGER J-Link OB (書き込み・デバッグ・仮想 UART 2 ポート)、nPM1300 PMIC、外付け 64 Mb フラッシュ、4 ボタン・4 LED

## Zephyr のボードターゲット

| ターゲット | 用途 |
|---|---|
| `nrf54l15dk/nrf54l15/cpuapp` | メインターゲット (本リポジトリが採用) |
| `nrf54l15dk/nrf54l15/cpuapp/ns` | TF-M 非セキュア側ビルド |
| `nrf54l15dk/nrf54l10/cpuapp`・`nrf54l15dk/nrf54l05/cpuapp` | 同一 DK で下位 SoC を模擬 (RAM が小さい) |

## WASM 実行への適性

WAMR の `simple` サンプル相当をこのターゲットでビルドした結果です。

- FLASH: 145,000 B / 1,524 KB (9.29%)。WASM ランタイム込みでも NVM の 1 割以下です
- RAM: 154,400 B / 256 KB (58.90%)。内訳の大半は `CONFIG_WAMR_GLOBAL_HEAP_SIZE=131072` (128 KB) と WASM 側の linear memory 64 KB です
- WAMR_BUILD_TARGET は `THUMB` に自動導出されました

評価として、256 KB の RAM なら default の 128 KB ヒープのまま余裕があり、64 KB などに下げればさらに余白を作れます。RRAM は実行コードと埋め込み WASM バイナリの置き場として十分です。

## 検証の範囲

本リポジトリではビルド (zephyr.hex 生成) までを確認済みです。実機への書き込み・UART 出力の確認は、DK がある環境で `west flash` を実行して行います。実機での動作は本リポジトリでは試していません。
