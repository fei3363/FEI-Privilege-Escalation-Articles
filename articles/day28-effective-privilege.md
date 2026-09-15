# Day 28：帳號名稱不代表真正權限 — 從 Capabilities、Group 到 Windows Token

## 背景知識

到了第 28 天，我們已經做過 20 個提權 Lab。你可能已經發現一件事：

> **`whoami` 只告訴你名字。你真正擁有的權限，遠比名字複雜。**

`fei-student` 看起來只是一個 Standard User。但在不同 Scenario 中，同一個帳號可以因為：
- 被加入 `disk` group → 存取 raw block device
- 被授予 `cap_setuid` capability → 改變 Process UID
- 被授予 `SeBackupPrivilege` → 繞過 NTFS ACL
- 找到 root credential → 切換身份
- 被允許 `sudo find` → root shell

這篇文章要建立的是全系列最重要的觀念：

```
Effective Privilege
=
Identity
+
Groups
+
Special Privileges / Capabilities
+
Accessible Privileged Resources
+
Trust Relationships
```

---

## OS 原理：Linux Privilege Stack

### 完整的 Linux 權限層次

![完整的 Linux 權限層次](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day28-effective-privilege-diagram-01.png)


### fei-student 在不同 Scenario 中的 Effective Privilege

| Scenario | 額外權限 | 可做什麼 |
|----------|---------|---------|
| L00 (Base) | 無 | 只有 Standard User |
| L01 | sudo find | root shell via -exec |
| L02 | SUID fei-l02-reader | 讀任意檔案 |
| L03 | sudo maintenance + writable PATH | 間接 root execution |
| L04 | 無（但 cron script writable） | 等待 root 執行 |
| L05 | fei-python3 cap_setuid | setuid(0) → root |
| L06 | sudo runner + writable config | 間接 root file read |
| L07 | 無（但找到 root password） | su → root |
| L08 | disk group | raw device read → 任意檔案 |
| L09 | sudo daemon-reload/restart + writable drop-in | 間接 root execution |

### 為什麼 `id` 是 Linux 提權的第一個指令

```
$ id
uid=1001(fei-student) gid=1001(fei-student) groups=1001(fei-student),6(disk)
```

這行告訴你：
- **UID 1001**：不是 root (0)
- **GID 1001**：primary group 是 fei-student
- **groups=...,6(disk)**：supplementary group 包含 disk

如果沒有仔細看 `groups=`，你可能以為 fei-student 只是普通使用者。但 `disk` group 讓他可以存取 raw block device — 等同於繞過所有檔案系統權限。

### Real UID vs Effective UID vs Saved UID

![Real UID、Effective UID 與 Saved UID](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day28-effective-privilege-diagram-02.png)


L02 (SUID) 和 L05 (Capabilities) 的差別就在這裡：
- SUID：**自動**設定 EUID = file owner
- Capabilities：Process **主動呼叫** setuid()，如果有 CAP_SETUID

---

## OS 原理：Windows Privilege Stack

### 完整的 Windows 權限層次

![完整的 Windows 權限層次](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day28-effective-privilege-diagram-03.png)


### fei-student 在不同 Scenario 中的 Token

| Scenario | Token 變化 | 效果 |
|----------|-----------|------|
| W00 (Base) | Standard User Token | Medium Integrity |
| W01 | 無 Token 變化 | 但 Service ACL 允許 config change |
| W02 | 無 Token 變化 | 但 Binary File ACL 可寫 |
| W03 | 無 Token 變化 | 但 Candidate dir 可寫 |
| W04 | 無 Token 變化 | 但 Action Target 可寫 |
| W05 | 無 Token 變化 | 但 MSI 以 Elevated 安裝 |
| W06 | 無 Token 變化 | 但 Registry ACL 可寫 |
| W07 | 切換到 fei-w07-admin | 新 Identity + Administrators |
| W08 | SeBackupPrivilege + Backup Operators | backup semantics |
| W09 | 無 Token 變化 | 但 DLL candidate dir 可寫 |

