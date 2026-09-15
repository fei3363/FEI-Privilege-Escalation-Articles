# Day 02：root、Administrator、SYSTEM 到底差在哪？


---

## 背景知識

### 為什麼要搞清楚這些？

在滲透測試報告中，你會看到「取得 root」或「取得 SYSTEM」。在 CTF 中，flag 通常放在 root 才能讀的地方。但如果有人問你：

> 「root 和 SYSTEM 是不是一樣的東西？」

你會怎麼回答？

如果你的回答是「差不多啦，都是最高權限」，那今天這篇會讓你重新理解。

### 權限模型的歷史脈絡

Unix（Linux 的前身）誕生於 1970 年代。它的權限模型從一開始就圍繞「使用者」和「群組」設計。root（UID 0）是系統管理員，可以做一切事情。簡單、直接、有效——也很危險。

Windows NT 誕生於 1993 年。它一開始就引入了更複雜的安全模型：Access Token、SID、Integrity Level、Mandatory Access Control。這不是因為 Windows 更安全，而是因為 Windows NT 設計時就考慮到多使用者企業環境。

兩者的哲學差異：
- **Linux**：預設信任 root 可以做一切
- **Windows**：即使是 Administrator 也有層層限制（UAC、Integrity Level）

---

## OS 原理（Linux）：UID 的世界

### Process Identity Chain

![Process Identity Chain](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day02-root-admin-system-diagram-01.png)


每個 Linux Process 都攜帶一組身分識別：


#### Real UID vs Effective UID

這是 Linux 權限模型中最重要的區別：

- **Real UID**：誰創造了這個 process。一般不會改變。
- **Effective UID**：Kernel 在做存取檢查時真正看的 UID。

大多數時候 RUID = EUID。但有些機制可以讓它們不同：

| 機制 | RUID | EUID | 說明 |
|------|------|------|------|
| 正常執行 | 1001 | 1001 | RUID = EUID |
| SUID binary | 1001 | 0 | EUID 變成檔案 owner |
| setuid() | 1001 | 0 | Process 自己改變 EUID |
| sudo | 1001 | 0 | sudo 設定新的 EUID |

### root (UID 0) 能做什麼？

```bash
fei-student@fei-privesc-linux:~$ id
uid=1001(fei-student) gid=1001(fei-student) groups=1001(fei-student)
```

fei-student 是 UID 1001。以下是 UID 0 (root) 可以額外做的事：

| 能力 | 說明 |
|------|------|
| **繞過檔案權限** | 讀寫任何檔案，不管 owner/group/other 設定 |
| **修改任何 Process** | kill 任何 process、attach debugger |
| **網路特權** | 綁定 1024 以下的 port、raw socket |
| **帳號管理** | 新增/刪除使用者、修改密碼 |
| **Kernel 操作** | 載入/卸載 module、mount filesystem |
| **系統設定** | 修改 hostname、network config、sysctl |

### Linux Capabilities：root 能力的拆分

從 Kernel 2.6.26 開始，Linux 把 root 的能力拆成約 40 個獨立的 Capabilities：

```
CAP_SETUID        — 可以改變 UID
CAP_NET_RAW       — 可以使用 raw socket
CAP_DAC_OVERRIDE  — 可以繞過檔案權限
CAP_SYS_ADMIN     — 超級雜項能力
...
```

這意味著一個 Process 可以「不是 root，但擁有 root 的某些能力」。Day 09 會深入探討。

---

## OS 原理（Windows）：Token 的世界

### Access Token 結構

![Access Token 結構](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day02-root-admin-system-diagram-02.png)


Windows 的權限不像 Linux 那樣只看 UID。它看的是一個複雜的 **Access Token**：


以上是 `fei-student` 在 FEI Lab 的實際 Token 內容（從 `whoami /all` 取得）。

### User SID

每個 Windows 帳號有一個唯一的 SID（Security Identifier）：

```
fei-student: S-1-5-21-162171792-3566788465-3636827614-1002
```

格式：`S-1-5-21-<machine/domain ID>-<RID>`

常見的 Well-Known SID：

