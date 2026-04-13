# D-Bus 筆記

## D-Bus 在哪裡？

D-Bus 在 **BMC 內部軟體層**，是跑在 BMC Linux 裡的 IPC 機制：

```
BMC 硬體（ARM 晶片）
  └─ BMC Linux OS
       └─ D-Bus（daemon: dbus-broker / dbus-daemon）
            ├─ phosphor-hwmon             （讀溫度/風扇/電壓感測器）
            ├─ phosphor-state-manager    （管主機開關機狀態）
            ├─ bmcweb                    （提供 Redfish / Web UI HTTP API）
            ├─ phosphor-host-ipmid       （處理 IPMI 指令）
            ├─ phosphor-network          （管理 BMC 網路設定）
            ├─ phosphor-fan-presence     （偵測風扇是否存在）
            ├─ phosphor-fan-control      （控制風扇轉速）
            ├─ phosphor-power            （電源管理、PSU 監控）
            ├─ phosphor-led-manager      （控制 BMC 上的 LED 燈號）
            ├─ phosphor-logging          （寫入 Event Log / SEL）
            ├─ phosphor-user-manager     （管理 BMC 使用者帳號）
            ├─ phosphor-certificate-manager（管理 TLS 憑證）
            ├─ obmc-console              （Serial-over-LAN / SOL）
            ├─ phosphor-software-manager （韌體更新管理）
            └─ entity-manager            （描述硬體拓撲，如哪顆 CPU 接在哪）
```

**D-Bus 是這些服務之間的通訊橋樑**，不對外，只在 BMC 內部。

---

## 服務之間怎麼通訊？事件驅動而非一直傳

直覺上可能以為：溫度服務會一直把資料傳給風扇服務。
但 OpenBMC 實際用的是 **事件驅動（Property Changed）** 的方式：

```
phosphor-hwmon
  └─ 讀到溫度值 → 更新 D-Bus 上的屬性（Property）
                        │
                        └─ D-Bus 發出 PropertiesChanged 訊號
                                  │
              ┌───────────────────┘
              │
phosphor-fan-control
  └─ 收到訊號 → 計算需要的風扇轉速 → 調整風扇 PWM
```

| 方式              | 說明                                                   |
| ----------------- | ------------------------------------------------------ |
| Polling（一直傳） | 風扇服務每隔 X 秒主動去問「現在幾度？」                |
| D-Bus 事件驅動    | 溫度服務更新數值後，D-Bus **自動通知**所有有訂閱的服務 |

OpenBMC 用的是後者。溫度沒變就沒有訊號，不浪費資源。

**把 D-Bus 想成「黑板 + 廣播系統」：**

```
phosphor-hwmon       → 在黑板上寫「CPU 溫度 = 85°C」
D-Bus                → 廣播「黑板上的 CPU 溫度欄位改了！」
phosphor-fan-control → 聽到廣播 → 去黑板查新值 → 加速風扇
```

---

## D-Bus 是什麼？

**不是晶片**，是一種 **行程間通訊（IPC）機制**，跑在 Linux 軟體層。

可以把它想像成 BMC 內部各服務之間的「對講機頻道」：

- 感測器服務、電源管理服務、Redfish API 服務各自獨立跑
- 它們透過 D-Bus 互相傳遞訊息、呼叫彼此的功能

---

## 跟 Log 的關係

|                                | 說明                                      |
| ------------------------------ | ----------------------------------------- |
| D-Bus 本身不寫 log             | 它只傳訊息，不存儲                        |
| 要看 D-Bus 活動 → `journalctl` | 各服務透過 D-Bus 通訊的結果會印到 journal |
| 即時監看 D-Bus 訊息            | SSH 進 BMC 後用 `busctl monitor`          |

---

## 常用指令

```bash
ssh root@<BMC-IP>

busctl list                          # 列出所有 D-Bus 服務
busctl monitor                       # 即時看所有 D-Bus 訊息流
busctl introspect xyz.openbmc_project.State.Host \
  /xyz/openbmc_project/state/host0   # 查看特定服務的介面與方法
```

---

## 學會後能做什麼？

- 新增自訂感測器或硬體監控功能
- 修改開關機流程邏輯
- 寫新的 D-Bus 服務讓 Redfish API 暴露新的端點
- 偵錯：用 `busctl` 直接查看 BMC 上所有服務狀態

---

## D-Bus 是硬體資訊進入軟體世界的轉運站

所有硬體狀態最終都匯聚到 D-Bus，你不用管底層是 I2C、SPI 還是 GPIO，統一從 D-Bus 查詢：

```
硬體世界                         BMC 軟體世界（D-Bus）
──────────────────────────────────────────────────────
BIOS POST Code  ──LPC/eSPI──▶ phosphor-host-postd  ──▶ D-Bus
溫度感測器      ──I2C/hwmon──▶ phosphor-hwmon       ──▶ D-Bus
風扇轉速        ──PWM/hwmon──▶ phosphor-hwmon       ──▶ D-Bus
電源狀態        ──GPIO──────▶ phosphor-state-manager──▶ D-Bus
                                                          │
                                                你在這裡撈 ◀───┐
                                            busctl / Python / Redfish
```

> 這就是 OpenBMC 架構的核心思想：**硬體抽象層，統一用 D-Bus 對外**。
