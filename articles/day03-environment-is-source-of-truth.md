# Day 03：我為什麼自己打造 20 個提權環境？Environment is the Source of Truth


---

## 背景知識

### 「照著做就好啦」的陷阱

HackTricks 很好。GTFOBins 很好。PayloadsAllTheThings 也很好。

但如果你把上面的 command 一字不差複製到自己的 Lab，結果是 `Access Denied`——你會知道為什麼嗎？

這就是我自己建立 20 個提權環境的原因。不是因為網路資源不夠好，而是因為：

> **「看起來可以」和「在目標 OS 上真的可以」之間，往往有一道巨大的鴻溝。**

在建立這套 Lab 的過程中，我遇到了超過 15 個「理論上應該可以但實際上不行」的情況。每一個都是寶貴的工程經驗。

### Environment is the Source of Truth

這句話是整個系列的最高原則：

```
網路文章說可以     ≠  實際環境一定可以
Command 看起來合理  ≠  實際 OS 一定這樣執行
有某個設定         ≠  一定可以成功提權
```

只有在**實際 VM 上**、用**實際低權限帳號**、**實際取得 flag**——才算數。

---

## OS 原理：為什麼需要隔離 Lab？

### 資安靶場的三個層次

| 層次 | 說明 | 風險 |
|------|------|------|
| **Production** | 真實環境 | 不能做任何攻擊測試 |
| **Cloud Lab** | HackTheBox、TryHackMe | 別人建好的，你不知道細節 |
| **Self-hosted Lab** | 自己建的 VM | 完全控制，但需要自己維護 |

Self-hosted Lab 的好處：
- 你知道每一個設定為什麼存在
- 你可以看到 setup/reset 的完整過程
- 你遇到問題時知道去哪裡找原因
- 你可以重複測試、做 regression

### 虛擬化隔離

![虛擬化隔離](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day03-environment-is-source-of-truth-diagram-01.png)



---

## Lab：完整環境介紹

### Host 規格

| 項目 | 規格 |
|------|------|
| **Host OS** | Windows 10 Pro |
| **Hardware** | Framework Laptop (13th Gen Intel Core) |
| **CPU** | Intel i7-1370P, 14 Cores / 20 Threads |
| **RAM** | 64 GB |
| **Disk** | 3.8 TB SSD |
| **Hypervisor** | VMware Workstation 17.5.2 |
| **共存** | Hyper-V hypervisor 同時啟用 |

### 網路架構

![網路架構](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day03-environment-is-source-of-truth-diagram-02.png)



### 兩台 VM

| VM | OS | IP | 帳號 |
|----|----|-----|------|
| FEI-PRIVESC-LINUX | Ubuntu 22.04.4 LTS | 192.168.77.10 | fei-student / fei-labadmin |
| FEI-PRIVESC-WINDOWS | Windows 10 Pro (19045) | 192.168.77.20 | fei-student / fei-labadmin |

### 實際環境確認

```bash
# Linux VM
fei-student@fei-privesc-linux:~$ id
uid=1001(fei-student) gid=1001(fei-student) groups=1001(fei-student)

fei-student@fei-privesc-linux:~$ uname -a
Linux fei-privesc-linux 5.15.0-94-generic #104-Ubuntu SMP x86_64
```

```cmd
# Windows VM
C:\> hostname
FEI-PRIV-WIN

C:\> whoami
fei-priv-win\fei-student
```

---

## 取得完整腳本：GitHub Repo

![GitHub Repo 目錄結構](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day03-environment-is-source-of-truth-diagram-03.png)


所有 Scenario 的腳本、設定檔、Framework 都開源：

```
https://github.com/fei3363/FEI-Privilege-Escalation-Lab
```

```bash
git clone https://github.com/fei3363/FEI-Privilege-Escalation-Lab.git
```

Repo 結構：


---

## 從零開始建置：跟著做

### 下載清單

| 軟體 | 下載連結 |
|------|---------|
| VMware Workstation Player（免費）| https://www.vmware.com/products/workstation-player.html |
| VMware Workstation Pro（試用）| https://www.vmware.com/products/workstation-pro.html |
| Ubuntu Server 22.04 LTS | https://ubuntu.com/download/server |
| Windows 10 ISO | https://www.microsoft.com/software-download/windows10 |

