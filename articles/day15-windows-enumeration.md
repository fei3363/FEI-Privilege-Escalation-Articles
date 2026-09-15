# Day 15：拿到 Windows Shell 後，不要急著跑工具：先看懂自己的 Token

### 場景準備

```powershell
# 1. 以 fei-labadmin 登入 Windows VM（VMware Console）
# 密碼：FEI-LabAdmin-2026!

# 2. 以系統管理員 PowerShell 執行 setup
cd C:\FEI-PrivEsc\Scenarios\FEI-W00-ENUMERATION
.\fei-setup.ps1

# 3. 確認就緒
.\fei-verify.ps1
# 應看到 FEI-W00 STATUS: READY

# 4. 登出，改以 fei-student 登入
# 密碼：FEI-Student-2026!
```

---

## 開場情境

你在一台 Windows 機器上拿到了 `fei-student` 的 shell。

你的第一反應可能是跑 WinPEAS、PowerUp、或 Seatbelt。

但在跑任何工具之前，你真的知道自己現在是「誰」嗎？

在 Windows 上，「你是誰」不只是 username。你的 **Access Token** 裡裝了你的完整身份：SID、Groups、Privileges、Integrity Level。

---

## 今天要解決的問題

1. Windows Access Token 到底裝了什麼？
2. `whoami /priv` 的每一行代表什麼？
3. Username 為什麼不能完整描述你的有效權限？
4. Integrity Level 怎麼影響你能做什麼？

---

## 背景知識：Windows 身份模型

### User vs Identity

Linux 用 UID/GID 識別身份。Windows 用 **SID**（Security Identifier）。

```
fei-student 的 SID:
S-1-5-21-162171792-3566788465-3636827614-1002

格式：S-{revision}-{authority}-{domain sub-authorities}-{RID}
```

### Access Token

![Windows Access Token](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day15-windows-enumeration-diagram-01.png)


每個 Windows Process 都有一個 Access Token，包含：


**每次存取資源，Windows 都比對這個 Token，不是 Username。**

---

## OS 原理：Token 各組成詳解

### User SID

```
fei-priv-win\fei-student
S-1-5-21-162171792-3566788465-3636827614-1002
```

這是 **你是誰**。

### Group SIDs

```
BUILTIN\Users              S-1-5-32-545     ← 所有普通使用者
Everyone                   S-1-1-0          ← 所有人
NT AUTHORITY\Authenticated Users  S-1-5-11  ← 已認證使用者
```

Group SID 決定你屬於哪些群組。ACL 可以基於 Group SID 授權。

### Privileges

```
SeShutdownPrivilege            已停用
SeChangeNotifyPrivilege        已啟用
SeUndockPrivilege              已停用
SeIncreaseWorkingSetPrivilege  已停用
SeTimeZonePrivilege            已停用
```

Privilege 是 **OS 層級的特殊能力**，和檔案權限不同。

每個 Privilege 有三種狀態：
- **不存在**：Token 中完全沒有
- **Disabled（已停用）**：存在但需要程式主動啟用
- **Enabled（已啟用）**：可以立即使用

### Integrity Level

```
Mandatory Label\Medium Mandatory Level    S-1-16-8192
```

| Level | 數值 | 代表 |
|-------|------|------|
| Low | 4096 | 沙箱、瀏覽器 tab |
| **Medium** | **8192** | **一般使用者 Process** |
| High | 12288 | 提升的 Administrator |
| System | 16384 | OS 核心服務 |

Medium = 一般使用者。你不能寫入 High/System integrity 的 object。

---

## 🔬 FEI Lab 環境

```
VM:         FEI-PRIVESC-WINDOWS
OS:         Windows 10 Pro (Build 19045)
起始帳號:    fei-student（Standard User）
IP:         192.168.77.20
Scenario:   FEI-W00-ENUMERATION
```

---

### cmd vs PowerShell：什麼時候用哪個？

