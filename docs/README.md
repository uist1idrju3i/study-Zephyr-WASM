# ドキュメント索引

調査・検討・検証の記録です。すべて日本語で書いています。

## research (調査)

- [zephyr-wasm.md](research/zephyr-wasm.md): Zephyr の WASM 実行環境の調査。in-tree にランタイムがないこと、WAMR の Zephyr ポートの構成、Kconfig オプション、メモリと性能の公開データをまとめています。
- [community.md](research/community.md): コミュニティの動向の調査。WAMR リポジトリの移管と活発さ、代替ランタイム (wasm3 など) の状態、WAMR on Zephyr を扱った外部プロジェクト・研究をまとめています。
- [nrf-connect-sdk.md](research/nrf-connect-sdk.md): nRF Connect SDK の調査。v3.4 系列が NCS 初の LTS であること、v3.4.2 の内容、WAMR が NCS マニフェストに含まれないことをまとめています。
- [nrf54l15-dk.md](research/nrf54l15-dk.md): nRF54L15-DK の調査。SoC 仕様 (Cortex-M33F・256 KB RAM・1.5 MB RRAM)、ボードターゲット名、WASM 実行への適性をまとめています。

## plan (検討・計画)

- [design-decisions.md](plan/design-decisions.md): 採用した構成とその理由、却下した選択肢、今後の拡張候補をまとめています。

## verification (検証)

- [build-2026-11-22.md](verification/build-2026-11-22.md): nrf54l15dk/nrf54l15/cpuapp 向けビルドの再現手順と結果。FLASH・RAM の使用量と未検証の範囲を記録しています。
