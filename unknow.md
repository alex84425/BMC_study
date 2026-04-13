# BMC 還不知道的主題

## 面試題清單

| #   | 題目                                                            | 狀態 |
| --- | --------------------------------------------------------------- | ---- |
| 1   | BMC 跟 BIOS 怎麼溝通？KCS / BT / SSIF 是什麼？                  | ✅   |
| 2   | Redfish 的資料結構長什麼樣？URI 怎麼設計？                      | ✅   |
| 3   | SNMP 是什麼，跟 IPMI/Redfish 差在哪？                           | ✅   |
| 4   | BMC Secure Boot 跟主機 Secure Boot 差在哪？                     | ✅   |
| 5   | UEFI 的 PK / KEK / DB / DBX 是什麼？                            | ✅   |
| 6   | I2C / SPI / LPC / eSPI 這些硬體介面分別用在哪？                 | ✅   |
| 7   | AST2600 是什麼？為什麼幾乎是業界標準？                          | ✅   |
| 8   | sdbusplus 是什麼？C++ 怎麼操作 D-Bus？                          | ✅   |
| 9   | entity-manager 和 JSON 設定檔怎麼描述硬體拓撲？                 | ✅   |
| 10  | OpenBMC 的 CI/CD 怎麼運作？PR 怎麼合入？                        | ✅   |
| 11  | BMC IP 設錯連不上，你有哪些方法救回來？                         | ✅   |
| 12  | ROT（Root of Trust）是什麼？跟 TPM / IBB / Secure Boot 的關係？ | ⬜   |
| 13  | MCTP 是什麼？NVMe/PCIe 設備怎麼跟 BMC 通訊？                    | ⬜   |
| 14  | PLDM 是什麼？跟 IPMI 差在哪？                                   | ⬜   |
| 15  | RAS 在 BMC 扮演什麼角色？                                       | ⬜   |
| 16  | 怎麼 debug 一個 D-Bus service 掛掉的問題？                      | ⬜   |
| 17  | 怎麼新增一個新的感測器到 OpenBMC？走哪些步驟？                  | ⬜   |
| 18  | phosphor-logging 怎麼寫自訂 event？                             | ⬜   |
| 19  | BMC 的攻擊面有哪些？怎麼加固（hardening）？                     | ⬜   |
| 20  | Redfish 用什麼認證方式？Session token 怎麼運作？                | ⬜   |
| 21  | 什麼時候該用 IPMI，什麼時候用 Redfish？                         | ⬜   |
| 22  | 主機 OS 跟 BMC 要共享資料，怎麼設計？                           | ⬜   |

---

IPMI 是協定，實際走的物理通道有三種：

| 介面     | 全名                      | 特性                              |
| -------- | ------------------------- | --------------------------------- |
| **KCS**  | Keyboard Controller Style | 最常見，走 I/O port，簡單但速度慢 |
| **BT**   | Block Transfer            | 速度較快，支援佇列                |
| **SSIF** | SMBus System Interface    | 走 I2C/SMBus                      |

KCS 最普遍，幾乎所有伺服器都用：

```
主機 CPU（BIOS/OS）
  └─ 透過 I/O port 寫資料到 KCS 暫存器
       └─ BMC 讀取並解析成 IPMI 指令 → 執行 → 回傳
```

```bash
ipmitool -I open mc info   # 走 KCS（本機，不需要網路）
ipmitool -I lan  mc info   # 走網路
```

---

## 2. Redfish 資料結構與 URI 設計

RESTful 風格，URI 是樹狀結構：

```
/redfish/v1/
  ├─ Systems/                  # 主機系統
  │    └─ system/
  │         ├─ Processors/     # CPU 資訊
  │         └─ Memory/         # 記憶體資訊
  ├─ Chassis/                  # 機箱/硬體
  │    └─ chassis/
  │         └─ Thermal/        # 溫度/風扇
  └─ Managers/                 # BMC 本身
       └─ bmc/
            └─ LogServices/    # Log
```

---

## 3. SNMP 是什麼，跟 IPMI/Redfish 差在哪

