# BlastEM 単体でのヘッドレス録画・録音

`--record` は、画面サーバー・音声デバイス・MCP・外部エンコーダーなしで、ゲーム内の時間を保った APNG と WAV を出力する。実時間待ちをせず、CPU と保存先の性能で収録する。エミュレーションの master clock を変える speed 操作とは別の経路。

```sh
mkdir -p .local/capture-demo
./blastem --record .local/capture-demo/run 3600 \
  --record-input .local/capture-demo/input.txt ROMFILE
```

メガCDのディスクを録画するときは、CUEを入力に渡す。地域を固定するには
`-r J` または `-r U` を使い、BlastEmの設定に対応する
`system.scd_bios_jp` または `system.scd_bios_us` のBIOSファイルを指定する。
セーブも実行ごとに分けるときは、専用の `XDG_DATA_HOME` を使う。

```sh
./blastem -r J --record .local/capture-demo/cd 2400 \
  --record-input .local/capture-demo/start.txt DISC.cue
```

入力ファイルは省略できる。指定する場合は次の形式で、フレーム番号の昇順に並べる。同じフレームの複数イベントは記載順に適用する。

```text
# 完了フレーム番号 pad番号 down|up ボタン
180 1 down a
186 1 up a
```

上の例は、180番目の映像フレームを出力した直後に A を押し、186番目の直後に放す。

- フレーム番号は 1 から始まり、録画終了フレーム未満に限る
- pad 番号は 1〜8
- ボタンは `up down left right a b c start x y z mode`

既存のプレイと保存先を分けたい場合は、`XDG_DATA_HOME` に専用ディレクトリの絶対パスを指定する。通常の設定・SRAM の読込み規則はそのまま適用される。

音源ごとの録音や書き込みログは [音源別の録音と書き込みログ](audio-stems.md) を参照（`--record-mute`、`--record-no-video`、`--record-sound-log`）。

## 入力列を作りながら録画する

操作を決める実行（MCP や制御ソケット）と録画を並行させる。操作列ができるのを待ってから録画し直す時間がなくなる。

```sh
mkfifo .local/capture-demo/input.pipe
./blastem --record .local/capture-demo/run MAX_FRAMES \
  --record-follow .local/capture-demo/input.pipe ROMFILE   # - なら標準入力
```

操作を決める側は、次の行を書くたびに flush する。

| 行 | 意味 |
|---|---|
| `FRAME PAD down\|up BUTTON` | 上と同じ入力。フレームの昇順 |
| `FRAME through` | `FRAME` までの入力は書き終え、実行はその先へ続く。録画はそこまで進んで待つ |
| `FRAME end` | `FRAME` を最終フレームにする。すべての入力と `through` より後で、`MAX_FRAMES` 以下 |

- `through` より前のフレームの入力、逆戻りする `through`、範囲外の `end` は失敗にする
- `end` の前に列が閉じたら失敗し、`complete` の録画にしない
- 同じ入力なら、映像・音声・JSON は `--record-input` とバイト単位で一致する
- `--record-input` とは併用できない

## 出力

- `PREFIX.apng`、`PREFIX.part0001.apng` 以降: 1,800フレーム単位のロスレス映像
  - NTSC は 320×240、PAL は 320×256
  - H32 の画面は中央に配置し、H40 との切替も同じサイズで収録する。初期設定の overscan 除去値を使う
- `PREFIX.wav`: メインのミキサー出力。メガCDではRF5C164なども含む。44,100 Hz、16 bit、stereo
- `PREFIX.json`: 完了フレーム数、映像寸法、正確なフレームレートの分子・分母、音声サンプル数、映像の分割数と分割フレーム数、正常完了フラグ、実表示領域（`active_areas`）
  - `active_areas` は、表示モード（H32/H40、V28/V30）が同じ区間ごとの `from`・`to`（1始まりのフレーム番号）と、320 幅の画像の中で実際にゲーム画面が描かれる矩形（`x`・`y`・`width`・`height`）

既存ファイルは上書きしない。音声は1,024サンプル単位で同期的に混合するため、終了時に1ブロック未満の差と、VDP のフレーム出力位相の差が残る。音声を水増しせず、実際のサンプル数を記録する。PC の音声デバイスの速度調整やコールバックは使わない。

## MP4 への結合

APNG のフレーム間隔は 16bit の有理数で丸められる。長編を MP4 にするときは、JSON に保存した正確なフレーム時間から timestamp を組み立て直す。

```sh
python3 tools/mux_capture.py .local/capture-demo/run .local/capture-demo/playthrough.mp4
```

この入口は次を確認してから、1280×960・30 fps の確認用動画を出力する。

- 完了フラグ
- 全映像分割の実際の decode フレーム数
- WAV のサンプル数
- 映像と音声の長さの差（40 ms 以内）

`--crop auto` は `active_areas` をすべて含む矩形で切り出す（H32 だけの録画なら 256×224、H32 と H40 が混ざる録画なら 320 幅）。`--crop WIDTH HEIGHT X Y` で任意の範囲も指定できる。切り出した範囲を 1280×960（4:3）へ拡大する。APNG demuxer が duration を返さない場合に備え、concat 入力に各分割の duration を明示する。結合後も 30 fps のフレーム数と尺を metadata と照合し、不完全な動画は失敗にする。

## 長時間 APNG の分割

単一の APNG が 2 GiB を超えると、FFmpeg が途中までしか decode できなかった。そのため 1,800フレームごとにファイルを閉じ、次を独立した APNG として始める。320×256 RGB を非圧縮で換算しても1ファイル約 422 MiB なので、この上限には達しない。音声と入力のフレーム番号は分割せず連続させる。既存の APNG 終了処理が acTL のフレーム数を書き換えたときに CRC を更新していなかった点も修正した。

## 保証範囲

- 対象は通常の MD カートリッジとメガCDのディスク起動、progressive 映像、SDL 版の本体。zlib が必要
- interlace、途中の NTSC/PAL 切替、別機種では明示的に失敗する
- 任意の savestate、外部制御からの速度変更、32X、全タイトルでの動作は未検証
- `-b` とは併用できない。WAV の RIFF サイズ上限は従来どおり適用される
- 終了条件はフレーム数。指定した END に達したかの判定は呼出側が行う。録画中に強制終了すると出力が未完了になり得るため、配布前に JSON の完了フラグを確認する
- 検証用のCAVCディスクでは、日本向けBIOSで本編の進行、RF5C164のテスト音、映像分割境界の再現性を確認した。北米向けBIOSでは録画ファイルは正常確定したが、同じデータの北米向けディスクで本編に進まなかった。WAV出力や録画終了の成否とは別に、ディスク固有の再生完了を確認する

## 試験

`tests/test_offline_recording.py` は fixture を使い、DISPLAY を外し、SDL の映像・音声ドライバーに存在しない名前を指定した環境で実行する。

- NTSC・PAL で各600フレームを記録する
- A を押すと背景色が赤に変わり、PSG 音が鳴る。放すと無音に戻る
- 同じ入力で、映像・音声ファイルの SHA-256 が一致する
- 音声サンプル数と WAV ヘッダーが一致し、映像との長さの差が 40 ms 未満
- 1,801フレームを 1,800+1 に分割し、両ファイルの全 PNG chunk CRC と decode フレーム数を確認する。分割した録画の MP4 結合も確認する
- 既存ファイルを上書きせず、順序が不正な入力を実行前に拒否する
- 少しずつ書いたパイプ・標準入力からの録画が、ファイル入力の録画と一致する。不正な列と途中で閉じた列を拒否する
