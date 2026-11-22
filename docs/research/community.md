# コミュニティの動向の調査

## WAMR リポジトリの活発さ

WAMR は `bytecodealliance/wasm-micro-runtime` から `wasm-micro-runtime/wasm-micro-runtime` 組織へ移管されており、現在も活発です (2026-11-22 時点で最新コミットは 2026-11-20)。

Zephyr ポートまわりでは、2026 年 9〜11 月にかけて整備が連続しています。

- 2026-08-07 `0e5b221`: WAMR を Zephyr モジュールとして公開 (`zephyr/module.yml`・`zephyr/Kconfig`・`zephyr/CMakeLists.txt`)
- 2026-09 上旬: `product-mini/platforms/zephyr` を独立した application リポジトリとして再構成、Ztest スイートの CI 接続、twister でのサンプル・プラットフォーム選択
- 2026-09 下旬〜10 月: ユーザーモード fault カバレッジ、カバレッジ測定パイプライン、Zephyr モジュール経由での user-mode サンプルビルドへの移行 (`#5143`)
- 2026-11 中旬: アプリケーション提供のスレッドプール (`#5126`)、ユーザーモード同期プール (`#5128`)、静的プールによる同期のバック (`#5129`)、Zephyr スイートの簡素化 (`#5131`)、ユーザーモードでの WAMR スレッド例 (`#5132`)

最新リリースタグは `WAMR-2.4.6` (2026-11-17) ですが、このタグには `zephyr/Kconfig` と `zephyr/CMakeLists.txt` が含まれず (`zephyr/module.yml` は ext モードのみ)、Zephyr モジュールとしての Kconfig 統合は main 系でのみ利用できます。タグ側の `simple` サンプルは、アプリの CMakeLists 内で `runtime_lib.cmake` を直接組み込み、生成済みの `test_wasm.h` をリポジトリに持つ旧方式です。本リポジトリは Kconfig 統合を使うため main 系の `06f6c8c` をピン留めしています。

## 外部プロジェクト・研究での採用例

- Absmach の Propeller: Zephyr 上の WAMR を前提にしたエッジ実行基盤のドキュメントを公開しています
- Halmstad 大学の学士論文 (2025): i.MX RT1180 上の Zephyr + WAMR で FFT・行列乗算を測定。インタプリタのオーバーヘッドが大きく、当時の Zephyr ポートに WASI がなく計算主体の用途には不向きという結論 (現在の main では WASI 系サンプル simple-file・simple-http が追加済み)

## 代替ランタイムの状況

- wasm3: 超軽量インタプリタとして 2020 年頃に Zephyr で動かす例がありましたが、upstream の更新が止まっており、Zephyr 専用の保守されたポートがありません
- その他の WASM ランタイム (Wasmi・WasmEdge・Wasmtime): MCU/RTOS ターゲットではありません

結論として、Zephyr 上で現在も保守され upstream 公式に Zephyr ポートを持つのは WAMR だけです。コミュニティでも WAMR + Zephyr の組み合わせが事実上の標準になっています。

## Zephyr 側の受け入れ状況

upstream Zephyr の manifest に WAMR は入っておらず、in-tree 化の動きも確認できませんでした。WAMR 側が Zephyr のモジュール規約に合わせる形で統合されているため、今後も外部モジュール方式が続く見込みです。
