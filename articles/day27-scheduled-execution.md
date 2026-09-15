# Day 27：Cron vs Scheduled Task — 排程真正危險的是「誰執行了什麼」

## 背景知識

Day 08 教了 Linux Cron（L04），Day 19 教了 Windows Scheduled Task（W04）。

兩者都是「排程執行」，但底層架構完全不同。今天不是重複這兩題，而是拉高到一個共同問題：

> 當一個系統自動排程以高權限執行某個動作，**低權限使用者能控制它實際執行的內容嗎？**

---

## OS 原理：Linux cron

### cron daemon 的執行流程

![cron daemon 的執行流程](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day27-scheduled-execution-diagram-01.png)


關鍵欄位（/etc/cron.d/ 格式）：

```
分 時 日 月 週 使用者 命令
*  *  *  *  *  root   /bin/bash /opt/.../fei-maintenance.sh
```

### User Crontab vs System Cron

| 特性 | User Crontab | System Cron (/etc/cron.d/) |
|------|-------------|---------------------------|
| 位置 | /var/spool/cron/crontabs/<user> | /etc/cron.d/<file> |
| 格式 | 沒有 user 欄位 | 有 user 欄位 |
| 執行身份 | crontab 擁有者 | 指定的 user |
| 編輯方式 | `crontab -e` | root 直接編輯 |
| 重點 | 使用者自己的排程 | 系統級排程（通常 root）|

### FEI Lab 實測（L04）

```
$ cat /etc/cron.d/fei-l04-maintenance
# FEI PrivEsc Lab — FEI-L04-CRON Training
* * * * * root /bin/bash /opt/fei-privesc/training/L04/fei-maintenance.sh

$ ls -la /opt/fei-privesc/training/L04/fei-maintenance.sh
-rwxrwxrwx 1 root root 464 ...    # ← 0777！任何人都能改！

$ ls -la /etc/cron.d/fei-l04-maintenance
-rw-r--r-- 1 root root ...         # ← cron definition 本身安全
```

**漏洞模型**：
- Cron Definition 安全（root:root 644）
- Executed Script 不安全（0777）
- 修改 Script → 等待 cron 自動執行 → root 執行攻擊者內容

---

## OS 原理：Windows Task Scheduler

### Task Scheduler 架構

![Task Scheduler 架構](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day27-scheduled-execution-diagram-02.png)


### Task 的四大組成

| 組成 | 說明 | 提權相關 |
|------|------|---------|
| **Trigger** | 什麼時候執行（時間、事件、登入） | 攻擊者需要知道何時觸發 |
| **Action** | 執行什麼（exe、script、cmd） | **如果 action target 可控 → 提權** |
| **Principal** | 以誰的身份執行（SYSTEM、特定使用者） | 決定提權目標 |
| **Settings** | 其他設定（逾時、重試） | 通常不影響提權 |

### FEI Lab 實測（W04）

```
C:\> schtasks /query /tn "\FEI\FEI-W04-Maintenance" /fo LIST
TaskName:    \FEI\FEI-W04-Maintenance
Run As User: SYSTEM
Schedule:    每 1 分鐘

C:\> icacls "C:\FEI-PrivEsc\Training\W04\fei-maintenance.cmd"
BUILTIN\Users:(M)    ← 可修改！

C:\> echo type "C:\FEI-PrivEsc\Flags\fei-w04-system-flag.txt" ^> "C:\Users\Public\fei-w04-proof.txt" >> "C:\FEI-PrivEsc\Training\W04\fei-maintenance.cmd"
```

等待 60 秒後：

```
C:\> type "C:\Users\Public\fei-w04-proof.txt"
FEI{WINDOWS_W04_SCHEDULED_TASK_SYSTEM_ACCESS}
```

---

## 共同 Mental Model

![Scheduled Execution 共同 Mental Model](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day27-scheduled-execution-diagram-03.png)


### 跨平台比較表

