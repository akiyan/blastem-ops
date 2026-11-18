# MCP でのシンボル名の指定

`blastem_start` の `symbols_path` にシンボルファイルを渡すと、アドレスの代わりにシンボル名を使える。応答には、アドレスを最も近いシンボルとオフセットで表した値（`main_loop+0x12` など）も付く。

## シンボルファイル

`nm -n` の出力形式（`アドレス 種類 名前`）か、`アドレス 名前` の2列。アドレスは16進。SGDK のビルドが出力する `symbol.txt` や、`m68k-elf-nm -n` の結果をそのまま使える。同じ名前が複数あるときは最初の行を使う。

## 使える場所

| ツール | 指定 | 応答に付く情報 |
|---|---|---|
| `ram_read` | `symbol` | `location` |
| `memory_read`、`memory_write` | `symbol` | `location`（read） |
| `breakpoint_set`、`breakpoint_remove` | `symbol` | `address`、`location` |
| `frame_trace` | 各 watch の `symbol`（名前を省略するとシンボル名になる） | — |
| `cpu_registers` | — | `pc_location`（PC の位置） |
| `symbol_lookup` | `name`（部分一致で検索）、`address`（位置の名前） | 一致したシンボル、`location` |

`symbol` は `名前`、`名前+オフセット`、`名前-オフセット` の形で、オフセットは10進か `0x` 付き16進。`address` と同時に指定したときは、両者が一致しなければエラーにする。シンボルファイルを読んでいないときにシンボルを指定するとエラーになる。

```json
{"symbol": "port_bgm_fade_frames", "size": 2}
{"watches": [{"symbol": "port_choice_active", "size": 2}], "until": {"watch": "port_choice_active", "op": "ne", "value": 0}, "frames": 600}
```

## 試験

- 読み込み、`+`・`-` のオフセット、未知の名前・不正なオフセット・範囲外の拒否、位置の名前付け、部分一致検索を Go の単体試験で確認する
- fixture で、シンボル名による RAM 読み出し・メモリ書き込み・ブレークポイント設定、停止後の `pc_location`、`frame_trace` のシンボル指定、`symbol_lookup` を確認する
