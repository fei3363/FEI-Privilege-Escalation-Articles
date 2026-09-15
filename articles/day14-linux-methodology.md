# Day 14：打完 10 個 Linux Lab，我發現提權其實一直在找同一件事

## 開場

過去 10 天，我們做了 10 個 Linux 提權 Lab：

- L00：Enumeration
- L01：sudo
- L02：SUID
- L03：PATH Hijacking
- L04：Cron
- L05：Capabilities
- L06：Weak File Permission
- L07：Credential Exposure
- L08：Dangerous Group
- L09：systemd Drop-in

表面上，每一題的技術都不同。但如果你退一步看，會發現它們一直在回答同一個問題：

> **我能控制什麼？誰信任它？那個「誰」的權限比我高嗎？**

今天不做新的 Lab。今天要把這 10 題整理成一套方法論。

---

## 背景知識：為什麼需要方法論

### Checklist 方法的問題

很多人學提權是背 checklist：

```
✓ sudo -l
✓ find -perm -4000
✓ getcap
✓ cat /etc/crontab
✓ ls -la /etc/passwd
✓ docker group?
```

checklist 有用，但有一個根本問題：**它只能找你知道的漏洞。**

如果今天遇到一個 checklist 上沒有的情境呢？

### 方法論 vs Checklist

| | Checklist | 方法論 |
|---|----------|--------|
| 回答什麼 | 「這個東西有沒有」| 「為什麼可能形成提權」|
| 遇到新情境 | 不知道怎麼辦 | 可以推理 |
| 適合 | 快速初步掃描 | 深入分析 |

---

## 10 個 Lab 的統一分類

### 分類框架

![Linux Privilege Escalation 分類框架](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day14-linux-methodology-diagram-01.png)



### 每個分類的核心 Pattern

![Delegated Execution 核心 Pattern](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day14-linux-methodology-diagram-02.png)

![Identity 與 Execution Context 核心 Pattern](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day14-linux-methodology-diagram-03.png)

![Executable Resolution 核心 Pattern](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day14-linux-methodology-diagram-04.png)

![Scheduled 與 Managed Execution 核心 Pattern](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day14-linux-methodology-diagram-05.png)

![Trusted Resource 核心 Pattern](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day14-linux-methodology-diagram-06.png)

![Credential Exposure 核心 Pattern](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day14-linux-methodology-diagram-07.png)




#### 1. Delegated Execution（L01）


#### 2. Identity / Execution Context（L02, L05, L08）


#### 3. Executable Resolution（L03）


#### 4. Scheduled / Managed Execution（L04, L09）


#### 5. Trusted Resource（L06）


#### 6. Credential Exposure（L07）


---




## 共同抽象：Trust Relationship

![共同抽象：Trust Relationship](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day14-linux-methodology-diagram-08.png)


所有 10 個 Lab 都可以歸結為一句話：

> **低權限使用者可以控制的東西，被高權限主體信任了。**


---

## Linux PrivEsc Decision Tree

![Linux PrivEsc Decision Tree](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day14-linux-methodology-diagram-09.png)



---

## False Positive 匯總

| 發現 | 是否一定可提權 | 需要再驗證什麼 |
|------|--------------|--------------|
| `sudo -l` 有結果 | 不一定 | command 是否有 escape 功能？ |
| 找到 SUID binary | 不一定 | 是標準 binary 還是異常的？ |
| 找到 capability | 不一定 | 是 CAP_NET_RAW（正常）還是 CAP_SETUID（危險）？ |
| 找到 writable file | 不一定 | 有沒有高權限 Process 會使用它？ |
| 找到 password string | 不一定 | 是 decoy/example 還是真實有效的？ |
| 在 docker group | 不一定 | Docker daemon 有在跑嗎？ |
| cron job 存在 | 不一定 | script 可寫嗎？用 absolute path 嗎？ |

共同判斷框架：

```
發現某個設定/權限
    ↓
問 4 個問題：
    1. 我真的能控制它嗎？
    2. 有高權限 Process 信任它嗎？
    3. 我能觸發那個信任關係嗎？
    4. 會真的產生 root context 嗎？
```

---

## OS 原理：為什麼 Linux 的提權機制這麼多？

Linux 的設計哲學是 **Flexibility**：

```
sudo    → 細粒度的權限委派
SUID    → per-binary 的身份切換
Cap     → 拆分 root 特權
Group   → 資源存取控制
Cron    → 定時自動化
systemd → 統一服務管理
```

每一個機制都是為了解決「不想給 full root，但需要某些特權操作」的需求。

