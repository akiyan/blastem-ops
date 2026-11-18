# 状態保存・リセット・フレーム指定の入力

同じ状態から同じ入力を与えて、同じ結果を再現するための機能。問題の場面の直前を名前付きで保存し、入力や修正を変えながら何度でもやり直せる。

## MCP

| ツール | 内容 |
|---|---|
| `state_save {name}` | フレーム制御で止めた状態で、機械の状態全体を名前付きで保存する。止まったまま |
| `state_load {name}` | 同じ ROM から保存した状態を復元し、止まったままにする |
| `state_list` | 保存済みの状態の一覧（名前・大きさ・更新時刻） |
| `reset` | 押している入力を放し、次に進めたときの最初のサイクルでリセット（本体のリセットボタン）をかける |
| `button_press {pad, button, frames}` | 押してから N フレーム（1〜600）進め、最後のフレーム境界で放す |
| `input_sequence {events, frames}` | フレーム位置を指定した押す・放すの列を、1回の呼び出しで最大36,000フレーム分実行する。終わると押している入力を放す（`keep_held` で残せる） |

名前は 1〜64 文字の英数字と `.`・`_`・`-` で、英数字で始める。状態のファイルは `blastem_start` の `state_dir`（絶対パス）に置く。省略するとセッション専用のディレクトリを使い、セッション終了で消える。ROM を作り直すと状態は使えないため、再ビルドをまたぐ途中再開にはゲーム側のセーブ（[SRAM](sram.md)）を使う。

`state_load` の直後の `paused_screenshot` は、読み込み前に描いた画面を返す。画面を確かめるときは1フレーム進める。

```json
{"frames": 120, "events": [
  {"frame": 0, "pad": 1, "button": "a", "action": "down"},
  {"frame": 6, "pad": 1, "button": "a", "action": "up"},
  {"frame": 60, "pad": 1, "button": "start", "action": "down"},
  {"frame": 66, "pad": 1, "button": "start", "action": "up"}
]}
```

## 本体: v2

| コマンド | params | capability |
|---|---|---|
| `save_state`、`load_state` | `name` | `state` |
| `reset` | `kind`（`soft` のみ。省略可） | `reset` |

- 状態は `BLASTEM_STATE_DIR`（絶対パス）の `<名前>.state` に書く。形式は、識別子 `AKSTATE1`、版、ROM の SHA-1、保存時のフレーム、データの長さと SHA-1、BlastEM の状態データ。一時ファイルへ書いて fsync し、rename とディレクトリの fsync を終えてから応答する
- 読み込みでは、ROM の SHA-1 が違えば `invalid_state`、識別子・長さ・SHA-1 が合わなければ `io_error`、ファイルがなければ `not_found` を返し、機械の状態は変えない
- 保存は、止まっている 68000 を命令の境界まで進めてから行う（最大で1命令ぶん）。読み込み後は、フレームの境界を数え直す基準もそのフレームに合わせる
- `reset` は、対話用のリセットボタンが使う乱数の遅延を使わず、次に進めたときの最初のサイクルでリセットする。VDP のフレーム番号は 0 に戻らない。電源の入れ直し（`hard`）は `unsupported`
- `status` は ROM の SHA-1（`rom_sha1`）を返す

## 制限

- GDB 起動時、threaded video では使えない。状態保存は CD・32X も対象外
- 状態の互換性は、同じ BlastEM のビルドと同じ ROM に限る

## 試験

- 状態を保存し、A を押して記録した値の列と、読み込んでから同じ入力で記録した値の列が完全に一致する（68000 RAM、Z80 RAM、VDP レジスタ、CRAM）
- 名前の検証、存在しない状態、壊れた状態、別の ROM の状態を拒否する
- 同じ状態からのリセットを2回行い、結果が一致し、fixture のフレームカウンターが 0 から数え直される
- MCP では、`input_sequence` と `frame_trace` を同じ状態から2回実行して一致すること、`button_press`、`reset`、`state_list` を確認する
