# Day 29：如果今天只給你 Low Privilege Shell，你下一步會查什麼？

## 背景知識

過去 28 天，每一篇都告訴你「今天是什麼題型」。你知道 Day 05 是 sudo，Day 18 是 Unquoted Path。

但在真實滲透測試中，沒有人會告訴你：

> 「這台機器的漏洞是 Weak Service ACL，請用 sc config。」

你拿到的只有：

```
$ whoami
fei-student
$
```

然後呢？

今天就是那個「然後呢」。

---

## 今天要解決的問題

1. 不知道漏洞類型時，如何系統性地找到提權路徑？
2. 什麼時候該停止某個方向、轉向另一個？
3. 如何避免只跑自動化工具而不理解結果？
4. 如何把 20 個 Lab 的經驗整合成一套可重用的方法？

---

## 完整 PrivEsc Decision Tree

![完整 PrivEsc Decision Tree](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day29-blind-privesc-analysis-diagram-01.png)

以下這棵 Decision Tree 涵蓋了本系列 20 個 Lab 的所有路徑。


### 每一步的具體指令

| 步驟 | Linux | Windows |
|------|-------|---------|
| 1. Identity | `id`, `whoami` | `whoami`, `whoami /user` |
| 2A. Groups | `id -Gn`, `groups` | `whoami /groups`, `net localgroup` |
| 2B. Privileges | `getcap -r / 2>/dev/null` | `whoami /priv` |
| 2C. Sudo | `sudo -l` | N/A |
| 3. SUID | `find / -perm -4000 -type f 2>/dev/null` | N/A |
| 4. Scheduled | `ls /etc/cron.d/`, `crontab -l` | `schtasks /query /fo LIST /v` |
| 5. Services | `systemctl list-units --type=service` | `sc query`, `sc qc <svc>` |
| 6. Writable | `find / -writable -type f 2>/dev/null` | `icacls`, `Get-Acl` |
| 7. Credentials | `grep -ri password /opt/ /etc/ /home/` | `findstr /si password C:\*.ini *.xml *.ps1` |
| 8. Policy | N/A | `reg query HKLM\...\Installer` |

---

## 假想場景 Walkthrough

### 場景 A：Linux — 「一看就很乾淨」

```
$ id
uid=1001(fei-student) gid=1001(fei-student) groups=1001(fei-student)
$ sudo -l
sorry, user fei-student is not allowed to run sudo
$ find / -perm -4000 -type f 2>/dev/null
(只有標準系統 SUID)
$ getcap -r / 2>/dev/null
/usr/bin/ping cap_net_raw=ep
(正常)
```

看起來什麼都沒有？繼續往下：

```
$ cat /etc/cron.d/*
* * * * * root /bin/bash /opt/maintenance.sh

$ ls -la /opt/maintenance.sh
-rwxrwxrwx 1 root root ...    ← 找到了！
```

→ 走 Decision Tree Step 4 → **L04 Cron 路徑成立**

**教訓**：不要在 Step 1-3 沒發現就放棄。Scheduled Execution 往往在 Step 4 才出現。

### 場景 B：Windows — 「Service 很多但都改不了」

```
C:\> whoami /groups
BUILTIN\Users
(沒有 Administrators)

C:\> sc qc FEISomeService
SERVICE_START_NAME : LocalSystem

C:\> sc config FEISomeService binPath= "test"
Access Denied            ← W01 不成立

C:\> icacls "C:\path\to\service.exe"
Users:(RX)               ← W02 不成立

C:\> sc qc (check ImagePath)
"C:\quoted\path\service.exe"  ← W03 不成立（有引號）
```

W01/W02/W03 全部被擋。怎麼辦？

```
C:\> reg query "HKLM\SOFTWARE\FEI\PrivEsc\W06" /v ActionPath
ActionPath = C:\...\fei-helper.exe

C:\> Get-Acl 'HKLM:\SOFTWARE\FEI\PrivEsc\W06' | ...
Users: WriteKey           ← 找到了！
```

→ 走 Decision Tree Step 5 (Services) → Registry 方向 → **W06 路徑成立**

**教訓**：Service 提權不只是 ACL/Binary/Path 三種。Registry Configuration 也是一個維度。

### 場景 C：Linux — 「找到密碼但不確定有效」

```
$ grep -ri "password" /opt/
/opt/backup/config.bak:password = changeme
/opt/backup/deploy.conf:password = FEI-Root-Something!
/opt/scripts/example.sh:# password: example-only
```

