# 2026-11-18 ulalume/blastem への追従

独自コミットを、ulalume/blastem の最新 main に載せ替えた。

| 項目 | 値 |
|---|---|
| 旧基準 | `4b71bfec3b5919367c5843f23c22c76ac24aa38c` |
| 新基準 | `5702187402276e7f0aea2ad4e6784ffb187c8c10`（+45コミット） |
| 旧独自コミット | `a6d033a06fbb2ada3c3001496b5a197cf9b69e7b`（タグ `akiyan-2026-11-18-4b71bfe`） |
| 新独自コミット | `ec7dca7`（cherry-pick） |

## 上流の主な変更

- upstream BlastEm の取り込み（v1.0.0 タグ、CHD の部分対応と zstd/lzma、32X・Sega CD・SMS・GG link port の修正、VDP の修正など）
- 制御ソケットの `burst <count> <prefix>`（連続フレーム撮影）、`script <path>`（フレーム指定の入力スクリプト）、`frame`、`logframes`、kit_prof の `prof lap`
- `BLASTEM_HIDDEN_WINDOW=1` でウィンドウを非表示で作る
- リリース用 CI の変更（macOS 15、draft 作成後に Latest を公開）

## 競合

`render.h` だけが競合した。上流の `render_save_screenshot_burst` と独自の `render_snapshot` の宣言を両方残した。`ctrl_fifo.c` は上流が `process_command` に追加したコマンドを残したまま、独自の v2 受付ループへ自動で統合された。上流は受付ループ（`ctrl_fifo_poll`）自体を変更していない。

## 検証

- clean build から `tools/akiyan-check.sh` がすべて成功した（本体ビルド、Go の gofmt・vet・race 試験、fixture、本体の Python 試験17件、MCP 統合試験）
- 利用側の MD ROM で、MCP によるフレーム送り・入力・停止中撮影を実行した。フレーム送りの結果と4画面は追従前と一致した。1画面だけ画面下端のデバッグ表示が異なったが、追従後の同じバイナリで2回実行しても同じ箇所が変わるため、実行ごとの変動で追従の影響ではない
