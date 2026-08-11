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
  - **2026-08-10: USBシリアルログでのハード切り分け用ビルドを追加・CI検証済み**
    （`build.yaml` に `copen2_L_dbg` として追加、CI自動ビルド成功）。
    - shield `copen2_l_dbg`（`config/boards/shields/copen2_l_dbg/`）を通常の
      `copen2_L` と組み合わせてビルド。devicetreeは変更せず、Kconfigのみ
      `CONFIG_ZMK_USB_LOGGING=y` + `CONFIG_LOG_DEFAULT_LEVEL=4` を追加し、
      USB CDC ACM経由でログコンソールを有効化。BLE/central接続は不要。
    - **CIビルドログで確認済みの注意点**: `CONFIG_ZMK_USB=y` はZMK本体のKconfig
      制約（`depends on !ZMK_SPLIT || (ZMK_SPLIT && ZMK_SPLIT_ROLE_CENTRAL)`）
      によりperipheralでは`n`に強制される（設定しても警告が出るだけで無効）ため、
      この診断ビルドでもL側はUSBキーボードとしては機能しない。ただし
      `CONFIG_ZMK_USB_LOGGING` は `CONFIG_USB_CDC_ACM` / `CONFIG_USB_DEVICE_STACK` /
      `CONFIG_LOG_BACKEND_UART` を独立して有効化するため、シリアルログ出力自体は
      正常に機能する（CIの`.config`ダンプで実際に有効化されていることを確認済み）。
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
         BLEにもUSBキーボードにもならないため通常使用不可）。
  - **2026-08-10: 実機で診断ビルドを実行し、ハードウェア問題を強く確証**
    - 実施内容: `copen2_L_dbg` を実機に書き込み、USBシリアルコンソール
      （`/dev/tty.usbmodem*`）に接続。`XIAO-SENSE`（ブートローダー/UF2ドライブ）
      ではなく通常のアプリとして起動していることを確認した上で、`cat` でログを
      監視しながらL側の全キーを押下。
    - 結果: **ファームウェアは正常に起動・USBシリアルも正常に開通しているにも
      関わらず、キー押下でログに一切反応なし**。
    - 意義: この手順は**BLE/central接続を完全に迂回**しているため、「central側との
      ペアリング失敗」の可能性を排除できる。ソフトウェア（起動・ロギング機構）は
      生きていることも確認済みなので、消去法で**物理的な配線・はんだ付け・
      ダイオードの問題である可能性がほぼ確定**した。
    - （途中経過のトラブルシューティングメモ: `screen`の`$TERM too long`エラーは
      無関係な既知の問題→`cat`か`TERM=xterm screen`で回避。デバイスが
      `XIAO-SENSE`として見える場合はブートローダーモードなので、Finderで
      アンマウント→リセットボタンに触れずに挿し直せば通常起動する。）
  - **2026-08-10: 「ハードウェア確定」の結論が実験で覆り、ファームウェア調査に方針転換**
    - ユーザーが `copen2_R.uf2`（右手用ファームウェア）を**物理的な左手基板**に
      書き込んだところ、**いくつかのキーが反応した**。これは配線が全滅している
      という説と矛盾する→ハードウェアは生きている可能性が高い。
    - `zmkfirmware/zmk` 本体のソースを直接確認（`/workspace/zmkfirmware/zmk` に
      クローン済み）:
      - `app/src/physical_layouts.c` の `zmk_physical_layout_kscan_callback` で
        キー押下は `LOG_DBG("Row: %d, col: %d, position: %d, pressed: %s", ...)`
        としてログされるはず（`LOG_MODULE_DECLARE(zmk, CONFIG_ZMK_LOG_LEVEL)`）。
      - `CONFIG_ZMK_LOG_LEVEL` はデフォルトで4(DEBUG)（`ZMK_LOGGING_MINIMAL=n`の場合）
        であり、前回の`copen2_l_dbg`ビルドのCIログでも実際に`CONFIG_ZMK_LOG_LEVEL=4`
        になっていたことを確認済み → **ログレベル設定ミスが原因ではなかった**
        （一時的に疑ったが否定された）。
      - `zmk_physical_layout_kscan_callback` には
        `if (dev != active->kscan) return;`という無警告returnがあり理論上は
        怪しいが、L/R共通コードなので単体では「Lだけ無反応」を説明しにくい。
    - → **左手自身のkscanログ(peripheral構成)は無反応なのに、右手ファームウェア
      (central構成)を同じ基板に書くと一部反応する**という矛盾が残っている。
      ハードウェアの断線・はんだ不良を疑うフェーズは一旦保留し、
      firmware/Kconfig側の切り分けを優先する方針に変更。
  - **2026-08-10: 追加の診断ビルド `copen2_L_standalone` を用意（CI: 要確認）**
    - shield `copen2_l_standalone`（`config/boards/shields/copen2_l_standalone/`）を
      通常の `copen2_L` と組み合わせてビルド。**devicetree(L実機の実配線)は一切
      変更せず**、Kconfigのみ `CONFIG_ZMK_SPLIT_ROLE_CENTRAL=y` + `CONFIG_ZMK_USB=y`
      を追加し、L基板をBLEペリフェラルではなく**スタンドアロンのUSBキーボード**
      として動作させる。
    - 目的: L実機の配線・overlay自体が単体で正しくキー入力を出せるかを、
      BLE/ペリフェラル接続を完全に排除した状態で検証する
      （ユーザーが手動で試した「Rファームウェアを左基板に書く」実験を、
      L自身の正しい配線定義を使う形でより正確に再現）。
    - **使い方**: `copen2_L_standalone-xiao_ble-zmk` (uf2) をL側にフラッシュ →
      USBでPCに接続 → テキストエディタを開いてL側の各キーを押す →
      文字が入力されるか確認（シリアルログ確認より簡単・確実）。
      対応表は本ドキュメント作成時のチャットlog参照（例: Tキー = row0,col4）。
      診断後は通常運用の `copen2_L.uf2` に書き戻すこと。
  - **次のアクション**: 上記 `copen2_L_standalone` の実機テスト結果待ち。
    - 正しく（または一部でも）入力できれば → L実機の配線・overlay自体は生きている
      ことが確定し、原因は peripheral role / BLE 側のロジックに完全に絞り込める。
    - 何も入力されなければ → overlay/devicetree側（col-gpios定義等）に何らかの
      不具合がある可能性が浮上する。
    - （保留中）マルチメーターでの導通チェックは、上記の結果を見てから
      必要に応じて再開する。

## 完了

- [x] **USB接続時、Layer 3 の Y キー（BT_SEL 0）を押すと `y` が入力されてしまう**
  - 2026-08-10: 症状改善を確認。完了扱いとする。
  - 右手(copen2_R)のBLE再接続不可の問題を `settings_reset.uf2` 書き込み＋
    PC側の古いペアリング削除で解消した際に、あわせて改善したと思われる
    （詳細な原因切り分けは未実施）。再発したら詳細調査を再開する。
