# 2026-11-22 作成中の入力列からの録画

| 項目 | 値 |
|---|---|
| 上流基準 | `5702187402276e7f0aea2ad4e6784ffb187c8c10` |
| 独自コミット | `d9320de7bb79182aa9d4c3e6d341d14648689adc` |
| 直前の版 | `0858b7af95fb5eb756ea8f1b29538586748e731e`（タグ `akiyan-2026-11-22-0858b7a`） |
| 実行環境 | Linux x86_64、GCC、Go 1.27.0 |

操作列を作る実行と録画を並行できるように、`--record-follow FILE` を追加した（[#29](https://github.com/akiyan/blastem/issues/29)、[説明](../docs/features/offline-recording.md#入力列を作りながら録画する)）。基準は変えず、独自コミットへ amend した。

## 検証

- `tools/akiyan-check.sh` がすべて成功（本体の Python 試験 27 件、Go の vet・race、MCP 統合試験）
  - `test_paths.py` は `zstd.h` を要するため、system に開発パッケージのない環境では同梱の `zstd/` を `C_INCLUDE_PATH` に加えて実行した
- fixture で、少しずつ書いた標準入力と名前付きパイプからの録画が、`--record-input` の録画と APNG・WAV・JSON ともバイト単位で一致した
- 不正な列と `end` の前に閉じた列は失敗し、`complete` にならなかった
- 利用側の MD ROM の 15,608 フレームで、入力列を分けて渡した録画が、直前の版の `--record-input` の録画の全 11 ファイルとバイト単位で一致した

## 制限

- 録画は入力列の確定を待つ。操作を決める実行が録画より遅ければ、その速さで進む
- `end` を書かずに終わった実行の録画は使えない
