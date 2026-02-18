---
type: technical_notes
project: zmk-corne-mini
date: 2026-02-19
author: op
tags:
  - bluetooth
  - corne-mini
  - roam
  - rpi5
---

# Corne Mini Bluetooth 接続ノート

## 概要

Corne Mini を roam（RPi5 16GB）に Bluetooth 接続した際の記録と調査結果。

---

## ハードウェア構成

| 項目 | 詳細 |
|------|------|
| キーボード | Corne Mini（nice!nano v2 x2） |
| ホスト | RPi5 16GB（roam） |
| RPi5 内蔵BT | BCM43455（CYW43455）BT 5.0 |
| BT アドレス（Corne） | `F3:A0:DC:21:3E:BF` (random) |
| BT 名 | `CorneCustom`（→ 次回ファーム書き込みで `Corne Mini` に変更予定） |
| TX Power | +8dBm（`CONFIG_BT_CTLR_TX_PWR_PLUS_8=y`） |

---

## 実際にフラッシュされているキーマップ

**重要**: `config/corne_custom.keymap`（リポジトリ上のファイル）は実機にフラッシュされているものと**異なる**。

実機には **Charybdis と同じ Callum 式 One-Shot キーマップ** がフラッシュされている。

### 判別に使用したテスト

| テスト | corne_custom.keymap の場合 | 実機の結果 | 判定 |
|--------|---------------------------|-----------|------|
| Z 長押し | Ctrl（`&mt LCTRL Z`） | Z リピート | Charybdis 式 |
| NAV + HJKL | なし（NAV に矢印なし） | 矢印キー | Charybdis 式 |
| NAV + SYM 同時 → F1 | 該当レイヤーなし | F1 検出（evtest） | tri-layer (FUN) |

### 実機のレイヤー構成（Charybdis 式）

```
Layer 0 (BASE): QWERTY + mo(NAV) / SPC / mo(SYM) / BSPC / RET
Layer 1 (NAV):  ESC, Swapper, Tabber, CAPS | HOME/PGDN/PGUP/END | Arrows | Undo/Cut/Copy/Paste/Redo
Layer 2 (SYM):  1-0 | =/\/`/;/' | [/]/{/}/|
Layer 3 (FUN):  F1-F12 | Mute/Vol | BT_SEL 0-2 / BT_CLR / OUT_TOG（NAV+SYM 同時で発動）
```

### BT キーの位置（FUN レイヤー・右手下段）

```
N → BT_SEL 0
M → BT_SEL 1
, → BT_SEL 2
. → BT_CLR
/ → OUT_TOG
```

### 参照キーマップ

- `zmk-for-charybdis/config/charybdis.keymap` が実機と同等の設計
- `zmk-corne-mini/config/corne_custom.keymap` はリポジトリ上のみ（未フラッシュ）

---

## BT ペアリング手順

### 成功した手順

1. Corne 側: FUN レイヤーで `BT_CLR`（NAV + SYM 同時押し → `.` キー）
2. Corne 側: nice!nano リセットボタン 2 度押し（ペアリングモードへ）
3. Corne に USB 給電（バッテリー不良のため外部電源必須）
4. roam 側: `expect` を使用して bluetoothctl を操作

```bash
expect -c '
spawn bluetoothctl
expect "#"
send "agent on\r"
expect "Agent registered"
send "default-agent\r"
expect "#"
send "scan on\r"
expect "CorneCustom"
send "pair F3:A0:DC:21:3E:BF\r"
expect "Pairing successful"
send "trust F3:A0:DC:21:3E:BF\r"
expect "#"
send "quit\r"
'
```

> **Note**: `bluetoothctl` をパイプや heredoc で操作すると agent 登録前に pair が実行されて `AuthenticationFailed` になる。`expect` が必須。

### ペアリング状態

```
Paired: yes
Bonded: yes
Trusted: yes
WakeAllowed: yes
```

---

## 判明した問題

### 1. RPi5 内蔵 BT (BCM43455) の BLE HID 不安定

**症状**: 接続が数分〜数十分で切断される。再接続を繰り返す。

**dmesg エラー**:
```
Bluetooth: hci0: unexpected cc 0x041a length: 7 > 1
```

**journalctl**: input デバイスが input8 → input9 → input10 と再登録される。

**原因**: RPi5 の CYW43455 チップは BT 5.0 の必須機能のみ実装。BLE HID デバイスとの長時間安定接続に問題がある（既知の問題）。

**対処**: USB Bluetooth ドングルを使用する（内蔵 BT を無効化してドングル経由で接続）。

### 2. Corne バッテリー不良

**症状**: USB 給電を外すと BT 接続が切れる。

**推定原因**: nice!nano v2 のバッテリーが死んでいる、または未接続。

**対処**: 常時 USB 給電が必要（roam の USB ポートではなく、別の電源から給電）。

### 3. corne_custom.keymap とフラッシュ済みファームウェアの不一致

**詳細**: リポジトリ上の `corne_custom.keymap` は hold-tap ベース + 4 レイヤー構成だが、実機には Charybdis 式 OSM + conditional layer (tri-layer) がフラッシュされている。

**対処**: 次回ファーム書き込み時にキーマップファイルを Charybdis 式に更新すること。

---

## USB BT ドングル（購入済み・到着待ち）

| 項目 | 詳細 |
|------|------|
| 製品 | TP-Link UB500 (UNVER) |
| BT バージョン | 5.4（アップデート後） |
| チップ | RTL8761B |
| roam kernel | 6.12.62+rpt-rpi-2712（ドライバー内蔵） |

### セットアップ予定手順

```bash
# 1. 認識確認
lsusb | grep -i bluetooth
bluetoothctl list

# 2. ドングルをデフォルトに設定
bluetoothctl select <hci1のアドレス>

# 3. 内蔵 BT を無効化
sudo hciconfig hci0 down

# 4. 既存ペアリング削除 & 再ペアリング
bluetoothctl remove F3:A0:DC:21:3E:BF
bluetoothctl scan on
bluetoothctl pair F3:A0:DC:21:3E:BF
bluetoothctl trust F3:A0:DC:21:3E:BF
bluetoothctl connect F3:A0:DC:21:3E:BF

# 5. 内蔵 BT の永続無効化
# /boot/firmware/config.txt に追記:
# dtoverlay=disable-bt
```

---

## 次回タスク

- [ ] TP-Link UB500 到着後、ドングル経由で BT 再ペアリング
- [ ] 内蔵 BT の永続無効化（`dtoverlay=disable-bt`）
- [ ] バッテリー状態の確認（nice!nano v2 のバッテリーコネクタ確認）
- [ ] `corne_custom.keymap` を Charybdis 式に更新（次回ファーム書き込み時）
- [ ] BT 名を `CorneCustom` → `Corne Mini` に変更（次回ファーム書き込み時）
