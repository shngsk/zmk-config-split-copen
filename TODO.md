# TODO

## 未解決の問題

- [ ] **左手デバイス（copen2_L）のキースイッチを押しても何も入力されない**

  ### 現在の状況（2026-08-11時点）

  **確定事項:**
  - ✅ **L基板のハードウェア（配線・はんだ・ダイオード）は正常** —
    `copen2_L_standalone` 診断ビルド（下記）でL基板を単体USBキーボードとして
    動かしたところ、**全キーが正しく入力できることを実機で確認済み**。
    「セルフビルドなのでハード不良だろう」という当初の想定は否定された。
  - ✅ **L側の devicetree/overlay（`copen2_L.overlay` の col-gpios、共通
    `copen2.dtsi` の row-gpios・matrix-transform）も正常** — 上記テストで
    実証済み。
  - ✅ **BLEボンド情報の破損が原因ではない** — `settings_reset.uf2` を
    左右両方に書き込んでボンドを完全クリアし、再ペアリングしても症状変わらず。
  - → **原因は split peripheral(L) ↔ central(R) 間のBLE接続・通信ロジックに
    絞り込まれている**（ハード・devicetree・BLEボンドは全て正常と確認済み）。

  **未確認（次にやること）:**
  L(peripheral) が実際にR(central)へBLE接続を試みているか/成功しているかを
  ログで直接確認できていない。`zmkfirmware/zmk` 本体ソースを調査したところ、
  以下のファイルに接続状態を示すログ行がある（`app/src/ble.c`,
  `app/src/split/bluetooth/peripheral.c`）:
  `"Connected %s"` / `"Disconnected from %s (reason 0x%02x)"` /
  `"advertising from %d to %d"` / `"BLUETOOTH FAILED (%d)"` /
  `"Security changed/failed"` など。

  次のセッションでは、L・R **両方に同時に電源を入れた状態**で
  `copen2_L_dbg`（下記）をL側で動かし、USBシリアルログを見ながら
  上記キーワードが出るか確認する（キー入力ではなく、BLE接続確立そのものを見る）。

  ### 診断用ビルド（`build.yaml` に追加済み、CI検証済み、いずれも通常運用の
  `copen2_L.uf2` に書き戻すまで一時的に使うもの）

  | shield名 | 何をするか | 使い道 |
  |---|---|---|
  | `copen2_l_dbg`（`copen2_L`と組合せ） | devicetree不変、`CONFIG_ZMK_USB_LOGGING=y`+`CONFIG_LOG_DEFAULT_LEVEL=4`のみ追加。peripheral roleのまま、USB CDC ACMでログ確認可能に | BLE接続ログ・kscanログの確認（次のアクションで使用） |
  | `copen2_l_standalone`（`copen2_L`と組合せ） | devicetree不変、`CONFIG_ZMK_SPLIT_ROLE_CENTRAL=y`+`CONFIG_ZMK_USB=y`を追加。L実機をスタンドアロンUSBキーボード化 | L実機の配線が単体で動くか検証（**検証済み・正常だった**） |

  どちらもartifact名は `copen2_L_dbg-xiao_ble-zmk` / `copen2_L_standalone-xiao_ble-zmk`。

  ### 参考: ZMK本体ソースの参照方法
  `zmkfirmware/zmk` を `/workspace/zmkfirmware/zmk` にクローン済み（このセッション
  固有の一時パスなので、別セッションでは無ければ
  `git clone --depth 1 https://github.com/zmkfirmware/zmk` で再取得すること）。
  関連ファイル: `app/src/physical_layouts.c`（kscan→position解決、ログ行あり）、
  `app/src/ble.c` / `app/src/split/bluetooth/peripheral.c`（BLE接続状態ログ）、
  `app/src/split/peripheral.c`。

  ### ハード側の確認（保留中・優先度低）
  ソフト/BLE側の切り分けが先。もし今後もハードを疑うなら:
  テスターでrow-gpios（4本）×col-gpios（6本）の導通・短絡チェック
  （詳細手順はチャット履歴参照）。ただし上記の通り現時点ではハード起因の
  可能性は低いと考えられる。

## 完了

- [x] **USB接続時、Layer 3 の Y キー（BT_SEL 0）を押すと `y` が入力されてしまう**
  - 2026-08-10: 症状改善を確認。完了扱いとする。
  - 右手(copen2_R)のBLE再接続不可の問題を `settings_reset.uf2` 書き込み＋
    PC側の古いペアリング削除で解消した際に、あわせて改善したと思われる
    （詳細な原因切り分けは未実施）。再発したら詳細調査を再開する。