| 維度 | Linux (L04) | Windows (W04) |
|------|-------------|---------------|
| **排程系統** | cron daemon | Task Scheduler Service |
| **排程定義** | /etc/cron.d/fei-l04-maintenance | \FEI\FEI-W04-Maintenance |
| **執行身份** | root | SYSTEM |
| **執行目標** | /bin/bash fei-maintenance.sh | cmd.exe /c fei-maintenance.cmd |
| **定義安全嗎？** | ✅ root:root 644 | ✅ SYSTEM task |
| **目標安全嗎？** | ❌ 0777 (world-writable) | ❌ Users:(M) |
| **攻擊方式** | 修改 .sh → 等 60s | 修改 .cmd → 等 60s |
| **Flag** | `FEI{LINUX_L04_CRON_ROOT_ACCESS}` | `FEI{WINDOWS_W04_SCHEDULED_TASK_SYSTEM_ACCESS}` |

---

## Attack Path 對照

### Linux L04

![Linux L04 攻擊路徑](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day27-scheduled-execution-diagram-04.png)


### Windows W04

![Windows W04 攻擊路徑](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day27-scheduled-execution-diagram-05.png)


---

## False Positive 分析

### Linux Cron False Positives

1. **Cron Job 存在但 Script 不可寫**
   - Cron 以 root 執行但 Script 是 root:root 755
   - fei-student 只能讀不能改 → 不成立

2. **Script 可寫但 Cron 以低權限使用者執行**
   - 即使能改 Script，執行身份和目前使用者相同 → 沒有提權效果

3. **Cron 定義檔案看起來可修改**
   - `/etc/cron.d/` 目錄本身通常是 root:root 755
   - 即使個別 cron 檔案意外可寫，修改 cron definition 是更嚴重的問題（不只是「script writable」）

### Windows Task False Positives

1. **Task 存在但 Action Target 不可寫**
   - 常見場景：Task Action 指向 `C:\Windows\System32\` 下的檔案
   - 標準使用者無法修改 → 不成立

2. **Action Target 可寫但 Task 不是 SYSTEM**
   - 如果 Task Run As 是普通使用者 → 沒有提權效果

3. **Task 可以被 `/run` 但學員不確定是否有 SYSTEM identity**
   - 需要確認 `Run As User` 和 Actual Process Identity

---

## Edge Case

### Linux：cron PATH 行為

cron 執行 Job 時的預設 PATH 非常受限：

```
PATH=/usr/bin:/bin
```

這和互動 Shell 的 PATH 不同。如果 cron job 的 Script 依賴某個不在 `/usr/bin:/bin` 的工具，可能會失敗。但這不影響 L04 的攻擊——因為 Script 內容完全由攻擊者控制。

### Windows：Task Scheduler 的 `/run` 權限

schtasks `/run` 需要 Task 本身允許該使用者 Run。SYSTEM task 預設不允許 Standard User `/run`。

在 L04 中，fei-student 不需要手動觸發 cron——它每分鐘自動跑。同樣在 W04 中，Task 每分鐘自動觸發。因此 `/run` 權限不是必要的。

### 共同 Edge Case：競爭條件

修改 Script 和 cron/Task 執行之間可能存在競爭。如果：
1. 攻擊者正在寫入 Script
2. cron/Task 同時開始執行

可能導致 Script 只被部分修改。在 Lab 環境中這不太可能發生（1 分鐘間隔足夠），但在生產環境中可能需要原子性操作。

---

## Troubleshooting

### Linux

| 問題 | 原因 | 解法 |
|------|------|------|
| 修改 Script 後等了 2 分鐘還沒結果 | cron daemon 未運行 | `systemctl status cron` |
| Script 被執行但沒有寫入 /tmp | Script 語法錯誤 | 先在 shell 測試 Script |
| Permission denied 寫 Script | Script 不是 world-writable | 確認 `ls -la` |
| cron log 找不到 | Ubuntu 22.04 用 systemd-journal | `journalctl -u cron` |

### Windows

| 問題 | 原因 | 解法 |
|------|------|------|
| Task 沒有觸發 | Task Scheduler Service 未運行 | `Get-Service Schedule` |
| cmd 修改後 proof 沒出現 | cmd 語法錯誤（`>` 要用 `^>`）| 先在 cmd 測試 |
| `schtasks /run` 被拒 | Standard User 不能 run SYSTEM task | 等待自動觸發 |
| proof 檔案是空的 | cmd 編碼問題 | 確認 append 語法 |

---

## Defense 防禦

### Linux

```
Detect:
  # 找出 root cron 引用的所有 script
  grep -rh "root" /etc/cron.d/ /etc/crontab | grep -v "^#" | awk '{print $NF}'
  
  # 檢查這些 script 的權限
  for f in $(above); do ls -la "$f"; done

