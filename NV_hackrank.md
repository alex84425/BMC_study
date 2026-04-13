# BMC 面試題整理

---

## D-Sub 與 D-Bus 的差別

|               | D-Sub                                | D-Bus                               |
| ------------- | ------------------------------------ | ----------------------------------- |
| 是什麼        | 硬體**接頭規格**（梯形金屬連接器）   | 軟體層的**行程間通訊（IPC）機制**   |
| 在哪裡        | 實體線材、顯示器接口（VGA 是 DB-15） | BMC Linux 內部，服務之間傳訊息用    |
| 跟 BMC 有關嗎 | 幾乎無關                             | 非常有關，是 OpenBMC 的核心通訊機制 |

---

## TPM（Trusted Platform Module）

- **硬體安全晶片**，焊在主板上，負責儲存加密金鑰、憑證、雜湊值
- 功能：
    - 儲存 Secure Boot 的信任根金鑰
    - 記錄開機各階段的量測值（PCR register）
    - 提供硬體亂數產生器
- 版本：TPM 1.2（舊）、TPM 2.0（現行標準）
- 與 BMC 關係：BMC 可透過 I2C/SPI 存取 TPM，管理平台安全狀態

---

## Secure Boot（安全開機）

- **UEFI 規範的功能**，確保每個開機階段載入的程式碼都有合法簽章
- 流程：
    ```
    UEFI firmware → 驗證 Bootloader 簽章
      └─ Bootloader → 驗證 OS kernel 簽章
           └─ 簽章不合法 → 拒絕執行，防止惡意程式在開機時植入
    ```
- 信任根（Root of Trust）存在 TPM 或 UEFI 的 Secure Boot DB 裡
- 與 BMC 關係：BMC 可管理 Secure Boot 的開啟/關閉狀態，並透過 Redfish API 暴露

### Secure Boot 防誰？

防的是**在開機流程中植入惡意程式碼的攻擊者**：

| 攻擊類型              | 說明                                             |
| --------------------- | ------------------------------------------------ |
| **Bootkit**           | 惡意程式藏在 Bootloader，OS 還沒起來就已經被控制 |
| **Rootkit（開機層）** | 竄改 kernel，讓 OS 起來後看不到惡意程式          |
| **供應鏈攻擊**        | 出廠前被替換成惡意 BIOS / Bootloader             |
| **實體存取攻擊**      | 有人拿到機器，換上惡意 USB 開機碟                |

**關鍵：**

```
一般防毒在 OS 起來之後才跑
  └─ 惡意程式在 OS 之前載入 → 防毒根本看不到

Secure Boot 在 OS 載入之前就驗簽章
  └─ 沒有合法簽章 → 直接拒絕執行
```

**防不了什麼：**

- OS 起來之後的攻擊（那是防毒 / EDR 的責任）
- 攻擊者拿到了合法簽章金鑰
- BMC 本身被攻擊（那是 BMC Secure Boot 另一條線）

---

## BIOS POST（Power-On Self Test）

- 主機上電後 BIOS/UEFI 執行的**硬體自我檢測流程**
- 順序：
    ```
    上電 → CPU 初始化 → 記憶體初始化（Memory Training）
      └─ PCIe 設備偵測 → 儲存設備偵測 → 傳遞控制權給 Bootloader
    ```
- POST 期間的錯誤會：
    - 發出嗶聲（Beep Code）
    - 亮 LED 燈號
    - 寫入 BMC 的 SEL（BIOS POST Code 可透過 IPMI 讀取）
- 透過 SOL（Serial-over-LAN）可即時看到 POST 過程輸出

---

## IBB（Initial Boot Block）

- BIOS 韌體中**最先被執行的那一塊程式碼**，存在 SPI Flash 的特定位置
- 功能：
    - CPU 重置後第一個跑的程式碼
    - 負責初始化最基本的硬體（快取、記憶體控制器）
    - 驗證後續 BIOS 程式碼的簽章（Secure Boot 信任鏈的起點）
- 安全性：IBB 本身必須是不可竄改的信任根，否則整條信任鏈都不可靠
- 與 BMC 關係：BMC 可透過 SPI 存取 Flash，能讀寫 BIOS（包含 IBB 所在區域）

---

## BMC 的三種對外介面

| 介面                    | 協定         | 說明                                             |
| ----------------------- | ------------ | ------------------------------------------------ |
| **IPMI over LAN**       | IPMI 2.0     | 最傳統，用 UDP port 623，指令列工具 `ipmitool`   |
| **Redfish（REST API）** | HTTPS / JSON | 現代標準，RESTful 風格，用 `curl` 或 Web UI 操作 |
| **SSH / Console**       | SSH          | 直接登入 BMC Linux shell，最高權限直接操作       |

> 三種都走 BMC 的獨立管理網路（不是主機的業務網路），主機 OS 掛掉也能連。
