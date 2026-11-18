# 2026-11-18 画像を見ずに検証するための観測機能

エージェントが画像を見ずに、数値・テキストで検証できるようにする機能を追加した（[#21](https://github.com/akiyan/blastem/issues/21)〜[#25](https://github.com/akiyan/blastem/issues/25)、優先度「高」）。基準は ulalume/blastem `5702187` のまま、独自コミットへ amend した。

| 機能 | 本体 | MCP / ツール | 文書 |
|---|---|---|---|
| フレームごとの値の記録と条件停止 | v2 `trace` | `frame_trace` | [frame-trace](../docs/features/frame-trace.md) |
| VDP の表示内容のテキスト化 | v2 `vdp_dump` | `vdp_layout` | [vdp-layout](../docs/features/vdp-layout.md) |
| 音源別の録音と書き込みログ | `--record-mute`、`--record-no-video`、`--record-sound-log` | `tools/record_stems.py` | [audio-stems](../docs/features/audio-stems.md) |
| 走査線単位の書き込み・割り込み | v2 `line_trace`、68000 JIT の RTE 観測 | `line_trace` | [line-trace](../docs/features/line-trace.md) |
| サウンド CPU の観測 | v2 `z80_state`・`z80_read`、Z80 のサイクル積算 | `z80_state`、`frame_trace` の `z80_frame` | [z80](../docs/features/z80.md) |

fixture に、C ボタンで毎ラインの H 割り込みが CRAM を書く動作、Z80 のカウンタープログラム、B ボタンで Z80 のバスを要求する動作を追加した。

## 検証

- `tools/akiyan-check.sh` がすべて成功（本体の Python 試験23件、Go の vet・race、MCP 統合試験）
- 利用側の MD ROM（サウンドノベルの移植版）で確認した値:
  - `vdp_layout`: H32、Plane A に文字面（837セル）、Plane B に背景（896セル、パレット別の内訳）、スプライト 56 件のリンク（表示中 0）
  - `frame_trace`: 選択肢が出るまでの変数の変化を条件停止で検出（132フレーム）
  - `line_trace`: 選択中の行の色替えで、CRAM の 15 番が line 35 で選択色、line 74 で元の色に戻る。H 割り込み処理は 225 回、平均 308・最大 812 サイクルで、1ライン（約 488 サイクル）を超える割り込みがある
  - `z80_state`: XGM2 ドライバで、1フレームあたり実行約 59,200、68000 のバス要求による停止約 500 サイクル
