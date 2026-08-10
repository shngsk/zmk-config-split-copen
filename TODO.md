# TODO

## 未解決の問題

- [ ] **USB接続時、Layer 3 の Y キー（BT_SEL 0）を押すと `y` が入力されてしまう**
  - USB接続時のみ発生？ BLE接続時の挙動は未確認。
  - Layer 3 は P ホールドまたは右端の `mo 3` でアクセスするレイヤー。
    Layer 3 の Y には `BT_SEL 0` が割り当てられているはず（`copen2.keymap` 参照）が、
    実際には default layer (Layer 0) の `y` が入力される＝レイヤー遷移が効いていない
    か、USB接続時に何らかの理由でレイヤーが正しく評価されていない可能性。

- [ ] **左手デバイス（copen2_L）のキースイッチを押しても何も入力されない**
  - kscan / matrix transform / split（セントラル・ペリフェラル）接続のどこかに問題がある可能性。
  - `copen2_L.overlay` の kscan col-gpios 設定、または split BLE 接続自体（ペリフェラル側が
    セントラルに認識されているか）を確認する必要あり。
