# Day 20：AlwaysInstallElevated — 兩個 Registry Policy 如何改變安裝權限

### 場景準備

```powershell
# 1. 以 fei-labadmin 登入，系統管理員 PowerShell
cd C:\FEI-PrivEsc\Scenarios\FEI-W05-ALWAYS-INSTALL-ELEVATED
.\fei-setup.ps1
.\fei-verify.ps1
# 應看到 FEI-W05 STATUS: READY

# 2. 登出，改以 fei-student 登入（密碼：FEI-Student-2026!）
# ⚠️ 必須重新登入讓 HKCU Policy 生效
```

---

## 開場情境

今天的題目是我整個 Lab 建置過程中最難搞的一題。

不是因為概念複雜——AlwaysInstallElevated 的原理其實很簡單。而是因為**「看起來對」到「真的能用」之間的距離，比任何一題都遠**。

我花了 6 次修正才讓這個 Lab 在實際 Windows 10 上真正運作。每一次修正都暴露了一個「只看文件不會知道」的 OS 行為。

---

## 今天要解決的問題

1. AlwaysInstallElevated 到底是什麼？
2. 為什麼需要 HKLM **和** HKCU 同時設定？
3. MSI 的 Custom Action 如何以 elevated 權限執行？
4. 為什麼這題花了 6 次修正才成功？（這是本系列最有價值的工程紀錄）

---

## 背景知識：Windows Installer

### MSI 是什麼？

MSI（Microsoft Installer）是 Windows 的標準軟體安裝格式。它是一個 **資料庫**，裡面描述了：

- 要安裝哪些檔案
- 要寫入哪些 Registry
- 要執行哪些自訂動作（Custom Actions）

### 安裝流程

![Windows Installer 安裝流程](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day20-always-install-elevated-diagram-01.png)



### 正常權限模型

**預設情況**：Standard User 安裝 MSI 時，Custom Actions 以**使用者自己的權限**執行。不是 SYSTEM，不是 Administrator。

