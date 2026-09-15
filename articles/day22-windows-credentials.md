# Day 22：Windows Credential Hunting：不用漏洞，也能從 User 走到 Local Admin

## 開場情境

前面 6 題 Windows 提權（W01-W06），最後都是拿到 SYSTEM。今天不一樣。

今天的問題不是「怎麼利用漏洞」，而是：

> 系統上有沒有人不小心把密碼留在你讀得到的地方？

而且這次的目標**不是 SYSTEM，是 Local Administrator**。

---

## 今天要解決的問題

1. 提權一定需要漏洞嗎？
2. Windows 上常見的 Credential 暴露位置有哪些？
3. 找到一組帳號密碼之後，還需要驗證什麼？
4. 為什麼這題的目標不是 SYSTEM？

---

## 背景知識：Credential Exposure vs Credential Dumping

這是非常重要的區分：

| | Credential Hunting（本題）| Credential Dumping（不在範圍）|
|--|---|---|
| 方法 | 搜尋檔案中的明文密碼 | 從記憶體/系統提取 hash |
| 工具 | findstr, grep, dir | Mimikatz, secretsdump |
| 所需權限 | 低（讀取權限即可）| 高（通常需 admin/SYSTEM）|
| 偵測難度 | 低（只是讀檔案）| 高（明顯的攻擊行為）|

本題只做 Credential Hunting —— 找明文密碼。不碰 LSASS、SAM、DPAPI。

---

## OS 原理：Windows Local Account Model

### Windows 帳號存放位置

Windows Local Account 的 hash 存在 SAM (Security Accounts Manager)：

```
C:\Windows\System32\config\SAM
```

但 SAM 檔案在 OS 運行時被鎖定，Standard User 無法讀取。

### 帳號與群組

```cmd
net user fei-w07-admin
net localgroup Administrators
```

一個帳號加入 `Administrators` 群組後，就有 Local Admin 權限 —— 但在 UAC 環境下，Token 可能是 Medium Integrity（標準）而非 High Integrity（提升）。

### runas 機制

```cmd
runas /user:fei-w07-admin cmd.exe
```

`runas` 建立新的 logon session，載入目標帳號的 Token。需要輸入密碼。

---

## Lab 環境

```
🔬 FEI Lab 環境
VM: FEI-PRIVESC-WINDOWS
OS: Windows 10 Pro (Build 19045)
起始帳號: fei-student (Standard User)
目標: fei-w07-admin (Local Administrator)
Flag: C:\FEI-PrivEsc\Flags\fei-w07-admin-flag.txt
Scenario: FEI-W07-CREDENTIALS
```



### 場景準備

`powershell
# 1. 以 fei-labadmin 登入 Windows VM（VMware Console）
# 密碼：FEI-LabAdmin-2026!

# 2. 以系統管理員開啟 PowerShell，進入場景目錄
cd C:\FEI-PrivEsc\Scenarios\FEI-W07-CREDENTIALS

# 3. 執行 setup
.\fei-setup.ps1

# 4. 確認場景就緒
.\fei-verify.ps1
# 應看到 W07-CREDENTIALS STATUS: READY

# 5. 登出，改以 fei-student 登入
# 密碼：FEI-Student-2026!
`

### 部署過程

1. 建立 `fei-w07-admin` 帳號，加入 Administrators
2. 在 Training 目錄放置 credential exposure 檔案（備份設定檔）
3. 放置 3 個 decoy（假的密碼）
4. Flag ACL 只允許 **SYSTEM + fei-w07-admin**（不是 Administrators 群組！）

> 注意：Flag 的 ACL 刻意排除 `BUILTIN\Administrators`，只允許特定使用者。這確保必須真正以 fei-w07-admin 身份存取。

---

## 如果我是攻擊者，我現在想知道什麼？

### Credential Hunting 思考流程

![Credential Hunting 思考流程](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day22-windows-credentials-diagram-01.png)



### 搜尋

```cmd
findstr /si "password" C:\FEI-PrivEsc\Training\W07\*.*
```

結果：

```
C:\FEI-PrivEsc\Training\W07\Backups\deploy-config.ini.bak:password = FEI-W07-Admin-Training-2026!
C:\FEI-PrivEsc\Training\W07\Configs\database.config:password = changeme123
C:\FEI-PrivEsc\Training\W07\Scripts\install-notes.txt:3. Run setup with default password: P@ssw0rd
```

