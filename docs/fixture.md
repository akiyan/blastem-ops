# 試験用 fixture ROM

`mcp/testdata/fixture/` は、本体と MCP の試験に使う 64 KiB の Mega Drive ROM のソース。生成物はリポジトリに入れず、GNU m68k binutils でソースからビルドする。独自のコード・素材だけで作っており、MIT ライセンス（`mcp/LICENSE`）で配布できる。

```sh
sudo apt-get install binutils-m68k-linux-gnu
make -C mcp fixture
```

`mcp/testdata/fixture/build/` に次を出力する。

- `blastem-mcp-fixture.bin`: ROM
- `blastem-mcp-fixture.elf`: デバッガー用シンボル
- `symbols.txt`: シンボルのアドレス
- `fixture.map`: リンカーマップ

## 観測できる内容

16個の既知タイルと、内容の異なる4つの CRAM パレットを初期化し、その組み合わせを4本の横帯に表示する。パッド1を毎フレーム読み取る。A を押している間は背景色を赤にし、PSG で約 437 Hz の音を鳴らす。32 KiB の SRAM を持ち、起動時に読み出し、A で書き込む。

| アドレス | シンボル | 意味 |
|---:|---|---|
| `0xFF0000` | `fixture_magic` | ASCII `BMCP`（`0x424D4350`） |
| `0xFF0004` | `fixture_frame_counter` | 検出した VBlank ごとに増える |
| `0xFF0008` | `fixture_input_state` | U,D,L,R,B,C,A,Start を bit 0〜7 に active-high で格納 |
| `0xFF000A` | `fixture_raw_high` | パッドの TH=high 時の生値 |
| `0xFF000B` | `fixture_raw_low` | パッドの TH=low 時の生値 |
| `0xFF0010` | `fixture_scratch` | 書込み確認用。初期値 `0x11223344` |
| `0xFF0024` | `fixture_hint_count` | C を押している間の H 割り込みの回数 |

起動時に Z80 RAM へ「0x1000 番地を増やし続けるループ」（`21 00 10 34 18 FD`）を書いて Z80 を動かす。B を押している間は 68000 が Z80 のバスを要求し続けるため、Z80 は止まる。

C を押している間は、毎ラインの H 割り込みを有効にし、割り込み処理で CRAM の 0 番（表示に使わない色）に回数を書く。離すと割り込みを止める。

`fixture_breakpoint` は毎フレーム、カウンターと入力の更新直前に1回実行される。試験ではコードのアドレスを固定値で書かず、`symbols.txt` から求める。
