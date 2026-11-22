# blastem-ops

[akiyan/blastem](https://github.com/akiyan/blastem/tree/akiyan) の運用リポジトリ。実装・試験・Issue は本体、変更の説明・導入・上流追従・検証記録はここで管理する。

本体は [ulalume/blastem](https://github.com/ulalume/blastem) の上に、Mega Drive ソフトをコーディングエージェントから操作・観測・検証するための独自差分を1コミットで載せている。

- エミュレーター本体: 応答付き制御プロトコル v2、SRAM の確実な保存、停止中の撮影、フレーム単位の停止・送り、音声録音、速度設定、work RAM 観測、ヘッドレス録画
- `mcp/`: これらと既存の入力・撮影・VDP・68000 デバッグを公開する MCP サーバー（Go、stdio）

## 導入

```bash
git clone --branch akiyan https://github.com/akiyan/blastem.git
cd blastem
make -j4 DEBUG=1
make -C mcp build fixture
tools/akiyan-check.sh   # 回帰試験。Xvfb・ffmpeg・m68k binutils が必要
```

MCP ホストには `mcp/blastem-mcp` を stdio サーバーとして登録する。詳細は [導入と MCP ホストの設定](docs/setup.md)。

## 文書の言語と置き場所

| 置き場所 | 言語 | 内容 |
|---|---|---|
| このリポジトリ（blastem-ops） | 日本語 | 変更情報、導入・運用・上流追従の手順、機能ごとの説明、検証記録 |
| 本体（akiyan/blastem の `docs/`、`mcp/AGENT_GUIDE.md`） | 英語 | 本体と一緒に配布する利用者・AI エージェント向けの説明（GDB Remote、制御ソケット、プロトコル仕様、MCP の操作ガイド） |

- このリポジトリの文書は日本語で書き、日本語で更新する。英語の説明が必要な内容は本体側に置き、ここからはリンクする
- 本体側の英語の文書は、同じ内容をこのリポジトリに日本語で複製しない。本体の機能を変えたら、本体の英語文書とこのリポジトリの日本語文書（変更情報・機能説明）をそれぞれ更新する
- [統合前の設計記録](docs/history/README.md) は、旧リポジトリの英語の原文をそのまま残している例外

## 文書

- [AI エージェント向けの操作ガイド（英語、本体の `mcp/AGENT_GUIDE.md`）](https://github.com/akiyan/blastem/blob/akiyan/mcp/AGENT_GUIDE.md)
- [akiyan fork の変更情報](blastem_akiyan_fork.md)
- [導入と MCP ホストの設定](docs/setup.md)
- 本体側の英語ドキュメント（`docs/`）: [制御ソケット（v2・従来コマンド・スクリプト・環境変数）](https://github.com/akiyan/blastem/blob/akiyan/docs/control-socket.md)、[GDB Remote](https://github.com/akiyan/blastem/blob/akiyan/docs/gdb.md)、[制御プロトコル v2 の仕様](https://github.com/akiyan/blastem/blob/akiyan/docs/control-protocol-v2.md)
- 機能: [SRAM](docs/features/sram.md)、[停止中の画面取得](docs/features/paused-screenshot.md)、[フレーム制御](docs/features/frame-control.md)、[音声録音](docs/features/audio-capture.md)、[速度と RAM](docs/features/speed-and-ram.md)、[ヘッドレス録画](docs/features/offline-recording.md)、[フレームごとの値の記録](docs/features/frame-trace.md)、[VDP 表示内容のテキスト化](docs/features/vdp-layout.md)、[音源別の録音と書き込みログ](docs/features/audio-stems.md)、[走査線単位の記録](docs/features/line-trace.md)、[Z80 の観測](docs/features/z80.md)、[シンボル名の指定](docs/features/symbols.md)、[状態保存・リセット・入力](docs/features/state-and-input.md)、[停止の種類と GDB 実行の中断](docs/features/stops.md)
- [試験用 fixture ROM](docs/fixture.md)
- [上流への追従](docs/upstream-sync.md)
- [統合時の検証記録](releases/2026-11-18.md)、[上流追従の記録](releases/2026-11-18-upstream-sync.md)、[観測機能の追加](releases/2026-11-18-observation.md)、[作成中の入力列からの録画](releases/2026-11-22-record-follow.md)
- [統合前の設計記録](docs/history/README.md)

## 公開する情報

市販 ROM、BIOS、ROM 由来の画像・音声・セーブはこのリポジトリと本体に置かない。試験には、ソースからビルドする自作の fixture ROM だけを使う。検証で生成した画像・録音・ログは各自の作業ディレクトリ（`.local/` など）に置く。

文書は MIT ライセンス。本体のライセンスは [変更情報](blastem_akiyan_fork.md#保証範囲とライセンス) を参照。