三個候選。哪一個是真的？

---

## False Positive：找到密碼字串 ≠ 找到有效 Credential

### Decoy 分析

| 檔案 | 密碼 | 帳號 | 有效？|
|------|------|------|------|
| database.config | `changeme123` | 未指定 | ❌ 預設值 |
| install-notes.txt | `P@ssw0rd` | installer | ❌ 安裝程式預設 |
| **deploy-config.ini.bak** | **`FEI-W07-Admin-Training-2026!`** | **fei-w07-admin** | **✅** |

判斷邏輯：
- `changeme123` — 明顯是預設值，沒有對應帳號名稱
- `P@ssw0rd` — installer default，文件還特別標示「only for INSTALLER」
- `FEI-W07-Admin-Training-2026!` — 有明確的 username + password 配對，且帳號名稱像是管理員

### 驗證帳號存在

```cmd
net user fei-w07-admin
```

帳號存在，且是 Administrators 成員。

---

## 實際驗證

### Step 1：讀取暴露檔案

```cmd
type "C:\FEI-PrivEsc\Training\W07\Backups\deploy-config.ini.bak"
```

```ini
; FEI Lab Deployment Configuration
[credentials]
; Service account for automated deployment
username = fei-w07-admin
password = FEI-W07-Admin-Training-2026!
```

### Step 2：確認直接讀 Flag 被拒

```cmd
type "C:\FEI-PrivEsc\Flags\fei-w07-admin-flag.txt"
```

```
存取被拒。
```

### Step 3：切換身份

```cmd
runas /user:fei-w07-admin cmd.exe
```

輸入密碼：`FEI-W07-Admin-Training-2026!`

### Step 4：確認新身份

```cmd
whoami
```

```
fei-priv-win\fei-w07-admin
```

### Step 5：讀取 Flag

```cmd
type "C:\FEI-PrivEsc\Flags\fei-w07-admin-flag.txt"
```

```
FEI{WINDOWS_W07_CREDENTIAL_EXPOSURE_ADMIN_ACCESS}
```

---

## Attack Path

![Windows Credential Exposure Attack Path](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day22-windows-credentials-diagram-02.png)



---

## 為什麼這題的目標不是 SYSTEM？

這是**刻意的教學設計**。

前面 W01-W06 每題最後都是 SYSTEM。但提權的定義不是「一定要到 SYSTEM」：

```
Privilege Escalation
= 跨越原本的權限邊界
≠ 一定要到 SYSTEM
```

`fei-student → fei-w07-admin` 已經是真正的提權：

- fei-student 不能讀 Flag → fei-w07-admin 可以讀
- fei-student 不是 Administrators → fei-w07-admin 是
- 這就是跨越了一條 Security Boundary

---

## Edge Case：runas 與 UAC

在 Windows 10 上，即使你 `runas` 切換到 Administrators 成員，新的 cmd.exe 可能仍然是 Medium Integrity（因為 UAC Split Token）。

```cmd
whoami /groups | findstr "Mandatory"
```

如果看到 `Medium Mandatory Level`，代表 Token 沒有完全提升。但在本 Lab 中，Flag 的 ACL 是設給特定使用者（fei-w07-admin），不是 Administrators 群組，所以 Medium Token 就夠用了。

---

## Troubleshooting

| 問題 | 原因 | 解決 |
|------|------|------|
| findstr 沒結果 | 路徑或檔案類型錯誤 | 用 `/s` 遞迴搜尋，確認路徑 |
| runas 密碼錯 | 拼字錯誤 | 仔細對照大小寫和特殊字元 |
| runas 後 whoami 不是目標 | 在原始 cmd 執行 | 確認在 runas 開啟的新 cmd 裡執行 |
| Flag 仍然 Access Denied | Token 問題 | 確認 `whoami` 確實是 fei-w07-admin |

---

## 深入一層：為什麼 Decoy 很重要

在真實滲透測試中，你會找到大量「看起來像密碼」的字串。大部分是：

