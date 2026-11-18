# 音源別の録音と音源への書き込みログ

ヘッドレス録画（`--record`）に、チャンネルのミュート、映像なしの録音、YM2612・PSG への書き込みログを追加した。`tools/record_stems.py` はこれを使って音源別の WAV と、フレームごとの音量を書き出す。効果音の音量、鳴り終わり、BGM の進行を、聴かずに数値で確認できる。

## 音源別の録音: `tools/record_stems.py`

```sh
python3 tools/record_stems.py ROM .local/stems/run 3600 --record-input input.txt
python3 tools/record_stems.py ROM .local/stems/ch 3600 --per-channel   # 11チャンネル別
```

同じ ROM・入力で、全体ミックスと各ステムを別々に収録する。エミュレーションは決定的で、ミュートは出力段だけに作用するため、すべての WAV は同じ時間軸になる。既定のステムは `fm`（FM 1〜6）、`dac`、`psg`（トーン3つとノイズ）。`--per-channel` では `fm1`〜`fm6`、`dac`、`psg1`〜`psg3`、`noise`。

出力（PREFIX は指定した接頭辞）:

| ファイル | 内容 |
|---|---|
| `PREFIX.wav`、`PREFIX.json` | 全体ミックスとメタデータ |
| `PREFIX.<ステム>.wav`、`.json` | ステムごとの録音 |
| `PREFIX.sound.jsonl` | 全体ミックスの収録時の、音源への書き込みログ |
| `PREFIX.levels.jsonl` | フレームごとの `[AC RMS, ピーク, DC 平均]`（16bit フルスケール）。ミックスと各ステム |
| `PREFIX.stems.json` | ステムごとの AC RMS（dBFS 付き）、ピーク、有音フレーム数、最初と最後の有音フレーム、ミックスとステム合計の差 |

YM2612 の出力には一定の DC 成分があるため、有音の判定は DC を除いた AC RMS で行う（既定のしきい値 1.0、`--threshold` で変更）。フレーム番号は1始まりで、`--record-input` と同じく「k 番目の映像フレームを描くまでに生成された音」を k とする。numpy があれば集計に使い、なければ標準ライブラリだけで計算する。

## 本体の録画オプション

| オプション | 内容 |
|---|---|
| `--record-mute LIST` | 指定チャンネルを出力から外す。`fm1`〜`fm6`、`dac`、`psg1`〜`psg3`、`noise`、まとめ指定の `fm`・`psg` をカンマで区切る。DAC が有効な間、FM6 の位置の出力は `dac` として扱う |
| `--record-no-video` | APNG を書かず、WAV と JSON だけを出力する |
| `--record-sound-log` | `PREFIX.sound.jsonl` に YM2612・PSG への書き込みを記録する |

どれも `--record` と併用する。JSON には `video`、`muted`、`sound_log` を記録する。

書き込みログの各行:

```json
{"frame":121,"cycle":7940160,"chip":"ym2612","port":0,"reg":40,"value":240}
{"frame":121,"cycle":7944000,"chip":"psg","value":148}
{"frame":121,"dac_writes":245}
```

- `frame` は1始まりの、書き込みが起きたフレーム
- `cycle` は各音源の内部サイクル。フレーム境界ごとに基準を差し引くため、同じフレーム内の前後関係の比較に使う
- DAC のサンプル書き込み（ポート0のレジスタ 0x2A）は数が多いため1件ずつは書かず、フレームごとの件数 `dac_writes` だけを記録する。キーオン（0x28）や DAC 有効化（0x2B）などはすべて記録する

## 制限

- ステムの数だけ収録を繰り返すため、時間はステム数に比例する
- ミックスとステム合計の差は、リサンプルの丸めによる ±1〜2 程度。ミックス段のクリップが起きる区間では差が大きくなる
- MCP の `audio_capture` には未対応（ホストミキサーの録音で、チャンネル別の指定はできない）

## 試験

`tests/test_offline_recording.py` で、fixture の PSG 音を A 入力で鳴らし、次を確認する。

- PSG ステムの有音区間が入力と一致する（121〜241フレーム）
- FM・DAC ステムは無音で、ミックスとステム合計の差が 2 以下
- フレームごとの音量と、書き込みログの PSG 音量レジスタ値の区間が一致する
- `--record-no-video` で APNG が作られず、JSON に `muted` が入る
- 不正なチャンネル名と、`--record` なしのオプション指定を拒否する