| | cmd | PowerShell |
|---|-----|-----------|
| 開啟方式 | 搜尋「cmd」| 搜尋「powershell」|
| 提權指令 | `sc`, `net`, `reg` | `Get-Service`, `Get-LocalUser` |
| 本系列 | 大多數攻擊指令 | 進階查詢和 Lab 管理 |

本系列以 **cmd** 為主（因為低權限帳號通常更容易開 cmd），PowerShell 用於進階查詢。

---

## 完整攻擊流程（Actual Output）

### 1. 我是誰？

```cmd
C:\Users\fei-student> whoami
fei-priv-win\fei-student
```

### 2. 我的 SID

```cmd
C:\Users\fei-student> whoami /user

USER INFORMATION
----------------
使用者名稱                SID
======================== =============================================
fei-priv-win\fei-student S-1-5-21-162171792-3566788465-3636827614-1002
```

### 3. 我在哪些 Group？

```cmd
C:\Users\fei-student> whoami /groups

GROUP INFORMATION
-----------------
群組名稱                                類型       SID          屬性
====================================== ========== ============ ==========
Everyone                               知名的群組 S-1-1-0      已啟用
BUILTIN\Users                          別名       S-1-5-32-545 已啟用
NT AUTHORITY\INTERACTIVE               知名的群組 S-1-5-4      已啟用
NT AUTHORITY\Authenticated Users       知名的群組 S-1-5-11     已啟用
Mandatory Label\Medium Mandatory Level 標籤       S-1-16-8192
```

注意：**沒有 `Administrators`**。這是 Standard User。

### 4. 我有什麼 Privilege？

```cmd
C:\Users\fei-student> whoami /priv

PRIVILEGES INFORMATION
----------------------
特殊權限名稱                   描述                狀況
============================= ================== ======
SeShutdownPrivilege           關閉系統            已停用
SeChangeNotifyPrivilege       略過周遊檢查        已啟用
SeUndockPrivilege             從擴充座移除電腦    已停用
SeIncreaseWorkingSetPrivilege 增加處理程序工作組  已停用
SeTimeZonePrivilege           變更時區            已停用
```

Standard User 的典型 privilege 集合——沒有危險的 privilege。

### 5. 有誰是 Administrator？

```cmd
C:\Users\fei-student> net localgroup Administrators

別名     Administrators
成員
-------------------------------------------------------------------------------
Administrator
fei-labadmin
命令已經成功完成。
```

`fei-student` **不在** Administrators group。

### 6. 系統資訊

```cmd
C:\Users\fei-student> hostname
FEI-PRIV-WIN

C:\Users\fei-student> ipconfig | findstr IPv4
   IPv4 位址 . . . . . . . . . . . . : 192.168.77.20
```

---

## Windows PrivEsc 初始 Decision Tree

![Windows PrivEsc 初始 Decision Tree](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day15-windows-enumeration-diagram-02.png)



---

## False Positive 分析

**「fei-student 在 Users group，這安全嗎？」**

不一定。Users group 是安全的 **前提是** 沒有其他弱點。但：

- 某些 Service 對 `BUILTIN\Users` 授予了過多權限
- 某些 Registry key 對 Users 有 SetValue
- 某些目錄對 Users 有 CreateFiles

Users group 本身不是問題，但系統上可能有資源對 Users 開放了過多存取。

---

## Edge Case

### UAC Split Token

![UAC Split Token](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day15-windows-enumeration-diagram-03.png)


如果 fei-student 是 Administrator：


Standard User 沒有 Split Token，只有一個 Token。

### Logon Type 影響 Token

| Logon Type | 場景 | Token 特性 |
|-----------|------|-----------|
| Interactive (Type 2) | Console login | 完整 Token |
| Network (Type 3) | SMB | 可能被 filtering |
| Batch (Type 4) | Scheduled Task | 某些 Privilege 受限 |
| Service (Type 5) | Service logon | 完整 Service Token |

本系列 Lab 主要使用 Interactive / Batch logon。

---

## Troubleshooting

