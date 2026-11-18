# GDB Remote（68000 のデバッグ）

メイン CPU（68000）を GDB Remote Serial Protocol で操作する。基本機能は ulalume fork から引き継いだもので、akiyan では Ctrl-C による実行中断と、停止中の制御ソケットの利用範囲を追加した。

## 起動

```sh
# TCP（localhost の指定ポート）で待ち受ける
BLASTEM_GDB_PORT=2345 ./blastem rom.bin -D
m68k-elf-gdb build/rom.elf -ex 'target remote localhost:2345'

# ポートを指定しないときは標準入出力のパイプで接続する
m68k-elf-gdb build/rom.elf -ex 'target remote | ./blastem rom.bin -D'
```

MCP では `blastem_start` の `debug: true` が同じ起動を行い、`cpu_registers`・`memory_read`/`memory_write`・`breakpoint_set`/`remove`・`watchpoint_set`/`remove`・`cpu_step`・`cpu_continue` から使う（[停止の種類と GDB 実行の中断](stops.md)）。

## 対応するパケット

| 機能 | パケット |
|---|---|
| レジスタ | `g`（全体の読み出し）、`p`/`P`（1つの読み書き。D0〜D7、A0〜A7、SR、PC）。`G` は非対応 |
| メモリ | `m`、`M`（24 ビットのアドレス空間）。バイナリの `X` には空で応答し、gdb は `M` に切り替える |
| ブレークポイント | `Z0`/`Z1`、`z0`/`z1` |
| ウォッチポイント | `Z2`/`z2`（書き込みのみ）。読み出し・アクセス（`Z3`/`Z4`）には空で応答する |
| 実行 | `c`、`s`、`vCont;c/s`、`?`。実行中の `c` は Ctrl-C（0x03）で中断できる（TCP 接続時。約1フレーム以内に止まり `S02` を返す）。`k` は BlastEM を終了する |
| 問い合わせ | `qSupported`、`qAttached`、`qOffsets`、`qC`、`qfThreadInfo`/`qsThreadInfo`、`qSymbol`、`qTStatus` |

書き込みウォッチポイントに当たると、停止応答は `T05watch:アドレス` になる。

## 注意

- 対応していないパケットを受けると BlastEM は「Command ... is not implemented」と出して終了する。標準の gdb は問題ないが、他のクライアントは上の範囲で使う
- GDB クライアントは1つだけ
- GDB で止めている間も制御ソケットは動く。`status` は `boundary: instruction` を返し、`ram_read`・`vdp_dump`・`z80_state`/`z80_read`・`snapshot`・`save_flush` を使える。フレーム単位の操作（`pause`・`advance`・`trace`・状態保存など）は `-D` 起動では `unsupported`
- `-D` 起動では、ROM のデバッグ出力（`KDEBUG`）と `KIT` 行は標準出力ではなく標準エラーに出る
- 対象はメインの 68000 だけ。Z80 は [Z80 の観測](z80.md) で状態を読む