Prevent:
  # Cron 引用的 Script 必須 root:root 700 或 750
  chmod 700 /path/to/maintenance.sh
  chown root:root /path/to/maintenance.sh

Verify:
  # 修復後確認：fei-student 不能修改
  su - fei-student -c "echo test >> /path/to/maintenance.sh"
  # 應該 Permission denied
```

### Windows

```
Detect:
  # 找出 SYSTEM Task 的 Action Target
  Get-ScheduledTask | Where-Object { $_.Principal.UserId -eq 'SYSTEM' } |
    ForEach-Object { $_.Actions } | Select-Object Execute, Arguments

  # 檢查每個 Action Target 的 ACL
  icacls "target_path"

Prevent:
  # Action Target 必須只有 Administrators/SYSTEM 可寫
  icacls "target.cmd" /inheritance:r /grant "Administrators:(F)" /grant "SYSTEM:(F)"

Verify:
  # 以 fei-student 嘗試修改 — 應該被拒
  echo test >> "target.cmd"
  # 應該 Access Denied
```

---

## Detection 偵測

### Linux

| 偵測點 | 方法 |
|--------|------|
| cron 引用的 Script 被修改 | `inotifywait` / AIDE / Tripwire |
| 新的 cron 定義出現 | 監控 `/etc/cron.d/` 的 inode 變化 |
| cron 執行異常命令 | `auditd` 追蹤 cron fork 的子 Process |

### Windows

| 偵測點 | Event ID |
|--------|----------|
| Task 建立/修改 | 4698 (Task Created) / 4702 (Task Updated) |
| Task 執行 | 200 (Action started) / 201 (Action completed) |
| 可疑 Process 由 SYSTEM 啟動 | Sysmon Event ID 1 (parent = svchost -k netsvcs) |
| Action Target 被修改 | Sysmon Event ID 2 (FileCreationTime changed) |

---

## Fix Verification 修復驗證

### Linux — 修復後完整驗證

```bash
# 1. Script 權限正確
ls -la /opt/.../fei-maintenance.sh
# 應該是 -rwx------ root root 或 -rwxr-x--- root root

# 2. fei-student 不能修改
su - fei-student -c "echo hack >> /opt/.../fei-maintenance.sh" 2>&1
# Permission denied

# 3. cron 仍然正常執行（功能不受影響）
tail -f /var/log/syslog | grep CRON
# 應該看到正常 cron 執行
```

### Windows — 修復後完整驗證

```powershell
# 1. Action Target ACL 正確
icacls "C:\...\fei-maintenance.cmd"
# 只有 SYSTEM:(F) 和 Administrators:(F)

# 2. fei-student 不能修改
# 以 fei-student 嘗試
echo test >> "C:\...\fei-maintenance.cmd"
# Access Denied

# 3. Task 仍然正常觸發
Get-ScheduledTaskInfo -TaskName "\FEI\FEI-W04-Maintenance"
# LastRunTime 應該在最近 1 分鐘內
```

---

## Exercise 練習

### 練習 1：列出你系統上的所有排程

**Linux：**

```bash
# 系統 cron
ls -la /etc/cron.d/
cat /etc/crontab

# 使用者 crontab
crontab -l

# 對每個 root cron 引用的 script，檢查權限
```

**Windows：**

```powershell
# 所有 SYSTEM Task
Get-ScheduledTask | Where-Object { $_.Principal.UserId -eq 'SYSTEM' } |
  Select-Object TaskName, State | Format-Table

