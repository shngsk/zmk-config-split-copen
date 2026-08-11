# zmk-config-split-copen — Project Notes

## 概要

copen2 キーボード（セルフビルドの左右分割 40% キーボード）の ZMK ファームウェア設定。
GitHub Actions でビルドし、`.uf2` をダウンロードしてフラッシュする運用。

## ビルド環境

- **ボード**: `xiao_ble//zmk`（Seeeduino XIAO BLE、`zmk` ボード修飾子付き）
  - ⚠️ `seeeduino_xiao_ble` は ZMK v0.3 では無効。`xiao_ble` を使うこと。
  - `xiao_ble//zmk` 修飾子自体はビルド可能（NVS/設定パーティション定義を持つ）。
    ZMK公式Discordの助言により採用（→ 下記「NVS/BLE永続化」参照）。
  - ⚠️ `build.yaml` で `xiao_ble//zmk` を使う場合、各エントリに `artifact-name:` を
    明示指定すること。ZMK reusable workflow `@v0.3` の `build-user-config.yml` には
    board名の `/` をそのままファイルパスに使ってしまうバグがあり（`main`ブランチでは
    `${board//\//_}` で修正済みだが、v0.3時点では未リリース）、`artifact-name` を
    指定しないと成果物コピー時に `cp: cannot create regular file` で CI が失敗する。
- **ビルド**: `.github/workflows/build.yml` → ZMK reusable workflow `@v0.3`
- **設定**: `build.yaml`（copen2_R, copen2_L, settings_reset）

## シールドファイル

```
config/boards/shields/copen2/
  copen2.dtsi          # 共通ハードウェア定義（matrix transform, kscan, encoder）
  copen2.keymap        # キーマップ（全レイヤー）
  Kconfig.defconfig    # ZMK_SPLIT, ZMK_POINTING 等
  Kconfig.shield       # SHIELD_COPEN2_L / SHIELD_COPEN2_R
  copen2_L.overlay     # 左側: kscan col-gpios, エンコーダー有効化
  copen2_L.conf        # 左側: BLE, EC11
  copen2_R.overlay     # 右側: SPI (PMW3610トラックボール), kscan
  copen2_R.conf        # 右側: BLE, ZMK Studio, GLOBE key
```

## PMW3610 トラックボール（重要）

**Zephyr 4.1 ネイティブドライバーを使用**（shngsk の ZMK コミュニティドライバーは使わない）。

`copen2_R.overlay` の trackball ノード:
```dts
compatible = "pixart,pmw3610";
motion-gpios = <...>;   // irq-gpios ではない（Zephyr 4.1 で改名）
zephyr,axis-x = <INPUT_REL_X>;
zephyr,axis-y = <INPUT_REL_Y>;
// zephyr,cpi は現在の Zephyr binding に存在しないため不使用
```

`copen2_R.conf` に `CONFIG_PMW3610_*` は書かない（shngsk ドライバー固有のため）。
`config/west.yml` に `zmk-pmw3610-driver` は残してあるが、Kconfig では有効化していない。

## NVS / BLE ボンド永続化（重要）

### 問題
`xiao_ble` を `build.yaml` で単体指定した場合、`xiao_ble_zmk_defconfig` が読み込まれず NVS が無効になる。
BLE のボンド情報（LTK）がフラッシュに書き込まれないため、**電源オフ→オンで BLE 再接続失敗**する。

### 対処方法（採用済み・2026-08〜）
`build.yaml` で `xiao_ble//zmk` を指定する（`zmk` ボード修飾子）。これにより
`xiao_ble_zmk_defconfig`相当のNVS/設定パーティション定義が自動的に適用される。
以前は「`xiao_ble//zmk` はCIビルドエラーになる」として `xiao_ble` 単体+手動conf
コピーの方式を採っていたが、ZMK公式Discordの助言と実際の検証の結果、
`xiao_ble//zmk` 自体は正常にビルドできることを確認したため方針転換した。

`xiao_ble//zmk` は `CONFIG_HW_STACK_PROTECTION=y` を有効化するが、これが
`CONFIG_RUNTIME_ERROR_CHECKS=y` 経由でCIビルドエラーを招く問題は、
`copen2_R.conf` / `copen2_L.conf` に以下を明示することで回避済み:
```conf
CONFIG_HW_STACK_PROTECTION=n
```

