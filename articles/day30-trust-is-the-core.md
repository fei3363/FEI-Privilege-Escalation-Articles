# Day 30：30 天後重新理解 Privilege Escalation — 技巧只是表面，信任才是核心

## 背景知識

30 天前，你可能覺得 Privilege Escalation 是一堆技巧：

```
sudo → find -exec
SUID → 讀任意檔案
cron → 改 script
Service ACL → sc config
Unquoted Path → 放 exe
```

30 天後，我希望你看到的是同一件事：

> **「我能控制什麼，而誰相信了它？」**

---

## 回顧：我們到底建了什麼

### 數據

```
20 個 Validated Scenarios
140 個檔案
11 個 Phase
32,884 行程式碼
Full Regression Testing
```

### Bug Fix 統計

| 場景 | Bug 數量 | 最複雜的 Fix |
|------|---------|-------------|
| VMware PCIe | 1 | e1000 網卡替代 |
| W01 | 1 | SDDL DC 位元 |
| L02 | 1 | gcc → cp /bin/cat |
| W05 | 6 | MSI 結構 + Custom Action + HKCU + UAC |
| W06 | 4 | -band + WriteKey + ACL + ImagePath |
| L08 | 3 | LVM symlink + udev + readlink |
| W08 | 3 | LSA API + UAC token + robocopy |

共 **19 個 bug**，每一個都是「理論上可行但實際環境不行」。

### 教訓

> **Environment is the Source of Truth。理論正確 ≠ 環境正確。**

---

## FEI Privilege Escalation Methodology — 完整版

### 八大分類

![FEI Privilege Escalation Methodology 八大分類](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day30-trust-is-the-core-diagram-01.png)


### 每類核心總結

| # | 分類 | 核心問題 | Linux | Windows |
|---|------|---------|-------|---------|
| 1 | Delegated Privilege | 授權的指令有 escape 嗎？ | sudo find -exec | — |
| 2 | Identity / Execution Context | 帳號有隱藏的高權限嗎？ | SUID, cap, group | Token, privilege |
| 3 | Executable Resolution | 要執行的名字有歧義嗎？ | PATH search | Unquoted path parsing |
| 4 | Scheduled Execution | 排程的目標可寫嗎？ | cron script | Task action |
| 5 | Trusted Configuration | 高權限讀的設定可改嗎？ | config, drop-in | Registry |
| 6 | Credential Exposure | 有暴露的 credential 嗎？ | backup, history | deploy config |
| 7 | Installation / Policy | 有危險的安裝政策嗎？ | — | AlwaysInstallElevated |
| 8 | Code Loading | 載入的 code 可控嗎？ | — | DLL Hijacking |

---

## 20 個 Lab 完整總覽

| # | ID | 漏洞類型 | 分類 | Flag |
|---|------|---------|------|------|
| 1 | L00 | (Enumeration) | — | — |
| 2 | L01 | sudo find | Delegated | `FEI{LINUX_L01_ROOT_ACCESS}` |
| 3 | L02 | SUID reader | Identity | `FEI{LINUX_L02_SUID_ROOT_ACCESS}` |
| 4 | L03 | PATH hijack | Resolution | `FEI{LINUX_L03_PATH_ROOT_ACCESS}` |
| 5 | L04 | Cron writable | Scheduled | `FEI{LINUX_L04_CRON_ROOT_ACCESS}` |
| 6 | L05 | cap_setuid | Identity | `FEI{LINUX_L05_CAPABILITIES_ROOT_ACCESS}` |
| 7 | L06 | Writable config | Trusted Config | `FEI{LINUX_L06_WEAK_FILE_PERMISSION_ROOT_ACCESS}` |
| 8 | L07 | Credential exposure | Credential | `FEI{LINUX_L07_CREDENTIAL_EXPOSURE_ROOT_ACCESS}` |
| 9 | L08 | disk group | Identity | `FEI{LINUX_L08_DANGEROUS_GROUP_ROOT_ACCESS}` |
| 10 | L09 | systemd drop-in | Trusted Config | `FEI{LINUX_L09_SYSTEMD_ROOT_ACCESS}` |
| 11 | W00 | (Enumeration) | — | — |
| 12 | W01 | Service ACL | Trusted Config | `FEI{WINDOWS_W01_SYSTEM_ACCESS}` |
| 13 | W02 | Binary ACL | Trusted Config | `FEI{WINDOWS_W02_SYSTEM_ACCESS}` |
| 14 | W03 | Unquoted Path | Resolution | `FEI{WINDOWS_W03_UNQUOTED_PATH_SYSTEM_ACCESS}` |
| 15 | W04 | Task writable | Scheduled | `FEI{WINDOWS_W04_SCHEDULED_TASK_SYSTEM_ACCESS}` |
| 16 | W05 | AlwaysInstallElevated | Policy | `FEI{WINDOWS_W05_ALWAYS_INSTALL_ELEVATED}` |
| 17 | W06 | Registry ACL | Trusted Config | `FEI{WINDOWS_W06_REGISTRY_ACL_SYSTEM_ACCESS}` |
| 18 | W07 | Credential exposure | Credential | `FEI{WINDOWS_W07_CREDENTIAL_EXPOSURE_ADMIN_ACCESS}` |
| 19 | W08 | SeBackupPrivilege | Identity | `FEI{WINDOWS_W08_SEBACKUP_PRIVILEGED_READ}` |
| 20 | W09 | DLL Hijacking | Code Loading | `FEI{WINDOWS_W09_DLL_HIJACKING_SYSTEM_ACCESS}` |