| 問題 | 原因 |
|------|------|
| whoami /priv 沒有 SeBackupPrivilege | 不在 Backup Operators 或沒有被 User Rights Assignment |
| whoami /groups 沒顯示 Administrators | 正確 — fei-student 是 Standard User |
| cmd vs PowerShell 輸出不同 | 某些 cmd 指令有中文翻譯，PowerShell 可能不同 |

---

## 防禦觀點

### Detect

```powershell
# 列出所有 Local Administrators
Get-LocalGroupMember -Group Administrators

# 檢查 User Rights Assignment
secedit /export /cfg C:\temp\policy.cfg /areas USER_RIGHTS
```

### Prevent

1. **Standard User 原則**——日常作業不使用 Administrator
2. **定期審計 Local Administrators Group**
3. **監控特權 Token 的建立**

### Fix Verification

```powershell
# 確認某使用者不在 Administrators
(Get-LocalGroupMember Administrators).Name -contains "fei-student"
# 應該是 False
```

---

## Detection

| 活動 | 偵測方式 |
|------|---------|
| `whoami /priv` 執行 | Process monitoring (Sysmon Event 1) |
| `net localgroup` 列舉 | 同上 |
| 異常帳號加入 Administrators | Event ID 4732 |
| Token manipulation | Event ID 4672 (Special logon) |

---

## 深入一層：Token vs ACL — Windows 如何決定「你能不能做某件事」

![Token 與 ACL 存取決策](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day15-windows-enumeration-diagram-04.png)



Token 是「我有什麼」，ACL 是「資源允許誰」。兩者匹配才能存取。

---

## Exercise

### 練習 1

你在一台 Windows 機器上看到：

```
whoami /priv 中有 SeImpersonatePrivilege (Disabled)
```

這代表什麼？能馬上利用嗎？

### 練習 2

列出你會在 Windows 初始 Enumeration 中執行的前 10 個指令（cmd 和 PowerShell 各 5 個），並說明每個指令的目的。

---

## Quiz

**Q1：** Windows 的 `whoami /priv` 顯示某個 Privilege 是「已停用」。這代表 Token 中沒有這個 Privilege 嗎？

### 答案
不是。「已停用」表示 Privilege **存在於** Token 中，但目前沒有被啟用。程式可以呼叫 AdjustTokenPrivileges 來啟用它（如果 Token 允許）。如果 Privilege 完全不在 Token 中，`whoami /priv` 不會顯示它。


**Q2：** 為什麼 Username 不能完整描述有效權限？

### 答案
因為有效權限由 Access Token 決定，Token 包含 User SID + Group SIDs + Privileges + Integrity Level。同一個 Username 在不同 logon type、不同 UAC 狀態、不同 Token 下可能有不同的有效權限。


**Q3：** Medium Integrity Level 的 Process 能寫入 High Integrity 的檔案嗎？

### 答案
不能。Windows Mandatory Integrity Control (MIC) 要求 Process 的 Integrity Level >= Object 的 Integrity Level 才能寫入。Medium < High，所以會被拒絕。




## 場景收尾

做完後記得 reset：

`powershell
# 以 fei-labadmin 的系統管理員 PowerShell
cd C:\FEI-PrivEsc\Scenarios\FEI-W00-ENUMERATION
.\fei-reset.ps1
.\fei-verify.ps1 -Mode reset
# 應看到 W00-ENUMERATION STATUS: RESET
`

---

## 今天真正要記住的 3 件事

1. **Username 只是 Token 的一部分**——Group、Privilege、Integrity Level 才是完整的身份
2. **`whoami /priv` 是 Windows Enumeration 的第一步**——看有沒有危險 Privilege
3. **Standard User 的 Token 是你的起點**——接下來 10 天要找的是「哪個 Trust Boundary 有裂縫」

---

## 給自己的問題

> 如果 fei-student 不是 Administrator，但系統上某個 SYSTEM Service 允許 Users 修改它的設定呢？

---

## 下一篇

Day 16：Windows Service ACL：能控制 Service，就可能控制 SYSTEM 嗎？
