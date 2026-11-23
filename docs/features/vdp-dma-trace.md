# 単体CLIのVDP DMAトレース

`--vdp-dma-trace FILE` は、通常のヘッドレス実行や録画にVDPの転送記録を追加します。`-e FILE` のBLSTELログも同時に使えます。トレースは明示指定したファイルに書き出します。

```sh
blastem -b 1200 -r J --vdp-dma-trace dma.tsv disc.cue
```

要求時のソースアドレスと宛先、バス読み出し、実際のVDPメモリ書き込み、完了を転送ごとに追えます。エミュレータのマスタークロックとVDPの走査位置を使って、VBlankをまたぐ転送や、CPUが転送後に行う書き戻しを調べます。

busy解除と最後の書き込みを別々に記録します。VDPが64 KiB VRAMへのワード書き込みを2スロットに分ける場合は、両方の時刻と区間が分かります。メガCDのWord RAMでは、要求アドレスと実際に返った値のアドレスを分けて確認できます。 [イベントと列の仕様](https://github.com/akiyan/blastem/blob/7cf8539472285567a6c615bbad827b45470ff109/docs/vdp-dma-trace.md)

メガCD実機の転送タイミングは、実機での計測が必要です。
