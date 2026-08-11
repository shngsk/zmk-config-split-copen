# TODO

## 未解決の問題

- [ ] **左手デバイス（copen2_L）のキースイッチを押しても何も入力されない**

  **作業ブランチ**: `claude/left-hand-device-task-62r2xh`

  ### 現在の状況（2026-08-11 更新）

  **確定事項:**
  - ✅ L 基板のハードウェア（配線・はんだ・ダイオード）は正常
    → `copen2_L_standalone` で全キー入力を実機確認済み
  - ✅ L の BLE 接続は成立している
    → `copen2_l_dbg`（LOG_DEFAULT_LEVEL=3）のシリアルログで確認:
    - `bt_hci_core: Identity: CC:12:78:64:3E:5A (random)` — BLE HW 正常
    - `Failed to start advertising (-120)` — -EALREADY（既に広告中）、無害
    - t=2.9s: `split_svc_pos_state_ccc: value 1` — R が L の通知を購読
    - t=2.9s: `security_changed: C5:62:0B:D8:DD:0B (random) level 2` — R-L BLE 接続確立
    - t=3.1s: `split_svc_select_phys_layout_callback` — R がレイアウトを書き込み
    - キー押下後: `split_peripheral_listener:` がエラーなし — GATT notify 成功
  - ✅ L の GATT notify はエラーなしで R に送信されている
    → 接続前は `-128 (ENOTCONN)`、接続後はエラーなし
  - ❌ **PC でキーが認識されない** — R 側の処理（GATT 受信→キーマップ→HID 送信）
    のどこかで失敗している。R のログを見たことがないため原因不明。

  ### 次にやること（未実施）

  **`copen2_R_dbg-xiao_ble-zmk.uf2` を R にフラッシュしてログを確認する**

  ブランチ `claude/left-hand-device-task-62r2xh` の最新 CI（コミット `146abd3`）で
  `copen2_R_dbg` ビルドが完成している。

  手順:
  1. `copen2_R_dbg-xiao_ble-zmk.uf2` を R にフラッシュ
     （studio-rpc-usb-uart なしのビルドなので ZMK Studio は使えない）
  2. R を Mac に USB 接続
  3. シリアルログ監視: `while true; do cat /dev/cu.usbmodem* 2>/dev/null; sleep 0.1; done`
  4. L（`copen2_L_dbg` または通常 `copen2_L`）を電源 ON して BLE 接続させる
  5. L のキーを押す
  6. R のログで以下を確認:
     - `split_central` 系のログ（L からの GATT 通知受信）
     - `zmk_keymap` 系のログ（キーマップ処理）
     - HID レポート送信ログ

  R ログで何も出ない場合 → R が GATT 通知を受け取れていない（BLE プロトコル問題）
  キーマップ処理まで出るが HID ログがない場合 → HID 送信の問題

  ### 診断用ビルド一覧（`build.yaml` に追加済み）

  | アーティファクト名 | 用途 | 状態 |
  |---|---|---|
  | `copen2_L_dbg-xiao_ble-zmk` | L の USB シリアルログ確認 | ✅ 使用済み・確認完了 |
  | `copen2_L_standalone-xiao_ble-zmk` | L 単体 USB キーボード動作確認 | ✅ 使用済み・正常確認 |
  | `copen2_R_dbg-xiao_ble-zmk` | R の USB シリアルログ確認 | **← 次に使う** |

  ※ `copen2_L_dbg` の設定: `CONFIG_ZMK_USB_LOGGING=y`, `CONFIG_LOG_DEFAULT_LEVEL=3`
  ※ `copen2_R_dbg` の設定: `CONFIG_ZMK_USB_LOGGING=y`, `CONFIG_LOG_DEFAULT_LEVEL=3`
  ※ R_dbg は studio-rpc-usb-uart スニペットなし（CDC ACM と競合するため）

  ### ハード側の確認（保留中・優先度低）
  現時点ではハード起因の可能性は低い（L 単体テストで全キー正常）。

## 完了

- [x] **USB接続時、Layer 3 の Y キー（BT_SEL 0）を押すと `y` が入力されてしまう**
  - 2026-08-10: 症状改善を確認。完了扱いとする。
  - 右手(copen2_R)のBLE再接続不可の問題を `settings_reset.uf2` 書き込み＋
    PC側の古いペアリング削除で解消した際に、あわせて改善したと思われる
    （詳細な原因切り分けは未実施）。再発したら詳細調査を再開する。