三個候選。哪個有效？

```
$ cat /opt/backup/config.bak
password = changeme
→ 看起來是預設密碼，可能無效

$ cat /opt/backup/deploy.conf
username = root
password = FEI-Root-Something!
→ 有 username + password，而且 username 是 root

$ echo "FEI-Root-Something!" | su -c "whoami" root
root
→ 有效！
```

→ 走 Decision Tree Step 7 → **L07 路徑成立**

**教訓**：`grep password` 只是第一步。找到字串後還需要：
1. 判斷是 username 的 password 還是其他用途
2. 驗證 credential 是否有效
3. 確認對應的 identity 權限是否更高

### 場景 D：Windows — 「什麼都試了，最後才發現是 DLL」

前 8 步都沒有直接發現。但在 Step 5 深入時：

```
C:\> sc qc FEIDLLTrainingService
BINARY_PATH_NAME: "C:\...\FEI-W09-Loader.exe"  ← 有引號，W03 不成立
SERVICE_START_NAME: LocalSystem

C:\> icacls "C:\...\FEI-W09-Loader.exe"
Users:(RX)                                       ← 不可改，W02 不成立
```

但如果進一步分析 Loader 的行為：

```
C:\> findstr /si "LoadLibrary\|Assembly.Load" C:\...\Loader.exe
(看到 DLL loading 行為)

C:\> icacls "C:\...\Bin\"
Users:(S,WD)                                     ← 可建立新檔案！
```

→ **W09 DLL Hijacking 路徑成立**

**教訓**：DLL Hijacking 是最隱蔽的路徑。通常在 Service 的 Binary/ACL/Path 都安全時才需要深入到 Code Loading 層。

---

## Troubleshooting：Decision Tree 走到死路時

### 重新來過 Checklist

```
□ 我真的執行了 id / whoami /groups /priv 嗎？
□ 我有沒有只看 username 就跳過 groups？
□ sudo -l 有沒有因為密碼要求而跳過？
□ getcap 有沒有只搜 /usr/bin 而漏掉 /opt？
□ cron 有沒有只看 user crontab 沒看 /etc/cron.d/？
□ Service 有沒有只看 ACL 沒看 Binary ACL / Path / Registry？
□ grep password 有沒有搜夠多目錄？
□ 我有沒有忘記檢查 systemd drop-in？
□ 有沒有跑過 find -writable？
```

### 最常遺漏的三個點

1. **Supplementary Groups**：`id` 輸出的最後幾個 group 很容易被忽略
2. **Capabilities**：很多人只找 SUID，不找 getcap
3. **Registry ACL**：Windows 使用者習慣只看 NTFS ACL，忘記 Registry 也有 ACL

---

## Defense 觀點：防禦者的 Decision Tree

防禦者可以用同一棵 Tree **反向**操作：

```
對每個步驟，驗證「攻擊者會在這裡找到什麼？」

Step 1: 有沒有 user 被意外加到 privileged group？
Step 2: 有沒有不必要的 SUID / Capabilities / Privileges？
Step 3: 有沒有 sudo 規則允許 escape？
Step 4: 有沒有 cron/Task 引用可寫 script？
Step 5: 有沒有 Service 有弱 ACL/Binary/Path/Registry/DLL？
Step 6: 有沒有高權限程式信任可寫 config？
Step 7: 有沒有明文 credential 在檔案中？
Step 8: 有沒有危險 Policy（AlwaysInstallElevated 等）？
```

---

## Exercise 練習

### 練習 1：Blind PrivEsc

在 FEI Lab 中：

1. 請 fei-labadmin 執行任何一個你**沒看過 instructor.md** 的 Scenario 的 `fei-setup.sh`
2. 以 fei-student 登入
3. **不看** student.md 和 instructor.md
4. **只用 Decision Tree** 找到提權路徑
5. 記錄你走了哪些步驟、在哪一步找到突破口

### 練習 2：完整 Enum Script

寫一個 shell script（Linux）或 PowerShell script（Windows），自動執行 Decision Tree 的所有步驟，輸出一份 report。

```bash
#!/bin/bash
echo "=== FEI PrivEsc Enum ==="
echo "--- Identity ---"
id
echo "--- Sudo ---"
sudo -l 2>&1
echo "--- SUID ---"
find / -perm -4000 -type f 2>/dev/null | grep -v /usr/
echo "--- Capabilities ---"
getcap -r / 2>/dev/null | grep -v /usr/bin/
echo "--- Cron ---"
cat /etc/cron.d/* 2>/dev/null
echo "--- Writable ---"
find /opt /etc -writable -type f 2>/dev/null
echo "--- Credentials ---"
grep -ri "password" /opt/ /home/ 2>/dev/null | head -20
```