如果 MSI 需要寫入 `C:\Program Files\` 或修改系統 Registry，使用者需要有足夠權限（或 UAC 提示輸入 Admin 密碼）。

---

## OS 原理：AlwaysInstallElevated

### 這個 Policy 做什麼？

```
HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer\AlwaysInstallElevated = 1
HKCU\Software\Policies\Microsoft\Windows\Installer\AlwaysInstallElevated = 1
```

當兩個值**同時為 1** 時：

> **任何使用者**安裝的 MSI 都以 **elevated 權限**（類似 SYSTEM）執行。

包含 MSI 裡的 Custom Actions。

### 為什麼需要兩個值？

| Registry | 意義 |
|----------|------|
| HKLM | 電腦層級：「本機允許」 |
| HKCU | 使用者層級：「此使用者啟用」 |

**雙因素設計**：只有兩者同時為 1，policy 才生效。這是為了防止單一設定被意外啟用。

### 攻擊條件

```
HKLM = 1  AND  HKCU = 1
+
攻擊者控制一個 MSI（含惡意 Custom Action）
=
Custom Action 以 elevated 權限執行
=
Privilege Escalation
```

---

## FEI Lab 環境

```
🔬 FEI Lab 環境
VM: FEI-PRIVESC-WINDOWS
Scenario: FEI-W05-ALWAYS-INSTALL-ELEVATED
Policy: HKLM + HKCU AlwaysInstallElevated = 1
MSI: C:\FEI-PrivEsc\Training\W05\FEI-W05-Training.msi
Flag: C:\FEI-PrivEsc\Flags\fei-w05-system-flag.txt
```

---

## 攻擊流程

### Step 1：檢查 Policy

```cmd
reg query HKCU\Software\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
```

```
AlwaysInstallElevated    REG_DWORD    0x1
```

```cmd
reg query HKLM\Software\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
```

```
AlwaysInstallElevated    REG_DWORD    0x1
```

> 兩個都是 1。條件滿足。

### Step 2：安裝 MSI

```cmd
msiexec /i "C:\FEI-PrivEsc\Training\W05\FEI-W05-Training.msi" /qn
```

### Step 3：讀取 Flag

```cmd
type "C:\Users\Public\fei-w05-proof.txt"
```

```
FEI{WINDOWS_W05_ALWAYS_INSTALL_ELEVATED}
```

---

## Attack Path

![AlwaysInstallElevated Attack Path](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day20-always-install-elevated-diagram-02.png)



---

## False Positive 分析

### 只看到 HKLM 有值就報漏洞？

**不對。** 需要 HKCU **也**有值。

| HKLM | HKCU | 漏洞？ |
|:---:|:---:|:---:|
| 1 | 0 或不存在 | ❌ |
| 0 或不存在 | 1 | ❌ |
| 1 | 1 | ✅ |

### 有 Policy 就一定能提權？

不一定。還需要：
- 使用者能安裝自己的 MSI
- MSI 包含可控制的 Custom Action
- Windows Installer Service 正常運作

---

## Edge Case

### msiexec 只能從 Interactive Session 執行

在我們的 Lab 中發現：batch logon（schtasks）呼叫 msiexec 會得到 Error 1601（Access Denied connecting to server）。

**msiexec 需要 interactive session**。這代表：
- 遠端 shell（如 WinRM）可能無法直接利用
- 需要 RDP 或 console 存取
- 自動化測試受限

### ConsentPromptBehaviorUser

Windows 10 Build 19045 的預設值是 `3`（自動拒絕標準使用者的 elevation 請求）。這會阻擋 silent elevated install。

Lab 中我們把它改為 `0`（不提示）才成功。

### Silent vs Interactive Install

`/qn`（完全 silent）的行為和 `/qb`（basic UI）不同。某些 Windows Build 對 silent elevated install 有額外限制。

---

## FEI Lab 工程筆記（本系列最詳盡的一篇）

這個 Lab 經歷了 **6 次修正**。每一次都暴露了一個「文件沒寫清楚」的 OS 行為。

### Bug 1：msiOpenDatabaseModeCreate = 1

**預期**：用 VBScript COM API 建立 MSI 資料庫。

```vbs
Const msiOpenDatabaseModeCreate = 1
Set database = installer.OpenDatabase(msiPath, msiOpenDatabaseModeCreate)
```

**實際**：`OpenDatabase` 呼叫失敗。

**原因**：`msiOpenDatabaseModeCreate` 的正確值是 **3**，不是 1。值 1 是 `msiOpenDatabaseModeTransact`（開啟現有資料庫），由於檔案不存在所以失敗。

**修正**：改為 `Const msiOpenDatabaseModeCreate = 3`

**教訓**：COM API 的常數值必須查文件，不能憑直覺。

---

### Bug 2：GUID 含非 hex 字元

**預期**：MSI 的 ProductCode GUID 格式正確。

```
{FEI05AA1-W005-4FEI-B05A-FEI05TRAINING}
```

**實際**：MSI 載入後報 "Invalid descriptor format"。

**原因**：GUID 必須只使用 hex 字元（0-9, A-F）。`I` 不是 hex。

**修正**：

```
{FE105AA1-B005-4FE1-B05A-FE105A000001}
```

**教訓**：MSI internal validator 很嚴格。GUID 不是隨便的 identifier。

---

### Bug 3：缺少 Feature / Component / Media 表

**預期**：MSI 只需要 Property + CustomAction + InstallExecuteSequence 表。

**實際**：Error 1625（安裝被系統原則拒絕）。

**原因**：Windows Installer 需要 Feature、Component、FeatureComponents、Media 表才能通過 InstallValidate 步驟，即使實際上沒有要安裝任何檔案。

**修正**：加入 minimal Feature/Component/FeatureComponents/Media 表（空的 dummy 條目）。

**教訓**：MSI 有嚴格的 internal schema。Custom-Action-only MSI 也需要基本的安裝結構。

---

### Bug 4：CustomAction Type 搞混

**預期**：Type 34 = exe from Property。

```vbs
record.IntegerData(2) = 34   ' exe from property
record.StringData(3) = "COMSPEC"
```

**實際**：Error 2727 "The directory entry 'COMSPEC' does not exist in the Directory table"。

**原因**：MSI Custom Action Type 的位元編碼：

| 類型 | Source Bits | 意義 |
|------|-----------|------|
| 0x20 | Directory | Source = Directory 表中的 key |
| 0x30 | Property | Source = Property 表中的 key |

- Type 34 = 0x02 (exe) + 0x20 (**Directory**) = 34
- Type 50 = 0x02 (exe) + 0x30 (**Property**) = 50

我需要的是 Type **50**，不是 34。

然後又發現：Type 50 是 **immediate** execution — 以使用者權限執行，不是 elevated。

最終使用 Type **3078** = 6 (VBScript from Binary) + 1024 (deferred) + 2048 (no impersonate) = **deferred VBScript**，在 elevated server process 中執行。

**教訓**：MSI Custom Action Type 是一個位元欄位組合，每個位代表不同含義。搞混一個位就會完全不同的行為。

---

### Bug 5：HKCU 寫入失敗

**預期**：setup 以 SYSTEM 執行時，透過 `reg load` 載入 fei-student 的 NTUSER.DAT，寫入 HKCU 值。

**實際**：值寫不進去。PowerShell 的 Registry cmdlet 保持 handle，`reg unload` 時 hive 沒有正確存檔。

**修正**：改用 `reg.exe add` 原生指令（不經過 PowerShell Registry Provider），避免 handle 佔用問題。

後來又發現：如果 fei-student 的 hive 已由 live session 載入，`reg load` 會失敗（檔案正在使用）。改為偵測 `HKU\<SID>` 是否已載入，如果是就直接寫。

**教訓**：跨使用者的 Registry 操作在 Windows 上比想像中複雜得多。

---

### Bug 6：ConsentPromptBehaviorUser = 3

**預期**：設好 AlwaysInstallElevated 後，msiexec 直接以 elevated 安裝。

**實際**：MSI log 顯示 "Elevation prompt disabled for silent installs"，然後 Error 1625。

**原因**：Windows 10 預設 `ConsentPromptBehaviorUser = 3`（自動拒絕標準使用者的 elevation）。即使 AlwaysInstallElevated = 1，UAC policy 仍然會阻擋。

**修正**：setup 中將 `ConsentPromptBehaviorUser` 改為 `0`（不提示）。

**教訓**：Windows 的安全機制是多層的。一個 policy 說「可以」，另一個 policy 可能說「不行」。

---

### 6 次修正的時間線

![AlwaysInstallElevated 六次修正時間線](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day20-always-install-elevated-diagram-03.png)



> 這就是為什麼我說「Environment is the Source of Truth」。網路文章告訴你「設兩個 Registry 值就好」，但在 Windows 10 Build 19045 上，你需要解決 MSI 結構、GUID 編碼、Custom Action 位元欄位、Registry hive 操作、和 UAC 多層 policy 才能真正成功。

---

## 防禦（Defense）

### Root Cause

管理員（或 Group Policy）同時啟用了 HKLM 和 HKCU 的 AlwaysInstallElevated。

### 修復

```cmd
reg delete HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated /f
reg delete HKCU\Software\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated /f
```

### 預防

- 企業環境不要啟用 AlwaysInstallElevated
- 使用 SCCM / Intune 等正式部署管道
- 定期掃描：

```powershell
$hklm = Get-ItemProperty "HKLM:\SOFTWARE\Policies\Microsoft\Windows\Installer" -Name "AlwaysInstallElevated" -EA SilentlyContinue
$hkcu = Get-ItemProperty "HKCU:\Software\Policies\Microsoft\Windows\Installer" -Name "AlwaysInstallElevated" -EA SilentlyContinue
if ($hklm.AlwaysInstallElevated -eq 1 -and $hkcu.AlwaysInstallElevated -eq 1) {
    Write-Warning "AlwaysInstallElevated is ENABLED - privilege escalation risk!"
}
```

---

## Detection

| 偵測點 | 方法 |
|--------|------|
| Registry 值被設定 | Event ID 4657 |
| MSI 被安裝 | Event ID 11707 (MsiInstaller) |
| msiexec.exe 被一般使用者執行 | Event ID 4688 |

---

## Fix Verification

刪除兩個 Registry 值後，以 fei-student 嘗試：

```cmd
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
```

應回傳「找不到指定的登錄機碼或值」。

---

## Exercise

### 練習：MSI Custom Action Type 計算

Type 3078 是由哪些位元組合而成？

### 解答

```
3078 = 6 + 1024 + 2048
     = VBScript (0x06)
     + Deferred in-script (0x400 = 1024)
     + No Impersonate (0x800 = 2048)
