# 2026-11-23 メガCDのヘッドレス録画

| 項目 | 値 |
|---|---|
| 上流基準 | `5702187402276e7f0aea2ad4e6784ffb187c8c10` |
| 独自コミット | [`8a2cd7803cd43aa135456251b84736b756706014`](https://github.com/akiyan/blastem/commit/8a2cd7803cd43aa135456251b84736b756706014) |
| 直前の版 | `d9320de7bb79182aa9d4c3e6d341d14648689adc`（タグ `akiyan-2026-11-23-d9320de`） |
| 実行環境 | Linux x86_64、SDL、GCC、Go 1.27.0 |

単体CLIの `--record` をメガCDディスクに広げた。従来のフレーム指定入力、
分割APNG、WAV、完了JSONが使える。メガCDでもVDPの有効表示矩形を
`active_areas` に記録する。ウィンドウなしの録画中にネイティブデバッガーが
画面更新を呼び出した場合は、その更新を省略する。
[ヘッドレス録画の説明](../docs/features/offline-recording.md)

## 検証

- `tools/akiyan-check.sh` が成功。本体のPython試験27件、Goのvet・race、
  XvfbとヘッドレスでのMCP統合試験を実行した。
- 日本向けのCAVC合成ディスクで68フレーム、17段階のパレットを順に表示した。
  `--record-input` を固定した1,900フレームの2回の録画で、APNGの
  1,800フレーム分割をまたいでデコード済みの画素・PCM・時刻と
  `-e` のBLSTELバイト列が一致した。HUDのsector slip、control desync、
  audio resyncは0で、表示cadenceには警告が残った。
- CAVCの `PCMTEST` を使い、WAVからRF5C164の246.5 Hzのテスト音を測定した。
- SDL dummyドライバーで `-d --record` を開始し、ウィンドウなしで
  テキストデバッガーの `quit` を実行して正常終了した。

## 未確認の範囲

北米向けCUEとBIOSでは1,800フレームのAPNG/WAVまで確定したが、
合成ディスクは `FFFE` の起動画面に留まった。同じ再生の900走査フレーム目に、
MainはCAVCの待機ループ、SubはBIOS内を実行していた。北米向けの本編進行と、
DMA・RF5C164の実機タイミングは別途検証する。制御ソケットv2のメガCD対応と
DMA転送元を識別する構造化ログも今後の範囲。
