# pkill 命令完整指南

**文檔版本**：1.0
**日期**：2026-01-17
**目的**：理解並正確使用 pkill 命令管理進程

---

## 目錄

1. [什麼是 pkill](#什麼是-pkill)
2. [pkill vs kill vs killall](#pkill-vs-kill-vs-killall)
3. [基本語法](#基本語法)
4. [信號說明](#信號說明)
5. [常用範例](#常用範例)
6. [安全注意事項](#安全注意事項)
7. [實戰應用](#實戰應用)

---

## 什麼是 pkill

### 基本概念

`pkill` 是一個 Unix/Linux 命令，用於**根據進程名稱或其他屬性終止進程**。

**特點**：
- ✅ 根據**進程名稱**查找並終止
- ✅ 支援**正則表達式**匹配
- ✅ 可以根據**使用者**、**終端**等屬性過濾
- ✅ 一次性處理**多個進程**

---

### pkill 的由來

`pkill` 是 `pgrep` 工具套件的一部分：

```
pgrep  → 查找進程 (process grep)
pkill  → 終止進程 (process kill)
```

---

## pkill vs kill vs killall

### 命令比較

| 命令 | 搜尋方式 | 適用場景 |
|------|----------|----------|
| **kill** | 進程 PID | 精確終止單一進程 |
| **pkill** | 進程名稱/屬性 | 根據名稱終止多個進程 |
| **killall** | 完整進程名稱 | 終止所有符合名稱的進程 |

---

### 使用範例對比

#### kill - 需要知道 PID

```bash
# 步驟 1：查找進程 PID
ps aux | grep sshd
# root  1234  ... /usr/sbin/sshd

# 步驟 2：使用 PID 終止
kill -HUP 1234
```

**優點**：精確控制單一進程
**缺點**：需要先查找 PID，步驟繁瑣

---

#### pkill - 直接使用名稱

```bash
# 一條命令完成
pkill -HUP sshd
```

**優點**：簡潔快速，支援模式匹配
**缺點**：可能誤殺同名進程

---

#### killall - 終止完整名稱

```bash
# 終止完全符合名稱的所有進程
killall -HUP sshd
```

**注意**：
- Linux: 類似 pkill，但需要**完整名稱**匹配
- Solaris/AIX: ⚠️ 會終止**所有進程**（非常危險！）

---

## 基本語法

### 標準格式

```bash
pkill [選項] [模式]
```

### 常用選項

| 選項 | 說明 | 範例 |
|------|------|------|
| `-signal` | 發送指定信號 | `pkill -9 process` |
| `-u user` | 限定使用者 | `pkill -u www-data apache` |
| `-t term` | 限定終端 | `pkill -t pts/0` |
| `-P ppid` | 限定父進程 | `pkill -P 1234` |
| `-f` | 匹配完整命令列 | `pkill -f "python script.py"` |
| `-x` | 精確匹配 | `pkill -x sshd` |
| `-n` | 終止最新的進程 | `pkill -n chrome` |
| `-o` | 終止最舊的進程 | `pkill -o firefox` |
| `-c` | 顯示匹配數量 | `pkill -c sshd` |
| `-l` | 列出進程名稱 | `pkill -l sshd` |

---

### 搜尋模式

```bash
# 完整名稱
pkill sshd

# 部分匹配
pkill ssh          # 匹配 sshd, ssh-agent 等

# 正則表達式
pkill "^ssh"       # 匹配以 ssh 開頭的進程

# 忽略大小寫
pkill -i SSH

# 精確匹配（完全相同）
pkill -x sshd      # 只匹配 "sshd"，不匹配 "sshd-session"
```

---

## 信號說明

### 什麼是信號？

信號是 Unix/Linux 進程間通訊的一種方式，用來**通知進程發生了特定事件**。

---

### 常用信號列表

| 信號 | 數字 | 名稱 | 說明 | 進程反應 |
|------|------|------|------|----------|
| **SIGHUP** | 1 | HUP | Hang Up（掛斷） | 重新讀取配置 |
| **SIGINT** | 2 | INT | Interrupt（中斷） | Ctrl+C 的效果 |
| **SIGQUIT** | 3 | QUIT | Quit（退出） | 生成 core dump |
| **SIGKILL** | 9 | KILL | Kill（強制終止） | **立即終止**（無法捕捉） |
| **SIGTERM** | 15 | TERM | Terminate（終止） | **正常終止**（預設） |
| **SIGUSR1** | 10 | USR1 | User-defined 1 | 使用者自定義 |
| **SIGUSR2** | 12 | USR2 | User-defined 2 | 使用者自定義 |

---

### SIGHUP (1) - 重新載入配置

**用途**：通知守護進程重新讀取配置檔案

```bash
# 重新載入 sshd 配置
pkill -HUP sshd

# 等同於
pkill -1 sshd
```

**效果**：
- ✅ 進程**不會終止**
- ✅ 重新讀取配置檔案
- ✅ **不影響現有連接**（對 sshd 而言）

**適用服務**：
- sshd (SSH 伺服器)
- nginx (Web 伺服器)
- rsyslog (日誌服務)
- dnsmasq (DNS 服務)

---

### SIGTERM (15) - 正常終止

**用途**：要求進程**優雅地關閉**（預設信號）

```bash
# 正常終止 nginx
pkill nginx

# 等同於
pkill -TERM nginx
pkill -15 nginx
```

**效果**：
- ✅ 進程有機會**清理資源**
- ✅ 保存未完成的工作
- ✅ 關閉網路連接
- ✅ 可以被進程**捕捉並處理**

**流程**：
```
pkill -TERM process
  ↓
進程收到 SIGTERM
  ↓
執行清理函數
  ↓
保存狀態
  ↓
正常退出
```

---

### SIGKILL (9) - 強制終止

**用途**：**強制立即終止**進程（最後手段）

```bash
# 強制終止無響應的進程
pkill -9 hung_process

# 等同於
pkill -KILL hung_process
```

**效果**：
- ✅ **立即終止**，無法被捕捉
- ❌ 無法執行清理操作
- ❌ 可能導致**資源洩漏**
- ❌ 可能造成**資料損壞**

**何時使用**：
- 進程無響應
- SIGTERM 無效
- 緊急情況

**注意**：
⚠️ **SIGKILL 無法被捕捉**，進程無法執行任何清理操作

---

### 信號發送流程

```
正常終止流程（SIGTERM）：
pkill -TERM nginx
  ↓
nginx 收到 SIGTERM
  ↓
nginx 執行清理：
  - 完成處理中的請求
  - 關閉網路連接
  - 寫入日誌
  ↓
nginx 正常退出

強制終止流程（SIGKILL）：
pkill -9 nginx
  ↓
作業系統強制終止 nginx
  ↓
無任何清理操作
  ↓
立即消失
```

---

## 常用範例

### 範例 1：重新載入 SSH 伺服器配置

**情境**：修改 `/etc/ssh/sshd_config` 後需要重新載入

```bash
# 重新載入配置（不中斷現有連接）
pkill -HUP sshd
```

**為什麼用 HUP？**
- ✅ 不會關閉現有 SSH 連接
- ✅ 新連接使用新配置
- ✅ 不需要重啟整個服務

---

### 範例 2：終止特定使用者的所有進程

```bash
# 終止使用者 john 的所有進程
pkill -u john

# 終止使用者 www-data 的 php 進程
pkill -u www-data php
```

---

### 範例 3：終止特定終端的所有進程

```bash
# 查看目前終端
tty
# /dev/pts/1

# 終止該終端的所有進程
pkill -t pts/1

# 終止所有 SSH 連接
pkill -t "pts/*"
```

---

### 範例 4：根據完整命令列終止

```bash
# 終止特定 Python 腳本
pkill -f "python /path/to/script.py"

# 終止特定參數的 Java 程式
pkill -f "java.*myapp.jar"
```

**注意**：需要使用 `-f` 選項匹配完整命令列

---

### 範例 5：終止最新或最舊的進程

```bash
# 終止最新啟動的 Chrome 進程
pkill -n chrome

# 終止最舊的 Firefox 進程
pkill -o firefox
```

---

### 範例 6：顯示將被終止的進程

```bash
# 查看會匹配哪些進程（使用 pgrep）
pgrep -a sshd

# 顯示匹配數量
pkill -c sshd

# 列出匹配的進程名稱
pkill -l sshd
```

---

### 範例 7：終止子進程

```bash
# 終止 PID 1234 的所有子進程
pkill -P 1234

# 終止 nginx 的所有 worker 進程
pkill -P $(pgrep -o nginx)
```

---

## 安全注意事項

### ⚠️ 潛在風險

#### 1. 誤殺關鍵進程

```bash
# ❌ 危險：可能終止多個重要進程
pkill ssh
# 會匹配：sshd, ssh-agent, ssh-keygen 等

# ✅ 更安全：精確匹配
pkill -x sshd
```

---

#### 2. 影響生產服務

```bash
# ❌ 危險：直接終止所有 nginx 進程
pkill -9 nginx

# ✅ 更好：先嘗試正常終止
pkill -TERM nginx
sleep 5

# 如果還有殘留，再使用 -9
if pgrep nginx > /dev/null; then
  pkill -9 nginx
fi
```

---

#### 3. 無意中終止自己的 Shell

```bash
# ❌ 危險：可能終止自己的 bash
pkill bash

# ✅ 更安全：排除自己
pkill -u otheruser bash
```

---

#### 4. Root 權限濫用

```bash
# ❌ 危險：root 可以終止所有使用者的進程
sudo pkill -u www-data

# ✅ 更好：明確指定進程名稱
sudo pkill -u www-data php-fpm
```

---

### 最佳實踐

#### ✅ 使用前先確認

```bash
# 步驟 1：使用 pgrep 預覽
pgrep -a sshd

# 步驟 2：確認無誤後執行
pkill -HUP sshd
```

---

#### ✅ 使用精確匹配

```bash
# 避免部分匹配
pkill -x sshd         # 只匹配 "sshd"

# 匹配完整命令
pkill -f "exact command line"
```

---

#### ✅ 優先使用 SIGTERM

```bash
# 先嘗試優雅終止
pkill -TERM process_name

# 等待一段時間
sleep 3

# 檢查是否還在運行
if pgrep process_name > /dev/null; then
  # 最後手段：強制終止
  pkill -9 process_name
fi
```

---

#### ✅ 記錄操作日誌

```bash
# 記錄將被終止的進程
pgrep -a nginx >> /var/log/pkill.log
echo "$(date): Reloading nginx" >> /var/log/pkill.log

# 執行操作
pkill -HUP nginx
```

---

## 實戰應用

### 應用 1：Docker 容器中重載 sshd

**背景**：Docker 容器中的 sshd 沒有 init 系統，無法使用 systemctl 或 service 命令

**問題**：
```bash
# ❌ 容器中無效
systemctl reload sshd
service sshd reload

# ❌ pid 文件不存在
kill -HUP $(cat /var/run/sshd.pid)
```

**解決方案**：
```bash
# ✅ 使用 pkill
pkill -HUP sshd

# ✅ 或使用 killall
killall -HUP sshd
```

**為什麼可行？**
- ✅ 不依賴 init 系統
- ✅ 不需要 pid 文件
- ✅ 直接向進程發送信號

---

### 應用 2：清理殭屍進程

**情境**：系統中有大量殭屍 (zombie) 進程

```bash
# 查找殭屍進程
ps aux | grep defunct

# 終止殭屍進程的父進程
pkill -HUP -P $(pgrep defunct)

# 如果無效，強制終止父進程
pkill -9 -P $(pgrep defunct)
```

---

### 應用 3：批次終止使用者會話

**情境**：需要登出特定使用者的所有會話

```bash
# 終止使用者 testuser 的所有進程
sudo pkill -u testuser

# 只終止該使用者的 bash 會話
sudo pkill -u testuser bash
```

---

### 應用 4：重啟 Web 服務的 Worker

**情境**：Nginx 或 uWSGI 需要優雅地重啟 worker

```bash
# 查找主進程 PID
nginx_master=$(pgrep -o nginx)

# 發送 HUP 信號重新載入
kill -HUP $nginx_master

# 或直接使用 pkill
pkill -HUP -o nginx
```

---

### 應用 5：定期清理過期進程

**腳本範例**：
```bash
#!/bin/bash
# cleanup_old_sessions.sh

# 查找運行超過 24 小時的 tmux 會話
old_sessions=$(ps -eo pid,etimes,cmd | awk '$2 > 86400 && $3 ~ /tmux/ {print $1}')

if [ -n "$old_sessions" ]; then
  echo "Cleaning up old tmux sessions: $old_sessions"
  for pid in $old_sessions; do
    kill -TERM $pid
  done
else
  echo "No old tmux sessions found"
fi
```

---

## 進階技巧

### 組合使用 pgrep 和 pkill

```bash
# 查找符合條件的進程
pids=$(pgrep -f "python.*server.py")

# 檢查是否有匹配
if [ -n "$pids" ]; then
  echo "Found processes: $pids"
  # 執行操作
  pkill -f "python.*server.py"
else
  echo "No matching processes"
fi
```

---

### 使用 xargs 進行精細控制

```bash
# 對每個匹配的進程單獨處理
pgrep nginx | xargs -I {} sh -c 'echo "Killing PID: {}"; kill -TERM {}'
```

---

### 條件性終止

```bash
#!/bin/bash
# 只在負載高時終止特定進程

load=$(uptime | awk -F'load average:' '{print $2}' | awk '{print $1}' | cut -d. -f1)

if [ $load -gt 10 ]; then
  echo "High load detected: $load"
  pkill -STOP heavy_process  # 暫停進程
else
  echo "Load normal: $load"
  pkill -CONT heavy_process  # 繼續進程
fi
```

---

## 疑難排解

### Q1: pkill 沒有終止任何進程？

**檢查步驟**：
```bash
# 1. 確認進程存在
pgrep -a process_name

# 2. 檢查權限
ps -o user,pid,cmd | grep process_name

# 3. 嘗試使用完整命令列
pkill -f "complete command"

# 4. 使用精確匹配
pkill -x exact_name
```

---

### Q2: pkill 終止了不該終止的進程？

**原因**：模式匹配太寬泛

**解決方案**：
```bash
# 使用更具體的模式
pkill -x exact_process_name

# 或使用完整命令列
pkill -f "/usr/bin/specific_process --with-args"

# 限定使用者
pkill -u specific_user process_name
```

---

### Q3: SIGHUP 後進程還是使用舊配置？

**可能原因**：
1. 進程不支援 SIGHUP 重載
2. 配置檔案有錯誤
3. 需要完全重啟

**檢查方法**：
```bash
# 查看進程日誌
tail -f /var/log/syslog | grep process_name

# 測試配置
process_name -t  # 某些程式支援配置測試

# 最後手段：完全重啟
pkill process_name
/path/to/process_name
```

---

### Q4: pkill 在 Docker 容器中無效？

**可能原因**：
1. 容器中沒有目標進程
2. 進程名稱不同
3. 權限不足

**調試步驟**：
```bash
# 1. 列出所有進程
ps aux

# 2. 確認進程名稱
ps -o comm=

# 3. 使用完整路徑
pkill -f "/usr/sbin/sshd"
```

---

## 相關命令

### pgrep - 查找進程

```bash
# 查找進程 PID
pgrep sshd

# 顯示進程名稱和 PID
pgrep -a sshd

# 顯示完整命令列
pgrep -af "python.*script"
```

---

### ps - 列出進程

```bash
# 顯示所有進程
ps aux

# 顯示特定格式
ps -eo pid,user,cmd

# 樹狀顯示
ps auxf
```

---

### kill - 發送信號

```bash
# 發送 SIGTERM
kill 1234

# 發送 SIGHUP
kill -HUP 1234

# 發送 SIGKILL
kill -9 1234
```

---

### killall - 根據名稱終止

```bash
# 終止所有 nginx 進程
killall nginx

# 發送特定信號
killall -HUP sshd
```

**注意**：
- Linux: 安全，類似 pkill
- Solaris/AIX: ⚠️ 危險，會終止所有進程！

---

## 總結

### 何時使用 pkill？

| 情境 | 推薦命令 | 原因 |
|------|----------|------|
| 知道確切 PID | `kill` | 最精確 |
| 根據名稱終止 | `pkill` | 最方便 |
| 需要複雜過濾 | `pkill` + 選項 | 最靈活 |
| Docker 容器中 | `pkill` | 無 init 系統 |
| 批次操作 | `pkill` | 一次處理多個 |

---

### 關鍵要點

1. **信號選擇**
   - SIGHUP (1): 重新載入配置
   - SIGTERM (15): 正常終止（預設）
   - SIGKILL (9): 強制終止（最後手段）

2. **安全使用**
   - 使用前用 `pgrep` 確認
   - 優先使用 `-x` 精確匹配
   - 避免在生產環境直接使用 `-9`

3. **Docker 環境**
   - `pkill` 不依賴 init 系統
   - 適合容器內使用
   - 注意檢查實際進程名稱

---

## 快速參考

```bash
# 重新載入配置
pkill -HUP process_name

# 正常終止
pkill -TERM process_name

# 強制終止
pkill -9 process_name

# 精確匹配
pkill -x exact_name

# 限定使用者
pkill -u username process_name

# 匹配完整命令列
pkill -f "full command line"

# 預覽將被終止的進程
pgrep -a process_name
```

---

**最後更新**：2026-01-17
**維護者**：系統管理團隊
