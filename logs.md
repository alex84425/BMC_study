# BMC 都看哪些 Log？

## Log 總覽

| Log 類型                    | 層級       | 記錄內容                                             |
| --------------------------- | ---------- | ---------------------------------------------------- |
| **SEL（System Event Log）** | 硬體層     | 開關機、溫度過高、風扇故障、電源異常、CPU/記憶體錯誤 |
| **Redfish Event Log**       | BMC OS 層  | BMC 服務啟動/崩潰、API 呼叫記錄                      |
| **journalctl**              | BMC 軟體層 | BMC 內部 Linux systemd 服務運作狀態                  |
| **SOL（Serial-over-LAN）**  | 主機啟動層 | BIOS/UEFI → Bootloader → OS 開機輸出                 |

## 啟動順序與對應 Log

```
AC 插上
  └─ BMC 啟動        → journalctl / Redfish Event Log 開始記錄
       └─ 硬體初始化  → SEL 記錄溫度、電源、硬體事件
            └─ 主機開機 → SOL 看到 BIOS → OS 開機畫面
```

> SOL 是唯一能看到**主機**開機過程的管道，其他三個都是 BMC 自己的 log。

---

## 各 Log 怎麼看

### 1. SEL — 硬體事件

```bash
# 列出所有事件
ipmitool -I lan -H <BMC-IP> -U admin -P password sel list

# 清除 SEL
ipmitool -I lan -H <BMC-IP> -U admin -P password sel clear
```

### 2. Redfish Event Log

```bash
curl -k -u admin:password \
  https://<BMC-IP>/redfish/v1/Managers/bmc/LogServices/Journal/Entries
```

### 3. journalctl（SSH 進 BMC）

```bash
ssh root@<BMC-IP>

journalctl -xe                          # 看所有系統 log
journalctl -u phosphor-ipmi-host        # 看特定服務
journalctl --since "10 minutes ago"     # 看最近 10 分鐘
```

### 4. SOL — 主機開機畫面

```bash
# 連上主機序列輸出（看 BIOS/OS 開機）
ipmitool -I lanplus -H <BMC-IP> -U admin -P password sol activate

# 離開按 Ctrl+]
```

---

## 各事件在哪些 Log 出現

| 事件                            | SEL               | Redfish Event Log   | journalctl                      | SOL                  |
| ------------------------------- | ----------------- | ------------------- | ------------------------------- | -------------------- |
| **溫度過高**                    | ✅ 硬體感測器觸發 | ✅ 服務偵測到後回報 | ✅ 服務印出 warning             | ❌                   |
| **風扇故障**                    | ✅ 硬體事件       | ✅                  | ✅                              | ❌                   |
| **電源異常（PSU 故障）**        | ✅                | ✅                  | ✅                              | ❌                   |
| **CPU 過熱/錯誤**               | ✅                | ✅                  | ✅                              | ❌                   |
| **記憶體 ECC 錯誤**             | ✅                | ✅                  | ✅                              | ❌                   |
| **硬碟插拔 / NVMe 異常**        | ✅（部分平台）    | ✅                  | ✅                              | ❌                   |
| **主機被遠端關機**              | ✅                | ✅                  | ✅                              | ❌                   |
| **主機異常重啟（crash）**       | ✅                | ✅                  | ✅                              | ✅ 可看 kernel panic |
| **主機開不起來（卡 BIOS）**     | ❌                | ❌                  | ❌                              | ✅ 唯一能看到的地方  |
| **BMC 服務崩潰**                | ❌                | ✅                  | ✅                              | ❌                   |
| **BMC 重啟**                    | ❌                | ✅                  | ✅（重啟後 journal 從頭）       | ❌                   |
| **有人登入 BMC**                | ❌                | ✅                  | ✅                              | ❌                   |
| **Redfish API 被呼叫**          | ❌                | ✅                  | ✅                              | ❌                   |
| **BMC 網路介面 up/down**        | ❌                | ✅                  | ✅（networkd/systemd-networkd） | ❌                   |
| **BMC IP 設定變更**             | ❌                | ✅                  | ✅                              | ❌                   |
| **網路封包異常（丟包/斷線）**   | ❌                | ❌                  | ✅（需看 networkd log）         | ❌                   |
| **韌體更新（Firmware Update）** | ❌                | ✅                  | ✅                              | ❌                   |
| **BIOS/UEFI 更新**              | ✅（部分）        | ✅                  | ✅                              | ✅ 可看更新畫面輸出  |

---

## 除錯情境對照

| 問題                    | 先看哪個 Log                   |
| ----------------------- | ------------------------------ |
| 主機突然關機            | SEL（找電源/溫度異常事件）     |
| 主機開不起來            | SOL（看 BIOS 卡在哪）          |
| 主機 kernel panic       | SOL（看 panic 訊息）           |
| BMC 服務異常 / API 回錯 | journalctl                     |
| 想知道誰登入過 BMC      | Redfish Event Log              |
| BMC 連不上網路          | journalctl（看 networkd）      |
| 硬碟/記憶體硬體錯誤     | SEL                            |
| 韌體更新失敗            | Redfish Event Log + journalctl |
