# Arduino 物件計數系統 | Arduino Object Counter System

<div align="center">

[![Arduino](https://img.shields.io/badge/Arduino-Uno%2FNano-00979D.svg?logo=arduino)](https://www.arduino.cc/)
[![Language](https://img.shields.io/badge/Language-C%2B%2B-blue.svg)](https://isocpp.org/)
[![Version](https://img.shields.io/badge/Version-3.0-green.svg)](https://github.com/FlyEddiegogo/Arduino_ClaudeCode)

**紅外線感測器 + 七段顯示器物件計數系統**
**IR Sensor + 7-Segment Display Object Counter**

[繁體中文](#中文說明) | [English](#english-documentation)

</div>

---

## 中文說明

### 📌 專案簡介

這是一個基於 Arduino 的物件計數系統，使用紅外線感測器偵測物體通過，並透過雙位七段顯示器顯示計數結果（0-99）。系統採用三狀態機設計，有效防止物體離開時的重複計數問題。

**核心特色：**
- 🔢 雙位七段顯示器（0-99 循環計數）
- 🚦 三狀態機制防止重複計數（IDLE → COUNTING → BLOCKING）
- ⚡ 硬體中斷驅動，反應快速
- 🛡️ 防抖動機制（100ms）+ 封鎖保護（300ms）
- 📊 Serial 即時統計資訊（總觸發、濾除次數、成功率）
- 🔧 完整中文註解，每行程式都有詳細說明

### 🛠️ 硬體需求

#### 主要元件
- **Arduino 開發板**: Uno / Nano（ATmega328P）
- **紅外線感測器**: 1 個（數位輸出型，支援 Active Low/High）
- **74LS47 BCD 解碼器**: 2 個（驅動七段顯示器）
- **共陽極七段顯示器**: 2 個（0.56" 或類似尺寸）
- **限流電阻**: 14 個（220Ω，每個 LED 段 1 個）
- **麵包板與杜邦線**: 用於電路組裝

#### 腳位配置

```
Arduino 腳位連接:
┌─────────────────────────────────────────────┐
│ D2  ← 紅外線感測器（支援硬體中斷 INT0）       │
│ D3  → 74LS47 #1 (個位) A (LSB)              │
│ D4  → 74LS47 #1 (個位) B                    │
│ D5  → 74LS47 #1 (個位) C                    │
│ D6  → 74LS47 #1 (個位) D (MSB)              │
│ D7  → 74LS47 #2 (十位) A (LSB)              │
│ D8  → 74LS47 #2 (十位) B                    │
│ D9  → 74LS47 #2 (十位) C                    │
│ D10 → 74LS47 #2 (十位) D (MSB)              │
└─────────────────────────────────────────────┘
```

### 🚀 快速開始

#### 1. 下載程式碼

```bash
# 克隆專案
git clone https://github.com/FlyEddiegogo/Arduino_ClaudeCode.git
cd Arduino_ClaudeCode/20251024_arduino_counter_fixed
```

#### 2. 開啟 Arduino IDE

1. 啟動 Arduino IDE（建議版本 1.8.19 或 2.x）
2. 開啟 `20251024_arduino_counter_fixed.ino`
3. 選擇正確的開發板：**工具 → 開發板 → Arduino Uno/Nano**
4. 選擇正確的序列埠：**工具 → 序列埠 → COMx**

#### 3. 上傳程式

1. 點擊 **上傳** 按鈕（→）
2. 等待編譯與上傳完成
3. 開啟 **工具 → 序列埠監視器**（鮑率設為 9600）

#### 4. 測試運行

```
系統初始化訊息範例:
====================================
物件計數系統 v3.0 Debug修正版
====================================
系統初始化中...
感測器邏輯: 遮擋時輸出LOW (Active Low)
防抖動時間: 100 ms
封鎖保護時間: 300 ms
系統初始化完成!
準備開始計數...
====================================
```

用手或物體遮擋感測器，觀察七段顯示器計數變化。

### 📊 狀態機設計

系統採用三狀態機制，確保計數準確無誤：

```
┌─────────┐  物體遮擋(防抖通過)  ┌──────────┐  物體離開  ┌──────────┐
│  IDLE   │ ──────────────────→ │ COUNTING │ ─────────→ │ BLOCKING │
│ (閒置)  │                      │ (計數中) │            │ (封鎖中) │
└─────────┘                      └──────────┘            └────┬─────┘
     ↑                                                         │
     │              封鎖時間 300ms 結束                         │
     └─────────────────────────────────────────────────────────┘
```

**狀態說明：**
- **IDLE**: 等待物體進入，檢測到遮擋且通過防抖動後計數 +1
- **COUNTING**: 物體通過中，等待物體完全離開
- **BLOCKING**: 封鎖期，忽略所有觸發，防止重複計數

### ⚙️ 參數調整

可在程式碼中調整以下參數以適應不同應用場景：

```cpp
// 防抖動時間（毫秒）- 物體通過後的最短間隔
const uint16_t DEBOUNCE_MS = 100;     // 建議範圍: 50-150ms

// 封鎖時間（毫秒）- 物體離開後的保護期
const uint16_t BLOCK_TIME_MS = 300;   // 建議範圍: 200-500ms

// 感測器邏輯
#define SENSOR_ACTIVE_LOW 1           // 1=遮擋時 LOW, 0=遮擋時 HIGH
```

**調整建議：**
- 仍有重複計數 → 增加 `BLOCK_TIME_MS` (300-500ms)
- 漏計快速物體 → 減少 `DEBOUNCE_MS` (50-80ms)
- 計數太敏感 → 增加 `DEBOUNCE_MS` (100-150ms)

### 📈 Serial 輸出資訊

系統會透過 Serial 輸出詳細的運行資訊：

```
[計數] 當前值: 5 | 總觸發: 12 | 濾除: 7 | 成功率: 41.7%
[狀態] 封鎖解除,回到閒置狀態
[計數] 當前值: 6 | 總觸發: 15 | 濾除: 9 | 成功率: 40.0%
```

**欄位說明：**
- **當前值**: 顯示器上的計數（0-99）
- **總觸發**: 中斷被觸發的總次數
- **濾除**: 被防抖動/封鎖機制濾除的次數
- **成功率**: 有效計數占總觸發的百分比

### 🔧 故障排除

#### 問題 1：顯示器不顯示或顯示錯誤數字
**可能原因：**
- 74LS47 BCD 輸入腳位接錯
- 限流電阻值不正確或未安裝
- 七段顯示器共陽/共陰型態不符

**解決方法：**
1. 檢查腳位連接是否與程式定義一致
2. 確認使用共陽極七段顯示器（74LS47 專用）
3. 每個 LED 段必須有 220Ω 限流電阻

#### 問題 2：計數重複或跳數
**可能原因：**
- 防抖動時間太短
- 封鎖時間不足
- 感測器訊號不穩定

**解決方法：**
1. 增加 `BLOCK_TIME_MS` 至 400-500ms
2. 檢查感測器電源是否穩定（5V）
3. 確認感測器與物體距離適當（2-30cm）

#### 問題 3：無法偵測物體
**可能原因：**
- 感測器邏輯設定錯誤（Active Low/High）
- D2 腳位連接問題
- 感測器未通電

**解決方法：**
1. 檢查 Serial 輸出確認感測器邏輯設定
2. 用三用電表測量 D2 腳位電壓變化
3. 確認感測器 VCC、GND、OUT 三線正確連接

### 📚 進階應用

#### 與 Python 整合（物聯網應用）
```python
import serial

# 連接 Arduino
ser = serial.Serial('COM3', 9600, timeout=1)

while True:
    line = ser.readline().decode('utf-8').strip()
    if '[計數]' in line:
        # 解析計數資訊
        # 傳送至資料庫或雲端平台
        print(f"收到計數: {line}")
```

#### 擴充功能建議
- 加入 LCD1602 顯示器顯示更多資訊
- 加入 EEPROM 儲存累計計數（斷電保持）
- 加入藍牙模組（HC-05）無線傳輸資料
- 整合 Python 視覺系統進行複合式檢測

### 📄 專案文件

- **[ANALYSIS_REPORT.md](./ANALYSIS_REPORT.md)** - 完整技術分析報告（測試覆蓋率、整合方案等）
- **[20251024_arduino_counter_fixed.ino](./20251024_arduino_counter_fixed/20251024_arduino_counter_fixed.ino)** - 主程式（詳細中文註解）

### 📝 版本資訊

- **版本**: v3.0 Debug 修正版
- **日期**: 2024-10-24
- **修正內容**:
  - 修正十位數和個位數顯示位置錯誤
  - 改進中斷觸發邏輯，避免重複計數
  - 加強防抖動機制
  - 完整中文註解

### 👤 作者資訊

- **開發者**: Fly Eddie
- **GitHub**: https://github.com/FlyEddiegogo/Arduino_ClaudeCode

---

## English Documentation

### 📌 Project Overview

An Arduino-based object counting system using an infrared sensor to detect passing objects, displaying the count (0-99) on dual 7-segment displays. The system employs a three-state machine design to effectively prevent double-counting when objects leave the detection zone.

**Key Features:**
- 🔢 Dual 7-segment display (0-99 cyclic counting)
- 🚦 Three-state machine to prevent double-counting (IDLE → COUNTING → BLOCKING)
- ⚡ Hardware interrupt-driven, fast response
- 🛡️ Debounce mechanism (100ms) + blocking protection (300ms)
- 📊 Real-time Serial statistics (total triggers, rejected, success rate)
- 🔧 Comprehensive Chinese comments for every line of code

### 🛠️ Hardware Requirements

#### Main Components
- **Arduino Board**: Uno / Nano (ATmega328P)
- **IR Sensor**: 1x (digital output, supports Active Low/High)
- **74LS47 BCD Decoder**: 2x (drives 7-segment displays)
- **Common Anode 7-Segment Display**: 2x (0.56" or similar)
- **Current Limiting Resistors**: 14x (220Ω, one per LED segment)
- **Breadboard & Jumper Wires**: For circuit assembly

#### Pin Configuration

```
Arduino Pin Connections:
┌─────────────────────────────────────────────┐
│ D2  ← IR Sensor (hardware interrupt INT0)   │
│ D3  → 74LS47 #1 (ones) A (LSB)              │
│ D4  → 74LS47 #1 (ones) B                    │
│ D5  → 74LS47 #1 (ones) C                    │
│ D6  → 74LS47 #1 (ones) D (MSB)              │
│ D7  → 74LS47 #2 (tens) A (LSB)              │
│ D8  → 74LS47 #2 (tens) B                    │
│ D9  → 74LS47 #2 (tens) C                    │
│ D10 → 74LS47 #2 (tens) D (MSB)              │
└─────────────────────────────────────────────┘
```

### 🚀 Quick Start

#### 1. Download Code

```bash
# Clone repository
git clone https://github.com/FlyEddiegogo/Arduino_ClaudeCode.git
cd Arduino_ClaudeCode/20251024_arduino_counter_fixed
```

#### 2. Open Arduino IDE

1. Launch Arduino IDE (version 1.8.19 or 2.x recommended)
2. Open `20251024_arduino_counter_fixed.ino`
3. Select board: **Tools → Board → Arduino Uno/Nano**
4. Select port: **Tools → Port → COMx**

#### 3. Upload Program

1. Click **Upload** button (→)
2. Wait for compilation and upload to complete
3. Open **Tools → Serial Monitor** (baud rate: 9600)

#### 4. Test Operation

```
System initialization message:
====================================
Object Counter System v3.0 Debug Fixed
====================================
Initializing system...
Sensor logic: Blocked = LOW (Active Low)
Debounce time: 100 ms
Block protection time: 300 ms
System initialized!
Ready to count...
====================================
```

Block the sensor with your hand or an object, observe the 7-segment display count changes.

### 📊 State Machine Design

The system uses a three-state machine to ensure accurate counting:

```
┌─────────┐  Object blocks    ┌──────────┐  Object leaves  ┌──────────┐
│  IDLE   │ ────────────────→ │ COUNTING │ ──────────────→ │ BLOCKING │
│         │  (debounce pass)  │          │                 │          │
└─────────┘                   └──────────┘                 └────┬─────┘
     ↑                                                           │
     │              Block time 300ms expires                     │
     └───────────────────────────────────────────────────────────┘
```

**State Descriptions:**
- **IDLE**: Waiting for object entry, counts +1 when blocking detected and debounce passed
- **COUNTING**: Object passing through, waiting for complete departure
- **BLOCKING**: Block period, ignores all triggers to prevent double-counting

### ⚙️ Parameter Tuning

Adjust these parameters in code for different scenarios:

```cpp
// Debounce time (ms) - minimum interval after object passes
const uint16_t DEBOUNCE_MS = 100;     // Recommended: 50-150ms

// Block time (ms) - protection period after object leaves
const uint16_t BLOCK_TIME_MS = 300;   // Recommended: 200-500ms

// Sensor logic
#define SENSOR_ACTIVE_LOW 1           // 1=LOW when blocked, 0=HIGH when blocked
```

**Tuning Recommendations:**
- Still double-counting → Increase `BLOCK_TIME_MS` (300-500ms)
- Missing fast objects → Decrease `DEBOUNCE_MS` (50-80ms)
- Too sensitive → Increase `DEBOUNCE_MS` (100-150ms)

### 📈 Serial Output

System outputs detailed runtime information via Serial:

```
[Count] Current: 5 | Total triggers: 12 | Rejected: 7 | Success: 41.7%
[State] Block released, return to IDLE
[Count] Current: 6 | Total triggers: 15 | Rejected: 9 | Success: 40.0%
```

**Field Descriptions:**
- **Current**: Count shown on display (0-99)
- **Total triggers**: Total interrupt trigger count
- **Rejected**: Count rejected by debounce/blocking
- **Success**: Valid count percentage

### 🔧 Troubleshooting

#### Issue 1: Display shows nothing or wrong numbers
**Possible causes:**
- 74LS47 BCD input pins miswired
- Incorrect or missing current-limiting resistors
- 7-segment display type mismatch (common anode/cathode)

**Solutions:**
1. Verify pin connections match code definitions
2. Confirm using common anode 7-segment displays (74LS47 compatible)
3. Each LED segment must have 220Ω resistor

#### Issue 2: Repeated or skipped counts
**Possible causes:**
- Debounce time too short
- Insufficient block time
- Unstable sensor signal

**Solutions:**
1. Increase `BLOCK_TIME_MS` to 400-500ms
2. Check sensor power supply stability (5V)
3. Verify sensor-to-object distance is appropriate (2-30cm)

#### Issue 3: Cannot detect objects
**Possible causes:**
- Incorrect sensor logic setting (Active Low/High)
- D2 pin connection issue
- Sensor not powered

**Solutions:**
1. Check Serial output to confirm sensor logic setting
2. Use multimeter to measure D2 pin voltage changes
3. Verify sensor VCC, GND, OUT connections

### 📚 Advanced Applications

#### Python Integration (IoT Application)
```python
import serial

# Connect to Arduino
ser = serial.Serial('COM3', 9600, timeout=1)

while True:
    line = ser.readline().decode('utf-8').strip()
    if '[Count]' in line:
        # Parse count info
        # Send to database or cloud platform
        print(f"Received count: {line}")
```

#### Extension Ideas
- Add LCD1602 display for more information
- Add EEPROM storage for persistent counting
- Add Bluetooth module (HC-05) for wireless data
- Integrate with Python vision system for compound detection

### 📄 Project Documentation

- **[ANALYSIS_REPORT.md](./ANALYSIS_REPORT.md)** - Complete technical analysis (test coverage, integration plans)
- **[20251024_arduino_counter_fixed.ino](./20251024_arduino_counter_fixed/20251024_arduino_counter_fixed.ino)** - Main program (detailed Chinese comments)

### 📝 Version Info

- **Version**: v3.0 Debug Fixed
- **Date**: 2024-10-24
- **Fixes**:
  - Fixed tens/ones digit display position error
  - Improved interrupt trigger logic to prevent double-counting
  - Enhanced debounce mechanism
  - Complete Chinese comments

### 👤 Author

- **Developer**: Fly Eddie
- **GitHub**: https://github.com/FlyEddiegogo/Arduino_ClaudeCode

---

<div align="center">

**Made with ❤️ for Embedded Object Counting**

[⬆ Back to Top](#arduino-物件計數系統--arduino-object-counter-system)

</div>
