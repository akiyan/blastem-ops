# BlastEM akiyan fork 変更情報

[ulalume/blastem](https://github.com/ulalume/blastem)（`5702187402276e7f0aea2ad4e6784ffb187c8c10`）を基礎にしています。ulalume fork の既存機能を引き継ぎ、その上に以下の独自差分を1コミットで載せています。

- GDB Remote（[説明](docs/features/gdb.md)）
- Unix-domain の制御ソケット（パッド入力、撮影、連続フレーム撮影 `burst`、フレーム指定の入力スクリプト `script`、kit_prof プロファイラー、VRAM/CRAM/VSRAM/レジスターのダンプ。[説明](docs/features/control-socket.md)）
- `BLASTEM_HIDDEN_WINDOW=1` によるウィンドウ非表示での起動
- 組み込み UI フォント
- ヘッドレス実行

本体は [akiyan/blastem の akiyan ブランチ](https://github.com/akiyan/blastem/tree/akiyan)。コミットメッセージはこのページの URL のみとし、変更の説明と運用文書はこのリポジトリで管理します。

変更は Mega Drive ソフト全般の開発・検証用です。特定タイトルのシナリオ、シンボル、アドレスの意味は本体と MCP に持たせません。

## 独自変更（エミュレーター本体）

### 1. 設定・保存先の環境変数指定

`paths.c`

- `XDG_CONFIG_HOME` が絶対パスなら `$XDG_CONFIG_HOME/blastem` を設定ディレクトリにする
- `XDG_DATA_HOME` が絶対パスなら、その値を `$USERDATA` に使う
- 未指定・空文字・相対パスの場合は従来どおりホーム配下を使う。Windows・Android の分岐は変更しない
- 検証や自動実行を、普段のプレイ環境の設定・SRAM から分離するために使う

### 2. 応答付き制御プロトコル v2 と SRAM の確実な保存

`ctrl_v2.c`、`ctrl_v2.h`、`ctrl_fifo.c`、`blastem.c`、`genesis.c`、`genesis.h`、`gdb_remote.c`、`Makefile`、`third_party/yyjson/`

- 既存の制御ソケット（`BLASTEM_CTRL_SOCK`）で、1行1オブジェクトの JSON による v2 要求を受け付ける。接続の最初の文字が `{` なら v2 として扱い、従来の入力コマンドと JSON は同じ接続に混在させない
- `hello` は protocol 番号、上限値、capability 一覧を返す。クライアントは version ではなく capability で機能を判定する
- `status` は system、frame、停止状態、停止位置の種別を返す。種別は `instruction`、`frame_sync`、`paused_loop`、`event_poll` のいずれか
- 次の不正入力を拒否する
  - 1行65,536バイト・深さ16・文字列4,096バイト・1オブジェクト64フィールドの上限超過
  - 重複 key、小数、壊れた UTF-8
  - 接続内で増加しない要求 ID
- 使えない要求 ID には `id: null` で応答する。未対応 version にはエラーを返して切断する
- `BLASTEM_SAVE_DIR` に絶対パスのディレクトリを指定すると、ROM 名・ROM hash によらず同じ SRAM 保存先を使う
- `save_flush` は SRAM を一時ファイルへ書き、次の処理をすべて終えてから応答する
  - 一時ファイルの fsync、rename による置換、親ディレクトリの fsync
  - エミュレーター上のバッファは byte swap で書き換えない
- 書込み・置換・同期の失敗は `io_error` を返す。ディレクトリ同期の失敗は、置換後でも成功扱いにしない
- GDB 停止中も、命令を実行せずに制御ソケットの要求を処理する
- JSON parser として yyjson 0.12.0（MIT）を同梱し、システムの JSON ライブラリを不要にする
- capability: `save_directory`、`sram_flush`
- 対象は POSIX 上の通常 MD カートリッジ SRAM。EEPROM・flash・CD BRAM・32X・Windows での flush は `unsupported` を返す
- 詳細: [プロトコル仕様](docs/control-protocol-v2.md)、[SRAM](docs/features/sram.md)

### 3. 停止中の画面取得

`ctrl_v2.c`、`render.h`、`render_sdl.c`

- `snapshot {"path": 絶対パス}` は、最後に描画を完了したプログレッシブフレームを PNG で保存する。撮影のために CPU やフレームを進めない
- 応答には width、height、画像の更新番号（sequence）、停止時の VDP frame、`image_boundary`、`paused` を含める
- 未描画なら `not_ready`、実行中なら `invalid_state`、書込み失敗なら `io_error` を返す。相対パスと NUL を含むパスは拒否する
- 通常の SDL audio/video 同期が対象。OpenGL と SDL fallback renderer で検証済み
- インターレース中はキャッシュを無効化する。audio_thread・external 同期と非 SDL 描画は対象外
- capability: `cached_screenshot`
- 詳細: [停止中の画面取得](docs/features/paused-screenshot.md)

### 4. フレーム単位の停止・送り・中断

`ctrl_v2.c`、`ctrl_v2.h`、`ctrl_fifo.c`、`blastem.c`、`genesis.c`

- `pause` はゲーム側のブレークポイントなしで、次の VDP フレーム更新後の CPU 同期点（`frame_sync`）で停止する
- `advance {"frames": N}` は1〜600フレーム進めて停止する
- `interrupt` は実行中の advance を中断させる
- 応答には frame、advanced、paused、interrupted、boundary を含める
- advance 中に status・interrupt 以外の要求が来たら `busy` を返す。接続が切れたら次の同期点で停止する
- 対象は単体 MD の通常 SDL 同期。GDB 起動時、CD・32X、threaded・external 同期では `unsupported` を返す
- capability: `frame_control`
- 詳細: [フレーム制御](docs/features/frame-control.md)

### 5. ホスト音声の録音と WAV writer の修正

`ctrl_v2.c`、`render_audio.c`、`render_audio.h`、`wave.c`

- `audio_start` は絶対パスと1〜30秒の上限を受け取り、上限に達したら書込みを止める。`audio_stop` で WAV ヘッダーを確定する
- 未開始の終了と二重開始は拒否する。手動 WAV 録音とは別ファイルに書く
- 既存 writer の不具合を修正した
  - ヘッダー書込長の誤判定
  - float 出力から録音する際の変換バッファの拡張条件
  - 確定時の seek・write・close の失敗を通知していなかった点
- 録音はホストのミキサー出力の観測（`boundary: host_mixer`）。VDP フレームとサンプル区間の一致は保証しない
- capability: `host_audio_capture`
- 詳細: [音声録音](docs/features/audio-capture.md)

### 6. 停止中の速度設定と work RAM 観測

`ctrl_v2.c`

- `speed {"percent": 25〜1000}` は停止中に既存の速度変更を使い、ホスト上の進行速度を変える。フレームやサイクルは省略しない
- `ram_read` は停止したフレーム境界で、64 KiB の work RAM（`FF0000..FFFFFF`）から1〜256バイトを hex で返す
- CPU のメモリバスを通さないため、I/O の副作用や CPU の実行を伴わない
- capability: `speed_control`、`paused_work_ram`
- 詳細: [速度と RAM](docs/features/speed-and-ram.md)

### 7. 単体 CLI でのヘッドレス録画・録音

`blastem.c`、`render.h`、`render_sdl.c`、`png.c`、`png.h`、`sound_trace.c`、`sound_trace.h`、`ym2612.c`、`psg.c`、`tools/mux_capture.py`、`tools/record_stems.py`

- `--record PREFIX FRAMES [--record-input FILE]` は、画面サーバー・音声デバイス・外部エンコーダーなしで APNG と WAV を出力する
- 実時間待ちをせず、ゲーム内の時間を保ったまま収録する
- 出力するファイル
  - `PREFIX.apng`: 1,800フレームごとに分割する。NTSC は 320×240、PAL は 320×256 で、H32 の画面は中央に配置する
  - `PREFIX.wav`: 44,100 Hz、16 bit、stereo
  - `PREFIX.json`: 完了フレーム数、寸法、正確なフレームレート、サンプル数、分割情報、正常完了フラグ
- 入力ファイルの各行は `完了フレーム番号 pad番号 down|up ボタン`。pad は1〜8、ボタンは `up down left right a b c start x y z mode`
- 既存ファイルは上書きしない
- 次の場合は明示的に失敗する: interlace、途中の NTSC/PAL 切替、別機種、`-b` との併用
- 既存 APNG の終了処理で、acTL のフレーム数を書き換えた後に CRC を更新していなかった点を修正した
- JSON に、表示モードが同じ区間ごとの実表示領域（`active_areas`）を記録する
- `tools/mux_capture.py` は、完了フラグ・全分割の decode フレーム数・音声の長さを確認し、正確なフレーム時間で MP4 に結合する。`--crop auto` は実表示領域から切り出し範囲を決める
- `--record-mute` でチャンネル（FM1〜6、DAC、PSG トーン1〜3、ノイズ）を出力から外す。`--record-no-video` は音声と JSON だけを書く。`--record-sound-log` は YM2612・PSG への書き込みを1始まりのフレーム番号付きで記録する（DAC サンプルはフレームごとの件数）
- `tools/record_stems.py` は、ステムごとに収録を繰り返して音源別の WAV、フレームごとの AC RMS・ピーク・DC、有音区間、ミックスとの差を出力する
- 詳細: [ヘッドレス録画](docs/features/offline-recording.md)、[音源別の録音と書き込みログ](docs/features/audio-stems.md)

### 8. フレームごとの値の記録と条件停止

`ctrl_v2.c`

- `trace` は停止中から最大36,000フレーム進め、1〜16個の値（work RAM、CRAM、VSRAM、VDP レジスタ、VRAM、Z80 RAM、Z80 のフレームごとのサイクル数）をフレーム境界ごとに読む
- 値が変わったフレームだけを JSONL ファイルに書く（開始時点は必ず記録）
- `until` で、値の一致・不一致・変化（マスク付き）を満たしたフレームに止める。`interrupt` と切断でも止まる
- capability: `frame_trace`
- 詳細: [フレームごとの値の記録](docs/features/frame-trace.md)

### 9. 画面サーバーなしでの起動

`render_sdl.c`

- OpenGL のウィンドウを作れない映像ドライバー（`SDL_VIDEODRIVER=dummy` など）では、OpenGL なしでウィンドウを作り直し、SDL のソフトウェア描画へ切り替える。高速な描画ドライバーがないときもソフトウェア描画を使う
- MCP の `blastem_start.headless` はこれと dummy 音声ドライバー、`BLASTEM_HIDDEN_WINDOW` を使い、Xvfb なしでセッションを動かす

### 10. 停止中の VDP ダンプ

`ctrl_v2.c`

- `vdp_dump` は停止中（フレーム制御または GDB による停止）に、その時点の VRAM・CRAM・VSRAM・レジスタを KITVDMP1 形式で書く。既存ファイルには書かない
- capability: `paused_vdp_dump`
- 詳細: [VDP の表示内容のテキスト化](docs/features/vdp-layout.md)

### 11. 走査線単位の VDP 書き込みと割り込みの記録

`vdp_trace.c`、`vdp_trace.h`、`vdp.c`、`genesis.c`、`m68k_core.c`、`m68k_core.h`、`m68k_core_x86.c`、`m68k_internal.h`、`ctrl_v2.c`

- `line_trace` は停止中から1〜600フレーム進め、CRAM・VSRAM・VDP レジスタへの書き込み（VDP が反映した時点）と、割り込みの受付・RTE を、フレーム・ライン（vcounter）・スロット付きで JSONL に書く
- RTE は直前の割り込み受付と対応付け、その間の 68000 サイクル数を記録する。RTE の検出のため、68000 の JIT が RTE ごとに観測用の関数を呼ぶ（記録していないときは何もしない）
- capability: `line_trace`
- 詳細: [走査線単位の記録](docs/features/line-trace.md)

### 12. サウンド CPU（Z80）の観測

`z80_to_x86.c`、`z80_to_x86.h`、`ctrl_v2.c`

- Z80 の実行関数で、実行・68000 のバス要求による停止・リセットの区間をマスタークロックで積算する
- `z80_state` は停止中に Z80 のレジスタ（PC を除く）、リセット・バス要求の状態、直前のフレームと累計のサイクル数を返す。`z80_read` は Z80 RAM を 1〜256 バイト読む
- `trace` の値に `z80_frame`（直前のフレームのサイクル数）を追加
- capability: `z80_observation`
- 詳細: [Z80 の観測](docs/features/z80.md)

### 13. 名前付きの状態保存とリセット

`ctrl_state.c`、`ctrl_state.h`、`ctrl_v2.c`

- `save_state`・`load_state` は停止中に、`BLASTEM_STATE_DIR` の `<名前>.state` へ機械の状態を保存・復元する。ROM の SHA-1 とデータの SHA-1 を持つヘッダーで、別の ROM や壊れたファイルを拒否する。保存は fsync・rename で確定してから応答する
- `reset` は乱数の遅延なしで、次に進めた最初のサイクルにリセットをかける
- `status` に ROM の SHA-1 を追加
- capability: `state`、`reset`
- 詳細: [状態保存・リセット・入力](docs/features/state-and-input.md)

### 14. GDB 実行の中断

`gdb_remote.c`、`gdb_remote.h`、`ctrl_v2.c`

- GDB で実行を再開している間、フレームごとに接続を確認し、Ctrl-C（0x03）を受けたら次の命令の前で止めて SIGINT（`S02`）を返す
- GDB の命令境界で止まっている間も、制御ソケットの `ram_read` を使える
- 詳細: [停止の種類と GDB 実行の中断](docs/features/stops.md)

## 独自変更（MCP サーバー）

`mcp/`（Go、MCP stdio）

### 15. セッション・入力・観測・68000 デバッグ

- 管理対象の BlastEM を1つ起動・確認・停止する: `blastem_start`、`blastem_status`、`blastem_stop`
- パッド入力: `button_down`、`button_up`、`release_all_buttons`
- 次に描画されるフレームの撮影: `screenshot`
- VDP の観測
  - `vdp_snapshot`: VRAM、CRAM、VSRAM、VDP レジスター
  - `vram_tiles`: CRAM の4パレット別にタイル一覧を PNG で返す。撮影時に画面とパレットの整合を取る
- 68000 のデバッグ（GDB RSP 経由）
  - `cpu_registers`、`memory_read`、`memory_write`
  - `breakpoint_set`、`breakpoint_remove`、`cpu_step`、`cpu_continue`
- 配布可能な自作 MD fixture ROM（ソースからビルド）: `mcp/testdata/fixture/`（[内容](docs/fixture.md)）

### 16. 本体 v2 機能の公開

- `blastem_start.save_dir`
  - 起動前に capability を確認し、保存 profile を OS のファイルロックで排他する。symlink 経由の二重起動も拒否する
  - セッションの後片付けでは保存先を削除しない
- `save_flush`。profile 付きセッションの `blastem_stop` は先に flush し、失敗を隠さず返す
- `paused_screenshot`
- `frame_pause`、`frame_advance`、`frame_interrupt`
  - interrupt だけ同じ v2 接続で応答 ID を分けて送る。応答の順序が要求と違っても、ID に対応する呼出元へ返す
  - 要求がキャンセルされたら、その要求専用の接続だけを閉じる
- `audio_capture {"frames": 1〜600}`
  - 停止・録音開始・指定フレームの進行・録音終了をまとめて行い、WAV を MCP の audio content で返す
- `speed_set`、`ram_read`
- `frame_trace`: 値に名前を付けて記録し、前の記録から変わった名前（`changed`）を付けて返す。応答に含める記録数を制限でき、全記録はファイルに残る
- `vdp_layout`: 停止中の表示内容をテキストで返す。表示モード、スクロール、Plane A/B・ウィンドウの名前表を画面の行ごとの文字列にしたもの、スプライトのリンク一覧（表示判定、1ラインの上限超え、リンク異常）。パターンデータの CRC32 も付けられる
- `line_trace`: 走査線単位の書き込み・割り込みを、割り込みレベル別の回数・ライン範囲・処理サイクル数の集計とともに返す。ライン範囲で絞り込める
- `z80_state`: Z80 のレジスタ・状態・フレームごとのサイクル数と、任意の Z80 RAM 範囲を返す
- シンボル名: `blastem_start.symbols_path` で `nm -n` 形式のファイルを読み、`ram_read`・`memory_read`/`memory_write`・ブレークポイント・`frame_trace` でシンボル名（オフセット付き可）を使える。応答に最寄りのシンボル位置を付け、`symbol_lookup` で検索できる（[詳細](docs/features/symbols.md)）
- 状態と入力: `state_save`・`state_load`・`state_list`・`reset`、フレーム数を指定した `button_press`、フレーム位置を指定した入力列を1回で実行する `input_sequence`。`blastem_start.state_dir` で保存先を指定できる
- `watchpoint_set`・`watchpoint_remove`: GDB の書き込みウォッチポイント（Z2）をシンボル名でも指定できる
- `cpu_continue` の `timeout_ms`: ブレークポイントに届かなければ CPU を中断し、`interrupted`・停止位置を返す
- `blastem_start.headless`: 画面サーバーなしで起動する（SDL dummy ドライバー、ソフトウェア描画）
- AI エージェント向けの英語の操作ガイド `mcp/AGENT_GUIDE.md` を同梱し（MCP の各ツールに加え、GDB の直接利用、制御ソケットの v2・従来コマンド、スクリプト形式、環境変数を含む）、その要約を MCP の初期化応答（instructions）で返す。本体の `.gitignore` は `*.md` を除外しているため、このファイルだけ例外にしている
- 使い方: [導入と MCP ホストの設定](docs/setup.md)、[AI エージェント向けガイド](https://github.com/akiyan/blastem/blob/akiyan/mcp/AGENT_GUIDE.md)

## 回帰試験と CI

`tests/`、`tools/akiyan-check.sh`、`.github/workflows/akiyan-check.yml`、`.github/ISSUE_TEMPLATE/bug_report.yml`

- `tools/akiyan-check.sh` で、本体のビルド、Go の gofmt・vet・race 試験、fixture の生成・検査、本体の Python 試験、MCP の統合試験を順に実行する
- 本体の Python 試験
  - `test_paths.py`: 保存先の環境変数
  - `test_control_v2.py`: v2 プロトコル・保存・停止中撮影・フレーム制御・録音・速度・RAM・フレームごとの記録・VDP ダンプ・走査線単位の記録・Z80・状態保存とリセット
  - `test_wave_writer.py`: WAV writer
  - `test_offline_recording.py`: ヘッドレス録画、MP4 結合、音源別録音と書き込みログ
- MCP の統合試験は Xvfb 上の fixture で、入力・画像・VDP・GDB・保存・フレーム操作・録音・RAM・値の記録・表示内容・走査線単位の記録・Z80・状態保存と入力列・GDB 実行の中断を確認する
- MCP 統合試験と本体の制御試験は、Xvfb 上と画面サーバーなしの両方で実行する
- GitHub Actions は `akiyan` ブランチへの push と PR で同じ試験を実行する。ulalume fork のリリース用 `build.yml` は `v*` タグでだけ動き、変更していない

## 保証範囲とライセンス

- 対象は Linux x86_64 の SDL 版、通常の MD カートリッジ
- CD、32X、Windows、big-endian ホスト、任意の savestate は検証記録の対象外
- ライセンス
  - BlastEM 本体と、本体ソースへの変更は GPL-3.0-or-later
  - ulalume fork の変更は The Unlicense（`COPYING.fork`）
  - `mcp/` の MCP サーバーと fixture は MIT（`mcp/LICENSE`）
  - yyjson は MIT（`third_party/yyjson/LICENSE`）

## 検証と運用

- [導入と MCP ホストの設定](docs/setup.md)、[上流への追従とコミット形式](docs/upstream-sync.md)
- [統合時の検証記録](releases/2026-11-18.md)、[上流追従の記録](releases/2026-11-18-upstream-sync.md)、[観測機能の追加](releases/2026-11-18-observation.md)
- [統合前の設計記録](docs/history/README.md)