> 💡 **怎麼開系統管理員 PowerShell？**
>
> 右鍵點擊「開始」按鈕 → 選「Windows PowerShell (系統管理員)」。
> 或搜尋 `powershell`，右鍵 → 以系統管理員身分執行。

### Step 1：環境確認

```powershell
# 在 Host 執行
(Get-CimInstance Win32_ComputerSystem).HypervisorPresent  # 應為 True
(Get-CimInstance Win32_Processor).NumberOfLogicalProcessors  # 至少 8
[math]::Round((Get-CimInstance Win32_OperatingSystem).TotalVisibleMemorySize / 1MB, 0)  # 至少 16 GB
```

### Step 2：建立 Host-Only 網路

VMware Virtual Network Editor（系統管理員）→ VMnet2 → Host-only → `192.168.77.0/255.255.255.0` → 停用 DHCP

### Step 3：建立 Linux VM

```
Name: FEI-PRIVESC-LINUX
CPU: 2 vCPU / RAM: 4 GB / Disk: 40 GB
Network: Custom (VMnet2)
網卡: e1000（不要用 vmxnet3）
SCSI: lsilogic
```

安裝 Ubuntu Server 22.04 → Hostname: `fei-privesc-linux` → 使用者: `fei-labadmin`

### Step 4：Linux Post-Install

```bash
sudo useradd -m -s /bin/bash fei-student
echo "fei-student:FEI-Student-2026!" | sudo chpasswd
sudo gpasswd -d fei-student sudo 2>/dev/null
sudo mkdir -p /opt/fei-privesc/{framework,scenarios,flags,backups,logs,docs,bin}
```

### Step 5：建立 Windows VM

```
Name: FEI-PRIVESC-WINDOWS
CPU: 4 vCPU / RAM: 8 GB / Disk: 80 GB
Network: Custom (VMnet2)
網卡: e1000
SCSI: lsisas1068（不要用 lsilogic）
Firmware: EFI
```

### Step 6：Windows Post-Install

```powershell
$pass = ConvertTo-SecureString "FEI-Student-2026!" -AsPlainText -Force
New-LocalUser -Name "fei-student" -Password $pass -PasswordNeverExpires
Rename-Computer -NewName "FEI-PRIV-WIN" -Force
```

### Step 7：Snapshot

為兩台 VM 建立 `FEI-01-LAB-BASE` snapshot。

---

## Attack Reasoning：為什麼 Scenario Framework 要這樣設計

### setup → verify → solve → reset 流程

![setup、verify、solve、reset 流程](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day03-environment-is-source-of-truth-diagram-04.png)


每個 Scenario 都遵循：


### Active Scenario 衝突保護

```
/opt/fei-privesc/framework/active-scenario    # Linux
C:\FEI-PrivEsc\Framework\active-scenario.txt  # Windows
```

如果已有場景 active，setup 會拒絕：
```
Another FEI scenario is active: FEI-L01
Reset it before continuing.
```

### Regression Testing

每新增一個場景，都要重新驗證之前的場景仍然正常。例如 Phase 11（L09 + W09）完成後，需跑 L00-L08 和 W00-W08 的 regression。

---

## Actual Output：5 個真實 Bug Fix

### Bug 1：VMware PCIe Slot

**預期**：vmxnet3 網卡啟動 VM

**實際**：`No PCIe slot available for Ethernet0`

**原因**：Hyper-V 共存模式下 PCIe slot 分配機制不同

**修正**：改用 `e1000`（PCI 介面，不需 PCIe slot）

### Bug 2：W01 SDDL 缺少 DC

**預期**：fei-student 可以 `sc config` 修改 Service

**實際**：`存取被拒`

**原因**：SDDL 的 BU ACE 是 `CCLCSWRPWPDTLOCRSDRCWDWO`，缺少 `DC`（SERVICE_CHANGE_CONFIG）

**修正**：改為 `CCDCLCSWRPWPDTLOCRSDRCWDWO`

### Bug 3：L02 需要 gcc

**預期**：在 VM 中編譯 C 程式作為 SUID binary

**實際**：VM 是 Host-Only 沒有 Internet，無法 apt install gcc

**修正**：改用 `cp /bin/cat /opt/fei-privesc/bin/fei-l02-reader`，教學價值不變

### Bug 4：W05 AlwaysInstallElevated — 6 次修正

