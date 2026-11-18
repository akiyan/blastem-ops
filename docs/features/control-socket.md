# 制御ソケット（Unix ドメインソケット）

`BLASTEM_CTRL_SOCK` にパスを指定すると、BlastEM はその Unix ドメインソケットで外部からの命令を受け付ける。ウィンドウのフォーカスや疑似キー入力なしで、パッド入力・撮影・観測を行える。MCP サーバーはこのソケットと GDB を使っている。MCP を使わないスクリプトから直接操作することもできる。

- 1フレームごと（停止中や GDB で止めている間も）にソケットを確認する
- 同時に接続できるクライアントは1つ。次のクライアントは、前の接続が閉じてから受け付けられる
- 接続の最初の空白以外の1文字が `{` なら応答付きの JSON（v2）、それ以外なら従来の行コマンドとして扱う。1つの接続で両方を混ぜない

## 応答付き JSON（v2）: akiyan

1行に1つの JSON オブジェクトを送り、同じ `id` の応答を1つ受け取る。`id` は接続内で増やしていく。最初に `hello` を送り、`capabilities` で使える機能を確かめる。

```text
{"id":1,"version":2,"command":"hello","params":{"client":"me"}}
{"id":2,"version":2,"command":"pause","params":{}}
{"id":3,"version":2,"command":"advance","params":{"frames":60}}
{"id":4,"version":2,"command":"ram_read","params":{"address":16711680,"size":4}}
```

| コマンド | 内容 | 詳細 |
|---|---|---|
| `status` | フレーム番号、停止状態、停止位置の種類、ROM の SHA-1 | [停止の種類](stops.md) |
| `pause`、`advance`、`interrupt` | フレーム単位の停止・送り・中断 | [フレーム制御](frame-control.md) |
| `trace` | フレームごとの値の記録と条件停止 | [フレームごとの値の記録](frame-trace.md) |
| `line_trace` | 走査線単位の書き込み・割り込みの記録 | [走査線単位の記録](line-trace.md) |
| `ram_read`、`speed` | work RAM の読み出し、速度設定 | [速度と RAM](speed-and-ram.md) |
| `vdp_dump` | 停止中の VDP ダンプ（KITVDMP1） | [VDP の表示内容](vdp-layout.md) |
| `z80_state`、`z80_read` | Z80 の状態と RAM | [Z80 の観測](z80.md) |
| `snapshot` | 停止中の画面（最後に描き終えたフレーム） | [停止中の画面取得](paused-screenshot.md) |
| `save_flush` | SRAM の確実な保存 | [SRAM](sram.md) |
| `save_state`、`load_state`、`reset` | 名前付きの状態保存・復元、決定的なリセット | [状態保存・リセット・入力](state-and-input.md) |
| `audio_start`、`audio_stop` | ホストミキサーの録音 | [音声録音](audio-capture.md) |

`advance`・`trace`・`line_trace` は終わってから応答する。その間に受け付けるのは `status` と `interrupt` だけで、ほかは `busy` を返す。接続を閉じると、実行中の処理は次のフレーム境界で止まる。エラーコードと入力の制限は [プロトコル仕様](../control-protocol-v2.md) を参照。

## 従来の行コマンド: ulalume fork

改行で区切った命令を送る。応答はなく、結果は標準出力の `KIT ...` 行やファイルで確かめる。

| コマンド | 内容 |
|---|---|
| `pad <番号> down\|up <ボタン>` | パッドの押下・解放（MCP のボタン操作もこれを使う） |
| `screenshot <パス>` | 次に描画するフレームを保存する（.png / .ppm） |
| `burst <枚数> <接頭辞>` | 連続するフレームを `<接頭辞>00000.png` から順に保存する。`burst 0` で取り消す |
| `script <パス>` | フレーム指定の入力スクリプトを読み込む（下記） |
| `frame` | `KIT FRAME frame=<N>` を出力する |
| `logframes on\|off` | ROM のデバッグ出力の末尾に ` @f<フレーム>` を付ける |
| `vramdump <パス>` | 次のフレーム境界で KITVDMP1 を書き、`KIT VDUMP` を出力する |
| `vramhash on\|off` | フレームごとに VRAM・CRAM・VSRAM の FNV-1a ハッシュ（`KIT VHASH`）を出力する |
| `prof bracket <名前> <開始PC> <終了PC>` | 2つの PC の間で使った 68000 のサイクル数をフレームごとに数える |
| `prof lap <名前> <PC>` | 同じ PC を通る間隔（メインループの周期など）を数える |
| `prof log on\|off`、`prof clear` | `KIT PROF frame=<N> total=<サイクル> 名前=<サイクル>:<回数> ...` の出力、登録の消去 |
| `hud on\|off`、`hud gamefps <PC>`、`hud idle <PC\|auto>` | ゲームの FPS と CPU 使用率をウィンドウのタイトルと `KIT HUD` 行に出す |
| `watchlog add <アドレス>` / `clear` | 書き込みのログ（`KIT WATCH`）。新しい CPU コア（`NEW_CORE` ビルド）だけで動く |

ボタン名は `up down left right a b c x y z start mode`。PC とアドレスは16進。

### 入力スクリプト（`script`）

フレーム番号はリセットからの映像フレーム数。同じ ROM で同じスクリプトを再生すると、同じ画像が得られる。

```text
# <フレーム> down|up <ポート> <ボタン（カンマ区切り）>
120 down 1 start
126 up 1 start
300 shot /abs/out/title.png
600 burst 30 /abs/out/fade-
650 log reached menu
anchor MENU READY        # この行より下は、ROM のデバッグ出力にこの文字列が出たフレームを 0 とする
10 down 1 a
```

読み込んだ時点で過ぎているフレームの行はすぐに実行され、`KIT SCRIPT warn` が出る（その実行は再現性がない）。

## 出力: KIT 行と ROM のデバッグ出力

`KIT ...` 行と ROM のデバッグ出力は標準出力に出る（GDB の `-D` 起動では標準エラー）。MCP のセッションでは、`blastem_status` が返す一時ディレクトリの `blastem.stdout.log` に残る。

ROM は VDP レジスタ 30 に文字を書くと `KDEBUG MESSAGE: ...` を出力でき、レジスタ 31 で `KDEBUG TIMER: <サイクル>` を測れる（Gens KMod の方式。SGDK の `KLog`・`KDebug_Alert` など）。

## 環境変数

| 変数 | 内容 |
|---|---|
| `BLASTEM_CTRL_SOCK` | 制御ソケットのパス |
| `BLASTEM_GDB_PORT` | `-D` 起動で、GDB を localhost の TCP で待ち受ける |
| `BLASTEM_SAVE_DIR` | ROM 名によらない SRAM の保存先（絶対パス） |
| `BLASTEM_STATE_DIR` | 名前付きの状態の保存先（絶対パス） |
| `BLASTEM_HIDDEN_WINDOW=1` | ウィンドウを非表示で作る |
| `BLASTEM_NO_GUI=1` | メニュー UI を使わない |
| `XDG_CONFIG_HOME`、`XDG_DATA_HOME` | 絶対パスを指定すると、設定と保存先を普段の環境から分離できる |
| `SDL_VIDEODRIVER=dummy`、`SDL_AUDIODRIVER=dummy` | 画面・音声デバイスなしで動かす（MCP の `headless` が設定する） |
