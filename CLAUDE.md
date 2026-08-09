# zmk-config-split-copen — Project Notes

## 概要

copen2 キーボード（セルフビルドの左右分割 40% キーボード）の ZMK ファームウェア設定。
GitHub Actions でビルドし、`.uf2` をダウンロードしてフラッシュする運用。

## ビルド環境

- **ボード**: `xiao_ble`（Seeeduino XIAO BLE）
  - ⚠️ `seeeduino_xiao_ble` は ZMK v0.3 では無効。`xiao_ble` を使うこと。
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
