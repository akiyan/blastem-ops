# 停止中の画面取得

MCP の `paused_screenshot` は、GDB またはエミュレーターの停止中に、最後に描画を完了したプログレッシブフレームを PNG で返す。撮影のために CPU やフレームを進めない。通常の `screenshot` は従来どおり、次に描画されるフレームを待つ。

## 本体

- capability は `cached_screenshot`。`hello` の後に `{"id":2,"version":2,"command":"snapshot","params":{"path":"/absolute/output.png"}}` を送る
- 成功応答には次を含める
  - `width`、`height`
  - `sequence`: プロセス内で画像キャッシュを更新した回数。ゲームのフレーム番号ではない
  - `frame`: 停止時の VDP frame
  - `image_boundary: last_completed_progressive_frame`、`paused`
- CPU は次フレームの途中で止まっていることがあるため、`frame` を画像が示す時刻と同一視しない
- 通常のスクリーンショットと同じ表示領域を使い、縮小や切り抜きはしない
- まだ描画がなければ `not_ready`、実行中は `invalid_state`、書込み失敗は `io_error`。相対パスと NUL を含むパスは拒否する

## 対象

通常の SDL audio/video 同期で、OpenGL と SDL fallback renderer を検証している。インターレース中はキャッシュを無効化する。audio_thread・external 同期、非 SDL 描画、zlib なしのビルドは対象外。

## 試験

fixture で2回撮影し、画像・sequence・frame が一致し、CPU レジスターが変わらないことを確認する。