```

Deferred + No Impersonate = 在 elevated server process 中執行，不降級為使用者身分。


---

## Quiz

**Q1**: 為什麼只有 HKLM AlwaysInstallElevated = 1 不夠？

A. 因為 Windows 的 bug
B. 因為需要雙因素確認（HKLM + HKCU 都必須為 1）
C. 因為 HKLM 的值會被忽略
D. 因為需要 Administrator 權限才能讀 HKLM

### 答案B。Windows 的設計是雙重確認：電腦層級和使用者層級都必須同意。

**Q2**: 為什麼 Immediate Custom Action 不能用於提權？

A. 因為它不會執行
B. 因為它以使用者權限執行，不是 elevated
C. 因為 Windows 10 禁止 Custom Action
D. 因為它需要 .NET

### 答案B。Immediate Custom Action 在 client-side（使用者 context）執行。需要 Deferred + No Impersonate 才能在 elevated server process 中執行。



## 場景收尾

做完後記得 reset：

`powershell
# 以 fei-labadmin 的系統管理員 PowerShell
cd C:\FEI-PrivEsc\Scenarios\FEI-W05-ALWAYS-INSTALL-ELEVATED
.\fei-reset.ps1
.\fei-verify.ps1 -Mode reset
# 應看到 W05-ALWAYS-INSTALL-ELEVATED STATUS: RESET
`

---

## 今天真正要記住的 3 件事

1. **AlwaysInstallElevated 需要 HKLM + HKCU 同時為 1**。只有一個不夠。
2. **MSI 的 Custom Action Type 是位元欄位組合**。Type 34 和 Type 50 完全不同。
3. **「在文件上看起來可以」和「在 Windows 10 上真的可以」之間可能差 6 個 Bug**。這就是 Environment is the Source of Truth。

---

## 給自己的問題

> 如果 SYSTEM 不是因為 Service 或 Policy，而是因為一個 Registry 設定值控制了高權限 Process 的行為呢？低權限使用者能改那個 Registry 值嗎？

---

## 下一篇

Day 21：Registry ACL — 能改 Registry，為什麼可能等於控制 SYSTEM？