### Privilege 的三種狀態

```
$ whoami /priv
特殊權限名稱                  狀況
============================= ======
SeShutdownPrivilege           已停用    ← Present + Disabled
SeChangeNotifyPrivilege       已啟用    ← Present + Enabled
SeBackupPrivilege             已停用    ← Present + Disabled (需要 enable)
```

- **Absent**：Token 裡沒有 → 無法使用
- **Present + Disabled**：Token 裡有但需要 enable → `AdjustTokenPrivileges`
- **Present + Enabled**：可以直接使用

W08 的教學重點就在這裡：`SeBackupPrivilege` 是 Present + Disabled，需要透過 `robocopy /B`（它內部會 enable）才能使用。

---

## Session / Token Refresh 完整教學

### 為什麼「改了設定但 Shell 看不到」？

這是 Phase 10（L08 + W08）學到的最重要教訓：

#### Linux：Group Membership Change

```
# 在 Session A 中：
$ id
groups=1001(fei-student)          ← 沒有 disk

# 管理員在另一個 Session 執行：
$ sudo usermod -aG disk fei-student

# 回到 Session A：
$ id
groups=1001(fei-student)          ← 還是沒有 disk！

# 必須重新登入（新 Session）：
$ ssh fei-student@192.168.77.10
$ id
groups=1001(fei-student),6(disk)  ← 現在有了！
```

**原因**：Linux Process 的 Group Token 在 login 時建立，之後不會自動更新。`usermod` 修改的是 `/etc/group`（persistent state），不是現有 Process 的 Token（runtime state）。

#### Windows：User Rights Assignment

```
# 管理員授予 SeBackupPrivilege：
PS> secedit /configure ...

# fei-student 的現有 Session：
C:\> whoami /priv
# 可能看不到 SeBackupPrivilege

# 重新登入後：
C:\> whoami /priv
SeBackupPrivilege    已停用     ← 現在有了
```

**原因**：Windows Access Token 在 logon 時建立。User Rights Assignment 修改的是 Security Policy，不是現有 Token。新 Token 只在新的 logon 時產生。

### 驗證清單

```
修改生效了嗎？

Linux:
  □ /etc/group 有改嗎？         ← persistent state
  □ 目前 Shell 的 id 有嗎？     ← runtime state
  □ 是否需要 re-login？

Windows:
  □ secedit export 有嗎？       ← persistent state (policy)
  □ whoami /priv 有嗎？         ← runtime state (token)
  □ 是否需要 re-login？
```

---

## 跨平台 Effective Privilege 比較

| 維度 | Linux | Windows |
|------|-------|---------|
| **身分** | UID (Real/Effective/Saved) | User SID |
| **群組** | GID + Supplementary Groups | Group SIDs in Token |
| **特殊能力** | Capabilities (cap_setuid, ...) | Privileges (SeBackup, ...) |
| **層級** | UID 0 vs non-zero | Integrity Level (Medium/High/System) |
| **Token 更新** | 需要新 Session (login) | 需要新 Logon (token creation) |
| **檢查指令** | `id`, `groups`, `getcap` | `whoami /priv`, `whoami /groups` |

### 最重要的一句話

```
Username alone does not describe effective privilege.
```

「fei-student」這個名字什麼都沒說。真正決定權限的是 Token / Process Attributes 裡面的所有層次加總。

---

## Lab 回顧：哪些 Scenario 利用了 Effective Privilege 的「隱藏層」？

| Scenario | 利用的「隱藏層」 | 表面看起來 |
|----------|----------------|-----------|
| L02 SUID | EUID 自動變 root | 檔案有個 `s` bit |
| L05 Capabilities | cap_setuid | 檔案沒有 SUID 但有 capability |
| L07 Credentials | 找到 root password → su | 完全不需要任何特殊權限 |
| L08 Dangerous Group | disk group → raw device | id 的 groups= 裡多了 `disk` |
| W07 Credentials | 找到 admin password → runas | Identity 切換 |
| W08 Token Privileges | SeBackupPrivilege | whoami /priv 裡多了一行 |

