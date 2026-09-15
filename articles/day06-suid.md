# Day 06：SUID — 普通使用者為什麼可以用別人的身分執行程式？


---

## 開場

我是 `fei-student`。沒有 sudo。不知道 root 密碼。

但系統裡有一個程式，它執行的時候，不是用我的身分——而是用 root 的身分。

為什麼？這就是 SUID 的故事。

---

## 今天要解決的問題

1. SUID 到底改變了什麼？
2. Real UID 和 Effective UID 有什麼不同？
3. 看到 SUID binary 就代表一定能提權嗎？
4. SUID script 和 SUID binary 有什麼差別？

---

## 背景知識

### 什麼是 SUID？

SUID 全名是 **Set User ID on Execution**。它是 Linux 檔案權限系統中的一個特殊 bit。

簡單說：

> **當你執行一個設有 SUID bit 的程式時，你的「有效身分」會暫時變成那個程式的擁有者。**

如果程式的擁有者是 root，你執行時就暫時擁有 root 的權限。

日常例子：`/usr/bin/passwd` 就是一個 SUID root 程式。每個使用者都需要能改自己的密碼，但密碼存在 `/etc/shadow`（只有 root 能改），所以 `passwd` 程式設了 SUID——讓一般使用者執行它時可以暫時以 root 身分修改 shadow 檔案。

### 辨認 SUID

```bash
$ ls -la /usr/bin/passwd
-rwsr-xr-x 1 root root 59976 ... /usr/bin/passwd
   ^
   s = SUID bit 設定
```

數字表示法是 `4755`：
- `4` = SUID bit
- `7` = owner rwx
- `5` = group r-x
- `5` = others r-x

---

## OS 原理：Real UID、Effective UID、Saved UID

Linux 中，每個 Process 同時持有三個 UID。這是理解 SUID 的關鍵。

### 三種 UID

| UID | 英文 | 用途 |
|-----|------|------|
| **Real UID (RUID)** | Real User ID | 記錄「這個 process 是哪個使用者啟動的」|
| **Effective UID (EUID)** | Effective User ID | **OS 實際用來做權限檢查的身分** |
| **Saved UID (SUID)** | Saved User ID | 讓程式可以在「提升」和「恢復」權限之間切換 |

### 一般執行（沒有 SUID）

```
fei-student (UID=1001) 執行 /bin/cat：
  RUID = 1001
  EUID = 1001    ← OS 用這個做權限檢查
  SUID = 1001

結果：cat 讀 /root/fei-flag.txt → Permission denied
（因為 EUID=1001，不是 root）
```

### SUID 執行

```
fei-student (UID=1001) 執行 root-owned SUID 程式：
  RUID = 1001    ← 不變，你還是 fei-student
  EUID = 0       ← 變成 root！因為程式 owner 是 root
  SUID = 0       ← 保存 root UID，供程式切換用

結果：程式內部的 fopen() 以 EUID=0 (root) 執行
→ 可以讀取任何檔案
```

### 關鍵：EUID 才是實際權限

![EUID 才是實際權限](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day06-suid-diagram-01.png)



### Saved UID 的用途

有些程式會先用 root 權限完成初始化，然後呼叫 `seteuid(original_uid)` 降低回一般權限。Saved UID 保留了 root UID，讓程式未來有需要時可以再 `seteuid(0)` 回到 root。

`passwd` 程式就是這麼做的：
1. 以 root EUID 開啟 `/etc/shadow`
2. 降回使用者 EUID 做互動
3. 寫入時再提升回 root

---

## 🔬 FEI Lab 環境

```
Scenario:   FEI-L02-SUID
VM:         FEI-PRIVESC-LINUX (Ubuntu 22.04.4 LTS)
起始帳號:   fei-student (uid=1001, 無 sudo)
目標:       root
Flag:       /root/fei-l02-flag.txt
```



### 場景準備