但每一個機制設定錯誤，都可能變成提權路徑。

---

## Exercise

### 練習 1：用 Decision Tree 分析

你拿到一台 Linux 機器的 shell（user: webapp）。執行以下指令後得到：

```bash
id
# uid=1000(webapp) gid=1000(webapp) groups=1000(webapp),999(docker)

sudo -l
# (ALL) NOPASSWD: /usr/bin/less /var/log/app.log

getcap -r / 2>/dev/null
# /usr/bin/python3.8 cap_setuid=ep

cat /etc/cron.d/backup
# * * * * * root /opt/backup/run.sh

ls -la /opt/backup/run.sh
# -rwxrwxrwx 1 root root 156 ... /opt/backup/run.sh
```

問題：
1. 列出所有可能的提權路徑
2. 排序風險/成功率
3. 你會先嘗試哪一條？為什麼？

### 練習 2：設計一個不在 Checklist 上的提權場景

用 Trust Relationship 框架，設計一個不在本系列 10 個 Lab 中的 Linux 提權場景。描述：
- 低權限使用者控制什麼？
- 高權限主體信任什麼？
- 如何觸發？

---

## Quiz

**Q1：** L04 (Cron) 和 L09 (systemd) 的核心差異是什麼？

### 答案
L04 的弱點是「被執行的 script 可寫」（Writable Executed Resource）。L09 的弱點是「Service 的 configuration（drop-in）可寫」（Writable Configuration）。前者是修改「執行什麼」，後者是修改「設定用什麼來執行」。


**Q2：** 為什麼 Checklist 方法有極限？

### 答案
Checklist 只能發現已知的漏洞模式。面對新的配置或不在清單上的場景時無法判斷。方法論（Trust Relationship 分析）可以從「我能控制什麼 → 誰信任它 → 能否觸發」推理出新的提權路徑。


**Q3：** 10 個 Linux Lab 的共同抽象是什麼？

### 答案
「低權限使用者可以控制的東西，被高權限主體信任了。」或者更精確地說：What can I control? → Who trusts it? → What privilege do they have? → Can I trigger that trust?


---

## 10 個 Linux Lab 總覽

| Lab | 分類 | 核心弱點 | Flag |
|-----|------|---------|------|
| L00 | Enumeration | — | — |
| L01 | Delegated Execution | sudo + binary escape | `FEI{LINUX_L01_ROOT_ACCESS}` |
| L02 | Identity Context | SUID root binary | `FEI{LINUX_L02_SUID_ROOT_ACCESS}` |
| L03 | Executable Resolution | PATH + writable dir | `FEI{LINUX_L03_PATH_ROOT_ACCESS}` |
| L04 | Scheduled Execution | Cron + writable script | `FEI{LINUX_L04_CRON_ROOT_ACCESS}` |
| L05 | Identity Context | CAP_SETUID | `FEI{LINUX_L05_CAPABILITIES_ROOT_ACCESS}` |
| L06 | Trusted Resource | Writable config + root runner | `FEI{LINUX_L06_WEAK_FILE_PERMISSION_ROOT_ACCESS}` |
| L07 | Credential Exposure | Plaintext root password | `FEI{LINUX_L07_CREDENTIAL_EXPOSURE_ROOT_ACCESS}` |
| L08 | Identity Context | disk group + raw device | `FEI{LINUX_L08_DANGEROUS_GROUP_ROOT_ACCESS}` |
| L09 | Managed Execution | systemd drop-in writable | `FEI{LINUX_L09_SYSTEMD_ROOT_ACCESS}` |

---

## 今天真正要記住的 3 件事

1. **所有提權都在問同一件事**：「我能控制什麼？誰信任它？」
2. **Checklist 幫你找已知的，方法論幫你推理未知的**
3. **10 個 Lab 的核心是 Trust Relationship，不是技術名稱**

---

## 給自己的問題

> 如果同樣的 Mental Model 套用到 Windows，Windows 的 Trust Boundary 在哪裡？Service、Registry、Token——Windows 的高權限主體信任什麼？

---

## 下一篇

> 💡 **切換到 Windows**
>
> 從明天起，我們轉戰 Windows。請確認：
> 1. FEI-PRIVESC-WINDOWS VM 可以正常開機
> 2. 可以用 `fei-student` / `FEI-Student-2026!` 登入
> 3. `C:\FEI-PrivEsc\` 目錄存在

Day 15：拿到 Windows Shell 後，不要急著跑工具：先看懂自己的 Token