---

## False Positive 分析

### 「看起來是 Standard User 但其實不是」

1. **Linux：user 在 `lxd` group**
   - `id` 顯示 `groups=...lxd`
   - 看起來只是「容器管理群組」
   - 實際上：可以建立 privileged container → mount host filesystem → root

2. **Windows：user 有 `SeImpersonatePrivilege`**
   - `whoami /priv` 顯示 SeImpersonate...已停用
   - 看起來只是「停用的 privilege」
   - 實際上：Service Account 常有此 privilege → Potato 系列攻擊 → SYSTEM

3. **Linux：user 在 `docker` group**
   - 看起來只是「開發者方便」
   - 實際上：等同 root（docker daemon 以 root 運行）

### 「看起來有特殊權限但其實沒用」

1. **Windows：user 有 `SeShutdownPrivilege`**
   - 只能關機/重啟
   - 不能提權

2. **Linux：user 在 `audio` group**
   - 只能存取音效裝置
   - 不能提權（除非有極特殊的音效 driver 漏洞）

3. **Windows：Mandatory Label = Medium**
   - 所有 Standard User 都是 Medium
   - 本身不是問題

---

## Edge Case

### Linux：newgrp 不需要重新登入

如果你被加入一個 group 但不想重新登入：

```bash
newgrp disk
```

這會啟動一個新的 shell，其 group token 包含 `disk`。但注意：
- 這只影響 **這個 shell 和它的子 Process**
- 其他已存在的 shell 不受影響
- 退出 newgrp 的 shell 後就失效

### Windows：Token Elevation 和 Integrity Level

即使 user 在 Administrators group，UAC 也會建立兩個 Token：
- **Filtered Token**：Medium Integrity，大部分 admin privilege 被移除
- **Full Token**：High Integrity，完整 admin privilege

只有透過 UAC 提示（或 `runas /user:admin`）才能使用 Full Token。

W07 的設計就利用了這個概念：fei-student 找到 fei-w07-admin 的密碼 → `runas` → 新 Token 包含 Administrators group。

### W08：UAC Token Split 對 Backup Operators 的影響

![UAC Token Split 對 Backup Operators 的影響](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day28-effective-privilege-diagram-04.png)

這是本系列最深的 Edge Case：


解決方案：
1. 停用 UAC（`EnableLUA=0`）→ 不再 split → privilege 可用
2. 使用 `robocopy /B`（system utility，能正確 enable privilege）

---

## Troubleshooting

| 症狀 | 可能原因 | 解法 |
|------|---------|------|
| `id` 沒顯示剛加入的 group | Session 沒有 refresh | 重新登入或 `newgrp` |
| `whoami /priv` 沒有新 privilege | Token 沒有 refresh | 重新登入 |
| SUID binary 沒有 root 效果 | Binary 是 script 不是 ELF | Kernel 忽略 script 的 SUID |
| `AdjustTokenPrivileges` 1300 | UAC token split | 停用 EnableLUA 或用 system utility |
| `su` 成功但 `sudo` 仍然被拒 | su 只改變 identity，不給 sudo | 使用 `su -` 取得完整 root shell |

---

## Defense 防禦

### 原則：最小權限（Least Privilege）

```
Detect:
  Linux:
    # 列出所有 non-root users 的 supplementary groups
    for user in $(awk -F: '$3 >= 1000 {print $1}' /etc/passwd); do
      echo "$user: $(id -Gn $user)"
    done
    
    # 列出所有有 capability 的檔案
    getcap -r / 2>/dev/null
    
    # 列出所有 SUID binary
    find / -perm -4000 -type f 2>/dev/null

  Windows:
    # 列出所有 user 的 privilege
    secedit /export /cfg audit.cfg /areas USER_RIGHTS
    
    # 列出 Backup Operators 成員
    Get-LocalGroupMember "Backup Operators"

Prevent:
  - 移除不必要的 group membership
  - 移除不必要的 SUID / Capabilities
  - 移除不必要的 User Rights Assignment
  - 定期審計

Verify:
  - 修改後重新登入
  - 確認 id / whoami /priv 反映正確
  - 嘗試原本的攻擊路徑 — 應該失敗
```