| SID | 名稱 | 說明 |
|-----|------|------|
| S-1-5-18 | SYSTEM | OS 核心身分 |
| S-1-5-32-544 | Administrators | 管理員群組 |
| S-1-5-32-545 | Users | 一般使用者群組 |
| S-1-1-0 | Everyone | 所有人 |

### Integrity Level

Windows Vista 引入的 Mandatory Integrity Control：

```
System   (S-1-16-16384) — SYSTEM Process
High     (S-1-16-12288) — Elevated Administrator Process
Medium   (S-1-16-8192)  — Standard User Process ← fei-student 在這裡
Low      (S-1-16-4096)  — 受限 Process（瀏覽器沙箱等）
```

即使一個帳號在 Administrators 群組，UAC 也會給它一個 **Medium Integrity** 的 filtered token。只有在「以系統管理員身分執行」時才會拿到 High Integrity token。

### Privileges：Token 裡的特殊能力

Privileges 是 Windows 給 Process 的額外能力，類似 Linux 的 Capabilities：

| Privilege | 說明 | 危險等級 |
|-----------|------|---------|
| SeBackupPrivilege | 用 backup semantics 讀任何檔案 | 高 |
| SeRestorePrivilege | 用 restore semantics 寫任何檔案 | 高 |
| SeImpersonatePrivilege | 冒充其他使用者的 token | 高 |
| SeDebugPrivilege | Debug 任何 process | 極高 |
| SeShutdownPrivilege | 關機 | 低 |
| SeChangeNotifyPrivilege | 略過周遊檢查 | 低 |

fei-student 只有 5 個低風險的 Privilege。但如果某個 Process 的 Token 包含 `SeBackupPrivilege`，即使不是 Administrator，也可能讀到不該看的東西（Day 23 會實際示範）。

---

## Lab：實際比較 fei-student 的身分

### Linux

```bash
fei-student@fei-privesc-linux:~$ id
uid=1001(fei-student) gid=1001(fei-student) groups=1001(fei-student)
```

解讀：
- UID 1001 → 不是 root (0)
- 只屬於自己的群組 → 沒有 sudo、docker、disk 等特權群組
- 沒有額外 Capabilities

### Windows

```
fei-priv-win\fei-student
SID: S-1-5-21-162171792-3566788465-3636827614-1002

Groups:
  Everyone, BUILTIN\Users, Authenticated Users
  Mandatory Label: Medium Mandatory Level

Privileges:
  SeShutdownPrivilege (Disabled)
  SeChangeNotifyPrivilege (Enabled)
  SeUndockPrivilege (Disabled)
  SeIncreaseWorkingSetPrivilege (Disabled)
  SeTimeZonePrivilege (Disabled)
```

解讀：
- 不在 Administrators 群組
- Integrity Level = Medium（標準使用者）
- 只有標準低風險 Privileges
- 沒有 SeBackupPrivilege、SeImpersonatePrivilege 等危險 Privileges

### 比較表

| 屬性 | Linux fei-student | Windows fei-student |
|------|-------------------|---------------------|
| 身分識別 | UID 1001 | SID S-1-5-21-...-1002 |
| 群組 | fei-student (1001) | Users (S-1-5-32-545) |
| 特權群組 | 無 | 無 |
| 特殊能力 | 無 Capabilities | 5 個低風險 Privileges |
| 存取 root flag | Permission denied | 存取被拒 |
| 等級 | Standard User | Medium Integrity |

---

## Attack Reasoning：拿到低權限後的思考

面對上面的身分狀態，攻擊者的第一個問題是：

```
「這個帳號看起來很普通。
 但 OS 有沒有在某個地方信任了它不該信任的東西？」
```

可能的方向：

```
Linux:
  sudo misconfiguration?
  SUID binary?
  Writable cron script?
  Dangerous group?
  Exposed credential?

Windows:
  Weak Service ACL?
  Writable Service Binary?
  Unquoted Service Path?
  AlwaysInstallElevated?
  Token Privileges?
```

這就是接下來 28 天要一一探索的。

---

## Actual Output：Administrator vs SYSTEM 的差異

![Administrator vs SYSTEM 的差異](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day02-root-admin-system-diagram-03.png)


在 Windows 上，`fei-labadmin` 是 Administrators 群組的成員：

```
net localgroup Administrators

Members:
  Administrator
  fei-labadmin
```

