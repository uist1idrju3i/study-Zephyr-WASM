# nRF Connect SDK の調査

## v3.4 系列は NCS 初の LTS

Nordic は nRF Connect SDK v3.4.0 を最初の LTS リリースとし、5 年間のパッチリリース (脆弱性とクリティカルなバグの修正のみ、破壊的変更なし) を提供します。LTS 系列では最新のマイナーリリースを使うのが公式の推奨です。

| リリース | 日付 | ベース |
|---|---|---|
| v3.4.0 | 2026-07 頃 | Zephyr 4.4, Mbed TLS 4.1.0, TF-M 2.3 |
| v3.4.1 | 2026-09-17 | Zephyr 4.4.2, Mbed TLS 4.1.1, TF-M 2.3.1 |
| v3.4.2 | 2026-11-04 | Zephyr 4.4.2 (ビルドログで確認) |

v3.4.2 は 2026-11-22 時点の最新パッチであり、本リポジトリが採用する版です。なお v3.4 系列は nRF52 シリーズをサポートする最後の NCS 系列でもあります (nRF52 は feature complete 扱い)。

## NCS と WASM の関係

NCS v3.4.2 のワークスペースを実際に確認しました。

- `zephyr/west.yml` と `nrf/west.yml` に WAMR・WASM の記述なし
- `zephyr/samples/modules/` に WASM 系サンプルなし (canopennode, chre, cmsis_dsp, lvgl, nanopb, tflite-micro, thrift のみ)
- `modules/` 以下に wasm 関連のモジュールなし (`modules/lib/liblc3/wasm` という無関係のディレクトリがあるのみ)

つまり NCS は WASM ランタイムを提供しないため、外部モジュールとして持ち込む必要があります。

## ツールチェーン

- `nrfutil sdk-manager` で SDK・ツールチェーンを管理します。本検証環境には v3.4.2 がインストール済みです
- `nrfutil sdk-manager toolchain launch --ncs-version v3.4.2 -- <コマンド>` でツールチェーン環境のコマンドを実行できます
- ツールチェーンの内容: Zephyr SDK 1.0.1 (GNU 14.3.0 arm-zephyr-eabi)、west 1.5.0、CMake 4.2.1、Python 3.12.4、dtc 1.5.0
- `toolchain launch` は `ZEPHYR_BASE` を設定しません。ワークスペースの外にあるアプリをビルドするときは、ワークスペース内で `west build -s <アプリ>` を実行するか、このリポジトリの T2 マニフェストで専用ワークスペースを作ります

## ビルド方式

NCS のアプリは既定で sysbuild 経由でビルドされます。nrf54l15dk/nrf54l15/cpuapp 向けの今回のビルドでも `_sysbuild` 構成が走り、`simple`/`app` イメージとして zephyr.elf・zephyr.hex・zephyr.bin が生成されました。ブートローダーは既定では含まれません。

## Zephyr の WASM 機能との位置づけ

「Zephyr の WASM 機能」は Zephyr 本体の機能ではなく、外部ランタイム (WAMR) の Zephyr ポートです。NCS v3.4.2 が提供する Zephyr 4.4.2 は WAMR ポートの要件 (Zephyr 3.7.0 以上) を満たし、本リポジトリで実ビルドが通ることを確認済みです。