---

## Detection 偵測

### Linux

| 偵測點 | 方法 |
|--------|------|
| Group membership 變更 | `auditd` 監控 `/etc/group` |
| Capability 設定變更 | `auditd` 監控 `setcap` syscall |
| SUID bit 變更 | `AIDE` / `Tripwire` |
| 異常的 `su` / `sudo` | `/var/log/auth.log` |

### Windows

| 偵測點 | Event ID |
|--------|----------|
| 使用者加入群組 | 4732 (Local Group Member Added) |
| User Rights 變更 | 4704 (User Right Assigned) |
| 異常 Logon | 4624 (Logon) + Logon Type |
| Privilege 使用 | 4672 (Special Privileges Assigned) |

---

## Fix Verification 修復驗證

![Fix Verification 修復驗證流程](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day28-effective-privilege-diagram-05.png)


---

## Exercise 練習

### 練習 1：完整 Effective Privilege 盤點

在你的 FEI Lab 中，以 fei-student 登入後執行：

**Linux：**
```bash
echo "=== Identity ==="
whoami
id

echo "=== Sudo ==="
sudo -l 2>&1

echo "=== SUID ==="
find / -perm -4000 -type f 2>/dev/null | grep -v /usr/

echo "=== Capabilities ==="
getcap -r / 2>/dev/null | grep -v /usr/bin/

echo "=== Writable by me ==="
find /opt/fei-privesc -writable -type f 2>/dev/null

echo "=== Cron ==="
ls -la /etc/cron.d/ 2>/dev/null
```

**Windows：**
```powershell
Write-Output "=== Identity ==="
whoami
whoami /groups
whoami /priv

Write-Output "=== Services ==="
Get-Service | Where-Object { $_.Status -eq 'Running' } | Select-Object Name, DisplayName

Write-Output "=== Scheduled Tasks ==="
Get-ScheduledTask | Where-Object { $_.Principal.UserId -eq 'SYSTEM' } | Select-Object TaskName
```

### 練習 2：思考題

如果一台 Linux 主機上的 `www-data` user（Web Server）屬於 `docker` group，這代表什麼？一個 Web Shell 可以如何利用？

### 練習 3：Session Refresh 實驗

1. Setup L08（disk group）
2. 在**同一個 SSH Session** 中執行 `id` — 看到 disk group 了嗎？
3. 登出再登入
4. 再次 `id` — 現在呢？
5. 這個差異在實戰中代表什麼？

---

## Quiz 測驗

**Q1**: Linux 的 Effective UID (EUID) 什麼時候和 Real UID (RUID) 不同？

### 答案
1. 執行 SUID binary 時（EUID = file owner）
2. Process 呼叫 setuid() 時（如果有 CAP_SETUID 或已經是 root）


**Q2**: Windows 的 `whoami /priv` 顯示一個 Privilege 為「已停用」，這代表什麼？

### 答案Privilege 在 Token 中存在（Present）但處於 Disabled 狀態。需要透過 AdjustTokenPrivileges API 或使用支援的工具（如 robocopy /B）才能啟用並使用。「已停用」≠「沒有」。

**Q3**: fei-student 在 L08 中被加入 `disk` group 後，為什麼已經開啟的 Shell 看不到新 group？

### 答案Linux Process 的 group token 在 login 時建立。`usermod -aG` 修改的是 `/etc/group`（persistent state），不會自動更新已存在 Process 的 token。需要重新登入（新 Session）才會建立包含新 group 的 token。

**Q4**: Windows W07 的目標為什麼不是 SYSTEM 而是 Local Admin？