`ash
# 1. 以 fei-labadmin 登入
ssh fei-labadmin@192.168.77.10
# 密碼：FEI-LabAdmin-2026!

# 2. 進入場景目錄並 setup
cd /opt/fei-privesc/scenarios/FEI-L02-SUID
sudo ./fei-setup.sh

# 3. 確認場景就緒
sudo ./fei-verify.sh
# 應看到 L02-SUID STATUS: READY

# 4. 切換到 fei-student
su - fei-student
# 密碼：FEI-Student-2026!
`

### 部署過程

setup 做了什麼：

1. 將 `/bin/cat` 複製為 `/opt/fei-privesc/bin/fei-l02-reader`
2. 設定 owner 為 `root:root`
3. 設定 mode 為 `4755`（SUID）
4. 建立受保護的 flag 檔案（`0600 root:root`）

注意：**原始的 `/bin/cat` 完全不受影響**。只有複製出來的 `fei-l02-reader` 有 SUID。

verify 確認了 11 項檢查，包括：

```
[PASS] fei-student exists
[PASS] fei-student UID is not 0
[PASS] FEI training helper exists (/opt/fei-privesc/bin/fei-l02-reader)
[PASS] helper owner is root
[PASS] SUID bit is set
[PASS] helper permissions are 4755
[PASS] flag exists (/root/fei-l02-flag.txt)
[PASS] fei-student cannot directly read flag
[PASS] no FEI-L01 sudo rule remains active
```

---

## 如果我是攻擊者，我現在想知道什麼？

```
我沒有 sudo，也不知道 root 密碼。

但 Linux 還有其他方式可以讓程式以更高權限執行嗎？

→ SUID。

有哪些檔案設了 SUID？
```

---

## 完整攻擊流程

### Step 1：確認身分

```bash
fei-student@fei-privesc-linux:~$ whoami
fei-student

fei-student@fei-privesc-linux:~$ id
uid=1001(fei-student) gid=1001(fei-student) groups=1001(fei-student)
```

沒有任何特殊群組。

### Step 2：確認無法直接讀取 flag

```bash
fei-student@fei-privesc-linux:~$ cat /root/fei-l02-flag.txt
cat: /root/fei-l02-flag.txt: Permission denied
```

### Step 3：尋找 SUID binary

```bash
fei-student@fei-privesc-linux:~$ find / -perm -4000 -type f 2>/dev/null
/usr/bin/newgrp
/usr/bin/sudo
/usr/bin/umount
/usr/bin/chfn
/usr/bin/gpasswd
/usr/bin/mount
/usr/bin/passwd
/usr/bin/pkexec
/usr/bin/chsh
/usr/bin/su
/usr/bin/fusermount3
/usr/libexec/polkit-agent-helper-1
/usr/lib/snapd/snap-confine
/usr/lib/openssh/ssh-keysign
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/opt/fei-privesc/bin/fei-l02-reader           ← 不是系統標準！
```

大部分都是系統正常的 SUID binary（`sudo`、`passwd`、`mount` 等）。

但最後一個 **`/opt/fei-privesc/bin/fei-l02-reader`** 不在系統標準目錄，非常可疑。

### Step 4：檢查可疑 binary

```bash
fei-student@fei-privesc-linux:~$ ls -la /opt/fei-privesc/bin/fei-l02-reader
-rwsr-xr-x 1 root root 35288 Sep 13 14:53 /opt/fei-privesc/bin/fei-l02-reader
```

確認：
- Owner: **root:root**
- Permission: **-rwsr-xr-x**（SUID set）
- 路徑不標準（`/opt/fei-privesc/bin/`）

嘗試執行看看它做什麼：

```bash
fei-student@fei-privesc-linux:~$ /opt/fei-privesc/bin/fei-l02-reader
FEI Lab File Reader
Usage: /opt/fei-privesc/bin/fei-l02-reader <filepath>
```

它是一個**檔案讀取器**。

### Step 5：建立 Attack Hypothesis

```
已知條件：
  1. fei-l02-reader 是 root-owned SUID binary
  2. 它可以讀取任意路徑的檔案
  3. 執行時 EUID = 0 (root)

假說：
  如果我餵給它 /root/fei-l02-flag.txt，
  它會以 root 權限讀取並輸出內容。
```