|      | SNMP             | IPMI                | Redfish        |
| ---- | ---------------- | ------------------- | -------------- |
| 年代 | 1988（最舊）     | 1998                | 2015（最新）   |
| 用途 | 網路設備監控為主 | 伺服器帶外管理      | 伺服器帶外管理 |
| 格式 | OID 數字編碼     | 二進位指令          | JSON / REST    |
| 現況 | 監控系統還在用   | 逐漸被 Redfish 取代 | 現行標準       |

---

## 4. BMC Secure Boot vs 主機 Secure Boot

|            | 主機 Secure Boot | BMC Secure Boot        |
| ---------- | ---------------- | ---------------------- |
| 保護什麼   | 主機 OS 開機程式 | BMC 韌體本身           |
| 在哪裡設定 | UEFI BIOS 設定   | BMC 內部，通常出廠鎖定 |
| 能關掉嗎   | ✅ 可以          | ❌ 通常不行            |

---

## 5. UEFI 金鑰資料庫

| 名稱    | 全名                         | 說明                         |
| ------- | ---------------------------- | ---------------------------- |
| **PK**  | Platform Key                 | 最高層金鑰，廠商持有，簽 KEK |
| **KEK** | Key Exchange Key             | 可以更新 DB/DBX 的金鑰       |
| **DB**  | Signature Database           | 允許執行的簽章白名單         |
| **DBX** | Forbidden Signature Database | 黑名單，被撤銷的簽章         |

```
PK → 簽 KEK → KEK 可以更新 DB/DBX → DB 決定哪些 Bootloader 可以跑
```

---

## 6. 硬體介面各用在哪

| 介面     | 用途                             |
| -------- | -------------------------------- |
| **I2C**  | 讀溫度/電壓感測器、控制 LED      |
| **SPI**  | 存取 BIOS/BMC Flash 韌體         |
| **LPC**  | 傳 POST Code、KCS IPMI（舊平台） |
| **eSPI** | LPC 的新版本，更快更省針腳       |
| **UART** | Serial console、除錯輸出         |
| **GPIO** | 電源按鈕、主機狀態信號           |

---

## 7. AST2600 是什麼

Aspeed 公司出的 BMC SoC，幾乎是業界標準：

- ARM Cortex-A7，1.2GHz
- 內建網路、USB、PCIe、I2C、SPI 等介面
- OpenBMC 官方支援，NVIDIA、Intel、Meta 的伺服器都在用
- 便宜、文件齊全、社群大

---

## 8. sdbusplus（C++ 操作 D-Bus）

OpenBMC 自己封裝的 C++ D-Bus 函式庫，比原生 libdbus 好用：

```cpp
auto bus = sdbusplus::bus::new_default();
auto method = bus.new_method_call(
    "xyz.openbmc_project.State.Host",
    "/xyz/openbmc_project/state/host0",
    "org.freedesktop.DBus.Properties",
    "Get");
method.append("xyz.openbmc_project.State.Host", "CurrentHostState");
auto reply = bus.call(method);
```

---

## 9. entity-manager 與硬體拓撲 JSON

`entity-manager` 讀取 JSON 設定檔，描述板子上有哪些硬體：

```json
{
    "Name": "CPU Temp Sensor",
    "Type": "TMP75",
    "Bus": 3,
    "Address": "0x48",
    "Thresholds": [
        { "Direction": "greater than", "Severity": 1, "Value": 85.0 }
    ]
}
```

entity-manager 讀到後，自動在 D-Bus 上建立對應物件，其他服務就能使用。

---

## 10. OpenBMC CI/CD 與 PR 流程

```
你 fork openbmc repo
  └─ 修改 → 推到自己的 branch
       └─ 開 PR 到上游
            └─ 自動跑 CI（bitbake 編譯、單元測試）
                 └─ Reviewer 審查（至少要 2 個 +2）
                      └─ 用 Gerrit 合入（不是 GitHub merge）
```

> OpenBMC 用 **Gerrit** 做 code review，不是 GitHub PR，要特別注意。