### 答案W07 教的是 Credential-based Privilege Escalation。找到 fei-w07-admin 的密碼 → 切換身份 → 跨越權限邊界。重點是「提權 = 跨越 Security Boundary」，不一定每次都要到 SYSTEM。從 Standard User 到 Local Admin 就是提權。

**Q5**: 如果一個 Windows user 同時在 Administrators 和 Backup Operators，UAC 會怎麼處理他的 Token？

### 答案UAC 會建立 split token：
- Filtered Token（預設使用）：移除 Administrators 特權，Backup Operators 的 SeBackupPrivilege 也被移除/無法啟用
- Full Token（UAC 提升後）：包含所有特權
使用者的互動 Session 預設使用 Filtered Token。

**Q6**: 列出至少三種「不用漏洞就能提權」的方式。

### 答案
1. Credential Hunting（L07/W07）— 找到暴露的高權限密碼
2. Dangerous Group（L08）— disk group 等同 raw device access
3. Token Privileges（W08）— SeBackupPrivilege 繞過 ACL
其他：Docker group、LXD group、SSH key 洩漏、history 中的密碼


**Q7**: `Effective Privilege = Identity + Groups + Privileges + Resources + Trust`。請用 L06 作為例子說明每個元素。

### 答案
- Identity: fei-student (UID 1001)
- Groups: fei-student (無特殊 group)
- Privileges: 有 sudo /usr/local/bin/fei-l06-runner
- Resources: /opt/fei-privesc/training/L06/fei-runner.conf 可寫
- Trust: root runner 信任 config 中的 TARGET_FILE

結合：fei-student 可寫 config → root runner 讀 config → root 執行 cat TARGET_FILE → 攻擊者控制 TARGET_FILE → 讀任意檔案


**Q8**: Linux `newgrp` 和重新登入有什麼差異？

### 答案
- `newgrp disk`：啟動一個新 shell，只有這個 shell 和子 Process 有新 group。退出後失效。不需要密碼（如果已經在 group 中）。
- 重新登入：整個 Session 的所有 Process 都有新 group。永久生效（直到登出）。


---

## Engineering Note 工程筆記

### L08 — Group Membership + LVM + udev

L08 建置時發現三個連鎖問題：

1. **LVM 裝置不歸屬 disk group**
   - `/dev/mapper/ubuntu--vg-ubuntu--lv` 預設 group 是 `root`
   - 需要 udev rule 讓 `dm-*` 裝置歸屬 `disk` group

2. **symlink 的 group 不是裝置的 group**
   - `/dev/mapper/` 下的是 symlink → `/dev/dm-0`
   - `stat symlink` 永遠是 root:root
   - 必須 `readlink -f` 再 `stat` 實際裝置

3. **udev rule 讓設定持久化**
   - 直接 `chgrp` 會被 udev 重設
   - 需要 `/etc/udev/rules.d/99-fei-l08-disk.rules`

### W08 — UAC Token Split 的完整故事

![W08 UAC Token Split 的完整故事](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day28-effective-privilege-diagram-06.png)

W08 經歷了三次修正：


教訓：
> **「whoami /priv 看到 privilege」和「能真正使用 privilege」之間，存在 UAC token split 這道牆。**

---

## 今天真正要記住的 3 件事

1. **`whoami` 只告訴你名字。真正的 Effective Privilege 是 Identity + Groups + Privileges + Resources + Trust 的總和**

2. **Persistent State（/etc/group, secedit policy）和 Runtime State（id output, whoami /priv）可能不同步。改了設定不代表目前 Session 立即生效**

3. **提權分析的第一步不是找漏洞，而是完整盤點「我到底有什麼」**

---

## 給自己的問題

> 如果你拿到了一台陌生 Linux 主機的 Low Privilege Shell，你沒有任何背景資訊。你會按什麼順序盤點自己的 Effective Privilege？能不能列出一個不超過 10 步的清單？

---

## 下一篇

Day 29 — 如果今天只給你 Low Privilege Shell，你下一步會查什麼？完整 Decision Tree。
