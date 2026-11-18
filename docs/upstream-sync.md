# 上流への追従

本体は [ulalume/blastem](https://github.com/ulalume/blastem) の main の上に、akiyan 独自の差分を1コミットだけ載せる。コミットメッセージは[変更情報ページ](../blastem_akiyan_fork.md)の URL のみとし、説明と運用文書はこのリポジトリで管理する。ulalume fork 自身のコミットは祖先としてそのまま保持する。

- `main`: ulalume/blastem の追従用。独自の変更は載せない
- `akiyan`: 既定ブランチ。`main` の固定した commit の上に独自差分の1コミットを載せる

初回の基準は ulalume/blastem の `4b71bfec3b5919367c5843f23c22c76ac24aa38c`。現在の基準は `5702187402276e7f0aea2ad4e6784ffb187c8c10`（2026-11-18 追従）。追従前の独自コミットは `akiyan-<日付>-<旧基準の短縮SHA>` のタグで保持する。

## 更新手順

1. 本体の未コミット変更を確認し、更新と混ぜない
2. `origin` と `ulalume` を fetch し、現在の `origin/akiyan` の SHA を控える
3. 稼働中の akiyan コミットにタグを付けて保持する
4. 新しい ulalume/blastem の commit から作業ブランチを作り、現在の独自コミットを cherry-pick する
5. 競合を解消し、`tools/akiyan-check.sh` を通す。上流の `build.yml` に影響する変更があれば、その変更も確認する
6. [変更情報](../blastem_akiyan_fork.md) の基準 SHA・対象パス・挙動を更新し、ops を先に push する。本体のコミットを amend し、メッセージを次の URL のみにする。独自差分が1コミットであることを確認する

   ```text
   https://github.com/akiyan/blastem-ops/blob/main/blastem_akiyan_fork.md
   ```

7. `main` を新しい基準へ進め、`akiyan` は控えたリモート SHA を指定した force-with-lease で公開する。リモートが動いていたら中止して確認する
8. 基準 SHA、独自コミットの SHA、実行した検証、既知の制限を `releases/` に記録する

## 機能を追加する場合

同じ1コミットへ amend し、変更情報ページに節を追加する。タイトル固有のシナリオ、シンボル、アドレスの意味は本体と MCP に入れない。利用側のプロジェクトは `akiyan` の commit SHA を固定して使う。

## 文書の更新

独自差分を変えたら、このリポジトリの日本語の文書（[変更情報](../blastem_akiyan_fork.md)、`docs/features/`、`releases/`）と、本体の英語の文書（`docs/`、`mcp/AGENT_GUIDE.md`）をそれぞれ更新する。言語と置き場所の分担は [README](../README.md#文書の言語と置き場所) を参照。

## 課題の管理

不具合と機能の Issue は [akiyan/blastem](https://github.com/akiyan/blastem/issues) で管理する。
