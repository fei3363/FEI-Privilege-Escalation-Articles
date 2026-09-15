# Day 25：Windows Service 提權不是一種漏洞：一次拆懂五種不同 Trust Boundary

## 開場情境

「Windows Service 有漏洞。」

這句話幾乎沒有意義。因為：

> SYSTEM Service 有很多種方式可以被利用。真正的問題不是「Service 存在」，而是「Service 信任了什麼？」

今天我們不做新的 Lab。我們把 W01-W09 中與 Service 相關的五個場景排在一起，看清楚一件事：

**同一個 SYSTEM Service，五個不同的 Trust Surface。**

---

## 今天要解決的問題

1. 「SYSTEM Service 存在」本身是漏洞嗎？
2. 攻擊者到底可以從哪些面向攻擊一個 Service？
3. 修好一層就夠了嗎？
4. 怎麼系統性地評估一個 Service 的安全？

---

## 背景知識：SYSTEM Service 的 Trust Surface

![SYSTEM Service Trust Surface 資訊圖](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day25-windows-service-trust-boundary-diagram-01.png)

一個 Windows Service 在運行時，信任的不只是自己的 exe：


每一層都是一個 **Trust Boundary**。

---

## 五種 Trust Surface 完整矩陣

| # | 場景 | 信任的資源 | 漏洞條件 | 攻擊動作 | 檢查工具 |
|---|------|-----------|---------|---------|---------|
| W01 | Service ACL | Service Configuration | `SERVICE_CHANGE_CONFIG` 給 Users | `sc config binPath=` | `sc sdshow` |
| W02 | Service Binary | Executable 檔案 | Users 可修改 exe | 替換 exe | `icacls exe` |
| W03 | Unquoted Path | ImagePath 解析 | 空格 + 無引號 + 可寫候選 | 放置 candidate exe | `sc qc` + `icacls dir` |
| W06 | Registry ACL | Registry Value | Users 有 WriteKey | `reg add` 修改值 | `Get-Acl Registry::` |
| W09 | DLL Loading | Plugin/DLL | App directory 可寫 | 放置 DLL | `icacls dir` |

---

## OS 原理：為什麼五層都要保護

### 層層防禦

![Service 層層防禦資訊圖](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day25-windows-service-trust-boundary-diagram-02.png)


修好 W01 的 Service ACL 只保護了第一層。如果 Binary 可寫（W02）、路徑可歧義（W03）、Registry 可改（W06）、DLL 可放（W09），Service 仍然不安全。

### 每一層的檢查

```
W01: sc sdshow <service> → 檢查 BU ACE 有沒有 DC
W02: icacls <exe path> → Users 只能 RX
W03: sc qc → ImagePath 有引號 + 路徑無空格
W06: Get-Acl → Registry Key ACL 限制 WriteKey
W09: icacls <exe directory> → Users 只能 RX，不能 CreateFiles
```

---

## 實際案例回顧

### W01：`sc config` 成功

```
sc config FEITrainingService binPath= "cmd.exe /c type flag > proof"
→ [SC] ChangeServiceConfig 成功
```

**壞掉的**：Service Object 的 DACL 給了 Users `SERVICE_CHANGE_CONFIG`。

**FEI Lab 工程筆記**：原始 SDDL 缺少 `DC`（CHANGE_CONFIG），`sc config` 在第一次測試時被拒絕。修正後：`CCLCSW...` → `CCDCLCSW...`

### W02：替換 Binary 成功

```
copy payload.exe FEITrainingBinaryService.exe /Y
→ 複製了 1 個檔案。
```

**壞掉的**：Service Binary 的 NTFS ACL 給了 Users `Modify`。

### W03：候選 exe 被執行

```
copy payload.exe "C:\FEI Training\W03.exe"
→ Windows 先嘗試 C:\FEI Training\W03.exe
```

**壞掉的**：ImagePath 無引號 + 含空格 + 中間目錄可寫。

### W06：Registry 被修改

```
reg add "HKLM\SOFTWARE\FEI\PrivEsc\W06" /v ActionPath /t REG_SZ /d "payload.exe" /f
→ 操作順利完成。
```

**壞掉的**：Registry Key 的 ACL 給了 Users `WriteKey`。

### W09：DLL 被載入

```
copy FEIW09Plugin.dll "C:\FEI-PrivEsc\Training\W09\Bin\"
→ SYSTEM 載入 DLL
```

**壞掉的**：Application Directory 的 ACL 給了 Users `CreateFiles`。

---

## False Positive：「SYSTEM Service 存在」不是漏洞

