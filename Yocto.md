# Yocto 筆記

## Yocto 是什麼

**從零開始客製化建置一個 Linux 系統映像檔的工具鏈**

```
你定義想要什麼
  └─ 哪個 kernel 版本
  └─ 裝哪些套件（phosphor-hwmon、bmcweb...）
  └─ 針對哪個 ARM 晶片編譯
       │
       ▼
  Yocto（bitbake）
       │
       ▼
  輸出一個可以燒進 BMC Flash 的 .img 映像檔
```

---

## 跟一般 Linux 發行版的差別

|          | Ubuntu / Debian      | Yocto                       |
| -------- | -------------------- | --------------------------- |
| 給誰用   | 通用桌機/伺服器      | 嵌入式/客製化硬體           |
| 大小     | 幾 GB                | 可以壓到幾十 MB             |
| 套件來源 | apt 線上裝           | 全部從 source code 編譯進去 |
| 彈性     | 低（包什麼就是什麼） | 高（自己決定每一個細節）    |

---

## 核心概念

| 概念                 | 說明                                           |
| -------------------- | ---------------------------------------------- |
| **Recipe（.bb 檔）** | 告訴 Yocto 怎麼編譯一個套件，像食譜            |
| **Layer**            | 一組 Recipe 的集合，OpenBMC 本身就是一個 Layer |
| **bitbake**          | Yocto 的建置指令，相當於 `make`                |
| **Image**            | 最終輸出的 .img 映像檔，燒進 BMC Flash         |

---

## Recipe（.bb）用什麼語言寫

BitBake 語法 = **Shell + Python 混合**，不是 C++ 也不是純 Python：

```bitbake
SUMMARY = "My BMC service"
LICENSE = "Apache-2.0"

SRC_URI = "git://github.com/xxx/my-service.git"

inherit systemd

do_install() {
    install -m 0755 my_service.py ${D}/usr/bin/   # shell 語法
}

python do_check_version() {                        # Python 語法
    version = d.getVar('PV')
    bb.note("version is " + version)
}
```

---

## 各層用什麼語言

| 層                     | 語言                           |
| ---------------------- | ------------------------------ |
| Yocto Recipe（.bb 檔） | BitBake 語法（Shell + Python） |
| 你寫的 BMC 服務        | Python 或 C++（自己決定）      |
| phosphor 現有服務      | C++                            |
| kernel / U-Boot        | C                              |

> Yocto 語法 和 服務邏輯（C++/Python）是不同的事，可以分開學。

---

## 基本操作

```bash
# 下載 OpenBMC
git clone https://github.com/openbmc/openbmc
cd openbmc

# 選擇目標平台（romulus 是 QEMU 模擬用）
. setup romulus

# 建置映像檔（第一次要幾小時）
bitbake obmc-phosphor-image

# 用 QEMU 跑起來測試
runqemu
```

---

## 為什麼學習曲線陡

- 第一次全量編譯要 1~3 小時
- 錯誤訊息不直觀，要回去看 build log
- 加一個新套件要寫 Recipe，有固定格式要學
- 不同 Layer 版本相容性問題複雜