---

## Defense 匯總：每個分類的防禦原則

| 分類 | 防禦原則 |
|------|---------|
| Delegated Privilege | 最小授權：只允許必要指令，檢查 escape 可能 |
| Identity / Execution Context | 移除不必要 SUID/Capabilities/Groups/Privileges |
| Executable Resolution | 使用絕對路徑、加引號、避免 PATH 含可寫目錄 |
| Scheduled Execution | 保護排程引用的每一個檔案和 Script |
| Trusted Configuration | 高權限信任的設定必須由高權限擁有並保護 |
| Credential Exposure | 刪除 ≠ 修復。必須 Rotate + 清理 + 審計 |
| Policy | 停用不必要的危險 Policy |
| Code Loading | 使用絕對 DLL 路徑、保護 application directory |

### 共同原則

```
1. Least Privilege — 只給必要的權限
2. Defense in Depth — 多層保護
3. Audit — 定期檢查權限狀態
4. Monitor — 監控異常的權限使用
5. Verify — 修復後驗證攻擊路徑確實斷掉
```

---

## Detection 匯總：紅藍對抗

### Linux Detection Points

| 偵測點 | 工具 | 對應場景 |
|--------|------|---------|
| sudo 異常使用 | auth.log / auditd | L01, L03, L06 |
| 新 SUID binary | AIDE / Tripwire | L02 |
| getcap 變化 | auditd setcap | L05 |
| cron script 修改 | inotifywait | L04 |
| /etc/group 修改 | auditd | L08 |
| systemd drop-in 出現 | inotifywait on .service.d/ | L09 |
| 密碼字串搜尋 | 這是防禦者也該做的事 | L07 |

### Windows Detection Points

| 偵測點 | Event ID | 對應場景 |
|--------|----------|---------|
| Service config 修改 | 7040 | W01 |
| Service binary 替換 | Sysmon 11 | W02, W09 |
| 新 exe 在 unquoted path | Sysmon 11 | W03 |
| Task 修改 | 4702 | W04 |
| MSI 安裝 | 1033, 1035 | W05 |
| Registry 值修改 | 4657 (Object Access) | W06 |
| 異常 Logon | 4624 | W07 |
| Privilege 使用 | 4672 | W08 |
| DLL 載入 | Sysmon 7 | W09 |

---

## Fix Verification 匯總

![Fix Verification 匯總](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day30-trust-is-the-core-diagram-02.png)

修復任何提權漏洞後的標準驗證流程：


---

## 全系列 Quiz 回顧

### 基礎觀念

**Q**: Privilege Escalation 的定義是什麼？

### 答案從一個較低權限的 Identity，跨越 Security Boundary，取得較高權限的 Identity 或能力。不一定是 root/SYSTEM — 任何向上的權限跨越都算。

**Q**: Administrator 和 SYSTEM 的差異是什麼？

### 答案Administrator 是「使用者帳號加入 Administrators 群組」，受 UAC 限制。SYSTEM 是 OS 本身的 identity（NT AUTHORITY\SYSTEM），不受 UAC 限制，權限更高。

### 方法論

**Q**: Effective Privilege 包含哪些層次？

### 答案Identity + Groups + Privileges/Capabilities + Accessible Resources + Trust Relationships

**Q**: 為什麼「看到 Unquoted Service Path」不等於「可以提權」？

### 答案需要四個條件同時成立：1) 未加引號、2) 路徑有空格、3) 候選位置可寫、4) Service 高權限。缺一不可。

### 進階

**Q**: W05 經歷了 6 次 bug fix。最核心的教訓是什麼？

### 答案「網路文章說 AlwaysInstallElevated + msiexec 就能提權」在實際 Windows 10 Build 19045 上需要：正確的 MSI 結構（SummaryInfo + Feature/Component）、正確的 Custom Action Type（deferred + noImpersonate = VBScript Binary）、HKCU 要寫到正確的 user hive、UAC ConsentPromptBehaviorUser 不能是 auto-deny。理論 ≠ 實際環境。

**Q**: L08 用的是 `disk` group 而不是 `docker` group。為什麼？

