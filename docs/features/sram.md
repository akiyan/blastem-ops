# SRAM 保存先と応答付き保存

本体は `BLASTEM_SAVE_DIR`、MCP は `blastem_start.save_dir` で、絶対パスのディレクトリを保存 profile として使う。ROM 名や ROM hash が変わっても同じ `save.sram` をロードするため、ROM を再ビルドしながらゲーム側のセーブで途中再開できる。

## 本体

- `BLASTEM_SAVE_DIR` が未指定・空なら従来の保存先規則（`ui.save_path`、既定 `$USERDATA/blastem/$ROMNAME`）を使う。指定値が絶対パスでない、または作成できない場合は起動を中止する
- v2 の `save_flush` は空の params を受け、次の処理をすべて終えてから `bytes`、`frame`、`paused` を返す
  - 保存先の一時ファイルへ書き込み、fsync する
  - rename で置換し、親ディレクトリを fsync する
- エミュレーター上のバッファは byte swap で書き換えない
- 書込み・置換・同期の失敗は `io_error`。ディレクトリ同期の失敗は置換後にも起こり得るため、その場合も成功とは扱わない。処理できた失敗では一時ファイルを消す。プロセス・電源断では未使用の一時ファイルが残り得る
- GDB 停止中も、命令を実行せずに制御ソケットの要求を処理する。その時の `paused: true` は命令境界で、フレーム境界とは限らない
- capability: `save_directory`、`sram_flush`

## MCP

- `blastem_start` に絶対パスの `save_dir` を渡すと、起動前に capability を確認する
- profile を OS のファイルロックで排他する。symlink 経由の二重起動も同じロックになり、MCP が異常終了しても OS がロックを解放する。MCP を介さず直接起動した本体はこのロックを使わない
- セッションの一時ディレクトリを削除しても profile は消さない
- `save_flush` ツールで保存を確定する。profile 付きセッションの `blastem_stop` は先に flush し、保存失敗を隠さずに返す。失敗してもプロセスを停止し、ロックを解放する
- profile を省略したセッションは従来の起動方式を保つ

## 対象外

POSIX 上の通常 MD カートリッジ SRAM が対象。EEPROM、flash、CD BRAM、32X、Windows での flush は `unsupported` を返す。

## 試験

- fixture の 32 KiB SRAM へ、MCP の入力・GDB 操作から書き込み、flush 後の保存画像を確認する
- ROM A で保存したバイトを、別名・別 hash の ROM B で読み戻す
- 2つの profile が混ざらないこと、二重起動の拒否、終了後のロック解放、保存失敗を返して停止することを確認する
- 不正 JSON、重複 key、小数、壊れた UTF-8、未対応 version の拒否は `tests/test_control_v2.py` で確認する