# 每個 Task 的 Action
Get-ScheduledTask -TaskName "*" | ForEach-Object {
  [PSCustomObject]@{
    Task = $_.TaskName
    RunAs = $_.Principal.UserId
    Action = ($_.Actions | Select-Object -First 1).Execute
  }
}
```

### 練習 2：思考題

如果 cron 每分鐘以 root 執行一個 Script，但那個 Script 裡面 `source /etc/app.conf`（讀取 config），而 `/etc/app.conf` 是 world-writable... 這和 L04 有什麼相似之處？和 L06 呢？

### 練習 3：偵測練習

在 FEI Lab 中：
1. Setup L04
2. 用 `auditd` 或 `journalctl -u cron` 觀察 cron 的執行
3. 修改 Script
4. 觀察 log 中是否能看出「內容被改過」

> 思考：防禦者有什麼方法在 cron 執行前偵測到 Script 被修改？

---

## Quiz 測驗

**Q1**: Linux cron 的 `/etc/cron.d/` 檔案和 user crontab 的格式差異是什麼？

### 答案`/etc/cron.d/` 格式比 user crontab 多了一個 **使用者欄位**（指定以誰的身份執行）。User crontab 預設以 crontab 擁有者身份執行。

**Q2**: 在 W04 中，fei-student 為什麼不需要 `schtasks /run` 權限？

### 答案Task 設定為每分鐘自動觸發。fei-student 只需要修改 Action Target 然後等待，不需要手動觸發 Task。

**Q3**: L04 和 L06 都涉及「root 執行 student 可修改的資源」，兩者的差異是什麼？

### 答案
- L04：觸發機制是 **cron**（自動定期），目標是 **Script**
- L06：觸發機制是 **sudo runner**（手動），目標是 **Configuration**
兩者共同點：高權限程式信任低權限使用者可控制的資源。


**Q4**: 如果 Cron Definition 本身也可以被 fei-student 修改，和 L04 有什麼不同？

### 答案如果 cron definition 本身可修改，攻擊者可以直接改排程內容（例如改成 `* * * * * root /bin/bash -c "cat flag > /tmp/out"`）。這比 L04 更嚴重，因為連 Script 都不需要存在。L04 的設計是：Definition 安全但 Executed Target 不安全。

---

## Engineering Note 工程筆記

L04 和 W04 都是 **Phase 6 零 bug 通過**的場景。

值得記錄的設計決策：

### L04 — 選擇 `/etc/cron.d/` 而非 user crontab

設計時考慮過用 `crontab -e` 建立 root 的 user crontab。但 `/etc/cron.d/` 更常見於：
- 套件安裝後自動建立的 maintenance job
- 部署工具自動產生的排程

因此選擇 `/etc/cron.d/` 更貼近真實場景。

### W04 — schtasks /run 的 UAC 限制

在自動化測試時發現：Standard User 無法 `schtasks /run` SYSTEM Task。這不是 bug — 這是正確的安全行為。因此 W04 設計為「每分鐘自動觸發」，學員不需要手動 `/run`。

在 Instructor Guide 中特別標注了 cmd 語法的 `^>` escape（在 cmd 的 echo 中，`>` 需要用 `^>` 避免被解析為重導向）。

---

## 今天真正要記住的 3 件事

1. **排程本身不是漏洞。漏洞在於「高權限排程執行了低權限使用者可控制的資源」**
2. **cron 和 Task Scheduler 的核心問題相同：Schedule Definition 安全 ≠ Executed Resource 安全**
3. **防禦的重點不只是保護排程定義，還要保護排程引用的每一個檔案、Script、Executable**

---

## 給自己的問題

> 如果一個 SYSTEM Scheduled Task 執行的是一個固定路徑、不可修改的 EXE，但那個 EXE 載入了一個攻擊者可以控制的 DLL... 這和 Day 24（W09 DLL Hijacking）是什麼關係？排程觸發和 DLL Loading 可以組合成更複雜的攻擊鏈嗎？

---

## 下一篇

Day 28 — 帳號名稱不代表真正權限：從 Capabilities、Dangerous Group 到 Windows Token，理解什麼才是「Effective Privilege」。