> 注意：自動化 enum script 是「加速」不是「替代」。你仍然需要理解每一行輸出的意義。

### 練習 3：場景判斷

看以下 enum 輸出，判斷最可能的提權路徑：

```
$ id
uid=1001(webuser) gid=1001(webuser) groups=1001(webuser),999(docker)
$ sudo -l
(sorry, not allowed)
$ find / -perm -4000 ...
(only standard)
```

答案是什麼？（提示：看 groups）

---

## Quiz 測驗

**Q1**: Decision Tree 的第一步為什麼是 `id` / `whoami` 而不是找漏洞？

### 答案因為你首先需要知道「我現在有什麼」才能判斷「我缺什麼」。如果 id 就已經顯示你在 docker group，根本不需要找其他漏洞。先 enumerate 再 exploit。

**Q2**: 場景 B 中，W01/W02/W03 全部不成立。接下來應該查什麼？（至少列三個方向）

### 答案
1. Registry ACL — Service 是否從 Registry 讀取設定？（W06）
2. DLL Loading — Service 是否載入可控 DLL？（W09）
3. Scheduled Task — 是否有 SYSTEM task 引用可寫 action？（W04）
4. Credentials — 是否有明文 credential 可切換身份？（W07）
5. AlwaysInstallElevated — Policy 是否允許 elevated MSI？（W05）


**Q3**: 如果 `grep -ri password` 找到 10 個結果，你怎麼判斷哪個值得嘗試？

### 答案
按以下優先順序過濾：
1. 有明確 username + password 配對的（不只是 password 字串）
2. 在 backup / .bak / .old 檔案中的（可能是舊設定）
3. username 對應到高權限帳號的（root, admin, service account）
4. 排除明顯的預設值（changeme, example, P@ssw0rd）
5. 排除程式碼中的變數名稱（$password = Read-Host）

然後用 `su` / `runas` 實際驗證每個候選。


**Q4**: 練習 3 的答案是什麼？

### 答案webuser 在 `docker` group (GID 999)。Docker daemon 以 root 運行，docker group 的成員可以透過 `docker run -v /:/mnt alpine cat /mnt/root/flag` 存取 host 的任意檔案。這等同 root 存取。走 Decision Tree Step 2A → Dangerous Group → docker。

**Q5**: 為什麼 Decision Tree 的最後一步是「Re-evaluate」？

### 答案因為：
1. 第一輪可能遺漏了某些檢查
2. 新發現的資訊可能改變之前的判斷（例如找到 credential 後可以 sudo）
3. 多個「弱點」單獨不成立，但組合起來可能成立
4. 某些路徑需要先拿到中間 identity 再繼續提權


---

## Engineering Note 工程筆記

### 建立 Decision Tree 的過程

這棵 Decision Tree 不是「看書設計出來」的。它是從 20 個 VALIDATED Scenario 反推回來的：

1. 先建立所有 Lab
2. 在 Real Environment Validation 中記錄每題的 Enum → Discover → Exploit 路徑
3. 找出共同的 Decision Point
4. 按「最常見 → 最隱蔽」排序
5. 驗證每個 Decision Point 至少對應一個 VALIDATED Scenario

這也是為什麼 Decision Tree 的順序是：

```
Identity → Groups/Privileges → SUID → Scheduled → Services → Writable → Credentials → Policy
```

而不是按技術複雜度排序。因為在實測中，Group Membership 和 sudo 是最常、最快找到的路徑。

---

## 今天真正要記住的 3 件事

1. **不知道漏洞類型時，Decision Tree 比 random search 有效得多**
2. **每一步都是在回答一個具體問題，不是在「跑指令」**
3. **找到線索後還需要驗證：可控嗎？高權限使用嗎？能觸發嗎？**

---

## 給自己的問題

> 如果你把這棵 Decision Tree 交給一個完全沒做過 CTF 的人，他能照著走嗎？如果不能，哪個步驟需要更多解釋？

---

## 下一篇

Day 30 — 30 天後重新理解 Privilege Escalation。把 20 個 Lab、所有 Bug Fix、所有跨平台比較，收斂成一個統一的方法論。
