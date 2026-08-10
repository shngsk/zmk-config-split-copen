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
  - ハード面で切り分けが難しい場合の追加手段（未実施・要検討）:
    - L側を一時的に `CONFIG_ZMK_USB=y` + `CONFIG_ZMK_USB_LOGGING=y` 等でUSBシリアル
      ログを有効化したデバッグビルドを作り、USB給電しながらキー入力時に
      row/col イベントがログに出るか確認する（BLEペリフェラル接続の成否と切り離して
      kscan単体の生死を確認できる）。Kconfigオプション名は要検証（ローカルにZMKソース
      が無く未確認のため、試す場合は個別ブランチでCI結果を見ながら調整すること）。

## 完了

- [x] **USB接続時、Layer 3 の Y キー（BT_SEL 0）を押すと `y` が入力されてしまう**
  - 2026-08-10: 症状改善を確認。完了扱いとする。
  - 右手(copen2_R)のBLE再接続不可の問題を `settings_reset.uf2` 書き込み＋
    PC側の古いペアリング削除で解消した際に、あわせて改善したと思われる
    （詳細な原因切り分けは未実施）。再発したら詳細調査を再開する。