### 答案因為 FEI Lab 的 Ubuntu 22.04 VM 沒有安裝 Docker（離線環境）。但 `disk` group 同樣危險 — 它允許存取 raw block device，繞過 filesystem ACL。Lab 必須基於實際可用的環境，不能假設某個套件存在。

---

## Exercise 最終練習

### 練習 1：寫出你自己的 PrivEsc Methodology

不要照抄本文。用你做過的 Lab 經驗，寫一份不超過一頁的 PrivEsc Methodology。

要求：
- 涵蓋至少 Linux 和 Windows
- 每個步驟有明確的「我在回答什麼問題？」
- 不能是純 Command List

### 練習 2：建立自己的 Scenario

嘗試設計一個新的 FEI Scenario（不一定要實作）：
1. 選擇一個提權 Root Cause
2. 設計 setup/verify/reset
3. 寫 student.md（只給提示）
4. 寫 instructor.md（完整解答）
5. 想清楚：你的 Scenario 和既有 20 個有什麼不同？

### 練習 3：Review

回去看 Day 04 的 L00 Enumeration。和你 Day 29 的 Decision Tree 比較。你現在的 Enumeration 流程和第一天有什麼不同？

---

## 統一四問框架

![統一四問框架](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day30-trust-is-the-core-diagram-03.png)


這四個問題涵蓋了全部 20 個 Lab：

| Scenario | 我控制什麼 | 誰信任它 | 什麼權限 | 怎麼觸發 |
|----------|-----------|---------|---------|---------|
| L01 | find 的參數 | sudo | root | sudo find -exec |
| L02 | SUID binary 的輸入 | kernel (EUID) | root | 直接執行 |
| L03 | PATH 中的 fei-backup | maintenance script | root | sudo maintenance |
| L04 | cron 的 script | cron daemon | root | 等待排程 |
| L05 | python3 的 code | cap_setuid | root UID | os.setuid(0) |
| L06 | config TARGET_FILE | root runner | root | sudo runner |
| L07 | 找到 root password | su | root | su - |
| L08 | (已經有 disk group) | kernel device ACL | raw read | debugfs |
| L09 | drop-in override | systemd | root | daemon-reload + restart |
| W01 | Service binPath | SCM | SYSTEM | sc config + restart |
| W02 | Service exe 檔案 | SCM loader | SYSTEM | 替換 + restart |
| W03 | 中間候選 exe | CreateProcess | SYSTEM | 放置 + restart |
| W04 | Task action script | Task Scheduler | SYSTEM | 等待排程 |
| W05 | 自訂 MSI | Windows Installer | Elevated | msiexec /i |
| W06 | Registry ActionPath | SYSTEM service | SYSTEM | reg add + restart |
| W07 | 找到 admin password | runas | Local Admin | runas /user: |
| W08 | (已經有 SeBackup) | NTFS bypass | Backup read | robocopy /B |
| W09 | DLL 在 app dir | SYSTEM loader | SYSTEM | 放置 + restart |

---

## 最後

30 天前你可能覺得 Privilege Escalation 是一堆 Exploit 技巧。

30 天後，我希望你記住的不是 `sudo find -exec` 或 `sc config binPath=`。

我希望你記住的是：

> 每一個提權，本質上都是在回答：
>
> **「我能控制什麼，而誰相信了它？」**

當你看到一台陌生的主機，你不再需要背誦 Checklist。你需要的只是問：

```
Who am I?
What can I control?
Who trusts what I control?
Can I trigger that trust?
```

如果四個答案指向同一個方向——你就找到了提權路徑。

技巧會過時。漏洞會被修。工具會更新。

但這個問題不會變：

> **誰控制什麼，以及誰相信了它。**

---

## 今天真正要記住的 3 件事

1. **20 個 Lab、8 大分類，全部可以用四個問題統一**：What can I control? Who trusts it? What privilege? Can I trigger?

2. **理論 ≠ 環境。19 個 bug 每一個都證明了「看起來可以」和「真的可以」之間的差距**

3. **Privilege Escalation 的核心不是技巧名稱，是信任關係**

---

## 給自己的問題

> 下一次你拿到一台陌生主機的 Low Privilege Shell，你的第一個念頭是什麼？
>
> 是「跑 LinPEAS / winPEAS」嗎？
>
> 還是「讓我先搞清楚——我到底能控制什麼？」

---

## 系列完結

```
========================================
 FEI PrivEsc Lab
 30 Days Complete
========================================

 20 Scenarios | 140 Files | 19 Bugs Found & Fixed
 
 What can I control?
 Who trusts it?
 What privilege do they have?
 Can I trigger that trust?

 → Privilege Escalation

========================================
```

> 感謝你陪我走完這 30 天。
>
> Lab 是開源的：https://github.com/fei3363/FEI-Privilege-Escalation-Lab
>
> 如果這系列對你有幫助，歡迎 star、fork、或建立你自己的 Scenario。
>
> 我們 Privilege Escalation 再見。