### 動作確認済み（2026-08-10）
`xiao_ble//zmk` 切り替え後、右手(copen2_R)でBLE接続不可になる事象が発生したが、
下記「BLE接続失敗時のリセット手順」（`settings_reset.uf2`書き込み＋PC側の古いペアリング削除）
で解消し、**電源オフ→オンでの自動再接続も実機で確認できた**。
これは新しいパーティションレイアウトと整合しない古いボンド情報が残っていたためと推測される。
今後 `xiao_ble//zmk` 関連でパーティション定義が変わるような変更（west.yml更新等）を行った際は、
念のため両手に `settings_reset.uf2` を書き込んでからペアリングし直すこと。

### build.yaml の `artifact-name` が必須な理由
ZMK reusable workflow `@v0.3`（`build-user-config.yml`）には、board名に含まれる
`/` をそのままアーティファクトのファイルパスに使ってしまうバグがある
（`main`ブランチでは `${board//\//_}` として修正済みだが、v0.3時点では未リリース。
2026-08時点でのリリースタグはv0.1〜v0.3のみ）。
`board: xiao_ble//zmk` をそのまま使うと成果物コピー時に
`cp: cannot create regular file '...xiao_ble//zmk-zmk.uf2': No such file or directory`
でCIが失敗するため、`build.yaml` の各エントリに `artifact-name:` を明示指定してこれを回避する:
```yaml
include:
  - board: xiao_ble//zmk
    shield: copen2_R
    snippet: studio-rpc-usb-uart
    artifact-name: copen2_R-xiao_ble-zmk
  - board: xiao_ble//zmk
    shield: copen2_L
    artifact-name: copen2_L-xiao_ble-zmk
  - board: xiao_ble//zmk
    shield: settings_reset
    artifact-name: settings_reset-xiao_ble-zmk
```

### BLE 接続失敗時のリセット手順
1. `settings_reset.uf2` を両側にフラッシュ（ボンド情報クリア）
2. `copen2_R.uf2` → `copen2_L.uf2` の順でフラッシュ
3. PC の BT 設定から copen2 を削除 → 新規ペアリング
4. 電源オフ→オンで自動再接続を確認

---

## west.yml

```yaml
projects:
  - name: zmk
    remote: zmkfirmware
    revision: main      # ZMK main ブランチ（Zephyr 4.1 ベース）
  - name: zmk-pmw3610-driver
    remote: shngsk
    revision: main      # DTS binding 提供のために残存（ドライバー自体は無効）
```

## キーマップ レイヤー構成

| レイヤー | 内容 | アクセス方法 |
|---------|------|-------------|
| 0 (default) | QWERTY + マウスボタン → コンマ・ドット | — |
| 1 | 数字・記号・ファンクション | SPACE ホールド / ENTER ホールド |
| 2 | 矢印・ナビゲーション・ZMK Studio unlock | MINUS ホールド / TAB ホールド |
| 3 | Bluetooth 切り替え | P ホールド / mo 3 キー（右端）|

### Layer 3 (BT) キー配置
- Y=BT_SEL 0, U=BT_SEL 1, I=BT_SEL 2, O=BT_SEL 3, P=BT_CLR
- H=studio_unlock

### 右親指エリア（Layer 0）
- `lt 1 ENTER`（ホールド=Layer1、タップ=Enter）
- `mt RIGHT_WIN LANG1`（ホールド=RIGHT_WIN、タップ=LANG1）
- `kp GLOBE`（macOS Globe/Fn キー）
- `mo 3`（Layer 3 モーメンタリ）

## PCB

`pcb/` 以下に KiCad プロジェクトを格納（ファームウェアとは独立）。

## 既知の問題（進行中の調査）

詳細・最新状況は `TODO.md` を参照。

- **左手デバイス（copen2_L）が無反応**: ハードウェア・devicetree・BLEボンドは
  いずれも正常と確認済み（`copen2_L_standalone` 診断ビルドで実機検証）。
  原因は split peripheral(L) ↔ central(R) 間のBLE接続ロジックに絞り込み中。
  診断用ビルド（`copen2_l_dbg`, `copen2_l_standalone`）が `build.yaml` に
  追加してある（通常運用には使わないこと、検証後は `copen2_L.uf2` に戻す）。