### Step 6：驗證

```bash
fei-student@fei-privesc-linux:~$ /opt/fei-privesc/bin/fei-l02-reader /root/fei-l02-flag.txt
FEI{LINUX_L02_SUID_ROOT_ACCESS}
```

**成功。**

---

## Attack Path

![SUID Attack Path](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day06-suid-diagram-02.png)



---

## 為什麼它真的成立？

壞掉的 Security Boundary 在這裡：

```
正常情況：
  fei-student → cat /root/flag → EUID=1001 → Denied

SUID 情況：
  fei-student → fei-l02-reader /root/flag → EUID=0 → Allowed
```

`cat` 和 `fei-l02-reader` 做的事完全一樣（讀檔案），但因為 `fei-l02-reader` 有 SUID root，Kernel 看到的 EUID 是 root，所以放行了。

**重點：程式的功能沒有變，但身分變了。**

---

## False Positive：看到 SUID 不代表一定能提權

這是新手最常犯的錯誤。

### `/usr/bin/passwd` 也是 SUID root，但它安全

```bash
$ ls -la /usr/bin/passwd
-rwsr-xr-x 1 root root 59976 ... /usr/bin/passwd
```

為什麼安全？因為 `passwd` 程式**只允許使用者改自己的密碼**。它內部有嚴格的身分檢查和操作限制，不會讓你讀任意檔案。

### 判斷 SUID binary 是否危險的關鍵

問這些問題：

| 問題 | 如果答案是 Yes |
|------|--------------|
| 它能讀/寫任意檔案嗎？ | 危險 |
| 它能執行 shell 或任意指令嗎？ | 危險 |
| 它有 `-exec` / `--exec` 等功能嗎？ | 危險 |
| 它能修改環境變數並影響子 process 嗎？ | 可能危險 |
| 它只做一件受限的事（如改密碼）？ | 通常安全 |

### 常見安全 SUID binary

| Binary | SUID 原因 | 為什麼安全 |
|--------|----------|-----------|
| `/usr/bin/passwd` | 需要寫 `/etc/shadow` | 只允許改自己的密碼 |
| `/usr/bin/mount` | 需要 mount syscall | 受限於 fstab 設定 |
| `/usr/bin/su` | 需要切換 UID | 要求目標密碼 |
| `/usr/bin/sudo` | 需要切換 UID | 受 sudoers 限制 |

### 常見危險 SUID binary（如果被錯誤設定）

`find`、`vim`、`python`、`perl`、`awk`、`nmap`（舊版）、`less`、`more` — 這些工具都有方法「逃逸」出原本的功能範圍。