但如果你用 `fei-labadmin` 開一個普通的 cmd.exe（不是「以系統管理員身分執行」），它的 token 仍然是 **Medium Integrity**。UAC 做了 token splitting：


所以「在 Administrators 群組 ≠ 擁有 Administrator 權限」。這是 Windows 提權中非常重要的觀念。

而 SYSTEM 完全不同：
- 它不是使用者帳號
- 它是 OS 核心服務的身分
- 它的 Integrity Level 是 System（最高）
- 它不受 UAC 限制
- 大多數 Windows Service 以 SYSTEM 身分執行

---

## False Positive

| 常見誤判 | 真相 |
|---------|------|
| 「fei-labadmin 在 Administrators，所以已經有最高權限」 | 錯。UAC 讓它用 Medium Integrity 的 filtered token |
| 「root 和 SYSTEM 是一樣的」 | 不完全。root 是使用者帳號（UID 0），SYSTEM 是 OS 身分 |
| 「Windows 的 Administrator 等於 Linux 的 root」 | 概念類似但機制完全不同。root 不受 UAC 限制 |
| 「知道 Administrator 密碼就等於有最高權限」 | 要看 UAC 設定和 Integrity Level |

---

## Edge Case

### UAC 對 Administrator 的影響

在 FEI Lab W08 的測試中，我們發現：

- `SeBackupPrivilege` 在 token 中顯示為 Disabled
- 即使用 `AdjustTokenPrivileges` API 嘗試啟用，也回傳 error 1300
- 原因：UAC token splitting 讓 Backup Operators 群組的 privilege 變成永久停用
- 修正：停用 `EnableLUA`（僅限 Lab 環境）

這在 Windows 10 Build 19045 上是確認的行為。不同 Windows 版本可能不同。

### Linux Namespace

在 Container 環境中，`id` 可能顯示 `uid=0(root)`，但這個 root 可能被 user namespace 限制在容器內，實際上在 Host 上是 UID 1000。

---

## Troubleshooting

| 問題 | 解法 |
|------|------|
| `whoami` 顯示 root 但不能讀 Host 檔案 | 你在 container 內，不是真正的 Host root |
| `whoami /groups` 顯示 Administrators 但 UAC 阻擋 | 用「以系統管理員身分執行」開 cmd |
| `id` 顯示在 docker 群組但 docker 指令失敗 | 需要重新登入（新 session）才能取得新 group token |
| Windows `whoami /priv` 沒有顯示 SeBackupPrivilege | Privilege 可能需要透過 Group Policy 設定，且需重新登入 |

---

## Defense

### 最小權限原則

```
Linux:
- 不要把使用者加入不必要的群組（sudo、docker、disk）
- sudoers 只開放必要的指令
- 使用 Capabilities 替代 SUID

Windows:
- 不要把使用者加入 Administrators
- 使用 Standard User 日常工作
- 保持 UAC 啟用
- 謹慎分配 User Rights（特別是 SeBackupPrivilege、SeImpersonatePrivilege）
```

### 監控

```
Linux:
- 監控 /var/log/auth.log 的 sudo 使用
- 監控 setuid()/setgid() 系統呼叫
- auditd 規則

Windows:
- 監控 Event ID 4672 (Special Logon)
- 監控 Event ID 4688 (Process Creation) 配合 Command Line
- 監控 Service 建立 (Event ID 7045)
```

---

## Detection

藍隊偵測提權的關鍵指標：

| 指標 | 說明 |
|------|------|
| **Child Process 權限 > Parent Process** | 低權限 Process 產生高權限子 Process |
| **異常 sudo 使用** | 非正常工時、異常 command |
| **Token 操作** | AdjustTokenPrivileges、ImpersonateLoggedOnUser |
| **Service 異常** | 新建 Service、Service 執行異常 binary |
| **SUID/Capability 變更** | 新增 SUID bit 或 Capability |

---

## Fix Verification

確認 least privilege 設定正確：

```bash
# Linux：確認 fei-student 的完整身分
id fei-student
# 預期：只有自己的群組

# 確認沒有不必要的 sudo 權限
sudo -l -U fei-student
# 預期：User fei-student is not allowed to run sudo

# 確認沒有意外的 SUID
find / -perm -4000 -user root -type f 2>/dev/null | grep -v '/usr/'
# 預期：沒有非系統的 SUID binary
```

