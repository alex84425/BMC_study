# OpenBMC 自學筆記

## BMC 基礎概念

- BMC（Baseboard Management Controller，基板管理控制器）是伺服器主板上的獨立晶片（通常是 ARM），負責帶外管理（Out-of-Band Management）
- 即使主機斷電，BMC 仍可透過 IPMI / Redfish / SNMP 協定與外界通訊

### 為什麼主機斷電 BMC 還能運作？

關鍵在於**電源架構**：伺服器插上 AC 電源線後，電供（PSU）會輸出兩種電：

| 電源 | 說明 |
|------|------|
| **主電（Main Power）** | 主機 CPU、記憶體、硬碟用，按下電源鍵才供電 |
| **待機電（Standby Power，+5VSB）** | 只要插著電源線就一直有電，**BMC 就靠這條活著** |

BMC 晶片直接接在待機電上，所以：
- 主機關機 → CPU 沒電，但 BMC 有待機電，繼續跑
- 完全拔掉電源線 → BMC 才真的斷電

這就是「帶外管理（Out-of-Band）」的核心概念：管理通道獨立於主機系統之外，不受 OS 狀態影響。
- 功能：遠端開關機、KVM、Serial-over-LAN、感測器監控、韌體更新

## OpenBMC 是什麼

- Linux Foundation 的開源 BMC 韌體專案，基於 **Yocto Project** 建置
- 用 **Python/C++** 撰寫服務，透過 **D-Bus** 做元件間通訊
- 支援 **Redfish**（RESTful API）與 IPMI 協定
- GitHub: https://github.com/openbmc/openbmc

### D-Bus 是啥？

**不是晶片**，是一種 **行程間通訊（IPC）機制**，跑在 Linux 軟體層。

可以把它想像成 BMC 內部各服務之間的「對講機頻道」：

- 感測器服務、電源管理服務、Redfish API 服務各自獨立跑
- 它們透過 D-Bus 互相傳遞訊息、呼叫彼此的功能

**學會後能做什麼？**

- 新增自訂感測器或硬體監控功能
- 修改開關機流程邏輯
- 寫新的 D-Bus 服務讓 Redfish API 暴露新的端點
- 偵錯：用 `busctl` 指令直接查看 BMC 上所有服務狀態

```bash
busctl list          # 列出所有 D-Bus 服務
busctl introspect xyz.openbmc_project.State.Host /xyz/openbmc_project/state/host0
```

---

### IPMI 協定 需要知道啥？

**IPMI（Intelligent Platform Management Interface）** 是比 Redfish 更古老的帶外管理標準，很多舊設備還在用。

| 概念         | 說明                                             |
| ------------ | ------------------------------------------------ |
| Channel      | 通訊管道，如 LAN、Serial                         |
| IPMI Command | 用數字編碼的指令，如 `0x06 0x01` = Get Device ID |
| RAW command  | 直接送原始 IPMI 指令                             |
| ipmitool     | 最常用的 IPMI 指令列工具                         |

**你需要會的基本操作：**

```bash
# 安裝 ipmitool
apt install ipmitool

# 查詢 BMC 資訊
ipmitool -I lan -H <BMC-IP> -U admin -P password mc info

# 查看感測器
ipmitool -I lan -H <BMC-IP> -U admin -P password sdr list

# 遠端重開機
ipmitool -I lan -H <BMC-IP> -U admin -P password chassis power reset
```

**學習重點優先順序：**

1. 理解 LAN channel 如何連線
2. `chassis`、`sdr`、`mc` 三個子指令
3. 再深入看 IPMI 2.0 規格的 Message Format（需要時再看即可）

## 自學路線

### 階段一：背景知識

- Linux 基礎（檔案系統、systemd、shell）
- ARM 嵌入式系統概念
- 網路協定：HTTP/REST、IPMI 基礎

### 階段二：工具鏈

- **Yocto Project** — 學會用 `bitbake` 建置映像檔
    - 官方文件: https://docs.yoctoproject.org
- **QEMU** — 模擬 BMC 硬體（不需要實體機）

### 階段三：OpenBMC 實作

```bash
# 下載並建置 OpenBMC (以 QEMU 為目標)
git clone https://github.com/openbmc/openbmc
cd openbmc
. setup romulus  # 選擇一個平台
bitbake obmc-phosphor-image
```

- 跑起來後用 Redfish API 操作：`https://<BMC-IP>/redfish/v1/`

### 階段四：深入開發

- 研究 **phosphor-dbus-interfaces** — 了解 D-Bus 架構
- 閱讀 **phosphor-host-ipmid** — IPMI 實作
- 挑一個感興趣的子系統貢獻 PR

## 學習資源

| 資源              | 連結                                               |
| ----------------- | -------------------------------------------------- |
| OpenBMC 官方文件  | https://github.com/openbmc/docs                    |
| Yocto 快速入門    | https://docs.yoctoproject.org/brief-yoctoprojectqs |
| DMTF Redfish 規格 | https://www.dmtf.org/standards/redfish             |
| OpenBMC Discord   | https://discord.gg/69Km47zH98                      |

## 實驗環境

不需要真實硬體，用 **QEMU** 模擬 `romulus` 或 `witherspoon` 平台即可開始實驗。