| # | Bug | 修正 |
|---|-----|------|
| 1 | `msiOpenDatabaseModeCreate = 1`（Transact） | 改為 `3`（Create）|
| 2 | GUID 含非 hex 字元 `I` | 改為合法 hex |
| 3 | 缺少 Feature/Component/Media 表 | 補齊 → Error 1625 → 1603 |
| 4 | CustomAction Type 34(Directory) | → 50(Property) → 3078(VBScript deferred) |
| 5 | HKCU offline hive handle 佔用 | → reg.exe + live HKU |
| 6 | `ConsentPromptBehaviorUser=3`(auto-deny) | → 0 |

這是整個系列最複雜的場景，6 次修正才通過 Real Validation。

### Bug 5：W08 UAC Token Split

**預期**：Backup Operators + SeBackupPrivilege → 讀取 ACL 受保護的檔案

**實際**：`AdjustTokenPrivileges` 回傳 error 1300

**原因**：UAC token splitting 讓 privilege 變成永久停用

**修正**：停用 `EnableLUA` + 改用 `robocopy /B`（系統工具能正確處理 privilege）

---

## False Positive：Lab 建置的常見誤判

| 看到的現象 | 為什麼不一定是問題 |
|-----------|-------------------|
| `verify.sh` 顯示 FAIL | 可能是 verify 腳本本身的 bug（W06 的 `-band` bitwise 問題）|
| setup 成功但 solve 失敗 | 可能是 OS 行為與預期不同（W05 MSI、W08 Token）|
| vmrun 可以跑但結果不對 | vmrun 的非互動 session token 和互動 session 不同 |
| Registry 值設定了但看不到 | 可能寫到了錯誤的 HKCU hive（不同使用者的 SID）|

---

## Edge Case：OS 版本差異

| 差異 | 影響 |
|------|------|
| Ubuntu 22.04 使用 `cron` 不是 `crond` | service 名稱需要驗證 |
| Windows 10 Build 19045 的 MSI 行為 | CustomAction deferred 在 silent install 中的表現不同 |
| LVM 的 `/dev/mapper/` 是 symlink | `stat` 顯示的 group 是 symlink 的 group，不是實際 device 的 |
| Hyper-V 共存影響 VMware 的 PCIe slot | vmxnet3/e1000e 可能無法使用 |

---

## Troubleshooting：建置常見問題

| 問題 | 原因 | 解法 |
|------|------|------|
| VM 啟動 `No PCIe slot` | Hyper-V 共存 | 網卡改 `e1000` |
| Windows 安裝找不到磁碟 | lsilogic 無驅動 | 改 `lsisas1068` |
| `sudo -l` 要密碼 | fei-student 不是 sudoer（正常） | 這是預期行為 |
| vmrun 回傳 `Invalid username or password` | VM 還在開機中 | 等幾秒重試 |
| Scenario setup 顯示 `Another scenario active` | 沒有先 reset | 先 reset 前一個 |
| secedit 設定的 privilege 看不到 | 需要 relogin | 登出重新登入 |
| `robocopy /B` 說沒有權限 | UAC token split | 需要 `EnableLUA=0`（僅 Lab）|

### 帳號密碼速查表

| 帳號 | 密碼 | 用途 |
|------|------|------|
| `fei-student` | `FEI-Student-2026!` | 攻擊者起始帳號（低權限）|
| `fei-labadmin` | `FEI-LabAdmin-2026!` | Lab 管理帳號（高權限）|

> ⚠️ 這些密碼只用於本地隔離 Lab，不得與任何真實服務共用。

### 部署 Scenario 到 VM

**Linux**（從 Host 的 PowerShell 執行）：

```powershell
# 用 SCP 複製所有 Linux Scenario
scp -r scenarios/FEI-L* fei-labadmin@192.168.77.10:/tmp/
# SSH 進去移到正確位置
ssh fei-labadmin@192.168.77.10
sudo cp -r /tmp/FEI-L* /opt/fei-privesc/scenarios/
sudo chown -R root:root /opt/fei-privesc/scenarios/
sudo find /opt/fei-privesc/scenarios -name "*.sh" -exec chmod 755 {} \;
# 修正換行符（Windows → Linux）
sudo find /opt/fei-privesc/scenarios -name "*.sh" -exec sed -i 's/\r$//' {} \;
```

**Windows**（從 Host 手動複製或用 USB）：

