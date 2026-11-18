# フレーム単位の停止・送り・中断

通常起動した Mega Drive を、ゲーム側のブレークポイントなしでフレーム単位に制御する。

## MCP ツール

- `frame_pause`: 次の VDP フレーム更新後に CPU を戻し、main loop で停止してから応答する
- `frame_advance {frames: N}`: 停止位置から 1〜600 フレーム進めて停止する
- `frame_interrupt`: 実行中の frame_advance に中断を要求する。受付応答の後、frame_advance の応答で最終的な停止状態を確認する
- `paused_screenshot`: 停止したまま、最後に完成した画面を確認する

## 本体

- capability は `frame_control`。コマンドは `pause`、`advance`、`interrupt`。`advance` の params は `{"frames":1}`、他は空オブジェクト
- 応答には `frame`、`advanced`、`paused`、`interrupted`、`boundary` を含める
- `frame` は VDP の 32bit カウンター。サイクル単位ではなく、VDP 更新後の CPU 同期点（`frame_sync`）で停止する。UI 操作による途中停止は `paused_loop` として区別する
- フレーム送りの途中に `status`・`interrupt` 以外の v2 要求が来たら `busy` を返す
- advance 中に接続が切れたら、次のフレーム同期点で停止する。タイムアウトで接続を閉じた場合も同じ

## MCP の通信

- 通常の操作は直列化する。interrupt だけは同じ v2 接続で応答 ID を分けて送り、応答の順序が要求と違っても ID に対応する呼出元へ返す
- 要求がキャンセルされたら、後続の新しい要求を巻き込まないよう、その要求専用の接続だけを閉じる
- 入力の保持・解放は MCP の既存の管理に従う

## 対象外

対象は単体 MD の通常 SDL 同期。GDB 起動時、CD・32X、threaded・external 同期では `unsupported` を返す。GDB の停止との使い分けは [停止の種類と GDB 実行の中断](stops.md) を参照。

## 試験

fixture で次を確認する: フレーム数 1・2・17、割り込み、busy、切断、キャンセル後の再操作、入力を保持したまま3フレーム進めたときの画面変化。
