# Zephyr の WASM 実行環境の調査

## Zephyr 本体に WASM ランタイムはない

Zephyr upstream (`zephyrproject-rtos/zephyr`) の `west.yml` を main と v4.4.0 の両方で確認しましたが、WASM ランタイムのプロジェクトは登録されていません。`samples/` にも WASM のサンプルはありません (2026-11-22 時点)。

したがって Zephyr で WASM を実行するには、外部ランタイムを Zephyr モジュールとして追加するのが公式に近い形です。追加の方法は 2 通りあります。

- アプリの west マニフェスト (T2) でランタイムを project として宣言する
- 既存ワークスペースのビルド時に `ZEPHYR_EXTRA_MODULES` でチェックアウトを指す

## ランタイムの選択肢

| ランタイム | Zephyr 対応 | 保守状況 | 評価 |
|---|---|---|---|
| WAMR (wasm-micro-runtime) | 公式ポート `product-mini/platforms/zephyr` あり。Zephyr モジュールとして組み込める | 活発 (2026-11 時点で Zephyr 関連コミットが連続) | 採用 |
| wasm3 | 過去に Zephyr で動かした例はあるが専用ポートなし | 低調 (実質更新が止まっている) | 不採用 |
| Wasmi / WasmEdge / Wasmtime | MCU・RTOS 向けでない | - | 対象外 |

WAMR を選ぶ理由は、Zephyr 対応がランタイム本家のツリー内で保守されている唯一の選択肢だからです。詳細は [community.md](community.md) にまとめています。

## WAMR の Zephyr ポートの構成

`wasm-micro-runtime/wasm-micro-runtime` リポジトリ内の `product-mini/platforms/zephyr` がポートです。2026-08-07 のコミット `0e5b221` ("zephyr: expose WAMR as a Zephyr module") で、リポジトリ直下の `zephyr/module.yml`・`zephyr/Kconfig`・`zephyr/CMakeLists.txt` が追加され、標準的な Zephyr モジュールとして組み込めるようになりました。

要件は Zephyr 3.7.0 以上です (ポートの README 記載)。本リポジトリでは Zephyr 4.4.2 (NCS v3.4.2) でビルドできることを確認済みです。

### サンプル (upstream 本家)

| サンプル | 内容 |
|---|---|
| `simple` | 最小構成。built-in libc で WASM モジュールを実行 |
| `simple-file` | WASI のファイルシステム API を Zephyr `fs_*` の上に実装 |
| `simple-http` | WASI のソケット API を Zephyr `zsock_*` の上に実装 |
| `user-mode` | Zephyr のユーザーモードスレッド内でランタイムを実行 |
| `user-mode-multi-thread` | 準備済みスレッドプールと同期オブジェクトでゲスト pthread を実行 |

このほか `tests/` 以下に Ztest スイート (runtime, usermode-faults, platform-api, thread-pool) があります。

### Kconfig オプション (main ブランチ)

`zephyr/Kconfig` が提供する主なオプションです。

- `CONFIG_WAMR`: ランタイムをアプリケーションイメージに組み込む
- `CONFIG_WAMR_INTERP` (既定 y): インタプリタ
- `CONFIG_WAMR_FAST_INTERP`: fast インタプリタ。速いがプリコンパイル済みバイトコード用のメモリを多く使う
- `CONFIG_WAMR_AOT` (既定 y): wamrc が生成する AOT モジュールの実行
- `CONFIG_WAMR_LIBC_BUILTIN` (既定 y): 最小の内蔵 libc (printf 系)
- `CONFIG_WAMR_LIBC_WASI`: WASI libc。ファイルシステムとソケットが必要
- `CONFIG_WAMR_LIB_PTHREAD`: WASM 側への pthread 提供
- `CONFIG_WAMR_GLOBAL_HEAP_POOL` / `CONFIG_WAMR_GLOBAL_HEAP_SIZE`: グローバルヒープの確保方式とサイズ
- `CONFIG_WAMR_REF_TYPES` など Wasm proposals の個別有効化

### ビルドターゲット

`zephyr/CMakeLists.txt` がボードのアーキテクチャから `WAMR_BUILD_TARGET` を自動導出します。ARM Cortex-M33 (nRF54L15) は `THUMB` になり、`THUMBV7` などのサブバリアントへ `-DWAMR_BUILD_TARGET=` で上書きできます。

### スレッドモデル

アプリケーションが作るスレッドと WAMR が作る worker は別系統です。`os_thread_*` API は WAMR のポーティング層で、初期化 (`wasm_runtime_init`) と終了 (`wasm_runtime_destroy`) の間でだけ有効です。ゲスト pthread や `wasm_runtime_spawn_thread()` には、アプリケーションが `WAMR_ZEPHYR_THREAD_POOL_DEFINE` で用意するスレッドプールが必要です。今回の simple サンプルは単一スレッドなのでプールは不要です。

## メモリと性能の公開データ

WAMR Wiki (nucleo_f767zi, Cortex-M7 での CoreMark 測定、2020-09):

- インタプリタのピークメモリ: classic 約 365 KB、fast 約 485 KB、wasm3 約 514 KB
- 実行速度: wamr-aot は wasm3 の約 19〜30 倍、wamr インタプリタは wasm3 と同等〜1.65 倍。つまり AOT はインタプリタに対して概ね 1〜2 桁速い
- WASM バイトコードは GCC ネイティブバイナリより小さい (coremark: 23,120 B → 10,457 B)

メモリモデルの要点 (`doc/memory_tune.md`):

- global heap: ランタイム内部構造・モジュール・実行環境・operand stack のためのヒープ。`wasm_runtime_full_init` の `RuntimeInitArgs` で確保方式 (ユーザーバッファ/allocator/os_malloc) を選べる
- operand stack: WASM スタックマシンのオペランド置き場
- linear memory: WASM モジュールが見る連続メモリ (data・補助スタック・ヒープ)。wasi-sdk では `--initial-memory` で初期サイズを決める

本リポジトリのビルドでは `CONFIG_WAMR_GLOBAL_HEAP_SIZE=131072` (128 KB) と WASM 側 `--initial-memory=65536` (64 KB) で RAM 使用量が 154,400 B / 256 KB (58.9%) でした。詳細は [../verification/build-2026-11-22.md](../verification/build-2026-11-22.md) を参照してください。

## AOT について

`CONFIG_WAMR_AOT=y` は既定で入りますが、AOT モジュールの生成にはホスト側で LLVM 連携した `wamrc` が要ります。`simple` サンプルが実行するのはインタプリタ向けの .wasm バイナリなので、今回の検証はインタプリタのみです。AOT の試行は今後の拡張候補です。