```powershell
# 在 Windows VM 中開 PowerShell
Copy-Item -Path "E:\scenarios\FEI-W*" -Destination "C:\FEI-PrivEsc\Scenarios\" -Recurse
```

### VM 關機與重開機

VM 正常關機後重新開機，已 setup 的 Scenario 仍然存在，不需要重新 setup。

如果你把 Lab 搞壞了（例如不小心刪了重要檔案），還原到 **FEI-01-LAB-BASE** Snapshot：

- VMware Workstation → 右鍵 VM → Snapshot → Revert to Snapshot → FEI-01-LAB-BASE

還原後需要重新部署 Scenario 檔案。

### 忘記 Reset 就做下一題？

如果你看到這個錯誤：

```
Another FEI scenario is active: FEI-L01
Reset it before continuing.
```

先回到上一題的目錄 reset：

```bash
cd /opt/fei-privesc/scenarios/FEI-L01-SUDO
sudo ./fei-reset.sh
```

然後再 setup 新題目。

---

## Defense：Lab 環境的安全設計

| 防護 | 說明 |
|------|------|
| Host-Only Network | VM 不接 Internet、不接 LAN |
| 禁用 Bridged | Lab 漏洞不會影響真實環境 |
| 禁用 Shared Folder | 不共享 Host 檔案 |
| Training Password | 所有密碼都是 Lab-only |
| 不用 Microsoft Account | Windows VM 使用離線帳號 |
| Scenario 隔離 | Active Scenario 機制防止衝突 |

---

## Detection：自動化驗證

每個 Scenario 的 `verify.sh/ps1` 是自動化偵測腳本：

```
[PASS] fei-student exists
[PASS] fei-student UID is not 0
[PASS] intended vulnerability exists
[PASS] no unintended privilege path
[PASS] flag protected
...
```

這就是 Lab 版本的 Detection：自動確認場景是否「如預期般脆弱」（或「如預期般安全」）。

---

## Fix Verification：Regression Testing

每次新增 Scenario 後：

```bash
# Linux：所有場景 setup → verify → reset → verify-reset
for s in FEI-L00 FEI-L01 ... FEI-L09; do
  cd /opt/fei-privesc/scenarios/$s
  sudo ./fei-setup.sh
  sudo ./fei-verify.sh    # 必須全 PASS
  sudo ./fei-reset.sh
  sudo ./fei-verify.sh --reset  # 必須全 PASS
done
```

如果任何一個 Scenario 出現 regression（之前 PASS 現在 FAIL），必須先修復才能繼續。

---

## 20 個 Scenario 總覽

| ID | 名稱 | 平台 | Flag |
|----|------|------|------|
| L00 | Enumeration | Linux | N/A |
| L01 | sudo Misconfiguration | Linux | `FEI{LINUX_L01_ROOT_ACCESS}` |
| L02 | SUID Binary | Linux | `FEI{LINUX_L02_SUID_ROOT_ACCESS}` |
| L03 | PATH Hijacking | Linux | `FEI{LINUX_L03_PATH_ROOT_ACCESS}` |
| L04 | Cron Job | Linux | `FEI{LINUX_L04_CRON_ROOT_ACCESS}` |
| L05 | Capabilities | Linux | `FEI{LINUX_L05_CAPABILITIES_ROOT_ACCESS}` |
| L06 | Weak File Permissions | Linux | `FEI{LINUX_L06_WEAK_FILE_PERMISSION_ROOT_ACCESS}` |
| L07 | Credential Exposure | Linux | `FEI{LINUX_L07_CREDENTIAL_EXPOSURE_ROOT_ACCESS}` |
| L08 | Dangerous Group | Linux | `FEI{LINUX_L08_DANGEROUS_GROUP_ROOT_ACCESS}` |
| L09 | systemd Service | Linux | `FEI{LINUX_L09_SYSTEMD_ROOT_ACCESS}` |
| W00 | Enumeration | Windows | N/A |
| W01 | Service ACL | Windows | `FEI{WINDOWS_W01_SYSTEM_ACCESS}` |
| W02 | Service Binary | Windows | `FEI{WINDOWS_W02_SYSTEM_ACCESS}` |
| W03 | Unquoted Path | Windows | `FEI{WINDOWS_W03_UNQUOTED_PATH_SYSTEM_ACCESS}` |
| W04 | Scheduled Task | Windows | `FEI{WINDOWS_W04_SCHEDULED_TASK_SYSTEM_ACCESS}` |
| W05 | AlwaysInstallElevated | Windows | `FEI{WINDOWS_W05_ALWAYS_INSTALL_ELEVATED}` |
| W06 | Registry ACL | Windows | `FEI{WINDOWS_W06_REGISTRY_ACL_SYSTEM_ACCESS}` |
| W07 | Credential Exposure | Windows | `FEI{WINDOWS_W07_CREDENTIAL_EXPOSURE_ADMIN_ACCESS}` |
| W08 | Token Privileges | Windows | `FEI{WINDOWS_W08_SEBACKUP_PRIVILEGED_READ}` |
| W09 | DLL Hijacking | Windows | `FEI{WINDOWS_W09_DLL_HIJACKING_SYSTEM_ACCESS}` |

