# 停止の種類と GDB 実行の中断

エミュレーターが止まる状態は2種類ある。どちらで止まっているかで使える操作が変わる。

| 停止 | 入り方 | `status.boundary` | 位置 |
|---|---|---|---|
| フレーム境界の停止 | `frame_pause`、`frame_advance` などの完了 | `frame_sync` | VDP がフレームを描き終えた時点 |
| 命令境界の停止 | GDB のブレークポイント、ステップ、中断（`debug: true` で起動したとき） | `instruction` | 68000 の命令の間。フレームの途中のこともある |

## 停止中に使える操作

| 操作 | フレーム境界 | 命令境界（GDB） |
|---|---|---|
| `ram_read`、`z80_state`、`vdp_layout`、`paused_screenshot`、`save_flush` | 可 | 可 |
| `cpu_registers`、`memory_read`/`write`、ブレークポイント、書き込みウォッチポイント、`cpu_step`、`cpu_continue` | —（GDB なしの起動では使えない） | 可 |
| `frame_advance`、`frame_trace`、`line_trace`、`audio_capture`、`speed_set`、`button_press`、`input_sequence`、`state_save`/`load`、`reset` | 可 | 不可（`unsupported`） |

命令境界で止まっているとき、`paused_screenshot` は最後に描き終えたフレームを返す。CPU はそのフレームより先の、次のフレームの途中にいることがある。`ram_read` の `frame` はその時点の VDP のフレーム番号。

## GDB 実行の中断

`cpu_continue` は、`timeout_ms`（既定 10,000、最大 600,000）以内にブレークポイントへ到達しなければ CPU を止める。応答の `interrupted` が true、`stop.signal` が 2（SIGINT）になり、`pc`・`pc_location` で止まった位置がわかる。到達しないブレークポイントや無限ループで、検証が止まり続けることはない。

本体は、GDB で実行を再開している間、フレームごとに GDB の接続を確認し、Ctrl-C（0x03）を受けたら次の命令の前で止めて `S02` を返す。そのため、中断までの遅れは最大で約1フレーム。

GDB を直接使う方法と対応するパケットは [GDB Remote](gdb.md) を参照。

## 書き込みウォッチポイント

`watchpoint_set {symbol|address, size}` は、その範囲へ 68000 が書き込んだ直後に CPU を止める。`cpu_continue` の停止応答は `T05watch:アドレス` になり、`cpu_registers` の `pc_location` で書き込んだ処理がわかる。読み出し・アクセスのウォッチポイントは本体が対応していない。

## 切断・期限切れ

- `frame_advance`・`frame_trace` などは、MCP の要求のキャンセルや接続の切断で、次のフレーム境界に止まる
- `cpu_continue` は MCP の要求がキャンセルされたときも中断を送り、止まった位置を返す
- `blastem_stop` と `reset` は、MCP が押している入力を放す。`input_sequence` は失敗やキャンセルでも押している入力を放す

## 制限

- GDB で起動したセッションでは、フレーム単位の操作を使えない。GDB の停止とフレーム境界の停止は1つのセッションで混在させない
- 中断の確認は TCP で接続したとき（MCP の `debug: true`）に有効。標準入出力のパイプ接続では行わない

## 試験

fixture で、実行されない例外処理にブレークポイントを置いて `cpu_continue` を 300 ms で中断し、SIGINT と停止位置が返ること、停止中に `ram_read` が使えること、その後に通常のブレークポイントへ到達できることを確認する。Go の単体試験では、偽の GDB サーバーで期限切れ時に 0x03 を送ることを確認する。
