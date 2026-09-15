# Day 01：提權到底在提什麼？從 Low Privilege 到 Root / SYSTEM

> **📋 前置需求**
>
> 跟著本系列實作需要：
> - 一台電腦（Windows 10/11 Pro，至少 16 GB RAM，140 GB 可用空間）
> - VMware Workstation 17+（[免費下載 Player](https://www.vmware.com/products/workstation-player.html) 或 [Pro 試用](https://www.vmware.com/products/workstation-pro.html)）
> - Ubuntu Server 22.04 LTS ISO（[官方下載](https://ubuntu.com/download/server)）
> - Windows 10 ISO（[Microsoft Media Creation Tool](https://www.microsoft.com/software-download/windows10)）
> - 基本 terminal 操作能力（會開 cmd / PowerShell、知道 cd / ls）
>
> Day 03 有完整建置指南。

---

## 背景知識

### 提權在滲透測試中的位置

如果你熟悉 MITRE ATT&CK 或任何滲透測試方法論，你一定看過一個共同的階段劃分：

```
偵察 → 初始存取 → 建立立足點 → 【提權】→ 橫向移動 → 資料存取 → 目標達成
```

「初始存取」通常給你的是一個低權限的 shell。你可能透過 Web 漏洞拿到了 `www-data`，或者透過釣魚拿到了一個 Standard User 的 cmd.exe。

但這個 shell 能做的事很有限：你不能讀取敏感設定、不能安裝後門、不能存取其他使用者的資料。

**Privilege Escalation（提權）** 就是從這個「能做一點事，但不夠」的狀態，跨越到「能做更多事」的狀態。

### 什麼是 Privilege？

在作業系統的世界裡，「Privilege（權限）」不是一個抽象概念。它是 OS 核心在每一次存取檢查時真正查看的東西：

- **Linux**：你的 Process 有一個 UID（User ID）。每次你嘗試讀一個檔案，Kernel 會比對你的 UID 和檔案的 owner/group/other 權限。UID 0（root）幾乎可以跳過所有檢查。

- **Windows**：你的 Process 有一個 Access Token。每次你嘗試開啟一個物件（檔案、Registry key、Service），Security Reference Monitor 會比對你 Token 裡的 SID、Groups、Privileges 和物件的 DACL。

所以「提權」的本質是：**改變你 Process 的有效身分或能力，讓它通過原本會被拒絕的存取檢查。**

### Vertical vs Horizontal

提權有兩個方向：

| 類型 | 說明 | 範例 |
|------|------|------|
| **Vertical（垂直）** | 從低權限跳到更高權限 | Standard User → root / SYSTEM |
| **Horizontal（水平）** | 同權限但切換到另一個身分 | User A → User B（同層級但存取不同資源）|

本系列專注在 **Vertical Privilege Escalation**：從 `fei-student` 這個 Standard User，跨越到 root（Linux）或 SYSTEM / Administrator（Windows）。

---

## OS 原理：兩種世界的權限模型

### Linux：Discretionary Access Control (DAC)

Linux 繼承自 Unix 的 DAC 模型，核心概念是：

```
每個 Process 有身分（UID/GID）
每個資源有存取控制清單（owner/group/other permissions）
Kernel 在每次存取時比對
```

完整的 Process Identity 包含：

| 屬性 | 說明 |
|------|------|
| **Real UID (RUID)** | 啟動 Process 的使用者 |
| **Effective UID (EUID)** | 用於存取檢查的 UID |
| **Saved UID (SUID)** | 用於在 RUID 和 EUID 之間切換 |
| **Real GID** | 群組身分 |
| **Effective GID** | 用於存取檢查的 GID |
| **Supplementary Groups** | 額外群組成員資格 |
| **Capabilities** | 細粒度的特權能力（Linux 2.6.26+）|

UID 0 是 root。擁有 UID 0 的 Process 可以：
- 讀寫任何檔案（繞過 DAC）
- 載入/卸載 Kernel Module
- 修改任何 Process
- 綁定任何 Port
- 修改任何系統設定

但「root」不只是一個使用者名稱——它是一種**身分狀態**。你可以透過各種方式讓你的 EUID 變成 0，而不一定需要知道 root 的密碼。

### Windows：Security Reference Monitor

Windows 的存取控制比 Linux 更複雜，核心是 **Access Token** 和 **Security Descriptor**：

```
每個 Process 有 Access Token
每個 Object 有 Security Descriptor（含 DACL）
Security Reference Monitor 在每次存取時比對
```

Access Token 包含：

| 屬性 | 說明 |
|------|------|
| **User SID** | 使用者安全識別碼 |
| **Group SIDs** | 所屬群組（Administrators、Users 等）|
| **Privileges** | 特殊能力（SeBackupPrivilege、SeImpersonatePrivilege 等）|
| **Integrity Level** | 完整性等級（Low / Medium / High / System）|
| **Logon SID** | 目前登入 Session |

Windows 有三個常見的「高權限身分」：

| 身分 | SID | 說明 |
|------|-----|------|
| **Administrator** | S-1-5-21-...-500 | 本機管理員帳號 |
| **Administrators Group** | S-1-5-32-544 | 管理員群組 |
| **SYSTEM (LocalSystem)** | S-1-5-18 | OS 本身的身分，比 Administrator 更高 |

> ⚠️ 重要：**Administrator ≠ SYSTEM**。Administrator 是使用者帳號，SYSTEM 是 OS 核心服務的身分。很多 Windows Service 以 SYSTEM 執行，這就是為什麼控制 Service 可能等於控制 SYSTEM。

---

## Lab：FEI PrivEsc 訓練環境

這個系列使用的所有 Lab 都是真實建立並驗證過的。

### 環境架構

![環境架構](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day01-what-is-privilege-escalation-diagram-01.png)



### 實際環境狀態

以下是 `fei-student` 在 Linux VM 上的實際身分：

```bash
fei-student@fei-privesc-linux:~$ whoami
fei-student

fei-student@fei-privesc-linux:~$ id
uid=1001(fei-student) gid=1001(fei-student) groups=1001(fei-student)
```

在 Windows VM 上：

```cmd
C:\Users\fei-student> whoami
fei-priv-win\fei-student

C:\Users\fei-student> whoami /priv
PRIVILEGES INFORMATION
SeShutdownPrivilege           關閉系統           已停用
SeChangeNotifyPrivilege       略過周遊檢查       已啟用
SeUndockPrivilege             從擴充座移除電腦   已停用
SeIncreaseWorkingSetPrivilege 增加處理程序工作組 已停用
SeTimeZonePrivilege           變更時區           已停用
```

兩邊都是低權限 Standard User。沒有 sudo、不在 Administrators、沒有危險 Privilege。

> 💡 **想跟著做？** Day 03 有完整的建置指南與 GitHub repo。

### 20 個 Scenario 預覽

本系列涵蓋 20 個 VALIDATED 提權場景：

| # | Linux | Windows | 主題 |
|---|-------|---------|------|
| 00 | L00 Enumeration | W00 Enumeration | 初始列舉 |
| 01 | L01 sudo | W01 Service ACL | 權限委派 |
| 02 | L02 SUID | W02 Service Binary | 身分切換 |
| 03 | L03 PATH | W03 Unquoted Path | 執行檔解析 |
| 04 | L04 Cron | W04 Scheduled Task | 排程執行 |
| 05 | L05 Capabilities | W05 AlwaysInstallElevated | 特殊權限 |
| 06 | L06 Weak File | W06 Registry ACL | 信任資源 |
| 07 | L07 Credentials | W07 Credentials | 憑證暴露 |
| 08 | L08 Dangerous Group | W08 Token Privileges | OS 權限模型 |
| 09 | L09 systemd | W09 DLL Hijacking | 服務/載入 |

每一個都經過：setup → verify → manual solve → flag → reset → regression。

---

## Attack Reasoning：提權的核心思維

![Attack Reasoning：提權的核心思維](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day01-what-is-privilege-escalation-diagram-02.png)


從現在開始，每次面對一台陌生的主機，請先問自己三個問題：


這三個問題會貫穿整個系列。

每一種提權技巧——不管是 sudo misconfiguration、SUID binary、writable cron script、weak Service ACL——本質上都是在回答：

> **「我目前能控制的東西，有沒有被比我更高權限的主體信任？」**

如果答案是「有」，並且你能觸發那個信任關係，就可能跨越 Security Boundary。

---

## Actual Output：Security Boundary 的真實面貌

讓我們看一個具體的例子。`fei-student` 嘗試讀取 root 的 flag：

```bash
fei-student@fei-privesc-linux:~$ cat /root/fei-flag.txt
cat: /root/fei-flag.txt: Permission denied
```

這就是 Security Boundary。Kernel 看到：
- 你的 EUID = 1001（fei-student）
- 檔案的 owner = 0（root）
- 權限 = 0600（只有 owner 可讀寫）
- 結果：**Access Denied**

提權就是想辦法讓這個檢查的結果從 Denied 變成 Allowed。

---

## False Positive：不要把這些當成提權

初學者常見的誤判：

| 看到的現象 | 為什麼不一定是漏洞 |
|-----------|-------------------|
| 有一個 shell | shell ≠ 有權限。低權限 shell 什麼敏感東西都讀不到 |
| `sudo` 指令存在 | `sudo` 存在不代表你有 sudo 權限。要看 `sudo -l` |
| 有 SUID binary | 系統本來就有很多 SUID binary（sudo、passwd、mount）。大多數是正常的 |
| Service 以 SYSTEM 執行 | 99% 的 Service 都以 SYSTEM 執行。「SYSTEM Service 存在」本身不是漏洞 |
| whoami 顯示 Administrator | 有 UAC 的話，你的 token 可能是 filtered 的（Medium Integrity）|

> 重點：**有某個設定存在 ≠ 可以被利用。** 必須同時滿足「可控制」+「被信任」+「可觸發」。

---

## Edge Case：不同 OS 的「最高權限」不同

| OS | 最高權限 | 說明 |
|----|---------|------|
| Linux | root (UID 0) | 幾乎可以做任何事 |
| Windows | SYSTEM (S-1-5-18) | OS 核心身分 |
| Windows | TrustedInstaller | 比 SYSTEM 更高（可以修改系統檔案）|
| macOS | root + SIP disabled | SIP 啟用時即使 root 也不能改某些東西 |
| Container | Container root | 可能被 namespace/cgroup 限制 |

在 Windows 上，「Administrator」和「SYSTEM」是不同的東西：
- Administrator 是使用者帳號，受 UAC 限制
- SYSTEM 是 OS 服務身分，不受 UAC 限制

在 Linux container 中，「root」可能不是真正的 root——它可能被 user namespace 限制在容器內。

---

## Troubleshooting：Lab 環境常見問題

| 問題 | 原因 | 解法 |
|------|------|------|
| VM 啟動失敗「No PCIe slot」 | Hyper-V 共存時 vmxnet3 無法取得 PCIe slot | 改用 `e1000` 網卡 |
| Windows 安裝找不到磁碟 | `lsilogic` SCSI 控制器無驅動 | 改用 `lsisas1068` |
| Linux VM ping 不通 | netplan 靜態 IP 未正確設定 | 確認介面名稱（ens33/ens97）|
| fei-student 密碼不對 | Training password 和 labadmin 不同 | fei-student: `FEI-Student-2026!` |

---

## Defense：縱深防禦的觀點

防禦提權不是靠單一機制，而是多層防護：

```
Layer 1: Least Privilege
         只給必要的權限

Layer 2: Configuration Hardening
         正確設定 sudoers、ACL、Service 權限

Layer 3: Monitoring & Detection
         監控異常的權限使用

Layer 4: Containment
         即使提權成功，限制影響範圍（namespace、AppArmor、Integrity Level）
```

### Detect

- 監控 `sudo` 日誌（Linux `/var/log/auth.log`）
- 監控 Service 建立/修改（Windows Event ID 7045）
- 監控異常 Process（高權限 Process 的 Parent 是低權限 Process）

### Prevent

- 最小權限原則（不給不必要的 sudo、不把使用者加入不必要的群組）
- 正確的檔案/Registry 權限
- 停用不必要的 Windows Installer Policy

### Verify

修復後如何確認：

```bash
# Linux：確認 fei-student 沒有不必要的 sudo 權限
sudo -l -U fei-student

# Windows：確認 fei-student 不在 Administrators
net localgroup Administrators
```

---

## Detection：藍隊視角

提權活動的常見偵測指標：

| 指標 | Linux | Windows |
|------|-------|---------|
| 異常 sudo 使用 | /var/log/auth.log | N/A |
| Service 建立/修改 | systemctl 日誌 | Event ID 7045, 4697 |
| 異常 SUID/Capability | auditd | N/A |
| 新帳號建立 | /var/log/auth.log | Event ID 4720 |
| Token 操作 | N/A | Event ID 4672 (Special Logon) |
| 排程任務修改 | Cron 日誌 | Event ID 4698 |

---

## Fix Verification：修好後怎麼確認

| 檢查項目 | Linux 指令 | Windows 指令 |
|----------|-----------|-------------|
| 使用者權限 | `id <user>`, `sudo -l -U <user>` | `whoami /all`, `net localgroup Administrators` |
| SUID 檔案 | `find / -perm -4000 -type f` | N/A |
| Service 權限 | `systemctl show <svc>` | `sc sdshow <svc>` |
| 排程 | `ls /etc/cron.d/`, `crontab -l` | `schtasks /query /fo LIST /v` |
| 敏感檔案權限 | `ls -la /etc/sudoers.d/` | `icacls <path>` |

---

## Exercise：延伸練習

1. **環境盤點**：列出你目前工作環境（或任何你有合法存取的系統）中的三個 Security Boundary。例如：一般使用者 vs 管理員、Application vs Database、Container vs Host。

2. **Mental Model 練習**：對任何一台你有合法 shell 的 Linux 主機，回答以下三個問題：
   - Who am I?（`whoami`, `id`）
   - What can I control?（`find / -writable 2>/dev/null | head -20`）
   - Who trusts what I control?（找出由 root 執行但讀取你可以修改的資源的 process）

3. **Windows Token 觀察**：在你的 Windows 電腦上開一個 cmd，執行 `whoami /all`。你的 Token 裡有多少 Groups？多少 Privileges？其中有幾個是 Enabled 的？

---

## Quiz：觀念測驗

**Q1**：以下哪一個描述最正確？
- (A) Privilege Escalation 就是取得 root
- (B) Privilege Escalation 是跨越 Security Boundary 取得更高權限
- (C) Privilege Escalation 一定需要利用 CVE 漏洞
- (D) 拿到 shell 就算 Privilege Escalation

### 答案(B) 提權是跨越 Security Boundary，不一定要到 root，也不一定需要 CVE。

**Q2**：Linux 的 root 和 Windows 的 SYSTEM 哪個描述正確？
- (A) 它們是完全一樣的東西
- (B) root 是 UID 0 的使用者帳號，SYSTEM 是 OS 核心服務身分
- (C) SYSTEM 就是 Windows 的 Administrator
- (D) root 不能做任何事

### 答案(B) root 是 UID 0，SYSTEM 是 S-1-5-18。

**Q3**：一台 Linux 主機上存在 SUID `/usr/bin/passwd`，這代表可以提權嗎？
- (A) 是，所有 SUID 都能提權
- (B) 不一定，passwd 是正常的 SUID binary
- (C) 完全不可能，SUID 和提權無關
- (D) 只有 root 才能執行 SUID

### 答案(B) passwd、sudo、mount 等都是正常的 SUID binary，不代表可以被濫用。要看 binary 的功能是否能被利用。

**Q4**：在 Windows 上看到一個 Service 以 SYSTEM 執行，這代表什麼？
- (A) 一定有漏洞
- (B) 這是正常的，大多數 Service 都以 SYSTEM 執行
- (C) 應該立刻停用
- (D) 代表 Administrator 密碼洩漏

### 答案(B) 99% 的 Windows Service 都以 SYSTEM 執行，這是正常設計。

**Q5**：「我能控制一個被 root 信任的資源」——這句話描述的是？
- (A) DoS 攻擊
- (B) 資料外洩
- (C) 可能的 Privilege Escalation 條件
- (D) 社交工程

### 答案(C) 控制 + 信任 + 觸發 = 提權的三個條件。

---

## Engineering Note：VMware PCIe Slot Bug

在建立 FEI Lab 時，第一個遇到的 Bug 就是 VMware 的 PCIe slot 問題。

**預期**：使用 vmxnet3 網卡（效能最佳）建立 VM。

**實際**：VM 啟動失敗，錯誤訊息 `No PCIe slot available for Ethernet0`。

**原因**：我們的 Host 同時啟用了 Hyper-V。VMware Workstation 在 Hyper-V 共存模式下以 User-mode Hypervisor 運行，PCIe slot 分配機制不同。vmxnet3 和 e1000e 都需要 PCIe slot，但在這個模式下無法分配。

**修正**：將網卡類型從 `vmxnet3` 改為 `e1000`（傳統 PCI 介面，不需要 PCIe slot）。對訓練靶場效能沒有影響。

```
# VMX 修正
ethernet0.virtualDev = "e1000"    # 原本是 vmxnet3
```

> 這是一個典型的「理論上應該可以，但實際環境不行」的案例。這也是為什麼我們堅持 **Environment is the Source of Truth**。

---

## 今天真正要記住的 3 件事

1. **提權不是找漏洞名稱**。它是問「我能控制什麼？誰信任了它？」

2. **Linux root 和 Windows SYSTEM 不一樣**。它們都是高權限，但機制完全不同。不要硬套。

3. **有 shell ≠ 有權限**。拿到 shell 只是開始，真正的挑戰從這裡才開始。

---

## 給自己的問題

> 如果你現在拿到一台 Linux 主機的 `www-data` shell，你第一個想知道的是什麼？你會執行什麼指令？為什麼？

---

## 下一篇

Day 02：root、Administrator、SYSTEM 到底差在哪？——我們會深入比較 Linux 和 Windows 的權限模型，搞清楚為什麼「管理員」在兩個 OS 上意思完全不同。