---

## Exercise

1. **建起你自己的 Lab**：clone repo，按照 Step 1-7 建立環境。執行 `host/fei-check-requirements.ps1` 確認環境。

2. **第一個 Scenario 測試**：部署 FEI-L01-SUDO，跑 setup → verify。確認所有 PASS。然後以 fei-student 登入嘗試解題。

3. **Bug 觀察**：在建立過程中，記錄你遇到的每一個「預期 vs 實際」差異。這些都是學習 OS 行為的最佳機會。

---

## Quiz

**Q1**：「Environment is the Source of Truth」是什麼意思？
- (A) 只要在 Lab 上成功就代表在任何環境都會成功
- (B) 只有在實際目標環境上驗證過的結果才能被信任
- (C) 網路上的文章比實際測試更可靠
- (D) Lab 環境不需要和 Production 一致

### 答案(B) 實際環境驗證 > 理論推測。

**Q2**：為什麼 FEI Lab 使用 Host-Only 網路？
- (A) 因為 Bridged 比較慢
- (B) 為了確保 Lab 中的漏洞不會影響真實網路環境
- (C) 因為 NAT 不支援
- (D) 沒有特別原因

### 答案(B) Host-Only 確保 VM 的漏洞環境完全隔離。

**Q3**：W05 AlwaysInstallElevated 經歷了幾次 Bug Fix？
- (A) 1 次
- (B) 3 次
- (C) 6 次
- (D) 0 次

### 答案(C) MSI 結構、GUID、CustomAction Type、HKCU hive、UAC policy，共 6 次。

**Q4**：Regression Testing 的目的是？
- (A) 讓新場景跑起來
- (B) 確認新場景不會破壞之前已驗證的場景
- (C) 只測試最新的場景
- (D) 節省時間

### 答案(B) Regression 確保改動不會造成副作用。

---

## Engineering Note：完整 Bug 統計

| Phase | 場景 | Bug 數量 | 關鍵 Bug |
|-------|------|---------|---------|
| 1 | VM 建立 | 2 | PCIe slot、lsilogic 磁碟 |
| 3 | W01 | 1 | SDDL DC |
| 4 | L02 | 1 | gcc dependency |
| 7 | W05 | 6 | MSI 結構（最複雜）|
| 8 | W06 | 4 | -band bitwise、WriteKey、ACL 繼承 |
| 9 | W07 | 0 | — |
| 10 | W08 | 3 | LSA API、UAC token、robocopy |
| 10 | L08 | 2 | LVM symlink、udev rule |
| 11 | L09 | 0 | — |
| 11 | W09 | 0 | — |

**總計：19 個 Bug，全部修正並記錄。**

> 資安教材真正困難的不是寫出 Command，而是**證明它真的成立**。

---

## 今天真正要記住的 3 件事

1. **「看起來可以」不等於「真的可以」**。W05 六次修正、W08 三次修正——每一個都是理論與實際的落差。

2. **Lab 不只是練習工具，也是學習 OS 行為的最佳途徑**。每一個 Bug 都教了一個 OS 機制的真實行為。

3. **Regression Testing 不是可有可無的**。當你有 20 個場景時，一個修改可能破壞另一個場景。

---

## 給自己的問題

> 你有沒有遇過「照著教學做但結果不同」的經驗？那個差異是什麼原因造成的？OS 版本？設定不同？權限不同？

---

## 下一篇

Day 04：拿到 Linux Shell 後，第一件事不是找 Exploit——我們會從 FEI-L00-ENUMERATION 開始，學習系統性的 Linux 列舉流程。