1. **預設值**：`changeme`、`password`、`admin`
2. **範例**：文件中的 placeholder
3. **已停用**：帳號已刪除或密碼已更換
4. **不同服務**：資料庫密碼，但你在 OS 提權，用不到

判斷 Credential 是否有用的 checklist：

- [ ] 有明確的 username + password 配對？
- [ ] Username 在目標系統存在？
- [ ] Password 目前有效（能認證）？
- [ ] 目標帳號權限比目前高？
- [ ] 有認證介面可用（runas、SSH、RDP）？

---

## FEI Lab 工程筆記

### Credential Verify 限制

W07 的 verify 腳本在 SYSTEM context（schtask）下用 `Start-Process -Credential` 驗證密碼，但這個方法在非互動 session 中會失敗。這是已知限制，不影響場景功能。

### Flag ACL 設計

Flag 刻意只允許 SYSTEM + fei-w07-admin（不是 Administrators 群組），確保學員必須真正切換身份才能讀取，不能用其他 admin 路徑繞過。

---

## 防禦者怎麼看？

### Detect

```powershell
# 搜尋潛在 Credential 暴露
Get-ChildItem C:\ -Recurse -Include *.config,*.ini,*.bak,*.old,*.ps1,*.bat -ErrorAction SilentlyContinue |
    Select-String -Pattern "password\s*=" -SimpleMatch |
    Select-Object Path, LineNumber, Line
```

### Prevent

1. **不要在明文設定檔中儲存密碼**
2. 使用 Secret Management（Windows Credential Manager、Azure Key Vault、HashiCorp Vault）
3. 刪除過時的備份和部署腳本
4. 限制 Training/Deployment 目錄的讀取權限

### Fix Verification

```
刪除暴露的檔案
≠
修復完成

還必須：
1. Rotation（更換密碼）
2. 審計（密碼是否被使用過）
3. 移除所有副本（backup, history）
```

---

## Detection

- **Sysmon Event ID 1**：Process Create with `findstr` + `password` pattern
- **PowerShell Script Block Logging**：搜尋包含 credential-hunting 關鍵字的指令
- **Windows Security Event 4648**：Explicit credential logon（runas）

---

## Exercise

1. 在你自己的工作目錄中搜尋 `password`、`secret`、`key`。有多少結果？其中有真正有效的 credential 嗎？
2. 設計一個比 `changeme` 更容易騙過自動化掃描器的 decoy credential
3. 如果找到了 credential，但 `runas` 不可用，還有什麼方式可以用這組密碼？

---

## Quiz

**Q1**：找到 `password=P@ssw0rd` 就代表可以提權嗎？

### 答案
不一定。還需要確認：哪個帳號？帳號存在嗎？密碼有效嗎？權限更高嗎？有認證介面嗎？


**Q2**：Credential Hunting 和 Credential Dumping 最大的差異是什麼？

### 答案
Hunting 是在檔案系統中搜尋明文密碼（低權限即可）；Dumping 是從記憶體或系統元件提取 hash/credential（通常需要 admin/SYSTEM）。


**Q3**：為什麼「刪除含密碼的檔案」不算完整修復？

### 答案
因為密碼已經暴露（可能已被讀取）。完整修復必須包含：刪除檔案 + 更換密碼（Rotation）+ 審計是否已被利用。




## 場景收尾

做完後記得 reset：

`powershell
# 以 fei-labadmin 的系統管理員 PowerShell
cd C:\FEI-PrivEsc\Scenarios\FEI-W07-CREDENTIALS
.\fei-reset.ps1
.\fei-verify.ps1 -Mode reset
# 應看到 W07-CREDENTIALS STATUS: RESET
`

---

## 今天真正要記住的 3 件事

1. **提權不一定需要漏洞** — 找到暴露的高權限 credential 就是提權
2. **Finding ≠ Valid** — 找到密碼字串後，還要驗證帳號、有效性、權限
3. **目標不一定是 SYSTEM** — Privilege Escalation = 跨越權限邊界

---

## 給自己的問題

> 如果你的帳號不能 sudo、不能改 Service、不能改 Binary、不能改 Registry、找不到密碼……但你的 Token 裡有一個特殊的 Privilege，那又怎樣？

---

## 下一篇

Day 23：`whoami /priv` 不只是清單 —— SeBackupPrivilege 到底給了什麼能力？
