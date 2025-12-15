# Dr.Doc
# ASUS TransBook T90CHI Linux 完全構築ガイド

ASUS TransBook **T90CHI** に Linux（Lubuntu 20.04）を導入し、  
**純正 Bluetooth キーボード / タッチ / スタイラス / 電源管理**まで  
すべて実用レベルで動作させた手順をまとめたドキュメントです。

> ※ 本資料は「なぜ動かないか」「なぜこの手順で動くのか」まで含めて記載します。

---

## 対象ハードウェア

- ASUS TransBook **T90CHI**
- CPU: Intel Atom Z3735F (Bay Trail)
- UEFI: **32bit**
- Bluetooth: **Broadcom BCM43341B0**
- 入力:
  - 純正 Bluetooth キーボード（Legacy HID）
  - タッチパネル
  - スタイラスペン

---

## 動作環境

- OS: **Lubuntu 20.04.5 LTS**
- Kernel: 5.4.x
- BlueZ: 5.53
- Display Server: Xorg

---

## できること / できないこと

### ✅ 動作確認済み
- 32bit UEFI 起動
- 画面回転固定
- タッチ入力（指）
- **スタイラス入力（座標補正済み）**
- **純正 Bluetooth キーボード（Legacy HID）**
- Wi-Fi
- SSH
- スクリーンセーバー無効化（放置フリーズ回避）

### ❌ 非対応
- Wayland（未検証）
- BlueZ 5.x 以外での保証

---

## 最大の問題点（要点）

T90CHI の純正 Bluetooth キーボードは **Legacy HID** であり、  
さらに **BCM43341B0 の firmware に固定 MAC アドレスが埋め込まれている**ため、

- BlueZ 5.x では「不正な MAC」として扱われ
- HCI が `DOWN RAW` のままになり
- キーボードが接続できない

という問題があります。

---

## 解決方法（結論）

1. **BCM43341B0.hcd を使用**
2. firmware 内の **固定 MAC アドレスを 1byte 書き換え**
3. BlueZ を **KeyboardOnly agent** で動作させる
4. レガシー HID は **キー入力をトリガに接続**

これにより **Ubuntu 20.04 標準カーネルのまま**  
純正キーボードが使用可能になります。

---

## インストール概要

### 1. Lubuntu 20.04 インストール
- amd64 ISO を使用
- 32bit UEFI 用 `bootia32.efi` を配置
- Minimal install 推奨

### 2. 初期安定化
- lxqt-powermanagement 無効化
- DPMS / スクリーンセーバー完全停止
- 放置時フリーズ回避

---

## Bluetooth キーボード（最重要）

### firmware 配置
```bash
mkdir -p /lib/firmware/brcm
cp BCM43341B0.hcd /lib/firmware/brcm/
cp BCM43341B0.hcd /lib/firmware/brcm/BCM.hcd
chmod 644 /lib/firmware/brcm/*.hcd
```

##  firmware 内 MAC アドレス修正（例）
printf '\xAB' | dd of=/lib/firmware/brcm/BCM43341B0.hcd \
  bs=1 seek=$((0x007B)) count=1 conv=notrunc

## Bluetooth 接続手順（重要）
bluetoothctl
power on
agent KeyboardOnly
default-agent
pairable on
discoverable on
scan on

connect は使用しない
キーボードでキー入力すると自動接続される

##  タッチ・スタイラス補正
##  スタイラスのみ座標補正
xinput set-prop "SYNA7508:00 06CB:11DD Stylus Pen (0)" \
  "Coordinate Transformation Matrix" \
  0 1 0 -1 0 1 0 0 1

autostart で永続化。

##  参考リンク
nomux2.net
drvlabo.jp
5ch notepc スレ（T90CHI）

##注意
本手順は ASUS T90CHI 固有の内容です。
他機種への適用は自己責任でお願いします。




---

# English Version

# ASUS TransBook T90CHI – Complete Linux Setup Guide

This document describes how to run Linux on **ASUS TransBook T90CHI**
with full support for:

- Original ASUS Bluetooth keyboard (Legacy HID)
- Touch input
- Stylus pen (calibrated)
- Stable power management (no freeze on idle)

This guide focuses not only on *how* to make it work,
but also *why* it does not work by default.

---

## Hardware

- ASUS TransBook **T90CHI**
- CPU: Intel Atom Z3735F (Bay Trail)
- UEFI: **32-bit**
- Bluetooth chipset: **Broadcom BCM43341B0**
- Input devices:
  - Original ASUS Bluetooth keyboard (Legacy HID)
  - Touchscreen
  - Stylus pen

---

## Software Environment

- OS: **Lubuntu 20.04.5 LTS**
- Kernel: 5.4.x
- BlueZ: 5.53
- Display server: Xorg

---

## What Works

- Booting on 32-bit UEFI
- Screen rotation (fixed)
- Touch input (finger)
- Stylus input (calibrated)
- **Original ASUS Bluetooth keyboard**
- Wi-Fi
- SSH
- Power management without idle freeze

---

## The Core Problem

The original ASUS T90CHI keyboard uses **Legacy Bluetooth HID**.
Additionally, the Broadcom **BCM43341B0 firmware contains a fixed MAC address**.

On BlueZ 5.x, this causes:

- The MAC address to be treated as invalid
- Bluetooth HCI to stay in `DOWN RAW` state
- Keyboard connection to fail

---

## The Solution

1. Use `BCM43341B0.hcd` firmware
2. **Patch the embedded MAC address inside the firmware (1 byte change)**
3. Use **KeyboardOnly agent** in BlueZ
4. Let **key input trigger the Legacy HID connection**

This allows the original keyboard to work
**without changing the stock Ubuntu kernel**.

---

## Bluetooth Keyboard Setup (Important)

### Firmware placement

```bash
mkdir -p /lib/firmware/brcm
cp BCM43341B0.hcd /lib/firmware/brcm/
cp BCM43341B0.hcd /lib/firmware/brcm/BCM.hcd
chmod 644 /lib/firmware/brcm/*.hcd

```

##  Patch the MAC address in firmware (example)
printf '\xAB' | dd of=/lib/firmware/brcm/BCM43341B0.hcd \
  bs=1 seek=$((0x007B)) count=1 conv=notrunc

##  Bluetooth Connection Procedure (Legacy HID)
bluetoothctl
power on
agent KeyboardOnly
default-agent
pairable on
discoverable on
scan on

⚠️ Do NOT use connect
When the keyboard appears:
Press keys on the physical keyboard
The HID connection is established automatically

##  Stylus Calibration
Only the stylus device needs calibration (touch input is already correct):

xinput set-prop "SYNA7508:00 06CB:11DD Stylus Pen (0)" \
  "Coordinate Transformation Matrix" \
  0 1 0 -1 0 1 0 0 1

Make it persistent using autostart.

##  Notes
This guide is specific to ASUS T90CHI
Legacy HID behavior is poorly documented and counter-intuitive
BlueZ 5.x does not automatically handle this device


##  References
nomux2.net
drvlabo.jp
5ch notepc thread (ASUS T90CHI)





  