```powershell
# Windows：確認 fei-student 的群組
net localgroup Administrators
# 預期：不包含 fei-student

# 確認 Token Privileges
whoami /priv
# 預期：只有標準低風險 Privileges
```

---

## Exercise

1. 在你的 Linux 系統上，執行 `id` 和 `groups`。你屬於哪些群組？其中有沒有可能是「危險群組」（docker、lxd、disk、libvirt）？

2. 在你的 Windows 系統上，執行 `whoami /all`。觀察你的 Token 裡有多少 Privileges。找出其中最危險的一個，查詢它的用途。

3. 比較：一個 UID 0 但在 container 內的 Process，和一個 UID 1000 但在 Host 上的 Process，哪個實際權限更大？為什麼？

---

## Quiz

**Q1**：以下關於 Linux UID 的描述，哪個正確？
- (A) Real UID 用於存取檢查
- (B) Effective UID 用於存取檢查
- (C) UID 和 GID 是一樣的東西
- (D) root 的 UID 是 1

### 答案(B) Effective UID 是 Kernel 在存取檢查時實際查看的。

**Q2**：Windows 的 SYSTEM 和 Administrator 的關係？
- (A) SYSTEM 就是 Administrator
- (B) SYSTEM 是 OS 核心身分，不是使用者帳號
- (C) Administrator 比 SYSTEM 權限更高
- (D) SYSTEM 不能做 Administrator 能做的事

### 答案(B) SYSTEM (S-1-5-18) 是 OS 本身的身分，比 Administrator 權限更高。

**Q3**：UAC (User Account Control) 做了什麼？
- (A) 禁止所有 Administrator 操作
- (B) 將 Administrator 的 token 分成 filtered 和 full 兩個
- (C) 只影響 SYSTEM
- (D) 只在 Windows Server 上有效

### 答案(B) UAC 建立 token splitting，預設使用 filtered (Medium) token。

**Q4**：Linux 的 Capabilities 目的是？
- (A) 讓 root 更強大
- (B) 將 root 的能力拆成細粒度的獨立能力
- (C) 替代 UID 系統
- (D) 只給 Container 使用

### 答案(B) Capabilities 把 root all-or-nothing 的特權拆成約 40 個獨立能力。

**Q5**：fei-student 在 Windows 上的 Integrity Level 是？
- (A) Low
- (B) Medium
- (C) High
- (D) System

### 答案(B) Standard User 的預設 Integrity Level 是 Medium。

---

## Engineering Note

在 FEI Lab W08 (SeBackupPrivilege) 的建置過程中，我們親身體驗了 UAC token splitting 的影響：

1. **secedit** 設定 User Rights Assignment 時，用 username 而非 SID → 無效 → 改用 **LSA API** (`LsaAddAccountRights`)
2. **Backup Operators 群組**成員的 SeBackupPrivilege 被 UAC token split → 無法 enable → 需停用 `EnableLUA`
3. 自訂 C# helper 的 `AdjustTokenPrivileges` 在 vmrun 非互動 session 中回傳 **error 1300** → 改用 `robocopy /B`（系統工具能正確處理 privilege）

這些都是「Administrator ≠ 想做什麼就做什麼」的真實案例。

---

## 今天真正要記住的 3 件事

1. **Linux 的 root (UID 0) 和 Windows 的 SYSTEM (S-1-5-18) 是不同的東西**。不要硬說等價。

2. **Windows 上的 Administrator 受 UAC 限制**。在 Administrators 群組 ≠ 擁有 High Integrity Token。

3. **帳號名稱不能完整描述有效權限**。真正的權限取決於 UID/EUID、Token 中的 Groups、Privileges、Integrity Level 等多重因素。

---

## 給自己的問題

> 如果一個帳號同時在 Administrators 群組和 Backup Operators 群組，但 UAC 啟用，它打開一個普通的 cmd.exe 時，實際擁有哪些能力？比 fei-student 多多少？

---

## 下一篇

Day 03：我為什麼自己打造 20 個提權環境？——包含完整的環境建置指南、GitHub repo，以及 5 個真實的 Bug Fix 工程筆記。
