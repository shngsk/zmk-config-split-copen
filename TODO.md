# TODO

## 未解決の問題

- [ ] **左手デバイス（copen2_L）のキースイッチを押しても何も入力されない**
  - ユーザー所感: **ハードウェア（セルフビルド）側の問題の可能性が高い**。
    ソフト（kscan / matrix transform / split接続）を疑う前に、まずハード面を確認する。
  - **2026-08-10: ソフトウェア側を精査済み、明確な不具合なし**
    （正常動作している copen2_R と突き合わせて確認）:
    - `copen2_L.overlay` col-gpios（列0-5, xiao_d 7/8/9/10 + gpio0 10/9）と
      `copen2_R.overlay` col-gpios（列6-10, `col-offset=6`）は列数・オフセットとも整合。
    - 共通 `copen2.dtsi` の row-gpios（PULL_DOWN + ACTIVE_HIGH）/
      `diode-direction = "col2row"` は col2row 配線のセオリー通り、L/R共通で問題なし。
    - `Kconfig.shield` のL側ブロック（`ZMK_SPLIT=y`、central指定なし＝peripheral）は
      split構成として正しい。
    - `left_encoder` 有効化、`zmk,physical-layout` 経由の kscan/matrix-transform 解決も
      dtsi 通りで問題なし。
    - → コード上に「左手だけ無反応」を説明できる欠陥は見当たらず、ハードウェア説が濃厚。
  - ハード側の確認ポイント（セルフビルドなので配線・はんだ不良が起きやすい）:
    - row/col の配線・はんだ付け（ブリッジ、イモはんだ、断線）
    - ダイオードの向き・破損
    - XIAO BLE 本体のはんだ付け（GPIOピンの浮き）
    - テスターで導通チェック（キー押下時に該当row-col間が導通するか）
  - **2026-08-10: USBシリアルログでのハード切り分け用ビルドを追加**
    （`build.yaml` に `copen2_L_dbg` として追加、CI自動ビルド）。
    - shield `copen2_l_dbg`（`config/boards/shields/copen2_l_dbg/`）を通常の
      `copen2_L` と組み合わせてビルド。devicetreeは変更せず、Kconfigのみ
      `CONFIG_ZMK_USB=y` + `CONFIG_ZMK_USB_LOGGING=y` + `CONFIG_LOG_DEFAULT_LEVEL=4`
      を追加し、USB CDC ACM経由でログコンソールを有効化。BLE/central接続は不要。
    - **使い方**:
      1. GitHub Actions の Artifacts から `copen2_L_dbg-xiao_ble-zmk` (uf2) を
         ダウンロードし、L側にフラッシュ（通常のフラッシュ手順と同じ）。
      2. データ通信対応のUSB-CケーブルでPCとL側を接続。
      3. シリアルターミナルで接続（Mac: `screen /dev/tty.usbmodemXXXX 115200`、
         Windows: PuTTY/TeraTermでUSB CDC ACMのCOMポートを開く）。
      4. 起動ログが出るか確認 → 出ればUSB/書き込み自体はOK。
      5. キースイッチを1個ずつ押し、押すたびにログへ反応（row/col関連イベント）が
         出るか確認。
         - 反応が出る → kscanハードは生きている。原因はBLE/central接続側に絞れる。
         - 起動ログは出るがキー押下に無反応 → 配線/ダイオード/はんだ不良の可能性大。
         - 起動ログすら出ない → USB接続/書き込み自体の問題を疑う。
      6. 診断が終わったら通常運用の `copen2_L.uf2` に書き戻すこと（このビルドは
         BLEに繋がらないため通常使用不可）。
    - ⚠️ Kconfigオプション名（`CONFIG_ZMK_USB_LOGGING` 等）はローカルにZMKソースが
      無く未検証。初回CIビルドが失敗する場合は、CIログのKconfig警告/エラーを見て
      `copen2_l_dbg.conf` を調整する必要あり。

## 完了

- [x] **USB接続時、Layer 3 の Y キー（BT_SEL 0）を押すと `y` が入力されてしまう**
  - 2026-08-10: 症状改善を確認。完了扱いとする。
  - 右手(copen2_R)のBLE再接続不可の問題を `settings_reset.uf2` 書き込み＋
    PC側の古いペアリング削除で解消した際に、あわせて改善したと思われる
    （詳細な原因切り分けは未実施）。再発したら詳細調査を再開する。
