# TODO

## 未解決の問題

- [ ] **左手デバイス（copen2_L）のキースイッチを押しても何も入力されない**

  ### 現在の状況（2026-08-12 更新）

  **確定事項:**
  - ✅ L 基板のハードウェア（配線・はんだ・ダイオード）は正常
    → L 単体 USB キーボードとして全キー入力を実機確認済み
  - ✅ L の BLE 接続は成立している
    → シリアルログで以下を確認:
    - `split_svc_pos_state_ccc: value 1` — R が L の通知を購読
    - `security_changed` level 2 — R-L BLE 接続確立
    - `split_svc_select_phys_layout_callback` — R がレイアウトを書き込み
    - キー押下後: `split_peripheral_listener:` がエラーなし — GATT notify 成功
  - ✅ L の GATT notify はエラーなしで R に送信されている
  - ❌ **PC でキーが認識されない** — R 側の処理（GATT 受信→キーマップ→HID 送信）
    のどこかで失敗している。R のログ未確認のため原因不明。

  ### 次にやること

  **R 側の USB シリアルログを取得して原因を絞り込む**

  `build.yaml` に R 用の診断ビルドエントリを一時的に追加してCIでビルドする:
  ```yaml
  - board: xiao_ble//zmk
    shield: copen2_R copen2_r_dbg
    artifact-name: copen2_R_dbg-xiao_ble-zmk
  ```
  シールドファイルは `config/boards/shields/copen2_r_dbg/` に用意済み
  （`claude/left-hand-device-task-62r2xh` ブランチを参照）。

  手順:
  1. `copen2_R_dbg-xiao_ble-zmk.uf2` を R にフラッシュ
     （studio-rpc-usb-uart なしのビルドなので ZMK Studio は使えない）
  2. R を Mac に USB 接続
  3. シリアルログ監視: `while true; do cat /dev/cu.usbmodem* 2>/dev/null; sleep 0.1; done`
  4. L を電源 ON して BLE 接続させる
  5. L のキーを押す
  6. R のログで以下を確認:
     - `split_central` 系（L からの GATT 通知受信）
     - `zmk_keymap` 系（キーマップ処理）
     - HID レポート送信ログ

  **ログ解釈:**
  - R ログで何も出ない → R が GATT 通知を受け取れていない（BLE プロトコル問題）
  - キーマップ処理まで出るが HID ログがない → HID 送信の問題

## 完了

- [x] **USB接続時、Layer 3 の Y キー（BT_SEL 0）を押すと `y` が入力されてしまう**
  - 2026-08-10: 症状改善を確認。完了扱いとする。
  - 右手(copen2_R)のBLE再接続不可の問題を `settings_reset.uf2` 書き込み＋
    PC側の古いペアリング削除で解消した際に、あわせて改善したと思われる
    （詳細な原因切り分けは未実施）。再発したら詳細調査を再開する。