查詢：[GTFOBins](https://gtfobins.github.io/) 列出了所有可被利用的 binary。

---

## Edge Case

### SUID Script vs SUID Binary

```bash
# 這不會有 SUID 效果：
chmod 4755 my-script.sh

# 原因：Linux kernel 在處理 #! (shebang) 時，
# 不會對 script 啟用 SUID。
# 它啟用 SUID 的是 interpreter（如 /bin/bash），
# 但 bash 本身會偵測並主動丟棄 SUID。
```

**為什麼？**

1. Kernel 看到 `#!/bin/bash` → 啟動 `/bin/bash`
2. bash 偵測到 RUID ≠ EUID → 認定是 SUID 呼叫
3. bash 主動呼叫 `setuid(getuid())` → **放棄提升的 EUID**
4. Script 最終以 RUID（一般使用者）執行

這是 bash 的安全機制（dash 也有）。

**教學意義**：只有 compiled binary 的 SUID 才有效。

### nosuid mount option

```bash
$ mount | grep /opt
/dev/sda1 on /opt type ext4 (rw,nosuid)
```

如果掛載點使用 `nosuid` option，該檔案系統上的所有 SUID bit 會被忽略。

在我們的 Lab 中，`/opt` 沒有使用 `nosuid`，所以 SUID 正常生效。

### Ubuntu 22.04 的 pkexec

你可能注意到 `/usr/bin/pkexec` 也有 SUID。2022 年的 CVE-2021-4034（PwnKit）讓 pkexec 可被利用提權。但 Ubuntu 22.04.4 已修補，所以在我們的環境中 pkexec 不構成威脅。

---

## Troubleshooting

### 學員常見問題

| 問題 | 解決 |
|------|------|
| 找不到異常 SUID | 確認 `find / -perm -4000` 有包含 `/opt` 目錄，不要只看 `/usr` |
| 以為需要寫 exploit | 這題不需要 buffer overflow。正常使用 SUID binary 的功能就夠了 |
| 執行 SUID binary 報 Permission denied | 確認 binary 的 others 有 `x` 權限 |
| 找到 SUID 但不知道它做什麼 | 試著不帶參數執行它看 usage |
| 嘗試對 shell script 設 SUID | 不會生效——kernel/bash 不允許 |

### 如果 SUID bit 被 nosuid 阻擋

```bash
# 檢查掛載選項
mount | grep $(df /opt/fei-privesc/bin/fei-l02-reader | tail -1 | awk '{print $1}')

# 如果看到 nosuid，SUID 不會生效
# 在 Lab 環境中不應該出現這個情況
```

---

## 防禦

### Detect：怎麼找到危險的 SUID？

```bash
# 列出所有 SUID binary
find / -perm -4000 -type f -ls 2>/dev/null

# 與已知基線比對
# 新增的 SUID binary 應立即調查
```

建議定期執行這個檢查，並與 baseline 比對。

### Prevent：怎麼配置才正確？

1. **移除不必要的 SUID**：
   ```bash
   chmod 0755 /path/to/unnecessary-suid-binary
   ```

2. **使用 Capabilities 替代 SUID**：
   ```bash
   # 只給特定能力，不給完整 root
   setcap cap_dac_read_search+ep /path/to/reader
   ```

3. **使用群組權限**：
   ```bash
   chgrp log-readers /var/log/specific.log
   chmod 640 /var/log/specific.log
   usermod -aG log-readers fei-student
   ```

4. **使用 POSIX ACL**：
   ```bash
   setfacl -m u:fei-student:r /var/log/specific.log
   ```

5. **AppArmor / SELinux**：限制 SUID 程式能存取的資源範圍。

### Fix Verification：修完怎麼確認？

```bash
# 移除 SUID 後
chmod 0755 /opt/fei-privesc/bin/fei-l02-reader

# 驗證 SUID 已移除
ls -la /opt/fei-privesc/bin/fei-l02-reader
# 應看到 -rwxr-xr-x（不是 -rwsr-xr-x）

# 驗證無法再提權
su - fei-student
/opt/fei-privesc/bin/fei-l02-reader /root/fei-l02-flag.txt
# 應該 Permission denied
```

---

## Detection：監控什麼？

| 監控項目 | 方法 |
|---------|------|
| 新增 SUID binary | 定期 `find -perm -4000` 比對 baseline |
| SUID binary 執行異常 | auditd rule：`-a always,exit -F path=/opt/fei-privesc/bin/fei-l02-reader -F perm=x` |
| `chmod +s` 操作 | auditd：`-a always,exit -S chmod -S fchmod -F a1&04000` |
| 非標準路徑的 SUID | 重點監控 `/opt`、`/tmp`、`/home` 下的 SUID |

---

## FEI Lab 工程筆記

### gcc → cp /bin/cat

原本的設計是用 C 語言寫一個簡單的 file reader，然後在 VM 裡用 gcc 編譯。

**預期**：setup.sh 用 `gcc -o fei-l02-reader reader.c` 編譯

**實際**：VM 是 Host-Only 網路，沒有 Internet，無法 `apt install gcc`

**原因**：Lab 設計原則是不依賴 Internet

**修正**：改為 `cp /bin/cat /opt/fei-privesc/bin/fei-l02-reader`，直接複製系統內建的 `cat` 並重新命名

**結果**：教學價值完全不變（SUID + 檔案讀取 = 權限繞過），但不再需要編譯器

**這件事教會我們**：Lab 設計必須考慮離線環境。任何依賴 Internet 的步驟都應該被消除，或提供離線替代方案。

---

## L01 sudo vs L02 SUID 比較

| 比較 | L01 sudo | L02 SUID |
|------|----------|----------|
| 提權機制 | sudo policy delegation | SUID bit on binary |
| 關鍵設定 | `/etc/sudoers.d/` | file mode `4755` |
| 權限來源 | sudoers 規則允許 | binary owner 的 UID |
| 設定方式 | `visudo` | `chmod +s` |
| 影響範圍 | 只影響被指定的使用者 | 影響所有能執行該 binary 的使用者 |
| 日誌 | sudo 有完整日誌 | 無特殊日誌（除非用 auditd）|
| 移除方式 | 刪除 sudoers 規則 | `chmod -s` |

共同點：**都是「信任委派錯誤」——把某個能力給了不該有的使用者。**

---

## Exercise：自己動手

1. 在 Lab 中，試著用 `fei-l02-reader` 讀 `/etc/shadow`。你能看到密碼 hash 嗎？
2. 試著對一個 shell script 設定 SUID，然後執行它。觀察 EUID 有沒有改變。
3. 用 `strace` 追蹤 `fei-l02-reader` 的系統呼叫，找出它在哪裡呼叫 `open()` 或 `openat()`。

---

## Quiz

**Q1**：一個由 root 擁有、設了 SUID bit 的 binary，當 fei-student 執行它時，它的 Real UID 和 Effective UID 分別是什麼？

### 答案

Real UID = 1001 (fei-student)，Effective UID = 0 (root)。
Real UID 不會因為 SUID 改變，但 Effective UID 會變成 binary owner 的 UID。



**Q2**：`/usr/bin/passwd` 也是 SUID root，為什麼它不算漏洞？

### 答案

因為 `passwd` 程式內部有嚴格的身分檢查，只允許使用者改自己的密碼。它雖然以 root EUID 執行，但不提供讀取任意檔案或執行任意指令的功能。SUID 本身不是漏洞，問題在於 SUID binary 的功能是否可以被濫用。



**Q3**：為什麼在 shell script 上設定 SUID bit 不會有效果？

### 答案

Linux kernel 在處理 shebang（`#!/bin/bash`）時，實際啟動的是 interpreter（bash）。bash 會偵測 RUID ≠ EUID 的情況，並主動呼叫 `setuid(getuid())` 丟棄提升的權限。所以 script 的 SUID bit 被 interpreter 的安全機制抵銷了。只有 compiled binary 的 SUID 才有效。




## 場景收尾

做完後記得 reset，避免影響下一題：

```bash
# 以 fei-labadmin 執行
ssh fei-labadmin@192.168.77.10
cd /opt/fei-privesc/scenarios/FEI-L02-SUID
sudo ./fei-reset.sh
sudo ./fei-verify.sh --reset
# 應看到 L02-SUID STATUS: RESET
```

---

## 今天真正要記住的 3 件事

1. **SUID 改變的是 Effective UID，不是 Real UID**。Kernel 用 EUID 做權限檢查。
2. **SUID binary 危不危險，取決於它的功能**。能讀任意檔案或執行指令的 SUID binary 等同於 root shell。
3. **找到 SUID 不等於找到漏洞**。你還需要確認：binary 的功能可以被利用、它確實有 SUID、掛載點沒有 nosuid。

---

## 給自己的問題

> SUID 把 root 的所有權限都給了 binary。有沒有辦法只給「其中一小塊」能力，而不是全部？

→ 這就是 Day 09 的 Linux Capabilities。

---

## 下一篇

Day 07：PATH Hijacking — Linux 最後到底執行了哪一個程式？

---

## Lab Reset

```bash
cd /opt/fei-privesc/scenarios/FEI-L02-SUID
sudo ./fei-reset.sh
sudo ./fei-verify.sh --reset
# [PASS] SUID helper removed
# [PASS] flag removed
# [PASS] active scenario cleared
# FEI-L02 STATUS: RESET
```
