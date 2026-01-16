# SSH Known Hosts 衝突解決指南

**文檔版本**：1.0
**日期**：2026-01-17
**目的**：理解並解決 SSH known_hosts 衝突問題

---

## 目錄

1. [什麼是 Known Hosts](#什麼是-known-hosts)
2. [為什麼會出現衝突](#為什麼會出現衝突)
3. [衝突錯誤訊息](#衝突錯誤訊息)
4. [解決方法](#解決方法)
5. [預防措施](#預防措施)
6. [安全考量](#安全考量)

---

## 什麼是 Known Hosts

### 基本概念

`known_hosts` 是 SSH 客戶端用來**驗證遠端主機身份**的檔案。

**檔案位置**：
```
~/.ssh/known_hosts          # 使用者層級
/etc/ssh/ssh_known_hosts    # 系統層級
```

### 工作原理

```
第一次連接到遠端主機：
┌──────────────────┐
│  SSH 客戶端      │
│  192.168.56.102  │
└─────────┬────────┘
          │ (1) 連接請求
          ▼
┌──────────────────┐
│  SSH 伺服器      │
│  192.168.100.50  │
│  回傳 Host Key   │
└─────────┬────────┘
          │ (2) Host Key: abc123...
          ▼
┌──────────────────┐
│  詢問使用者      │
│  是否信任？      │
│  [yes/no]        │
└─────────┬────────┘
          │ (3) yes
          ▼
┌──────────────────┐
│  儲存到          │
│  known_hosts     │
│  192.168.100.50  │
│  → abc123...     │
└──────────────────┘
```

### Known Hosts 檔案格式

```bash
# 標準格式
192.168.100.50 ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAAB...

# 雜湊格式（更安全）
|1|JfKTdBh7rNbXkVAQup... ssh-rsa AAAAB3NzaC1yc2EAAA...

# 包含端口號
[192.168.213.31]:2222 ssh-rsa AAAAB3NzaC1yc2EAAAADAQAB...
```

**欄位說明**：
1. **主機名稱/IP**：可以是 hostname 或 IP
2. **密鑰類型**：ssh-rsa, ecdsa-sha2-nistp256, ssh-ed25519 等
3. **公鑰內容**：Base64 編碼的公鑰

---

## 為什麼會出現衝突

### 常見原因

#### 1. 主機重新安裝或重建
```
舊的主機：192.168.100.50 → Host Key: abc123
  ↓ (重新安裝系統或容器重建)
新的主機：192.168.100.50 → Host Key: xyz789

known_hosts 中還是舊的 abc123 → ❌ 衝突！
```

#### 2. Docker 容器重建
```bash
# 容器每次重建都會生成新的 SSH Host Key
docker stop bastion-ssh
docker rm bastion-ssh
docker run ...              # 新的 Host Key 生成

# 下次連接時會出現衝突
ssh root@192.168.213.31
# ❌ WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!
```

#### 3. 跳板機切換
```
原本使用：192.168.213.136 → 192.168.100.50
  ↓
改用：    192.168.213.31  → 192.168.100.50

透過不同跳板機，目標主機的 Host Key 可能看起來不同
```

#### 4. IP 地址重複使用
```
舊設備：192.168.100.50 (Router A)
  ↓ (除役並換成新設備)
新設備：192.168.100.50 (Router B)

相同 IP，不同的 Host Key → 衝突
```

#### 5. DNS 劫持或中間人攻擊（安全威脅）
```
⚠️ 惡意情況：有人試圖冒充該主機
正常主機：server.com → abc123
  ↓
惡意主機：server.com → xyz789 (假冒)

SSH 偵測到 Host Key 不符，發出警告
```

---

## 衝突錯誤訊息

### 完整錯誤範例

```
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@    WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!     @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
IT IS POSSIBLE THAT SOMEONE IS DOING SOMETHING NASTY!
Someone could be eavesdropping on you right now (man-in-the-middle attack)!
It is also possible that a host key has just been changed.
The fingerprint for the RSA key sent by the remote host is
SHA256:NNvF7BoRcKC24WVtdFz6pu8rSFP2kLtpGEBWJ5GVK0I.
Please contact your system administrator.
Add correct host key in /home/geek/.ssh/known_hosts to get rid of this message.
Offending RSA key in /home/geek/.ssh/known_hosts:5
  remove with:
  ssh-keygen -f "/home/geek/.ssh/known_hosts" -R "192.168.100.50"
Password authentication is disabled to avoid man-in-the-middle attacks.
Keyboard-interactive authentication is disabled to avoid man-in-the-middle attacks.
UpdateHostkeys is disabled because the host key is not trusted.
cisco123@192.168.100.50: Permission denied (publickey,keyboard-interactive,password).
```

### 錯誤訊息分析

| 部分 | 說明 |
|------|------|
| **WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!** | 主機身份已變更 |
| **Offending RSA key in ... :5** | 衝突的舊 key 在第 5 行 |
| **remove with: ssh-keygen -f ... -R ...** | 建議的移除命令 |
| **Permission denied** | 因安全考量拒絕連接 |

---

## 解決方法

### 方法 1：使用 ssh-keygen 移除（推薦）

**語法**：
```bash
ssh-keygen -f <known_hosts 檔案路徑> -R <主機名稱或IP>
```

**範例**：
```bash
# 移除特定 IP
ssh-keygen -f ~/.ssh/known_hosts -R 192.168.100.50

# 移除特定主機名
ssh-keygen -f ~/.ssh/known_hosts -R server.example.com

# 移除特定端口的主機
ssh-keygen -f ~/.ssh/known_hosts -R "[192.168.213.31]:2222"
```

**輸出結果**：
```
# Host 192.168.100.50 found: line 5
/home/geek/.ssh/known_hosts updated.
Original contents retained as /home/geek/.ssh/known_hosts.old
```

**優點**：
- ✅ 安全：只移除指定主機的 key
- ✅ 自動備份：保留 `.old` 備份檔案
- ✅ 精確：不影響其他主機的 key

---

### 方法 2：手動編輯 known_hosts

```bash
# 使用編輯器打開
vi ~/.ssh/known_hosts

# 找到並刪除衝突的那一行
# (根據錯誤訊息中的行號，例如：line 5)
```

**優點**：
- ✅ 完全控制
- ✅ 可以同時檢查其他 key

**缺點**：
- ❌ 手動操作容易出錯
- ❌ 沒有自動備份

---

### 方法 3：刪除整個 known_hosts（不推薦）

```bash
# ⚠️ 這會刪除所有已知主機的記錄
rm ~/.ssh/known_hosts
```

**何時使用**：
- 實驗室環境
- 測試環境
- 已確認所有主機都重建過

**缺點**：
- ❌ 失去所有主機的驗證記錄
- ❌ 下次連接所有主機都需要重新確認

---

### 方法 4：使用 StrictHostKeyChecking=no（不安全）

```bash
# ⚠️ 不驗證主機身份，存在安全風險
ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null user@host
```

**適用情境**：
- ✅ 自動化腳本（搭配其他安全措施）
- ✅ 頻繁重建的測試環境
- ✅ 內部隔離網路

**注意事項**：
- ⚠️ 容易遭受中間人攻擊
- ⚠️ 不應該在生產環境使用
- ⚠️ 不會將 key 儲存到 known_hosts

---

## 預防措施

### 1. Docker 容器使用持久化 Host Key

**問題**：容器每次重建都生成新的 SSH Host Key

**解決方案**：掛載 Host Key 到容器外部

```dockerfile
# Dockerfile
VOLUME /etc/ssh/host_keys
```

```bash
# docker-compose.yml
services:
  bastion:
    volumes:
      - ./host_keys:/etc/ssh/host_keys
```

```bash
# entrypoint.sh
if [ -f /etc/ssh/host_keys/ssh_host_rsa_key ]; then
  cp /etc/ssh/host_keys/* /etc/ssh/
else
  ssh-keygen -A
  cp /etc/ssh/ssh_host_* /etc/ssh/host_keys/
fi
```

---

### 2. 使用雜湊格式（增強安全性）

```bash
# 在 ~/.ssh/config 中配置
Host *
    HashKnownHosts yes
```

**效果**：
```
# 原本（可讀）
192.168.100.50 ssh-rsa AAAAB3NzaC1yc2EAAA...

# 雜湊後（隱藏主機名）
|1|JfKTdBh7rNbXkVAQup... ssh-rsa AAAAB3NzaC1yc2EAAA...
```

**優點**：
- ✅ 防止洩漏你連接過哪些主機
- ✅ 增強隱私保護

**缺點**：
- ❌ 難以手動編輯
- ❌ 必須使用 ssh-keygen -R 移除

---

### 3. 定期備份 known_hosts

```bash
# 手動備份
cp ~/.ssh/known_hosts ~/.ssh/known_hosts.backup

# 自動備份（加入 .bashrc）
alias ssh='cp ~/.ssh/known_hosts ~/.ssh/known_hosts.backup.$(date +%Y%m%d) 2>/dev/null; /usr/bin/ssh'
```

---

### 4. 針對測試環境設定特殊規則

**~/.ssh/config**：
```
# 生產環境：嚴格檢查
Host *.prod.company.com
    StrictHostKeyChecking yes

# 測試環境：寬鬆檢查
Host *.test.company.com
    StrictHostKeyChecking accept-new
    UserKnownHostsFile /dev/null

# 本地開發：不檢查
Host 192.168.*.* 10.0.*.*
    StrictHostKeyChecking no
    UserKnownHostsFile /dev/null
```

---

## 安全考量

### 何時應該警惕？

#### ⚠️ 高風險情境

1. **生產環境主機** Host Key 突然改變
2. **公共網路**連接到遠端伺服器時出現衝突
3. **重要系統**（金融、醫療等）突然出現警告
4. **沒有重新安裝**或重建，但 Host Key 改變

**正確做法**：
```
❌ 不要：直接刪除並重新連接
✅ 應該：
   1. 聯繫系統管理員確認
   2. 檢查是否有授權的變更
   3. 使用其他方式（如 console）驗證主機
   4. 確認後再更新 known_hosts
```

---

#### ✅ 低風險情境

1. **剛重新安裝**系統
2. **Docker 容器重建**
3. **測試環境**頻繁變動
4. **已知的 IP 重複使用**

**正確做法**：
```
✅ 可以：使用 ssh-keygen -R 移除並重新連接
✅ 建議：記錄變更原因和時間
```

---

### 中間人攻擊 (MITM) 檢測

Known Hosts 機制的主要目的是**防止中間人攻擊**：

```
正常連接：
Client ←──────────────→ Real Server
       (Host Key 匹配)

中間人攻擊：
Client ←───→ Attacker ←───→ Real Server
       ❌ (Host Key 不匹配！)
       SSH 阻止連接，發出警告
```

---

## 常見問題與解答

### Q1: 為什麼不能自動接受新的 Host Key？

**A**: 這會讓你容易遭受中間人攻擊。Host Key 變更必須是有意識的決定。

---

### Q2: StrictHostKeyChecking 有哪些選項？

**A**:
```bash
# 嚴格模式（預設，最安全）
StrictHostKeyChecking yes
# - 已知主機 Key 必須匹配
# - 未知主機直接拒絕

# 詢問模式
StrictHostKeyChecking ask
# - 已知主機 Key 必須匹配
# - 未知主機詢問是否接受

# 自動接受新主機（較不安全）
StrictHostKeyChecking accept-new
# - 已知主機 Key 必須匹配
# - 未知主機自動接受並記錄

# 完全不檢查（危險）
StrictHostKeyChecking no
# - 永不驗證 Host Key
# - 容易遭受 MITM 攻擊
```

---

### Q3: 使用 ProxyCommand 時的 Known Hosts 管理？

**A**:
透過跳板機連接時，Host Key 驗證仍然有效：

```bash
# 跳板機的 Host Key
ssh root@bastion → 驗證 bastion 的 Host Key

# 目標主機的 Host Key
ssh -o ProxyCommand='ssh -W %h:%p root@bastion' user@target
→ 驗證 target 的 Host Key（非 bastion）
```

**注意**：
- 跳板機和目標主機的 Host Key **分別記錄**
- 更換跳板機**不影響**目標主機的 Host Key

---

### Q4: 如何查看某個主機的 Host Key？

```bash
# 方法 1：從 known_hosts 中查找
grep "192.168.100.50" ~/.ssh/known_hosts

# 方法 2：使用 ssh-keyscan 掃描
ssh-keyscan 192.168.100.50

# 方法 3：連接時顯示
ssh -v user@192.168.100.50 2>&1 | grep "Server host key"
```

---

### Q5: 雜湊格式的 known_hosts 如何移除特定主機？

```bash
# 使用 ssh-keygen -R（支援雜湊格式）
ssh-keygen -f ~/.ssh/known_hosts -R 192.168.100.50

# 或使用 -H 選項搜尋
ssh-keygen -H -F 192.168.100.50
```

---

## 實戰範例

### 範例 1：Docker 跳板機重建後的處理

**情境**：Docker bastion-ssh 容器重建，導致 Host Key 改變

**步驟**：
```bash
# 1. 移除舊的 Host Key
ssh-keygen -f ~/.ssh/known_hosts -R "[192.168.213.31]:2222"

# 2. 重新連接（會提示接受新的 Host Key）
ssh -i ~/.ssh/docker-bastion_key -p 2222 root@192.168.213.31

# 3. 確認並接受
The authenticity of host '[192.168.213.31]:2222' can't be established.
RSA key fingerprint is SHA256:abc123...
Are you sure you want to continue connecting (yes/no)? yes
```

---

### 範例 2：切換跳板機導致目標主機 Key 衝突

**情境**：從 .136 切換到 .31 跳板機，目標主機 192.168.100.50 的 Key 看起來不同

**步驟**：
```bash
# 1. 清除舊的目標主機 Host Key
ssh-keygen -f ~/.ssh/known_hosts -R "192.168.100.50"

# 2. 透過新跳板機連接
ssh -o ProxyCommand='ssh -W %h:%p -p 2222 -i ~/.ssh/docker-bastion_key root@192.168.213.31' \
    cisco123@192.168.100.50
```

---

### 範例 3：批次清理多個主機

```bash
# 清理整個網段
for ip in {1..254}; do
  ssh-keygen -f ~/.ssh/known_hosts -R "192.168.100.$ip"
done

# 清理特定主機列表
hosts=(
  "192.168.100.50"
  "192.168.100.51"
  "192.168.100.52"
)

for host in "${hosts[@]}"; do
  ssh-keygen -f ~/.ssh/known_hosts -R "$host"
done
```

---

## 最佳實踐總結

### ✅ 建議做法

1. **定期備份** `known_hosts` 檔案
2. **Docker 容器使用持久化** Host Key
3. **測試環境和生產環境分開管理**
4. **遇到衝突時先調查原因**，再決定如何處理
5. **使用 ssh-keygen -R** 移除，不要手動編輯

---

### ❌ 避免做法

1. ❌ 生產環境使用 `StrictHostKeyChecking=no`
2. ❌ 隨意刪除整個 `known_hosts` 檔案
3. ❌ 不調查原因就直接接受新 Key
4. ❌ 在公共網路忽略 Host Key 警告
5. ❌ 共用 `known_hosts` 檔案（多人共用同一帳號）

---

## 相關資源

### 官方文檔
- [OpenSSH Manual: ssh(1)](https://man.openbsd.org/ssh)
- [OpenSSH Manual: ssh-keygen(1)](https://man.openbsd.org/ssh-keygen)
- [OpenSSH Manual: sshd_config(5)](https://man.openbsd.org/sshd_config)

### 延伸閱讀
- SSH Host Key 驗證機制原理
- 中間人攻擊 (MITM) 防護
- SSH 安全最佳實踐

---

**最後更新**：2026-01-17
**維護者**：系統管理團隊
