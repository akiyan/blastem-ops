# 導入と MCP ホストの設定

対象は Linux x86_64。必要なものは次のとおり。

- Go 1.27 以上
- X11 ディスプレイ。画面がない環境では Xvfb を使う
- 本体の [akiyan ブランチ](https://github.com/akiyan/blastem/tree/akiyan)

## 依存パッケージ（Ubuntu/Debian）

```sh
sudo apt-get update
sudo apt-get install -y pkg-config libsdl2-dev libglew-dev libgl-dev
# fixture ROM・統合試験・録画試験に使う
sudo apt-get install -y binutils-m68k-linux-gnu xvfb xauth ffmpeg
```

## ビルド

```sh
git clone --branch akiyan https://github.com/akiyan/blastem.git
cd blastem
make -j4 DEBUG=1          # エミュレーター本体: ./blastem
make -C mcp build fixture # MCP サーバー: mcp/blastem-mcp、試験用 ROM: mcp/testdata/fixture/build/
```

回帰試験は `tools/akiyan-check.sh [出力ディレクトリ]` でまとめて実行する。ビルドと次の試験を順に行う。

- Go の gofmt・vet・race 試験
- fixture ROM の生成と検査
- Xvfb 上での本体の Python 試験
- MCP の統合試験

試験中の設定と保存先は出力ディレクトリ内に分け、普段の BlastEM 設定・セーブには触れない。Unix ソケットのパス長の制限があるため、一時ファイルは短い `/tmp/akiyan-check.*` に作り、終了時に削除する。

## MCP ホストへの登録

MCP ホストから stdio サーバーとして起動する。stdout は MCP の通信に使うため、バナーや診断を stdout に出すラッパーで包まない。

```json
{
  "mcpServers": {
    "blastem": {
      "command": "/absolute/path/to/blastem/mcp/blastem-mcp",
      "args": []
    }
  }
}
```

最初のツール呼び出しでエミュレーターを起動する。

```json
{
  "binary_path": "/absolute/path/to/blastem/blastem",
  "rom_path": "/absolute/path/to/local-or-fixture.bin",
  "debug": false,
  "save_dir": "/absolute/path/to/save-profile",
  "symbols_path": "/absolute/path/to/symbol.txt"
}
```

`debug: true` は CPU レジスター、メモリ、ブレークポイント、ステップ実行を使う場合だけ指定する。GDB 起動中は、フレーム制御・録音・速度設定を使えない。`save_dir` は省略できる（[SRAM](features/sram.md)）。`symbols_path` を渡すと、各ツールでアドレスの代わりにシンボル名を使える（[シンボル名の指定](features/symbols.md)）。

## ツール一覧

| ツール | 用途 | 条件 |
|---|---|---|
| `blastem_start` | 管理対象のエミュレーターを1つ起動する | 本体と ROM のパス |
| `blastem_status` | PID、ROM、一時ディレクトリ、接続先を確認する | なし |
| `blastem_stop` | 保持中の入力を放し、停止・後片付けする | 起動中 |
| `button_down` / `button_up` / `release_all_buttons` | パッド入力 | 起動中 |
| `screenshot` | 次に描画されるフレームを PNG で返す | 起動中 |
| `paused_screenshot` | 最後に完成したフレームを停止したまま返す | 停止中 |
| `vdp_snapshot` | VRAM/CRAM/VSRAM/レジスター（`KITVDMP1`） | 起動中 |
| `vram_tiles` | CRAM の4パレット別のタイル一覧 PNG | 起動中 |
| `frame_pause` / `frame_advance` / `frame_interrupt` | フレーム単位の停止・送り・中断 | GDB なし |
| `audio_capture` | 指定フレーム数を進めて WAV を返す | GDB なし |
| `speed_set` | 停止中に進行速度を 25〜1000% に設定 | GDB なし、停止中 |
| `ram_read` | work RAM の 1〜256 バイトを読む | 停止中 |
| `vdp_layout` | 表示モード・スクロール・Plane/ウィンドウの名前表・スプライト一覧をテキストで返す | 停止中 |
| `symbol_lookup` | シンボルの検索と、アドレスの位置の名前付け | `symbols_path` 付きで起動 |
| `z80_state` | Z80 のレジスタ・状態・フレームごとのサイクル数・RAM を返す | 停止中 |
| `line_trace` | 走査線単位の CRAM・VSRAM・レジスタ書き込みと割り込み処理時間を記録する | GDB なし、停止中 |
| `frame_trace` | 値をフレームごとに記録し、変化したフレームを返す。条件で停止できる | GDB なし、停止中 |
| `save_flush` | SRAM を確実に保存する | `save_dir` 付き |
| `cpu_registers` / `memory_read` / `memory_write` | 68000 のレジスター・メモリ | debug、CPU 停止中 |
| `breakpoint_set` / `breakpoint_remove` / `cpu_step` / `cpu_continue` | 68000 のデバッグ | debug |

ボタン名は `up down left right a b c x y z start mode`。同じパッドで反対方向の同時押しは拒否する。

## 一時ファイルとログ

セッションごとに専用の一時ディレクトリを作る。その絶対パスは `blastem_start` と `blastem_status` が返す。ディレクトリには次を置く。

- `blastem.stdout.log`、`blastem.stderr.log`
- 制御用の Unix ソケット
- セッション中に作った画像・VDP ダンプ・録音

`blastem_stop` で削除されるため、調査に必要なログは事前に複製する。`save_dir` の profile は削除しない。

## 安全と制限

- 本体と ROM はローカルの通常ファイルだけを受け付ける
- GDB Remote は loopback の動的ポートだけで待ち受ける
- MCP サーバー1プロセスにつき、管理できるセッションは1つ
- メモリ読み書きは1回あたり 64 KiB まで
- ROM は信頼できる入手元のものを使い、一般ユーザー権限で実行する
- `cpu_continue` を使う前に、到達可能なブレークポイントを設定する。実行中の CPU を任意の位置で中断することはできない

## トラブルシューティング

- **ディスプレイを開けない:** `DISPLAY` を設定するか、`xvfb-run -a` 下で起動する。`BLASTEM_NO_GUI=1` でも X ディスプレイが必要
- **ALSA・音声デバイスのエラー:** 画面のない試験では `SDL_AUDIODRIVER=dummy` を使う（統合試験では自動で設定する）
- **前回のセッションが残っている:** `blastem_status`、`blastem_stop` の順に呼ぶ
- **画像取得が時間切れになる:** 本体が描画を続けているか、`blastem.stderr.log`、`TMPDIR` の空き容量を確認する
