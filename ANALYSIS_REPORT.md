# Arduino 物件計數系統 - 全面技術分析報告

**分析日期:** 2025-11-22
**專案版本:** v3.0 Debug修正版
**分析者:** Claude Code

---

## 目錄

1. [專案概述](#1-專案概述)
2. [測試覆蓋率評估](#2-測試覆蓋率評估)
3. [代碼結構分析與 Python 整合接點](#3-代碼結構分析與-python-整合接點)
4. [通訊協議實現審查](#4-通訊協議實現審查)
5. [硬體相容性與限制分析](#5-硬體相容性與限制分析)
6. [測試與整合策略](#6-測試與整合策略)
7. [與 image_processing_analysis_claude 整合方案](#7-與-image_processing_analysis_claude-整合方案)

---

## 1. 專案概述

### 1.1 系統功能
這是一個基於 Arduino 的紅外線物件計數系統，具備以下核心功能：
- 使用紅外線感測器偵測物體通過
- 雙位七段顯示器顯示計數 (0-99)
- 透過 74LS47 BCD 解碼器驅動顯示器
- 三狀態機制防止重複計數
- Serial 通訊提供除錯輸出

### 1.2 硬體配置
```
┌─────────────────────────────────────────────────────────┐
│                    Arduino Uno/Nano                      │
├─────────────────────────────────────────────────────────┤
│  D2  ← IR Sensor (中斷 INT0)                            │
│  D3  → 74LS47 #1 (個位) A                               │
│  D4  → 74LS47 #1 (個位) B                               │
│  D5  → 74LS47 #1 (個位) C                               │
│  D6  → 74LS47 #1 (個位) D                               │
│  D7  → 74LS47 #2 (十位) A                               │
│  D8  → 74LS47 #2 (十位) B                               │
│  D9  → 74LS47 #2 (十位) C                               │
│  D10 → 74LS47 #2 (十位) D                               │
│  TX  → Serial 輸出 (9600 baud)                          │
└─────────────────────────────────────────────────────────┘
```

### 1.3 狀態機設計
```
┌─────────┐    物體遮擋      ┌──────────┐    物體離開     ┌──────────┐
│  IDLE   │ ───────────────→ │ COUNTING │ ──────────────→ │ BLOCKING │
│ (閒置)  │    (防抖通過)    │ (計數中) │                 │ (封鎖中) │
└─────────┘                  └──────────┘                 └────┬─────┘
     ↑                                                          │
     │                    封鎖時間 300ms 結束                    │
     └──────────────────────────────────────────────────────────┘
```

---

## 2. 測試覆蓋率評估

### 2.1 當前測試狀態
**測試覆蓋率: 0%**

目前專案**完全沒有自動化測試**：
- 無單元測試檔案
- 無整合測試檔案
- 無模擬測試框架
- 僅有註解中的手動測試步驟

### 2.2 需要單元測試的領域

| 模組/函式 | 優先級 | 測試類型 | 說明 |
|-----------|--------|----------|------|
| `outputBCD()` | **高** | 單元測試 | BCD 編碼轉換邏輯，可完全離線測試 |
| `displayNumber()` | **高** | 單元測試 | 數字分離邏輯 (十位/個位) |
| `sensorISR()` 狀態轉換 | **高** | 單元測試 | 狀態機邏輯驗證 |
| 防抖動邏輯 | **中** | 單元測試 | 時間閾值判斷 |
| 計數溢位處理 | **中** | 單元測試 | 99→0 循環邏輯 |
| 封鎖時間管理 | **中** | 單元測試 | 時間計算邏輯 |

### 2.3 需要整合測試的領域

| 測試場景 | 優先級 | 測試類型 | 說明 |
|----------|--------|----------|------|
| 完整計數週期 | **高** | 整合測試 | IDLE→COUNTING→BLOCKING→IDLE |
| Serial 輸出格式 | **高** | 整合測試 | 驗證輸出格式與 Python 解析相容 |
| 中斷與主迴圈同步 | **高** | 整合測試 | volatile 變數存取安全性 |
| 快速連續觸發 | **中** | 壓力測試 | 防抖動機制有效性 |
| 長時間運行穩定性 | **中** | 耐久測試 | 記憶體洩漏、計數溢位 |

### 2.4 建議的測試框架

```plaintext
推薦框架:
├── Arduino 端: AUnit (ArduinoUnit 的改進版)
│   ├── 支援模擬時間函式 (mock millis())
│   ├── 支援斷言和測試套件
│   └── 與 PlatformIO 整合良好
│
└── Python 端: pytest + pyserial
    ├── 整合測試 Serial 通訊
    ├── 模擬感測器觸發
    └── 驗證數據解析
```

---

## 3. 代碼結構分析與 Python 整合接點

### 3.1 代碼架構評估

**優點:**
- ✅ 清晰的模組化函式設計
- ✅ 完整的中文註解
- ✅ 使用 volatile 保護共享變數
- ✅ 臨界區保護 (noInterrupts/interrupts)
- ✅ 狀態機設計清晰

**缺點:**
- ❌ 單一檔案設計，缺乏模組分離
- ❌ 硬編碼的 Serial 輸出格式
- ❌ 缺乏命令解析機制
- ❌ 無雙向通訊協議

### 3.2 Python 整合接點識別

#### 接點 1: Serial 輸出數據流 (現有)
```cpp
// 位置: loop() 函式，第 205-218 行
Serial.print(F("[計數] 當前值: "));
Serial.print(currentCount);
Serial.print(F(" | 總觸發: "));
...
```

**當前輸出格式:**
```
[計數] 當前值: 15 | 總觸發: 20 | 濾除: 5 | 成功率: 75.0%
[狀態] 封鎖解除,回到閒置狀態
```

#### 接點 2: 建議新增的命令接口 (需開發)
```cpp
// 建議新增的命令處理機制
void processCommand(String cmd) {
    if (cmd == "GET_COUNT") {
        Serial.println(g_count);
    } else if (cmd == "RESET") {
        g_count = 0;
        displayNumber(0);
    } else if (cmd == "GET_STATS") {
        // JSON 格式輸出
        Serial.print("{\"count\":");
        Serial.print(g_count);
        Serial.print(",\"triggers\":");
        Serial.print(g_totalTriggers);
        Serial.println("}");
    }
}
```

#### 接點 3: 事件觸發點 (供影像同步)
```cpp
// 位置: sensorISR() 第 299-302 行
g_count = nextCount;
g_needUpdate = true;
// ← 此處是最佳的 Python 通知點
// 可新增: Serial.println("EVENT:COUNT_UPDATE");
```

### 3.3 整合架構建議

```
┌──────────────────┐     Serial      ┌─────────────────────┐
│   Arduino        │ ◄─────────────► │  Python 影像處理    │
│   計數系統       │    9600 baud    │  (image_processing) │
└──────────────────┘                 └─────────────────────┘
        │                                      │
        │  事件: COUNT_UPDATE                  │
        │ ─────────────────────────────────► │
        │                                      │  觸發影像擷取
        │  命令: GET_COUNT                     │
        │ ◄─────────────────────────────────  │
        │                                      │
        │  回應: {"count": 15}                 │
        │ ─────────────────────────────────► │
        │                                      │  同步計數顯示
```

---

## 4. 通訊協議實現審查

### 4.1 當前實現評估

| 項目 | 當前狀態 | 評分 | 建議 |
|------|----------|------|------|
| 鮑率設定 | 9600 baud | ⭐⭐⭐ | 可升級至 115200 |
| 數據格式 | 人類可讀文字 | ⭐⭐ | 改用結構化格式 |
| 錯誤處理 | 無 | ⭐ | 需新增校驗 |
| 雙向通訊 | 僅單向輸出 | ⭐ | 需新增命令解析 |
| 流量控制 | 無 | ⭐⭐ | 軟體流控足夠 |

### 4.2 數據傳輸穩定性分析

**潛在問題:**

1. **訊息碎片化風險**
   ```python
   # Python 端可能收到不完整的行
   "[計數] 當前值: 15 | 總觸發:"  # 第一次讀取
   " 20 | 濾除: 5 | 成功率: 75.0%"  # 第二次讀取
   ```

2. **時序問題**
   - ISR 中更新 `g_totalTriggers++` 可能與主迴圈的讀取競爭
   - 解決方案：已使用 `noInterrupts()` 保護，但統計變數未保護

3. **緩衝區溢位風險**
   ```cpp
   // Serial 緩衝區僅 64 bytes
   // 若 Python 讀取速度不足，數據可能丟失
   ```

### 4.3 建議的協議改進

```cpp
// 建議採用 JSON 格式 + 校驗碼
// 格式: {"type":"count","data":{"value":15,"ts":12345},"crc":"A3"}

struct Message {
    char type[16];
    uint8_t value;
    uint32_t timestamp;
    uint8_t crc;
};
```

**Python 端解析範例:**
```python
import serial
import json

class ArduinoCounter:
    def __init__(self, port='/dev/ttyUSB0', baud=9600):
        self.ser = serial.Serial(port, baud, timeout=1)

    def read_count(self):
        line = self.ser.readline().decode('utf-8').strip()
        if line.startswith('{'):
            data = json.loads(line)
            return data.get('count')
        return None

    def send_command(self, cmd):
        self.ser.write(f"{cmd}\n".encode())
```

---

## 5. 硬體相容性與限制分析

### 5.1 硬體相容性矩陣

| Arduino 板型 | 相容性 | 注意事項 |
|-------------|--------|----------|
| Arduino Uno | ✅ 完全相容 | D2 支援 INT0 |
| Arduino Nano | ✅ 完全相容 | 相同腳位配置 |
| Arduino Mega | ✅ 相容 | D2 為 INT4 |
| Arduino Leonardo | ⚠️ 需修改 | D2 為 INT1 |
| ESP32 | ⚠️ 需大幅修改 | GPIO 編號不同 |
| ESP8266 | ❌ 不建議 | GPIO 數量不足 |

### 5.2 時序限制分析

```plaintext
關鍵時序參數:
├── 防抖動時間: 100ms (DEBOUNCE_MS)
│   └── 最大偵測速率: 10 物件/秒
│
├── 封鎖時間: 300ms (BLOCK_TIME_MS)
│   └── 有效最大速率: ~2.5 物件/秒
│
├── 主迴圈延遲: 10ms
│   └── 顯示更新延遲: 最大 10ms
│
└── Serial 輸出延遲: ~10ms/行 @ 9600 baud
    └── 潛在瓶頸於高頻觸發時
```

**時序瓶頸識別:**
```
物體通過速度 > 2.5/秒 時:
  └── BLOCKING 狀態可能錯過有效觸發

Serial 輸出速度 @ 9600 baud:
  └── ~100 字元/秒 輸出能力
  └── 每次計數輸出約 60 字元
  └── 理論最大: ~1.6 次計數/秒 (Serial 限制)
```

### 5.3 資源限制分析

| 資源 | 使用量 | 上限 (Uno) | 使用率 |
|------|--------|------------|--------|
| Flash (程式碼) | ~4 KB | 32 KB | 12.5% |
| SRAM (變數) | ~200 bytes | 2 KB | 10% |
| EEPROM | 未使用 | 1 KB | 0% |
| GPIO 數位腳位 | 9 pins | 14 pins | 64% |
| 硬體中斷 | 1 (INT0) | 2 | 50% |

**記憶體使用詳細:**
```cpp
volatile uint8_t  g_count           // 1 byte
volatile uint32_t g_lastTriggerMs   // 4 bytes
volatile uint32_t g_blockStartMs    // 4 bytes
volatile uint8_t  g_state           // 1 byte
volatile bool     g_needUpdate      // 1 byte
uint8_t           g_lastShown       // 1 byte
uint32_t          g_totalTriggers   // 4 bytes
uint32_t          g_rejectedTriggers// 4 bytes
// 總計: 20 bytes 全域變數
// + Serial 緩衝區 64 bytes
// + 堆疊空間 ~100 bytes
```

### 5.4 潛在硬體問題

1. **感測器誤觸發**
   - 環境光干擾
   - 電源雜訊
   - 建議: 增加硬體濾波電容

2. **顯示器亮度不均**
   - 74LS47 驅動能力限制
   - 建議: 每段使用 220Ω 限流電阻

3. **長導線問題**
   - 感測器線超過 1m 可能產生雜訊
   - 建議: 使用屏蔽線或差分信號

---

## 6. 測試與整合策略

### 6.1 測試階段規劃

```
Phase 1: 單元測試 (Week 1)
├── 建立 PlatformIO 測試環境
├── outputBCD() 函式測試
├── displayNumber() 函式測試
└── 狀態機邏輯測試

Phase 2: 模擬測試 (Week 1-2)
├── 模擬 millis() 時間函式
├── 防抖動邏輯驗證
└── 計數溢位測試

Phase 3: 硬體整合測試 (Week 2)
├── 實際感測器測試
├── 顯示器驗證
└── Serial 通訊測試

Phase 4: Python 整合測試 (Week 2-3)
├── 建立 Serial 通訊橋接
├── 數據解析驗證
└── 影像同步測試
```

### 6.2 建議的測試檔案結構

```
Arduino_ClaudeCode/
├── 20251024_arduino_counter_fixed/
│   └── 20251024_arduino_counter_fixed.ino
├── test/
│   ├── test_bcd/
│   │   └── test_bcd.cpp          # BCD 編碼單元測試
│   ├── test_display/
│   │   └── test_display.cpp      # 顯示邏輯測試
│   ├── test_state_machine/
│   │   └── test_state_machine.cpp # 狀態機測試
│   └── test_integration/
│       └── test_serial.py        # Python 整合測試
├── lib/
│   └── Counter/
│       ├── Counter.h             # 模組化後的標頭檔
│       └── Counter.cpp           # 模組化後的實作
├── platformio.ini                # PlatformIO 配置
└── ANALYSIS_REPORT.md
```

### 6.3 PlatformIO 配置建議

```ini
; platformio.ini
[env:uno]
platform = atmelavr
board = uno
framework = arduino

[env:native]
platform = native
build_flags = -std=c++11
lib_deps =
    throwtheswitch/Unity@^2.5.2

[env:test]
platform = native
test_framework = unity
```

### 6.4 單元測試範例

```cpp
// test/test_bcd/test_bcd.cpp
#include <unity.h>

// 模擬 Arduino 函式
uint8_t mock_pins[14] = {0};
void digitalWrite(uint8_t pin, uint8_t val) {
    mock_pins[pin] = val;
}

// 包含待測函式
void outputBCD(uint8_t digit, bool toRight);

void test_bcd_zero() {
    outputBCD(0, true);
    TEST_ASSERT_EQUAL(0, mock_pins[3]);  // RIGHT_A
    TEST_ASSERT_EQUAL(0, mock_pins[4]);  // RIGHT_B
    TEST_ASSERT_EQUAL(0, mock_pins[5]);  // RIGHT_C
    TEST_ASSERT_EQUAL(0, mock_pins[6]);  // RIGHT_D
}

void test_bcd_five() {
    outputBCD(5, true);  // 5 = 0101
    TEST_ASSERT_EQUAL(1, mock_pins[3]);  // A=1
    TEST_ASSERT_EQUAL(0, mock_pins[4]);  // B=0
    TEST_ASSERT_EQUAL(1, mock_pins[5]);  // C=1
    TEST_ASSERT_EQUAL(0, mock_pins[6]);  // D=0
}

void test_bcd_nine() {
    outputBCD(9, false);  // 9 = 1001, to left display
    TEST_ASSERT_EQUAL(1, mock_pins[7]);  // LEFT_A=1
    TEST_ASSERT_EQUAL(0, mock_pins[8]);  // LEFT_B=0
    TEST_ASSERT_EQUAL(0, mock_pins[9]);  // LEFT_C=0
    TEST_ASSERT_EQUAL(1, mock_pins[10]); // LEFT_D=1
}

int main() {
    UNITY_BEGIN();
    RUN_TEST(test_bcd_zero);
    RUN_TEST(test_bcd_five);
    RUN_TEST(test_bcd_nine);
    return UNITY_END();
}
```

---

## 7. 與 image_processing_analysis_claude 整合方案

### 7.1 整合架構

```
┌─────────────────────────────────────────────────────────────────┐
│                        整合系統架構                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌──────────────┐         ┌──────────────────────────────┐    │
│   │   Arduino    │         │   image_processing_analysis   │    │
│   │   Counter    │◄───────►│         _claude               │    │
│   └──────┬───────┘  Serial └──────────────┬───────────────┘    │
│          │                                 │                     │
│          │                                 │                     │
│    ┌─────▼─────┐                    ┌─────▼─────┐               │
│    │ IR Sensor │                    │  Camera   │               │
│    │ + Display │                    │  Module   │               │
│    └───────────┘                    └───────────┘               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 7.2 通訊協議設計

**Arduino → Python (事件通知):**
```
格式: EVENT:<type>:<data>\n
範例:
  EVENT:COUNT:15
  EVENT:STATE:IDLE
  EVENT:TRIGGER:VALID
  EVENT:TRIGGER:REJECTED
```

**Python → Arduino (命令):**
```
格式: CMD:<command>:<params>\n
範例:
  CMD:GET_COUNT:
  CMD:RESET:
  CMD:SET_DEBOUNCE:150
  CMD:SET_BLOCK:400
```

**Arduino → Python (回應):**
```
格式: RSP:<status>:<data>\n
範例:
  RSP:OK:15
  RSP:OK:{"count":15,"triggers":20,"rejected":5}
  RSP:ERROR:INVALID_CMD
```

### 7.3 Python 整合模組

```python
# arduino_bridge.py
"""Arduino 計數器與 Python 影像處理整合橋接模組"""

import serial
import threading
import queue
import time
import json
from typing import Callable, Optional, Dict, Any

class ArduinoCounterBridge:
    """Arduino 計數器通訊橋接類"""

    def __init__(self, port: str = '/dev/ttyUSB0', baud: int = 9600):
        self.port = port
        self.baud = baud
        self.serial: Optional[serial.Serial] = None
        self.event_queue: queue.Queue = queue.Queue()
        self.running = False
        self._read_thread: Optional[threading.Thread] = None
        self._callbacks: Dict[str, Callable] = {}

    def connect(self) -> bool:
        """建立 Serial 連接"""
        try:
            self.serial = serial.Serial(
                self.port,
                self.baud,
                timeout=1,
                bytesize=serial.EIGHTBITS,
                parity=serial.PARITY_NONE,
                stopbits=serial.STOPBITS_ONE
            )
            time.sleep(2)  # 等待 Arduino 重置
            self.running = True
            self._read_thread = threading.Thread(target=self._read_loop, daemon=True)
            self._read_thread.start()
            return True
        except serial.SerialException as e:
            print(f"連接失敗: {e}")
            return False

    def disconnect(self):
        """關閉連接"""
        self.running = False
        if self._read_thread:
            self._read_thread.join(timeout=2)
        if self.serial:
            self.serial.close()

    def _read_loop(self):
        """Serial 讀取迴圈 (執行於獨立線程)"""
        while self.running:
            try:
                if self.serial and self.serial.in_waiting:
                    line = self.serial.readline().decode('utf-8', errors='ignore').strip()
                    if line:
                        self._process_message(line)
            except Exception as e:
                print(f"讀取錯誤: {e}")
            time.sleep(0.01)

    def _process_message(self, message: str):
        """處理接收到的訊息"""
        # 解析 EVENT 訊息
        if message.startswith('EVENT:'):
            parts = message.split(':')
            if len(parts) >= 3:
                event_type = parts[1]
                event_data = ':'.join(parts[2:])
                self._trigger_callback(event_type, event_data)

        # 解析舊格式 [計數] 訊息
        elif '[計數]' in message:
            count = self._parse_count_message(message)
            if count is not None:
                self._trigger_callback('COUNT', str(count))

        # 其他訊息放入佇列
        self.event_queue.put(message)

    def _parse_count_message(self, message: str) -> Optional[int]:
        """解析舊格式計數訊息"""
        try:
            # [計數] 當前值: 15 | 總觸發: 20 | ...
            if '當前值:' in message:
                value_part = message.split('當前值:')[1]
                value_str = value_part.split('|')[0].strip()
                return int(value_str)
        except (IndexError, ValueError):
            pass
        return None

    def register_callback(self, event_type: str, callback: Callable[[str], None]):
        """註冊事件回調函式"""
        self._callbacks[event_type] = callback

    def _trigger_callback(self, event_type: str, data: str):
        """觸發回調函式"""
        if event_type in self._callbacks:
            try:
                self._callbacks[event_type](data)
            except Exception as e:
                print(f"回調執行錯誤: {e}")

    def send_command(self, command: str, params: str = '') -> bool:
        """發送命令到 Arduino"""
        if not self.serial:
            return False
        try:
            msg = f"CMD:{command}:{params}\n"
            self.serial.write(msg.encode('utf-8'))
            return True
        except serial.SerialException:
            return False

    def get_count(self) -> Optional[int]:
        """取得當前計數值"""
        self.send_command('GET_COUNT')
        # 等待回應
        try:
            for _ in range(10):  # 最多等待 1 秒
                try:
                    msg = self.event_queue.get(timeout=0.1)
                    if msg.startswith('RSP:OK:'):
                        return int(msg.split(':')[2])
                except queue.Empty:
                    continue
        except (ValueError, IndexError):
            pass
        return None

    def reset_counter(self) -> bool:
        """重置計數器"""
        return self.send_command('RESET')


# 整合範例
class ImageProcessingIntegration:
    """影像處理整合類"""

    def __init__(self, arduino_port: str):
        self.bridge = ArduinoCounterBridge(arduino_port)
        self.last_count = 0

    def start(self):
        """啟動整合系統"""
        if self.bridge.connect():
            # 註冊計數更新回調
            self.bridge.register_callback('COUNT', self.on_count_update)
            print("整合系統啟動成功")
            return True
        return False

    def on_count_update(self, count_str: str):
        """計數更新時的回調處理"""
        try:
            count = int(count_str)
            if count != self.last_count:
                print(f"計數更新: {self.last_count} -> {count}")
                self.last_count = count
                # 觸發影像擷取
                self.trigger_image_capture(count)
        except ValueError:
            pass

    def trigger_image_capture(self, count: int):
        """觸發影像擷取 (供 image_processing_analysis_claude 呼叫)"""
        # 這裡實現與影像處理模組的整合
        print(f"觸發影像擷取 - 計數: {count}")
        # TODO: 呼叫影像處理模組 API

    def stop(self):
        """停止整合系統"""
        self.bridge.disconnect()


# 使用範例
if __name__ == '__main__':
    integration = ImageProcessingIntegration('/dev/ttyUSB0')

    if integration.start():
        try:
            print("系統運行中，按 Ctrl+C 停止...")
            while True:
                time.sleep(1)
        except KeyboardInterrupt:
            print("\n停止系統...")
        finally:
            integration.stop()
```

### 7.4 Arduino 端協議擴展建議

```cpp
// 建議新增到 Arduino 程式碼中的命令處理函式

// 新增全域變數
String g_inputBuffer = "";  // Serial 輸入緩衝區

// 在 loop() 中新增命令處理
void loop() {
    // ... 現有程式碼 ...

    // 處理 Serial 輸入命令
    while (Serial.available() > 0) {
        char c = Serial.read();
        if (c == '\n') {
            processCommand(g_inputBuffer);
            g_inputBuffer = "";
        } else if (c != '\r') {
            g_inputBuffer += c;
        }
    }
}

// 命令處理函式
void processCommand(String cmd) {
    cmd.trim();

    if (cmd.startsWith("CMD:GET_COUNT:")) {
        Serial.print(F("RSP:OK:"));
        Serial.println(g_count);
    }
    else if (cmd.startsWith("CMD:RESET:")) {
        noInterrupts();
        g_count = 0;
        g_totalTriggers = 0;
        g_rejectedTriggers = 0;
        g_needUpdate = true;
        interrupts();
        displayNumber(0);
        Serial.println(F("RSP:OK:RESET_DONE"));
    }
    else if (cmd.startsWith("CMD:GET_STATS:")) {
        Serial.print(F("RSP:OK:{\"count\":"));
        Serial.print(g_count);
        Serial.print(F(",\"triggers\":"));
        Serial.print(g_totalTriggers);
        Serial.print(F(",\"rejected\":"));
        Serial.print(g_rejectedTriggers);
        Serial.println(F("}"));
    }
    else if (cmd.startsWith("CMD:SET_DEBOUNCE:")) {
        // 需要額外實現參數解析
        Serial.println(F("RSP:ERROR:NOT_IMPLEMENTED"));
    }
    else {
        Serial.println(F("RSP:ERROR:UNKNOWN_CMD"));
    }
}

// 在計數更新時發送事件 (修改 loop() 中的輸出)
if (needUpdate && (currentCount != g_lastShown)) {
    displayNumber(currentCount);
    g_lastShown = currentCount;

    // 發送結構化事件
    Serial.print(F("EVENT:COUNT:"));
    Serial.println(currentCount);

    // 保留詳細輸出 (可選)
    Serial.print(F("[計數] 當前值: "));
    // ... 其餘輸出 ...
}
```

### 7.5 整合測試計劃

```python
# test/test_integration/test_serial.py
"""Arduino Serial 通訊整合測試"""

import pytest
import serial
import time

class TestArduinoIntegration:
    """Arduino 整合測試類"""

    @pytest.fixture
    def arduino(self):
        """建立 Arduino 連接"""
        ser = serial.Serial('/dev/ttyUSB0', 9600, timeout=2)
        time.sleep(2)  # 等待 Arduino 重置
        yield ser
        ser.close()

    def test_connection(self, arduino):
        """測試基本連接"""
        assert arduino.is_open

    def test_initial_output(self, arduino):
        """測試初始化輸出"""
        output = arduino.readline().decode('utf-8')
        assert '物件計數系統' in output or '====' in output

    def test_get_count_command(self, arduino):
        """測試 GET_COUNT 命令"""
        arduino.write(b'CMD:GET_COUNT:\n')
        time.sleep(0.1)
        response = arduino.readline().decode('utf-8').strip()
        assert response.startswith('RSP:OK:')

    def test_reset_command(self, arduino):
        """測試 RESET 命令"""
        arduino.write(b'CMD:RESET:\n')
        time.sleep(0.1)
        response = arduino.readline().decode('utf-8').strip()
        assert 'RSP:OK' in response

    def test_stats_json_format(self, arduino):
        """測試統計資料 JSON 格式"""
        arduino.write(b'CMD:GET_STATS:\n')
        time.sleep(0.1)
        response = arduino.readline().decode('utf-8').strip()
        assert 'RSP:OK:{' in response
        assert '"count":' in response
```

---

## 8. 總結與建議優先級

### 8.1 立即行動項目 (高優先級)

| 項目 | 說明 | 預估工作量 |
|------|------|------------|
| 1. 建立測試框架 | 配置 PlatformIO + Unity | 2-4 小時 |
| 2. 實現雙向通訊 | 新增命令處理函式 | 4-6 小時 |
| 3. 模組化重構 | 分離 .h/.cpp 檔案 | 4-6 小時 |
| 4. Python 橋接模組 | 實現基本通訊類 | 4-6 小時 |

### 8.2 短期改進項目 (中優先級)

| 項目 | 說明 | 預估工作量 |
|------|------|------------|
| 5. JSON 協議實現 | 結構化數據傳輸 | 2-4 小時 |
| 6. 單元測試完善 | 覆蓋核心函式 | 4-8 小時 |
| 7. 整合測試腳本 | Python pytest | 4-6 小時 |
| 8. 提升鮑率 | 9600 → 115200 | 1-2 小時 |

### 8.3 長期優化項目 (低優先級)

| 項目 | 說明 | 預估工作量 |
|------|------|------------|
| 9. ESP32 移植 | WiFi 無線通訊 | 8-16 小時 |
| 10. 資料持久化 | EEPROM 儲存 | 2-4 小時 |
| 11. Web 儀表板 | 即時監控界面 | 8-16 小時 |

---

## 附錄 A: 代碼品質指標

| 指標 | 數值 | 評估 |
|------|------|------|
| 代碼行數 (LOC) | 445 | 適中 |
| 註解密度 | ~60% | 優秀 |
| 循環複雜度 | 低 | 優秀 |
| 函式數量 | 5 | 適中 |
| 全域變數數量 | 8 | 可接受 |
| 魔術數字 | 少量 | 良好 |

## 附錄 B: 腳位分配圖

```
Arduino Uno 腳位分配:
────────────────────────────────────────
D0  │ RX (保留給 Serial)
D1  │ TX (保留給 Serial)
D2  │ ← IR 感測器輸入 (INT0)
D3  │ → 74LS47 個位 A
D4  │ → 74LS47 個位 B
D5  │ → 74LS47 個位 C
D6  │ → 74LS47 個位 D
D7  │ → 74LS47 十位 A
D8  │ → 74LS47 十位 B
D9  │ → 74LS47 十位 C
D10 │ → 74LS47 十位 D
D11 │ (未使用)
D12 │ (未使用)
D13 │ (未使用/LED)
────────────────────────────────────────
```

---

*報告結束*