| 聲稱 | 正確嗎？ |
|------|---------|
| 「這個 Service 以 SYSTEM 執行，所以有漏洞」 | ❌ SYSTEM Service 是正常的 |
| 「Service 的 ImagePath 沒有引號」 | ⚠️ 還要確認有空格 + 候選位置可寫 |
| 「Registry Key 有 Users 可寫」 | ⚠️ 還要確認哪個 value + 誰讀它 + 控制什麼 |
| 「Bin 目錄可以建立檔案」 | ⚠️ 還要確認 process 會載入該目錄的 DLL |

每一種都需要完整的條件鏈才成立。

---

## Edge Case

### 多層同時存在

在真實環境中，可能同時有多種弱點。例如：

```
Service ACL 弱 (W01)
+ Binary ACL 弱 (W02)
+ Unquoted Path (W03)
```

攻擊者會選最簡單的路徑。修復者必須全部修好。

### Service Recovery Options

有些 Service 設定了 Recovery Options（失敗時自動重啟）。即使攻擊者沒有 SERVICE_START 權限，Service 重啟也可能觸發 payload。

---

## Troubleshooting

### Service 安全評估 SOP

![Service 安全評估 SOP 資訊圖](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day25-windows-service-trust-boundary-diagram-03.png)


---

## 深入一層：SDDL 解讀速查

![SDDL 解讀速查資訊圖](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day25-windows-service-trust-boundary-diagram-04.png)

SDDL 看起來像亂碼，但結構很規則：


重要權限 bits：
- `DC` = SERVICE_CHANGE_CONFIG（W01 關鍵）
- `RP` = SERVICE_START
- `WP` = SERVICE_STOP

如果 `BU` 的 ACE 包含 `DC`，任何 User 都可以 `sc config`。

---

## 防禦者怎麼看？

### 完整 Service 安全清單

```
[ ] Service Account — 使用 Managed Service Account 而非 LocalSystem
[ ] Service ACL — 限制 CHANGE_CONFIG
[ ] Binary Path — 有引號
[ ] Binary ACL — 只有 admin 可寫
[ ] Binary Directory — 不允許 CreateFiles
[ ] Registry（如果使用）— 限制 WriteKey
[ ] DLL/Plugin paths — 不在可寫目錄
[ ] Recovery Options — 不自動執行任意程式
```

### Detection

```powershell
# 快速掃描所有 Service 的五種 Trust Surface
Get-Service | ForEach-Object {
    $name = $_.Name
    $cfg = sc.exe qc $name 2>&1
    # 檢查 ImagePath quoting, User, etc.
}
```

### Fix Verification

每修一層，都重新跑五項檢查。不要修了 W01 就停。

---

## Exercise：分析一個真實 Service

挑選你系統上的任何一個 SYSTEM Service（例如 `wuauserv`），完成以下檢查：

1. `sc qc wuauserv` — 誰在執行？ImagePath 有引號嗎？
2. `sc sdshow wuauserv` — Users 有什麼權限？有 DC 嗎？
3. `icacls <binary>` — Binary 可寫嗎？
4. `icacls <directory>` — 目錄可建立檔案嗎？
5. 如果這個 Service 讀 Registry 設定，那些 Key 的 ACL 安全嗎？

---

## Quiz

**Q1**：修好了 Service ACL（W01），Binary 也不可寫（W02），ImagePath 也加了引號（W03）。Service 現在安全嗎？

### 答案
不一定。還要檢查：Registry 設定的 ACL（W06）、DLL/Plugin 搜尋路徑的 ACL（W09）。五層都要保護。


**Q2**：SDDL 中的 `DC` 代表什麼？為什麼重要？

### 答案
DC = SERVICE_CHANGE_CONFIG。擁有此權限可以修改 Service 的 binPath，指向攻擊者控制的程式。這是 W01 的攻擊路徑。


**Q3**：為什麼「SYSTEM Service 存在」不是漏洞？

### 答案
Windows 有大量正常的 SYSTEM Service。Service 以 SYSTEM 執行是設計，不是漏洞。真正的問題是 Service 信任的資源（config、binary、path、registry、DLL）是否被低權限使用者控制。


---

## 今天真正要記住的 3 件事

1. **「Service 以 SYSTEM 執行」不是漏洞** — 問的是「它信任什麼？」
2. **五層 Trust Surface 都要保護** — ACL / Binary / Path / Registry / DLL
3. **修好一層不夠** — 攻擊者會找最弱的那一層

---

## 給自己的問題

> 到目前為止，Windows 的 W01-W09 都在問同一個核心問題：「低權限使用者能控制什麼？高權限 Process 信任什麼？」
>
> 那 Linux 呢？L01-L09 是不是也在問同一件事？

---

## 下一篇

Day 26：Linux PATH vs Windows Unquoted Path —— 兩個 OS 都在回答「到底執行誰？」
