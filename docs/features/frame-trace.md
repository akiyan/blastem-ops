# フレームごとの値の記録と条件停止

指定した値を毎フレーム記録し、値が変わったフレームだけを返す。条件を満たしたフレームで止めることもできる。画面を見ずに「何フレーム目に何が変わったか」を数値で確認するための機能。

## MCP: `frame_trace`

`frame_pause` で止めてから呼ぶ。

```json
{
  "frames": 600,
  "watches": [
    {"name": "fade", "address": 16711936, "size": 2},
    {"name": "backdrop", "space": "vdp_reg", "address": 7},
    {"name": "color1", "space": "cram", "address": 2}
  ],
  "until": {"watch": "fade", "op": "eq", "value": 0},
  "max_records": 500
}
```

| 項目 | 内容 |
|---|---|
| `frames` | 最大で進めるフレーム数。1〜36000 |
| `watches` | 1〜16個。`space` は `ram`（既定。68000 アドレス FF0000〜FFFFFF）、`cram`（バイト位置 0〜127）、`vsram`（バイト位置）、`vdp_reg`（レジスタ 0〜23）、`vram`（バイトアドレス）、`z80_ram`（0〜1FFF）。`size` は 1・2・4 バイトのビッグエンディアン。既定は cram・vsram が 2、それ以外が 1 |
| `until` | 省略可。`watch`（名前または番号）、`op`（`eq`・`ne`・`changed`）、`value`（eq・ne で必須）、`mask`（比較前に適用するビットマスク） |
| `max_records` | 応答に含める記録数。既定 500、最大 5000。全記録はファイルに残る |

応答:

- `trace`: 停止時の `frame`、進めたフレーム数 `advanced`、記録数 `records`、停止理由 `stop`（`frames`・`until`・`interrupted`）
- `records`: 開始時点と、値が変わったフレームだけの一覧。`values` は名前ごとの値、`changed` は前の記録から変わった名前
- `truncated`: `max_records` を超えて省略したか
- `artifact`: 全記録の JSONL ファイル

`frame_interrupt` で途中停止できる。停止後は `frame_pause` と同じ状態で、そのまま `ram_read` や `paused_screenshot` を使える。

## 本体: v2 `trace`

capability は `frame_trace`。params は `frames`、`path`（絶対パス。既存ファイルには書かない）、`watches`（`space`・`address`・`size`）、`until`（`watch` は番号）。完了応答は `advance` の結果に `records` と `stop` を加えたもの。ファイルの各行は `{"frame":N,"values":[...]}`。

値はフレーム境界（VDP がフレームを描き終え、68000 がその時点まで同期した位置）で読む。読み出しはエミュレーター内部の配列から行い、CPU のバス・I/O の副作用はない。

## 制限

- GDB 起動時、CD・32X、threaded video は対象外（`unsupported`）
- 1 フレームより細かい変化（フレーム途中の書き込み）は記録しない。走査線単位の記録は [#24](https://github.com/akiyan/blastem/issues/24) で扱う
- `until` の `changed` は、前フレームとの比較。開始時点では判定しない

## 試験

- fixture のフレームカウンターで、31 記録が連続フレームになり、値が 1 ずつ増える
- 変化しない値は開始時点の 1 記録だけになる
- A を押すと VDP レジスタ 7 が 2 になるフレームで停止する。離したときの `changed`（マスク付き）でも停止する
- 不正な範囲・幅・空間・条件、相対パス、既存ファイル、実行中の他要求（`busy`）、`interrupt` を確認する
- MCP では、名前付きの値、`changed`、`max_records` による省略を確認する
