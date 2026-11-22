# study-Zephyr-WASM

nRF54L15-DK (`nrf54l15dk/nrf54l15/cpuapp`) で、Zephyr の WebAssembly (WASM) 実行を試すリポジトリです。基盤は nRF Connect SDK v3.4.2 です (NCS 初の LTS 系列、Zephyr 4.4.2)。

Zephyr には in-tree の WASM ランタイムがありません。実行には外部ランタイムを Zephyr モジュールとして追加します。このリポジトリは [wasm-micro-runtime](https://github.com/wasm-micro-runtime/wasm-micro-runtime) (WAMR) を使います。WAMR は Bytecode Alliance 発の WASM マイクロランタイムで、2026年8月に Zephyr モジュールとしてのビルド定義 (`zephyr/module.yml`・Kconfig・CMake) が整いました。NCS のマニフェストには含まれないため、west.yml でピン留めした版を `modules/wasm-micro-runtime` に追加しています。

## 構成

- `west.yml`: T2 型の west マニフェストです。`sdk-nrf` v3.4.2 を import し、`wasm-micro-runtime` を `modules/wasm-micro-runtime` に追加します。
- `app/`: WAMR 本家の `product-mini/platforms/zephyr/simple` サンプルを写した最小アプリケーションです。ビルド時に wasi-sdk でコンパイルした WASM バイナリをヘッダーに埋め込み、WAMR のインタプリタで実行して結果を UART コンソールに出します。
- `docs/`: 調査・検討・検証の記録です。索引は [docs/README.md](docs/README.md) にあります。

## 必要なもの

- nRF Connect SDK v3.4.2 ツールチェーン
  (`nrfutil sdk-manager toolchain install --ncs-version v3.4.2`)
- [wasi-sdk](https://github.com/WebAssembly/wasi-sdk) 29 (`share/cmake/wasi-sdk.cmake` を含む版。WAMR の CI は wasi-sdk-29 を使います。wasi-sdk-33 までは `wasi-sdk.cmake` がありますが、最新の wasi-sdk-34.0 では p1/p2/p3 のバリアントのみとなりそのままでは使えません)

  ```sh
  curl -LO https://github.com/WebAssembly/wasi-sdk/releases/download/wasi-sdk-29/wasi-sdk-29.0-x86_64-linux.tar.gz
  tar xzf wasi-sdk-29.0-x86_64-linux.tar.gz
  export WASI_SDK_DIR=<展開先>/wasi-sdk-29.0-x86_64-linux
  ```

## ビルド

### 方法 A: このリポジトリのマニフェストで専用ワークスペースを作る

```sh
west init -l <このリポジトリのパス>
cd <ワークスペース>
west update   # sdk-nrf・zephyr・wasm-micro-runtime (modules/wasm-micro-runtime) を取得

west build -b nrf54l15dk/nrf54l15/cpuapp -s study-Zephyr-WASM/app
```

`west update` は NCS 全体を clone します (数 GB)。

### 方法 B: 既存の NCS v3.4.2 ワークスペースを使う

NCS のワークスペースに WAMR を extra module として重ねます。

```sh
cd <NCS ワークスペース>
west build -b nrf54l15dk/nrf54l15/cpuapp -s <このリポジトリ>/app \
  -- -DZEPHYR_EXTRA_MODULES=<wasm-micro-runtime のチェックアウト>
```

WAMR のチェックアウトは west.yml と同じコミット `06f6c8c6e64e75eee6b8a3ce06ac2c7d10d015bf` に合わせてください。

## 動作の内容

起動すると WAMR のランタイムが初期化され、埋め込んだ WASM モジュール (`wasm-app/main.c` をコンパイルしたもの) をインタプリタで実行します。WASM 側の `printf` が `Hello world!` と検証用の文字列を出し、malloc と文字列比較の結果が戻り値としてホストへ返り、ホスト側がその結果を UART コンソールに出します。

実機での実行は本リポジトリでは試していません。ビルドの検証結果は [docs/verification/](docs/verification/) に記録しています。
