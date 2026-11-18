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

## 文書

- [akiyan fork の変更情報](blastem_akiyan_fork.md)
- [導入と MCP ホストの設定](docs/setup.md)
- [制御プロトコル v2](docs/control-protocol-v2.md)
- 機能: [SRAM](docs/features/sram.md)、[停止中の画面取得](docs/features/paused-screenshot.md)、[フレーム制御](docs/features/frame-control.md)、[音声録音](docs/features/audio-capture.md)、[速度と RAM](docs/features/speed-and-ram.md)、[ヘッドレス録画](docs/features/offline-recording.md)、[フレームごとの値の記録](docs/features/frame-trace.md)、[VDP 表示内容のテキスト化](docs/features/vdp-layout.md)
- [試験用 fixture ROM](docs/fixture.md)
- [上流への追従](docs/upstream-sync.md)
- [統合時の検証記録](releases/2026-11-18.md)、[上流追従の記録](releases/2026-11-18-upstream-sync.md)
- [統合前の設計記録](docs/history/README.md)

## 公開する情報

市販 ROM、BIOS、ROM 由来の画像・音声・セーブはこのリポジトリと本体に置かない。試験には、ソースからビルドする自作の fixture ROM だけを使う。検証で生成した画像・録音・ログは各自の作業ディレクトリ（`.local/` など）に置く。

文書は MIT ライセンス。本体のライセンスは [変更情報](blastem_akiyan_fork.md#保証範囲とライセンス) を参照。
