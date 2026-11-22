# 設計の決定事項

## 採用した構成

- 基盤: nRF Connect SDK v3.4.2 (NCS 初の LTS 系列の最新パッチ、Zephyr 4.4.2)
- ターゲット: `nrf54l15dk/nrf54l15/cpuapp`
- WASM ランタイム: wasm-micro-runtime (WAMR)、main 系コミット `06f6c8c6e64e75eee6b8a3ce06ac2c7d10d015bf` をピン留め
- アプリ: WAMR 本家の `product-mini/platforms/zephyr/simple` サンプルを `app/` に写し、`bin_to_header.cmake` の参照をモジュール内のパスに修正
- モジュールの組み込み: T2 west マニフェスト (`west.yml`、sdk-nrf v3.4.2 import + wasm-micro-runtime) を主経路とし、`ZEPHYR_EXTRA_MODULES` による既存ワークスペースへの重ね合わせを副経路として併記
- WASM バイナリ: ビルド時に wasi-sdk 29 で `wasm-app/main.c` をコンパイルして生成する upstream 方式を踏襲

## 決定の根拠と却下した選択肢

### ランタイムは WAMR

upstream Zephyr に in-tree のランタイムはなく、Zephyr 向けの保守されたポートを持つのは WAMR だけです。wasm3 は更新が止まっており、Wasmi・WasmEdge 系は MCU/RTOS 向けではありません。

### WAMR は最新リリースではなく main 系のコミット

最新タグ `WAMR-2.4.6` (2026-11-17) には `zephyr/Kconfig`・`zephyr/CMakeLists.txt` がなく、`zephyr/module.yml` は ext モードのため `CONFIG_WAMR` による統合が使えません。タグ側のサンプルはアプリ内で `runtime_lib.cmake` を組み込み `test_wasm.h` をリポジトリに持つ旧方式です。main 系は `CONFIG_WAMR=y` だけでランタイムが組み込まれ、`WAMR_BUILD_TARGET` もボードから自動導出されるため、今後の保守方向に沿う新方式を採用しました。main の HEAD ではなく検証済みコミットを固定値でピン留めしているため再現性は保たれます。

### 起点サンプルは `simple`

libc-builtin の最小経路が最も依存が少なく、nrf54l15 で無設定でビルドできます。`simple-file` (WASI fs) と `simple-http` (WASI socket) はファイルシステム・ネットワークの設定が要るため今後の拡張候補です。

### WASM バイナリはビルド時生成、あらかじめ生成済みヘッダーにはしない

upstream の現行方式に従い、wasi-sdk で `wasm-app/` をネストビルドして `test_wasm.h` を生成します。バイナリをコミットしないので WASM 側の変更がそのまま追跡できます。代償として wasi-sdk (29 系、wasi-sdk-34.0 は `wasi-sdk.cmake` を含まない) が必要です。

### global heap は既定の 128 KB のまま

nrf54l15 の 256 KB RAM に対し 58.9% で収まるため、upstream サンプルの `CONFIG_WAMR_GLOBAL_HEAP_SIZE=131072` を維持しました。RAM を他用途へ回すなら `CONFIG_WAMR_GLOBAL_HEAP_SIZE` と WASM 側 `--initial-memory` の両方を下げます。

### T2 マニフェストの採用

兄弟リポジトリ study-Zephyr-rust と同じ形で、リポジトリ自身が west マニフェストを持ちます。`west init -l` + `west update` で NCS v3.4.2 と WAMR の完全なワークスペースを再現でき、既存ワークスペースへの `ZEPHYR_EXTRA_MODULES` も代替手順として残します。

## 今後の拡張候補

- `simple-file`・`simple-http` での WASI 経由の fs/net 利用 (要: littlefs 等の fs 構成、ネットワークスタック)
- `user-mode`・`user-mode-multi-thread` でのユーザーモード・ゲスト pthread 動作 (要: `CONFIG_USERSPACE`、スレッドプール、メモリドメイン設定)
- AOT: ホストで `wamrc` (LLVM 連携) をビルドして .aot モジュールを生成し、インタプリタとの速度を比較
- fast インタプリタ (`CONFIG_WAMR_FAST_INTERP`) との速度・メモリ比較
- ヒープ削減チューニング (global heap 128 KB → 縮小)
- 実機 nRF54L15-DK への `west flash` と UART 出力確認
