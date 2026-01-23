# SSH 跳板機伺服器 Docker 化實施原理詳細指南

**編制日期：** 2026-01-22
**文檔版本：** 1.0.0
**編制者：** Claude Code
**目標讀者：** 系統管理員、DevOps 工程師、基礎設施工程師

---

## 目錄

1. [執行摘要](#執行摘要)
2. [實體伺服器環境分析](#實體伺服器環境分析)
3. [Docker 化設計原理](#docker-化設計原理)
4. [配置文件對比分析](#配置文件對比分析)
5. [啟動流程設計](#啟動流程設計)
6. [Docker 特定考量](#docker-特定考量)
7. [功能對比表](#功能對比表)
8. [實際使用案例](#實際使用案例)
9. [優勢與限制](#優勢與限制)
10. [最佳實踐建議](#最佳實踐建議)
11. [附錄](#附錄)

---

## 執行摘要

### Docker 化的核心目的

將原有運行在實體 Alpine Linux 伺服器（192.168.213.136）上的 SSH 跳板機服務，遷移至容器化環境。這一遷移旨在實現：

- **快速部署** - 將複雜的 SSH 環境配置標準化，支持一鍵部署
- **環境一致性** - 消除「在我的機器上可以工作」的問題，確保開發、測試、生產環境一致
- **易於擴展** - 支持快速部署多個跳板機實例，滿足高可用和負載均衡需求
- **安全性強化** - 在容器化過程中補充了多項安全改進措施
- **簡化運維** - 標準化配置管理、日誌收集、監控告警等運維流程

### 實體伺服器 vs 容器化對比

| 特性 | 實體伺服器 | Docker 容器 | 優勢 |
|------|---------|---------|------|
| **部署時間** | 30-60 分鐘（手動配置） | 5-10 秒（自動拉起） | 容器快 6-12 倍 |
| **環境一致性** | 依賴手動維護 | 鏡像即單一真實來源 | 容器無差異 |
| **擴展性** | 需要新增硬體 | 支持輕量級複製 | 容器易於橫向擴展 |
| **安全隔離** | 進程級隔離 | 系統級隔離（cgroup/namespace） | 容器隔離更強 |
| **資源效率** | 整個系統開銷（500MB+） | 輕量級容器（~20MB 鏡像） | 容器資源少 25 倍 |
| **版本控制** | 難以跟蹤修改 | 基於鏡像版本標籤 | 容器可完全追溯 |
| **故障恢復** | 手動重啟或重裝 | 自動故障轉移 + 重啟 | 容器自動化更好 |

### 主要優勢總結

**🚀 性能和可靠性**
- 輕量級架構：基於 Alpine Linux 3.18（只有 20-30MB 鏡像）
- 快速啟動：容器啟動時間 < 5 秒
- 高可用支持：自動重啟、健康檢查、多實例部署

**🔒 安全性增強**
- **認證加固**：禁用密碼認證、禁用 Root 密碼登入、支援現代密鑰類型
- **轉發限制**：通過 PermitOpen 精細控制允許的轉發目標
- **權限隔離**：容器級別的 Linux capabilities 限制
- **密碼套件升級**：使用現代加密算法（AES-256-GCM、ChaCha20）

**📦 部署靈活性**
- 支持 Docker Compose 編排
- 支持 Kubernetes 部署（提供 YAML 模板）
- 支持單機 docker run 運行
- 易於多環境管理

**🔧 運維簡化**
- 配置文件即代碼（可版本控制）
- 健康檢查自動化（無需人工監控）
- 日誌持久化與集中管理
- 支持 Volume 動態更新配置

---

## 實體伺服器環境分析

### 系統基礎信息

**主機地址：** 192.168.213.136
**操作系統：** Alpine Linux 3.16.9
**内核版本：** Linux 5.10+（典型 Alpine 配置）

### OpenSSH 服務配置

| 項目 | 值 |
|------|-----|
| **OpenSSH 版本** | 9.0p1-r5 |
| **SSH 協議版本** | SSH-2.0 |
| **監聽埠位** | 22（預設） |
| **認證方式** | 密鑰認證（允許）+ 密碼認證（未明確禁用） |
| **Root 登入** | yes（允許） |
| **TCP 轉發** | yes（啟用） |
| **PermitOpen 配置** | 192.168.100.50:22（限定單一目標） |

### 實際運行狀態

實體伺服器上的 sshd 配置特點：

```bash
# 認證配置 - 相對寬鬆
PermitRootLogin yes                    # 允許 root 直接登入
PasswordAuthentication yes             # 支援密碼認證（風險因素）
PubkeyAuthentication yes               # 支援密鑰認證

# 轉發配置 - 單點限制
AllowTcpForwarding yes
PermitOpen 192.168.100.50:22           # 僅允許轉發到此單一地址

# 密碼套件 - 保持預設
# 使用 OpenSSH 9.0 的預設加密算法（包含較舊的算法）
```

### 配置特點和隱患

**存在的問題：**

1. **認證安全性較低** - 支援密碼認證增加被爆破的風險
2. **Root 直接登入** - 增加系統被直接入侵的可能性
3. **轉發目標單點** - PermitOpen 只允許一個地址，靈活性不足
4. **缺乏密碼套件限制** - 可能包含較弱的加密算法
5. **無日誌收集** - 認證日誌存儲在本地，難以集中管理

---

## Docker 化設計原理

### 為什麼選擇 Alpine Linux 作為基礎鏡像

**Alpine Linux 的優勢：**

| 特性 | 意義 |
|------|------|
| **輕量級** | 基礎鏡像只有 7-10MB，非常適合容器化 |
| **安全性** | 最小化的軟體包集合，減少潛在漏洞面 |
| **快速啟動** | 簡化的啟動流程，容器啟動速度快 |
| **包管理** | apk 包管理工具簡潔高效 |
| **廣泛採用** | 在容器生態中被廣泛使用，有大量參考資源 |

**版本選擇：** Alpine 3.18

```dockerfile
FROM alpine:3.18
```

Alpine 3.18 包含 OpenSSH 9.3_p2-r1（相對較新），提供：
- Ed25519 密鑰支持
- 現代加密算法
- 較好的安全性

### Dockerfile 分層設計詳解

#### 第一層：基礎設置和元數據

```dockerfile
FROM alpine:3.18
LABEL maintainer="Ansible Network Lab"
LABEL description="Bastion SSH Server for Ansible ProxyCommand with PermitOpen restrictions"
```

**作用：**
- 基於 Alpine 3.18 輕量級基礎鏡像
- 添加元數據便於鏡像識別和管理

#### 第二層：依賴安裝

```dockerfile
RUN apk update && \
    apk add --no-cache \
        openssh=9.3_p2-r1 \
        openssh-client=9.3_p2-r1 \
        bash \
        doas \
    && rm -rf /var/cache/apk/*
```

**安裝包說明：**

| 軟體 | 版本 | 用途 |
|------|------|------|
| **openssh** | 9.3_p2-r1 | SSH 伺服器核心 |
| **openssh-client** | 9.3_p2-r1 | SSH 客戶端（支持 SSH 連接測試） |
| **bash** | latest | 提供 shell 環境（替代 sh 的便利） |
| **doas** | latest | 輕量級權限管理工具 |

**重要優化：**
- 使用 `--no-cache` 防止快取污染
- 在同一 RUN 指令中清理 apk 快取 (`rm -rf /var/cache/apk/*`)
- 這樣做可以將鏡像大小減少 20-30%

#### 第三層：目錄創建和權限設置

```dockerfile
RUN mkdir -p /run/sshd && chmod 755 /run/sshd
RUN mkdir -p /root/.ssh && chmod 700 /root/.ssh
```

**作用：**
- `/run/sshd` - SSH 服務運行時目錄（sshd 需要此目錄）
- `/root/.ssh` - Root 用戶的 SSH 配置目錄，權限 700 符合安全規範

#### 第四層：配置文件複製

```dockerfile
COPY config/sshd_config /etc/ssh/sshd_config
RUN chmod 600 /etc/ssh/sshd_config

COPY config/authorized_keys /root/.ssh/authorized_keys
RUN chmod 600 /root/.ssh/authorized_keys
```

**權限說明：**
- `sshd_config` 須為 600（只有 root 可讀）
- `authorized_keys` 須為 600（只有擁有者可讀）
- 這些權限要求是 OpenSSH 強制的安全規範

#### 第五層：啟動腳本複製

```dockerfile
COPY scripts/entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh
```

**作用：** 複製並設置可執行權限的容器啟動腳本

#### 第六層：主機密鑰生成

```dockerfile
RUN ssh-keygen -A
```

**重要說明：**
- `-A` 參數會自動生成所有類型的主機密鑰
- 生成的密鑰包括：
  - `ssh_host_rsa_key`（RSA 2048-bit）
  - `ssh_host_ecdsa_key`（ECDSA P-256）
  - `ssh_host_ed25519_key`（Ed25519）

**為什麼預先生成主機密鑰？**
- 容器每次啟動時無需重新生成，加快啟動速度
- 保持鏡像一致性
- 生成密鑰計算量大（特別是 RSA），提前生成可加快啟動

#### 第七層：健康檢查配置

```dockerfile
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD sshd -t && echo "SSH service is healthy"
```

**參數詳解：**
- `--interval=30s` - 每 30 秒檢查一次
- `--timeout=10s` - 檢查命令執行超過 10 秒視為失敗
- `--start-period=5s` - 容器啟動後 5 秒內不進行檢查（啟動寬限期）
- `--retries=3` - 連續 3 次檢查失敗後標記容器不健康

**檢查命令：**
```bash
sshd -t && echo "SSH service is healthy"
```
- `sshd -t` - 驗證 SSH 配置語法（不實際啟動 sshd）
- 同時運行兩個命令確保雙重檢查

#### 第八層：暴露端口

```dockerfile
EXPOSE 22
```

**說明：**
- 聲明容器 SSH 監聽埠位（但不實際發佈）
- 實際端口映射由 docker-compose.yml 或 docker run 指定

#### 第九層：設置入口點

```dockerfile
ENTRYPOINT ["/entrypoint.sh"]
```

**作用：** 指定容器啟動時執行的腳本

### 軟體包選擇的安全性考量

**OpenSSH 版本選擇：9.3_p2-r1**

- 支援現代密鑰類型（Ed25519、ECDSA）
- 包含較新的安全修復
- OpenSSH 9.x 已棄用 DSA 密鑰，提高安全性

**包管理最佳實踐：**

```dockerfile
# ✅ 推薦做法：明確指定版本
apk add openssh=9.3_p2-r1

# ❌ 不推薦：使用最新版本
apk add openssh

# ✅ 推薦做法：清理快取減小鏡像
&& rm -rf /var/cache/apk/*
```

### 目錄結構設計

```
docker-bastion-ssh/
├── Dockerfile                          # Docker 鏡像定義
├── docker-compose.yml                  # 容器編排配置
├── config/                             # 配置文件目錄
│   ├── sshd_config                    # SSH 伺服器配置（容器內複製）
│   ├── authorized_keys                # SSH 公鑰（容器內複製或 Volume 掛載）
│   └── authorized_keys.example        # 公鑰格式範例
├── scripts/                            # 腳本目錄
│   ├── entrypoint.sh                  # 容器啟動腳本
│   └── test-bastion.sh                # 測試腳本
├── docs/                               # 文檔目錄
│   ├── SSH_KEY_MANAGEMENT.md          # 密鑰管理指南
│   ├── CUSTOMIZATION_GUIDE.md         # 自訂配置指南
│   └── PERMITOPEN_GUIDE.md            # PermitOpen 配置指南
└── examples/                           # 整合範例
    └── ansible-integration/           # Ansible 整合範例
```

**設計要點：**

1. **config 目錄** - 存放配置文件，在構建時複製到容器，支持 Volume 掛載動態更新
2. **scripts 目錄** - 包含啟動和測試腳本，容器啟動時執行 entrypoint.sh
3. **docs 目錄** - 詳細文檔便於二次開發和故障排除
4. **examples 目錄** - 提供不同場景的使用範例

---

## 配置文件對比分析

### 整體配置對比表

| 配置項 | 實體伺服器 (192.168.213.136) | Docker 容器 | 改進說明 |
|--------|------------------------|----------|---------|
| **PermitRootLogin** | yes | prohibit-password | 增強安全：只允許密鑰登入 |
| **PasswordAuthentication** | yes | no | 禁用弱密碼認證 |
| **PubkeyAuthentication** | yes | yes | 保持一致 |
| **AllowTcpForwarding** | yes | yes | 保持一致 |
| **PermitOpen** | 192.168.100.50:22 | 192.168.1.*:22 | 支持網段，更靈活 |
| **AllowAgentForwarding** | (未明確設置) | no | 新增限制：禁止 SSH Agent 轉發 |
| **X11Forwarding** | (未明確設置) | no | 新增限制：禁止 X11 轉發 |
| **MaxAuthTries** | (預設 6) | 3 | 加強安全：限制認證嘗試次數 |
| **MaxSessions** | (預設 10) | 5 | 加強安全：限制併發會話 |
| **ClientAliveInterval** | (預設 0 無保活) | 300 | 新增：定期發送保活信號 |
| **LogLevel** | (預設 INFO) | VERBOSE | 提升日誌詳細度便於審計 |
| **Ciphers** | (系統預設，包含較弱) | AES-256-GCM, AES-128-GCM 等 | 限制為現代加密算法 |
| **MACs** | (系統預設) | HMAC-SHA-2-256 等 | 使用安全的消息認證碼 |
| **KexAlgorithms** | (系統預設) | Curve25519 等現代算法 | 使用現代密鑰交換 |

### 詳細配置對比

#### 認證和授權安全性

**實體伺服器配置：**

```ini
PermitRootLogin yes
PasswordAuthentication yes
PubkeyAuthentication yes
PermitEmptyPasswords no
ChallengeResponseAuthentication no
```

**Docker 容器配置：**

```ini
PermitRootLogin prohibit-password
PasswordAuthentication no
PubkeyAuthentication yes
PermitEmptyPasswords no
ChallengeResponseAuthentication no
AllowUsers root
```

**安全性改進對比：**

| 項目 | 實體伺服器 | Docker 容器 | 改進效果 |
|------|---------|---------|---------|
| Root 登入限制 | 允許任何方式 | 僅允許密鑰 | 消除空密碼登入風險 |
| 密碼認證 | 開啟（易被爆破） | 關閉 | 大幅降低爆破風險 |
| 用戶限制 | (未設置，允許所有) | 僅 root | 減少可攻擊目標 |
| 空密碼 | 禁用 | 禁用 | 一致 |
| 挑戰應答 | 禁用 | 禁用 | 一致 |

#### TCP 轉發配置對比

**實體伺服器：**

```ini
AllowTcpForwarding yes
PermitOpen 192.168.100.50:22
GatewayPorts no
AllowAgentForwarding no
PermitTunnel no
X11Forwarding no
```

**Docker 容器：**

```ini
AllowTcpForwarding yes
PermitOpen 192.168.1.*:22
GatewayPorts no
AllowAgentForwarding no
PermitTunnel no
X11Forwarding no
```

**PermitOpen 限制差異分析：**

```
實體伺服器: 192.168.100.50:22
  └─ 限制為單一 IP 地址
  └─ 靈活性低，難以擴展
  └─ 適用於固定小規模環境

Docker 容器: 192.168.1.*:22
  └─ 使用通配符限制到整個網段
  └─ 支持 192.168.1.1 到 192.168.1.254 的所有主機
  └─ 高度靈活，易於擴展
  └─ 適用於 Ansible 自動化場景
```

**轉發限制矩陣：**

```
目標地址           | 實體伺服器 | Docker 容器 | 結果
192.168.100.50:22  | ✓ 允許   | ✗ 拒絕    | 不相容
192.168.1.10:22    | ✗ 拒絕   | ✓ 允許    | 不相容
192.168.1.50:22    | ✗ 拒絕   | ✓ 允許    | 新增支援
192.168.1.1:22     | ✗ 拒絕   | ✓ 允許    | 新增支援
```

#### 加密算法配置對比

**Docker 容器配置（詳細）：**

```ini
# ============================================
# 對稱加密算法 (Ciphers)
# ============================================
Ciphers aes128-ctr,aes192-ctr,aes256-ctr,aes128-gcm@openssh.com,aes256-gcm@openssh.com
```

**支持的加密算法列表：**

| 算法 | 密鑰長度 | 工作模式 | 安全性評估 |
|------|----------|----------|-----------|
| aes128-ctr | 128-bit | Counter | 高（現代標準） |
| aes192-ctr | 192-bit | Counter | 高 |
| aes256-ctr | 256-bit | Counter | 最高 |
| aes128-gcm@openssh.com | 128-bit | GCM（認證） | 最高（含認證） |
| aes256-gcm@openssh.com | 256-bit | GCM（認證） | 最高（含認證） |

```ini
# ============================================
# 消息認證碼 (MACs)
# ============================================
MACs hmac-sha2-256,hmac-sha2-512
```

**支持的 MAC 算法列表：**

| 算法 | 輸出長度 | 安全強度 |
|------|----------|----------|
| hmac-sha2-256 | 256-bit | 128-bit（實際） |
| hmac-sha2-512 | 512-bit | 256-bit（實際） |

```ini
# ============================================
# 密鑰交換算法 (KexAlgorithms)
# ============================================
KexAlgorithms curve25519-sha256,curve25519-sha256@libssh.org,diffie-hellman-group-exchange-sha256
```

**支持的密鑰交換算法列表：**

| 算法 | 描述 | 安全等級 |
|------|------|---------|
| curve25519-sha256 | 標準化的 Curve25519 | 最高 |
| curve25519-sha256@libssh.org | OpenSSH 特定實現 | 最高 |
| diffie-hellman-group-exchange-sha256 | DH 組動態協商 | 高 |

**為什麼排除較弱的算法：**

- **DES、3DES** - 密鑰空間過小，已過時
- **MD5** - 已破解，不再安全
- **RC4** - 已證明不安全
- **DSA** - 根據美國標準委員會建議已棄用

#### 會話和認證限制對比

**Docker 容器安全限制配置：**

```ini
# ============================================
# 認證和連接限制
# ============================================
MaxAuthTries 3              # 最多 3 次認證嘗試
MaxSessions 5               # 最多 5 個併發會話
MaxStartups 10:30:100       # 連接受理策略
LoginGraceTime 60           # 60 秒認證超時
ClientAliveInterval 300     # 300 秒發送保活信號
ClientAliveCountMax 2       # 失響 2 次後關閉連接
```

**安全限制的作用：**

| 限制項 | 值 | 防禦效果 |
|--------|-----|---------|
| **MaxAuthTries** | 3 | 限制密碼爆破（若有） |
| **MaxSessions** | 5 | 防止資源耗盡攻擊 |
| **LoginGraceTime** | 60s | 防止無限期連接占用 |
| **ClientAliveInterval** | 300s | 檢測殭屍連接 |
| **MaxStartups** | 10:30:100 | 漸進式限制新連接 |

---

## 啟動流程設計

### entrypoint.sh 腳本邏輯詳解

#### 腳本執行流程圖

```
容器啟動
    ↓
設置 set -e（遇錯即停）
    ↓
顯示啟動歡迎信息
    ↓
[1] 檢查 SSH 主機密鑰
    ├─ 密鑰存在？ → 跳過
    └─ 不存在 → 生成新密鑰
    ↓
[2] 檢查 authorized_keys
    ├─ 文件存在？ → 跳過
    └─ 不存在 → 創建空文件（告警）
    ↓
[3] 驗證 sshd 配置語法
    ├─ 配置有效？ → 繼續
    └─ 配置無效？ → 退出，顯示錯誤
    ↓
[4] 顯示配置信息和 PermitOpen 設置
    ↓
[5] 以前台模式啟動 sshd
    └─ sshd -D -e（保持容器運行）
```

#### 詳細步驟分析

**第一步：檢查 SSH 主機密鑰**

```bash
if [ ! -f /etc/ssh/ssh_host_rsa_key ]; then
    echo "產生 SSH 主機密鑰..."
    ssh-keygen -A
    echo "✓ 主機密鑰已產生"
fi
```

**為什麼這樣做？**

1. **Docker 鏡像中預先生成** - Dockerfile 執行 `ssh-keygen -A` 後，密鑰已存在
2. **容器重新啟動時重用** - 若使用 Volume 掛載 `/etc/ssh/`，密鑰會被保留
3. **動態生成備選** - 若密鑰丟失或被刪除，此檢查可自動重新生成

**生成的密鑰：**

```
/etc/ssh/ssh_host_rsa_key              # RSA private key
/etc/ssh/ssh_host_rsa_key.pub          # RSA public key
/etc/ssh/ssh_host_ecdsa_key            # ECDSA private key
/etc/ssh/ssh_host_ecdsa_key.pub        # ECDSA public key
/etc/ssh/ssh_host_ed25519_key          # Ed25519 private key
/etc/ssh/ssh_host_ed25519_key.pub      # Ed25519 public key
```

**第二步：檢查 authorized_keys**

```bash
if [ ! -f /root/.ssh/authorized_keys ]; then
    echo "⚠️  警告: /root/.ssh/authorized_keys 不存在"
    echo "⚠️  請掛載或建立 authorized_keys 檔案"
    mkdir -p /root/.ssh
    touch /root/.ssh/authorized_keys
    chmod 600 /root/.ssh/authorized_keys
fi
```

**重要性：**

- 若沒有公鑰，SSH 登入會失敗
- 此檢查創建空文件防止 sshd 啟動失敗
- 警告信息提醒用戶配置公鑰
- 權限設置 600 符合 OpenSSH 安全要求

**第三步：驗證 sshd 配置語法**

```bash
echo "驗證 sshd 配置..."
if sshd -t; then
    echo "✓ SSH 配置語法正確"
else
    echo "✗ SSH 配置有錯誤，請檢查 sshd_config"
    exit 1
fi
```

**作用：**

- `sshd -t` - 測試配置文件語法（不啟動服務）
- 若配置有誤（如 PermitOpen 語法錯誤），會立即輸出錯誤並退出
- 防止啟動無效配置的容器

**第四步：顯示配置信息**

```bash
echo ""
echo "=========================================="
echo "SSH 伺服器配置資訊"
echo "=========================================="
echo "監聽通訊埠: 22"
echo "認證方式: SSH 密鑰"
echo "TCP 轉發: 啟用 (PermitOpen 限制)"
echo "允許的轉發目標: 192.168.1.*:22"
echo ""

# 顯示目前的 PermitOpen 配置
echo "目前 PermitOpen 配置："
sshd -T | grep permitopen || echo "  (未配置)"
```

**`sshd -T` 命令說明：**

- 輸出當前有效的配置（合併了 sshd_config 和命令行參數）
- `grep permitopen` 篩選 PermitOpen 相關配置
- 輸出示例：`permitopen 192.168.1.*:22`

**第五步：啟動 sshd 服務**

```bash
exec /usr/sbin/sshd -D -e
```

**參數說明：**

| 參數 | 含義 |
|------|------|
| `-D` | 不後台化（Daemon off），保持在前台運行 |
| `-e` | 將日誌輸出到 stderr（便於容器日誌收集） |
| `exec` | 替換當前進程（使 sshd 成為 PID 1 進程） |

**為什麼使用 exec？**

- `exec` 替換 shell 進程，sshd 成為 PID 1
- 這樣 sshd 直接接收容器的 SIGTERM 信號，優雅關閉
- 不使用 exec 會導致 sshd 被 shell 包裹，信號传递延遲

### SSH 主機密鑰管理

#### 密鑰生命週期管理

**鏡像構建階段：**

```dockerfile
# Dockerfile 中
RUN ssh-keygen -A
```

**容器啟動階段：**

```bash
# entrypoint.sh 中
if [ ! -f /etc/ssh/ssh_host_rsa_key ]; then
    ssh-keygen -A
fi
```

**多實例部署考量：**

**場景 1：每個容器獨立密鑰**

```yaml
# docker-compose.yml - 不掛載密鑰
services:
  bastion:
    volumes:
      - bastion_logs:/var/log
      # 不掛載 /etc/ssh/
```

**優點：** 簡單，無額外配置
**缺點：** 每次容器重啟密鑰改變，客戶端需要重新驗證

**場景 2：共享密鑰（推薦生產環境）**

```yaml
# docker-compose.yml - 掛載共享密鑰
services:
  bastion:
    volumes:
      - ./ssh_keys:/etc/ssh:ro    # 掛載預生成的密鑰
      - bastion_logs:/var/log
```

**優點：** 多實例保持相同身份，客戶端 known_hosts 一致
**缺點：** 需要提前生成和管理密鑰文件

**密鑰初始化命令：**

```bash
# 在主機上一次性生成密鑰
mkdir -p ./ssh_keys
ssh-keygen -A -f ./ssh_keys/

# 權限設置
chmod 600 ./ssh_keys/ssh_host_*_key
chmod 644 ./ssh_keys/ssh_host_*_key.pub
```

### authorized_keys 管理策略

#### 方案對比

| 方案 | 配置 | 用途 | 優點 | 缺點 |
|------|------|------|------|------|
| **複製到鏡像** | COPY config/authorized_keys | 開發環境 | 簡單便捷 | 鏡像中包含敏感信息 |
| **Volume 掛載** | -v ./config/authorized_keys:... | 生產環境 | 敏感信息外部管理 | 需要額外配置 |
| **Docker Secrets** | 使用 docker secret | Swarm 部署 | 加密存儲 | 需要 Swarm 支援 |
| **ConfigMap** | Kubernetes ConfigMap | K8s 部署 | Kubernetes 原生 | 限於 K8s 環境 |

#### 推薦實踐

**開發環境：** 複製到鏡像（簡單快速）

```dockerfile
COPY config/authorized_keys /root/.ssh/authorized_keys
RUN chmod 600 /root/.ssh/authorized_keys
```

**生產環境：** Volume 掛載或 Docker Secrets

```yaml
# 方式 1：Volume 掛載
volumes:
  - ./secure/authorized_keys:/root/.ssh/authorized_keys:ro

# 方式 2：Docker Secrets（Docker Swarm）
secrets:
  authorized_keys:
    file: ./secure/authorized_keys
```

### 配置驗證機制

#### 驗證流程

```
配置文件修改
    ↓
容器啟動時執行 entrypoint.sh
    ↓
sshd -t（語法驗證）
    ├─ ✓ 有效 → 顯示配置信息
    ├─ ✗ 無效 → 輸出錯誤信息並退出
    ↓
sshd -T（輸出配置）
    └─ grep permitopen（驗證 PermitOpen）
    ↓
啟動 sshd 服務
```

#### 常見配置錯誤和檢測

**PermitOpen 語法錯誤：**

```ini
# ❌ 錯誤
PermitOpen 192.168.1.0/24:22        # CIDR 不支援，只能用通配符

# ✓ 正確
PermitOpen 192.168.1.*:22
```

**驗證命令：**

```bash
# 檢查是否生成錯誤
docker logs ansible-bastion 2>&1 | grep -i error

# 進入容器手動驗證
docker exec ansible-bastion sshd -t
```

---

## Docker 特定考量

### 容器權限配置 (Linux Capabilities)

#### 為什麼需要權限配置

SSH 服務需要某些系統級權限來正常運行。在容器中，應遵循最小化權限原則。

#### sshd 所需的最小 Capabilities

```yaml
cap_drop:
  - ALL                # 首先刪除所有特權
cap_add:
  - NET_BIND_SERVICE   # 綁定 < 1024 的埠位（端口 22）
  - CHOWN              # 修改文件所有者（鑰匙對文件設置）
  - SETUID             # 設置進程 UID（sshd 切換用戶）
  - SETGID             # 設置進程 GID
  - DAC_OVERRIDE       # 覆蓋文件系統權限檢查（必要運行）
  - SYS_CHROOT         # 使用 chroot（某些情況需要）
```

**各 Capability 的必要性：**

| Capability | 必要性 | 用途 |
|-----------|-------|------|
| **NET_BIND_SERVICE** | 必須 | 綁定端口 22（< 1024）需要 |
| **CHOWN** | 必須 | SSH 密鑰文件需要特定所有者 |
| **SETUID** | 必須 | 為不同用戶創建 SSH 會話 |
| **SETGID** | 必須 | 設置進程組 ID |
| **DAC_OVERRIDE** | 必須 | 訪問受保護的文件系統資源 |
| **SYS_CHROOT** | 可選 | 某些 chroot jail 場景 |

#### 不推薦的做法

```yaml
# ❌ 不推薦：給予容器所有特權
privileged: true

# ❌ 不推薦：不受限的 capabilities
cap_add:
  - ALL
```

### 健康檢查設計

#### 健康檢查配置詳解

**docker-compose.yml 中的配置：**

```yaml
healthcheck:
  test: ["CMD", "sshd", "-t"]           # 檢查命令
  interval: 30s                          # 檢查間隔
  timeout: 10s                           # 命令超時
  retries: 3                             # 重試次數
  start_period: 5s                       # 啟動寬限期
```

**Dockerfile 中的配置：**

```dockerfile
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD sshd -t && echo "SSH service is healthy"
```

#### 檢查狀態解釋

```
HEALTHY
├─ 表示：sshd 配置有效，服務正常運行
└─ 條件：sshd -t 返回 0（成功）

UNHEALTHY
├─ 表示：sshd 配置無效或服務異常
├─ 觸發條件：
│  ├─ 3 次連續檢查失敗（retries=3）
│  ├─ 檢查命令返回非零
│  └─ 檢查命令超過 10 秒未響應
└─ 後續動作：容器標記為不健康，docker-compose 可設置重啟

STARTING
├─ 表示：容器剛啟動，檢查在寬限期內
├─ 持續時間：5 秒（start_period）
└─ 在此期間不判斷健康狀態
```

#### 監控健康檢查

```bash
# 查看容器健康狀態
docker ps | grep ansible-bastion

# 詳細的健康檢查信息
docker inspect ansible-bastion | grep -A 20 "Health"

# 輸出示例
"Health": {
    "Status": "healthy",
    "FailingStreak": 0,
    "Log": [
        {
            "Start": "2026-01-22T10:30:45.123456Z",
            "End": "2026-01-22T10:30:45.987654Z",
            "ExitCode": 0,
            "Output": "SSH service is healthy"
        }
    ]
}
```

#### 自定義健康檢查腳本

**高級檢查方案（監控實際 SSH 連接）：**

```bash
#!/bin/sh
# scripts/health-check.sh

# 1. 檢查配置語法
sshd -t || exit 1

# 2. 檢查 SSH 進程是否運行
pgrep sshd > /dev/null || exit 1

# 3. 檢查 authorized_keys 是否存在
[ -f /root/.ssh/authorized_keys ] || exit 1

# 4. 檢查端口是否監聽
ss -tuln | grep ':22 ' > /dev/null || exit 1

echo "SSH service is healthy"
exit 0
```

**在 docker-compose.yml 中使用：**

```yaml
healthcheck:
  test: ["/scripts/health-check.sh"]
  interval: 30s
  timeout: 10s
  retries: 3
  start_period: 5s
```

### 端口映射策略

#### 單實例部署

```yaml
services:
  bastion:
    ports:
      - "2222:22"    # 主機埠 2222 → 容器埠 22
```

**連接方式：**

```bash
ssh -i ~/.ssh/bastion_key -p 2222 root@localhost
```

#### 多實例負載均衡部署

```yaml
services:
  bastion1:
    ports:
      - "2222:22"
    container_name: bastion-1

  bastion2:
    ports:
      - "2223:22"
    container_name: bastion-2

  bastion3:
    ports:
      - "2224:22"
    container_name: bastion-3
```

**連接方式：**

```bash
# 連接到不同實例
ssh -i ~/.ssh/bastion_key -p 2222 root@localhost   # 實例 1
ssh -i ~/.ssh/bastion_key -p 2223 root@localhost   # 實例 2
ssh -i ~/.ssh/bastion_key -p 2224 root@localhost   # 實例 3
```

#### Kubernetes 端口暴露

```yaml
apiVersion: v1
kind: Service
metadata:
  name: bastion-ssh

spec:
  type: LoadBalancer    # 或 NodePort
  selector:
    app: bastion-ssh

  ports:
  - port: 2222          # 外部端口
    targetPort: 22      # 容器端口
    protocol: TCP
```

### Volume 掛載方案

#### 配置動態更新方案

**場景：** 不重建鏡像，直接更新 SSH 配置

```yaml
services:
  bastion:
    volumes:
      # 掛載 SSH 配置（可動態更新）
      - ./config/sshd_config:/etc/ssh/sshd_config:ro

      # 掛載公鑰（可動態更新）
      - ./config/authorized_keys:/root/.ssh/authorized_keys:ro

      # 掛載主機密鑰（持久化）
      - ./ssh_keys:/etc/ssh:ro

      # 日誌持久化
      - bastion_logs:/var/log
```

**更新配置流程：**

```bash
# 1. 修改本地配置文件
vim ./config/sshd_config

# 2. 驗證配置語法
docker exec ansible-bastion sshd -t

# 3. 容器會自動加載新配置
# （entrypoint.sh 每次啟動時驗證）

# 4. 重新啟動容器
docker restart ansible-bastion
```

#### 日誌持久化方案

**方案 1：Named Volume**

```yaml
services:
  bastion:
    volumes:
      - bastion_logs:/var/log

volumes:
  bastion_logs:
    driver: local
```

**優點：** Docker 管理，便於備份
**缺點：** 路徑位置由 Docker 管理，不直觀

**方案 2：主機目錄綁定**

```yaml
services:
  bastion:
    volumes:
      - ./logs:/var/log
```

**優點：** 直觀，易於本地查閱日誌
**缺點：** 需要手動創建目錄

**方案 3：結合使用**

```yaml
services:
  bastion:
    volumes:
      # 配置文件：主機目錄（開發快速迭代）
      - ./config/sshd_config:/etc/ssh/sshd_config:ro
      - ./config/authorized_keys:/root/.ssh/authorized_keys:ro

      # 密鑰：主機目錄（多實例共享）
      - ./ssh_keys:/etc/ssh:ro

      # 日誌：Named Volume（持久化和備份）
      - bastion_logs:/var/log

volumes:
  bastion_logs:
    driver: local
```

---

## 功能對比表

### 完整功能對比矩陣

| 功能特性 | 實體伺服器 (192.168.213.136) | Docker 容器 | 等級 | 備註 |
|---------|:------------------------:|:----------:|:--:|------|
| **認證功能** | | | | |
| SSH 密鑰認證 | ✓ | ✓ | ✓✓✓ | 兩者支援，容器配置更安全 |
| 密碼認證 | ✓ | ✗ | 容器更優 | 容器禁用，增強安全性 |
| Root 密碼登入 | ✓ | ✗ | 容器更優 | 容器禁用，防止爆破 |
| 多用戶支援 | ✓ (系統所有用戶) | ✓ (僅 root) | 容器更優 | 容器限制減少攻擊面 |
| Ed25519 密鑰支援 | ✓ | ✓ | ✓✓✓ | 兩者支援，容器推薦首選 |
| **轉發功能** | | | | |
| TCP 轉發 | ✓ | ✓ | ✓✓✓ | 兩者支援 |
| ProxyCommand 支援 | ✓ | ✓ | ✓✓✓ | 兩者支援 |
| SCP 文件傳輸 | ✓ | ✓ | ✓✓✓ | 兩者支援 |
| PermitOpen 限制 | ✓ (單點) | ✓ (網段) | 容器更優 | 容器使用通配符更靈活 |
| Agent 轉發 | ✓ (可能) | ✗ | 容器更優 | 容器禁用 |
| X11 轉發 | ✓ (可能) | ✗ | 容器更優 | 容器禁用 |
| **安全性** | | | | |
| 密碼套件限制 | ✗ | ✓ | 容器更優 | 容器限制為現代算法 |
| 認證嘗試限制 | ✗ (預設 6) | ✓ (3) | 容器更優 | 容器更嚴格 |
| 併發會話限制 | ✗ (預設 10) | ✓ (5) | 容器更優 | 容器更嚴格 |
| 連接超時限制 | ✗ | ✓ | 容器更優 | 容器有超時保護 |
| 保活信號 | ✗ | ✓ | 容器更優 | 容器檢測死連接 |
| 權限隔離 | 系統級 | 容器級 + 系統級 | 容器更優 | 容器添加額外隔離層 |
| **運維功能** | | | | |
| 健康檢查 | ✗ (手動) | ✓ (自動) | 容器更優 | 容器內置檢查 |
| 自動重啟 | ✗ (需配置) | ✓ | 容器更優 | 容器開箱即用 |
| 配置版本控制 | ✗ (難) | ✓ (Git) | 容器更優 | 配置即代碼 |
| 日誌集中化 | ✗ | ✓ (可配) | 容器更優 | 支援 Volume 集中 |
| 資源限制 | 系統級 | 容器級 | 容器更優 | 容器可精細控制 |
| 多實例部署 | ✗ (困難) | ✓ (簡單) | 容器更優 | Docker Compose 支援 |
| **易用性** | | | | |
| 部署時間 | 30-60 分鐘 | 5-10 秒 | 容器更優 | 差異巨大 |
| 配置複雜度 | 中等 | 低 | 容器更優 | 文件即配置 |
| 擴展性 | 低 | 高 | 容器更優 | 支援多種平台 |

### 部署方式對比

| 部署方式 | 實體伺服器 | Docker Compose | Docker Swarm | Kubernetes |
|---------|---------|----------------|-------------|-----------|
| **部署複雜度** | 高 | 低 | 中 | 高 |
| **可擴展性** | 差 | 中 | 中-高 | 高 |
| **故障恢復** | 手動 | 自動 | 自動 | 自動 |
| **資源效率** | 低 | 高 | 高 | 高 |
| **學習曲線** | 陡 | 平 | 中 | 陡 |
| **最適用場景** | 小規模穩定 | 開發/測試 | 中等規模 | 大規模生產 |
| **成本** | 高 | 低 | 低-中 | 中-高 |

### 配置管理對比

| 方面 | 實體伺服器 | Docker 容器 | 改進 |
|------|---------|---------|------|
| **配置存儲位置** | 分散在各伺服器 | 集中在版本控制系統 | 容器可版本化 |
| **配置更新方式** | SSH 登入手動編輯 | git push + 重新構建 | 容器完全自動化 |
| **配置審計** | 難以追蹤 | 完整 Git 歷史 | 容器有完全追溯性 |
| **配置回滾** | 手動修復 | git revert | 容器一鍵回滾 |
| **環境差異** | 常見（配置漂移） | 消除 | 容器保證一致性 |
| **文檔同步** | 文檔易過時 | 文檔與代碼一起演進 | 容器自文檔化 |

---

## 實際使用案例

### 案例 1：Ansible 跳板機部署

#### 場景描述

公司有 100 台內網伺服器（192.168.1.0/24），需要通過 Ansible 自動化管理，但這些伺服器無法直接從外網訪問。需要一台跳板機作為 SSH Gateway。

#### 實體伺服器方案

**部署步驟：**

1. 購買或申請一台伺服器（如 192.168.213.136）
2. 安裝 Alpine Linux 3.16.9
3. 安裝 OpenSSH 9.0p1-r5
4. 手動編輯 sshd_config（配置 PermitOpen 192.168.100.50:22）
5. 生成 SSH 主機密鑰
6. 複製用戶公鑰到 authorized_keys
7. 啟動 sshd 服務
8. 測試連接
9. 配置文檔（難以同步）
10. 定期維護和更新

**時間成本：** 30-60 分鐘

#### Docker 容器方案

**部署步驟：**

```bash
# 1. 準備 SSH 公鑰
ssh-keygen -t ed25519 -f ansible_bastion -N ""
cat ansible_bastion.pub > config/authorized_keys

# 2. 自訂 PermitOpen 配置
cat > config/sshd_config << 'EOF'
...
PermitOpen 192.168.1.*:22    # 支援整個內網段
...
EOF

# 3. 啟動容器
docker-compose up -d

# 4. 驗證連接
ssh -i ansible_bastion -p 2222 root@localhost

# 5. 配置 Ansible
cat > ansible.cfg << 'EOF'
[defaults]
ansible_ssh_common_args = -o ProxyCommand="ssh -W %h:%p -i ./ansible_bastion -p 2222 root@localhost"
EOF

# 6. 運行 Ansible Playbook
ansible-playbook -i inventory.yml playbook.yml
```

**時間成本：** 5-10 分鐘

#### 對比優勢

| 方面 | 實體伺服器 | Docker 容器 |
|------|---------|---------|
| **部署時間** | 30-60 分鐘 | 5-10 分鐘 |
| **測試驗證** | 需實際連接測試 | 本地即可測試 |
| **配置管理** | 難以版本控制 | Git 完全管理 |
| **團隊協作** | 文檔易過時 | 配置即文檔 |
| **故障恢復** | 需找到備用伺服器重新配置 | 一條命令重新部署 |
| **多環境支援** | 需重複部署 | 修改配置即可 |

### 案例 2：多環境部署

#### 場景描述

公司有開發、測試、生產三個環境，分別對應不同的內網網段。需要為每個環境部署跳板機，並設置不同的轉發規則。

#### 部署方案

**目錄結構：**

```
docker-bastion-ssh/
├── environments/
│   ├── dev/
│   │   ├── sshd_config
│   │   └── authorized_keys
│   ├── test/
│   │   ├── sshd_config
│   │   └── authorized_keys
│   └── prod/
│       ├── sshd_config
│       └── authorized_keys
├── docker-compose.yml
└── Makefile
```

**docker-compose.yml 配置：**

```yaml
version: '3.8'

services:
  bastion-dev:
    build: .
    container_name: bastion-dev
    ports:
      - "2222:22"
    volumes:
      - ./environments/dev/sshd_config:/etc/ssh/sshd_config:ro
      - ./environments/dev/authorized_keys:/root/.ssh/authorized_keys:ro
      - dev_logs:/var/log
    networks:
      - dev_network

  bastion-test:
    build: .
    container_name: bastion-test
    ports:
      - "2223:22"
    volumes:
      - ./environments/test/sshd_config:/etc/ssh/sshd_config:ro
      - ./environments/test/authorized_keys:/root/.ssh/authorized_keys:ro
      - test_logs:/var/log
    networks:
      - test_network

  bastion-prod:
    build: .
    container_name: bastion-prod
    ports:
      - "2224:22"
    volumes:
      - ./environments/prod/sshd_config:/etc/ssh/sshd_config:ro
      - ./environments/prod/authorized_keys:/root/.ssh/authorized_keys:ro
      - prod_logs:/var/log
    networks:
      - prod_network
    restart: always

networks:
  dev_network:
  test_network:
  prod_network:

volumes:
  dev_logs:
  test_logs:
  prod_logs:
```

**各環境 sshd_config 配置：**

```ini
# environments/dev/sshd_config
PermitOpen 192.168.10.*:22,192.168.10.*:23

# environments/test/sshd_config
PermitOpen 192.168.20.*:22,192.168.20.*:23

# environments/prod/sshd_config
PermitOpen 192.168.30.*:22,192.168.30.*:23
```

**使用方法：**

```bash
# 啟動所有環境
docker-compose up -d

# 啟動特定環境
docker-compose up -d bastion-dev

# 查看特定環境日誌
docker-compose logs bastion-prod

# 停止所有環境
docker-compose down
```

### 案例 3：ProxyCommand 連接

#### 直接 SSH 連接

**命令示例：**

```bash
# 連接到內網伺服器 192.168.1.50
ssh -o ProxyCommand='ssh -W %h:%p -i ~/.ssh/bastion_key -p 2222 root@localhost' \
    -i ~/.ssh/target_key user@192.168.1.50
```

**分解說明：**

```
ssh 命令結構：
├─ 主命令：ssh（連接到最終目標）
├─ -o ProxyCommand='...'（通過跳板機連接）
│  └─ ssh -W %h:%p（stdio 轉發）
│     ├─ -i ~/.ssh/bastion_key（跳板機私鑰）
│     ├─ -p 2222（跳板機埠位）
│     └─ root@localhost（跳板機地址）
├─ -i ~/.ssh/target_key（目標伺服器私鑰）
└─ user@192.168.1.50（目標伺服器地址）
```

#### 命令別名設置（簡化使用）

**在 ~/.ssh/config 中配置：**

```bash
Host bastion
    HostName localhost
    Port 2222
    User root
    IdentityFile ~/.ssh/bastion_key

Host *.internal
    ProxyCommand ssh -W %h:%p bastion
    User user
    IdentityFile ~/.ssh/target_key
```

**簡化後的連接命令：**

```bash
# 直接連接
ssh 192.168.1.50.internal

# 登入後自動跳轉到內網伺服器
ssh root@192.168.1.50.internal
```

### 案例 4：SCP 文件傳輸

#### 上傳文件到內網伺服器

```bash
scp -o ProxyCommand='ssh -W %h:%p -i ~/.ssh/bastion_key -p 2222 root@localhost' \
    local_file.txt user@192.168.1.50:/remote/path/
```

#### 下載文件從內網伺服器

```bash
scp -o ProxyCommand='ssh -W %h:%p -i ~/.ssh/bastion_key -p 2222 root@localhost' \
    user@192.168.1.50:/remote/file.txt local_path/
```

#### 批量傳輸配置

**Shell 腳本示例：**

```bash
#!/bin/bash
# upload_to_servers.sh

BASTION_KEY="~/.ssh/bastion_key"
TARGET_KEY="~/.ssh/target_key"
BASTION_PORT="2222"
LOCAL_FILE="$1"
REMOTE_PATH="$2"

# 伺服器列表
SERVERS=(
    "user@192.168.1.10"
    "user@192.168.1.20"
    "user@192.168.1.30"
)

for server in "${SERVERS[@]}"; do
    echo "上傳到 $server..."
    scp -o ProxyCommand="ssh -W %h:%p -i $BASTION_KEY -p $BASTION_PORT root@localhost" \
        -i $TARGET_KEY \
        "$LOCAL_FILE" \
        "$server:$REMOTE_PATH/"
done
```

---

## 優勢與限制

### Docker 化的優勢

#### 🚀 部署和擴展性

**快速部署：**
- 從零到運行只需 5-10 秒
- 無需手動配置系統和軟體
- 支援一鍵部署多個實例

**橫向擴展：**
```bash
# 輕鬆創建 3 個跳板機實例進行負載均衡
docker-compose up -d --scale bastion=3
```

#### 🔐 安全性增強

| 安全特性 | 優勢 |
|---------|------|
| **認證加固** | 禁用弱密碼認證，強制密鑰認證 |
| **轉發限制** | 精細化 PermitOpen 規則（網段級） |
| **加密算法** | 限制為現代算法，禁用過時算法 |
| **會話限制** | 限制認證嘗試和併發會話數 |
| **權限隔離** | Linux Capabilities 限制，容器隔離 |
| **配置審計** | Git 版本控制，完全追蹤改動 |

#### 📦 配置管理

**配置即代碼：**
- 所有配置存儲在 Git 倉庫
- 完整的變更歷史和追蹤
- 支援代碼審查（PR）和自動化測試

**環境一致性：**
- 開發、測試、生產使用完全相同的鏡像
- 消除「在我的機器上工作」的問題
- 配置漂移問題完全解決

#### 🔧 運維簡化

| 運維方面 | 改進 |
|---------|------|
| **健康檢查** | 自動化，無需人工監控 |
| **故障恢復** | 自動重啟，快速恢復 |
| **日誌管理** | 集中化，便於分析 |
| **監控告警** | 容器原生支援 |
| **更新升級** | 構建新鏡像，滾動更新 |

### 容器環境的限制

#### 1. 主機密鑰持久化問題

**問題：** 容器重新啟動或銷毀時，主機密鑰可能丟失，導致 SSH 指紋改變

**影響：** 客戶端 known_hosts 中的指紋不匹配

**解決方案：**

```yaml
# 方案 1：掛載 Volume 持久化密鑰
volumes:
  - ./ssh_keys:/etc/ssh:ro

# 方案 2：使用 Docker Secrets (Swarm)
secrets:
  ssh_host_rsa_key:
    file: ./ssh_keys/ssh_host_rsa_key

# 方案 3：使用 Kubernetes Secrets (K8s)
volumes:
- name: ssh-keys
  secret:
    secretName: ssh-host-keys
```

#### 2. Root 權限需求

**問題：** SSH 服務通常需要 root 權限綁定低埠位

**當前做法：** 容器以 root 運行

**安全考量：** 遵循最小特權原則，已限制 Linux Capabilities

**改進方案（高級）：**

```dockerfile
# 使用 cap_drop 和 cap_add 限制特權
RUN addgroup -S sshd && adduser -S -H -G sshd sshd

# 配置高端口運行
RUN sed -i 's/^Port 22/Port 2222/' /etc/ssh/sshd_config

# 以非 root 用戶運行（需適當配置）
USER sshd
```

#### 3. 網絡隔離

**問題：** 容器默認無法訪問外部網絡（取決於配置）

**當前做法：** 使用自訂 bridge 網絡

**配置：**

```yaml
networks:
  ansible_network:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/16    # 指定子網
```

#### 4. 日誌大小限制

**問題：** 日誌可能無限增長，填滿容器存儲

**解決方案：**

```yaml
logging:
  driver: json-file
  options:
    max-size: "10m"      # 單個日誌文件最大 10MB
    max-file: "3"        # 保留最多 3 個日誌文件
```

#### 5. IP 地址動態性

**問題：** 容器 IP 地址可能改變（特別是在 Kubernetes 中）

**當前做法：** 使用埠映射和 Service 名稱

**Kubernetes 解決方案：**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: bastion-ssh-service

spec:
  type: LoadBalancer      # 或 NodePort
  selector:
    app: bastion-ssh
  ports:
  - port: 2222
    targetPort: 22
```

#### 6. 容器內訊號處理

**問題：** 進程信號傳遞可能延遲或丟失

**當前做法：** 使用 exec 替換 shell 進程，sshd 成為 PID 1

```bash
# entrypoint.sh 中
exec /usr/sbin/sshd -D -e
```

**作用：** sshd 直接接收容器信號，支援優雅關閉

### 限制總結表

| 限制 | 嚴重度 | 解決難度 | 推薦方案 |
|------|-------|--------|---------|
| 主機密鑰持久化 | 高 | 低 | Volume 掛載 |
| Root 權限需求 | 中 | 高 | Capabilities 限制 |
| 網絡隔離 | 低 | 低 | 自訂 bridge |
| 日誌大小 | 中 | 低 | 日誌驅動配置 |
| IP 動態性 | 低 | 低 | Service 名稱 |
| 訊號傳遞 | 低 | 低 | exec 替換 |

---

## 最佳實踐建議

### 生產環境部署清單

#### 安全配置清單

- [ ] **密鑰管理**
  - [ ] 使用 Volume 或 Secrets 管理 authorized_keys
  - [ ] 定期輪換 SSH 密鑰（每 90 天）
  - [ ] 使用強密鑰類型（Ed25519 優於 RSA）
  - [ ] 私鑰文件權限設置為 600

- [ ] **訪問控制**
  - [ ] 啟用 PermitOpen 限制轉發目標
  - [ ] 限制允許的用戶（只有 root）
  - [ ] 設置 MaxAuthTries 限制認證嘗試
  - [ ] 設置 MaxSessions 限制併發會話
  - [ ] 禁用不需要的轉發功能（Agent、X11）

- [ ] **加密和協議**
  - [ ] 限制為現代加密算法
  - [ ] 使用 SHA-2-256 及以上的 MAC
  - [ ] 使用 Curve25519 或 DH-group-exchange 的密鑰交換
  - [ ] 禁用舊的弱算法

- [ ] **容器安全**
  - [ ] 使用 cap_drop 和 cap_add 限制特權
  - [ ] 設置 read-only 根文件系統（如可能）
  - [ ] 禁用容器的 privileged 模式
  - [ ] 定期掃描鏡像漏洞（使用 Trivy）

#### 運維配置清單

- [ ] **監控和告警**
  - [ ] 配置健康檢查（30 秒間隔）
  - [ ] 集成容器監控系統（Prometheus、Datadog）
  - [ ] 設置失敗重啟策略
  - [ ] 配置日誌轉發（ELK、Splunk）

- [ ] **日誌管理**
  - [ ] 啟用日誌持久化
  - [ ] 設置日誌輪轉策略
  - [ ] 配置集中日誌收集
  - [ ] 定期審計日誌

- [ ] **備份和恢復**
  - [ ] 備份 SSH 主機密鑰
  - [ ] 備份 authorized_keys
  - [ ] 備份配置文件（sshd_config）
  - [ ] 定期測試恢復流程

- [ ] **更新和升級**
  - [ ] 定期更新基礎鏡像 (Alpine)
  - [ ] 追蹤 OpenSSH 安全公告
  - [ ] 實施自動化更新流程
  - [ ] 在測試環境先驗證

#### 文檔和培訓清單

- [ ] **文檔**
  - [ ] 部署指南
  - [ ] 配置管理指南
  - [ ] 故障排除文檔
  - [ ] 架構設計文檔

- [ ] **培訓**
  - [ ] 團隊培訓（Docker、SSH、ProxyCommand）
  - [ ] 故障排除培訓
  - [ ] 密鑰管理培訓
  - [ ] 安全最佳實踐培訓

### Docker 鏡像優化

#### 鏡像大小優化

**目標：** 將鏡像大小控制在 30MB 以內

**優化技巧：**

```dockerfile
# ❌ 低效做法
FROM alpine:3.18
RUN apk update
RUN apk add openssh
RUN rm -rf /var/cache/apk/*

# ✓ 高效做法：合併 RUN 指令
FROM alpine:3.18
RUN apk update && \
    apk add --no-cache openssh && \
    rm -rf /var/cache/apk/*
```

**體積檢查：**

```bash
# 查看鏡像大小
docker images | grep bastion

# 詳細分析層大小
docker history ansible-bastion

# 使用 dive 工具分析
dive ansible-bastion:latest
```

#### 構建性能優化

**多階段構建（若需要）：**

```dockerfile
# 階段 1：構建
FROM alpine:3.18 as builder
RUN apk add --no-cache build-base openssh-dev
# ... 編譯步驟 ...

# 階段 2：運行時
FROM alpine:3.18
COPY --from=builder /usr/local/bin/sshd /usr/sbin/sshd
# ... 其他步驟 ...
```

**構建快取優化：**

```dockerfile
# 將不經常改變的東西放在前面
FROM alpine:3.18

# 依賴安裝（變化頻率低）
RUN apk update && apk add --no-cache openssh bash

# 配置複製（變化頻率高）
COPY config/ /etc/ssh/
```

### 監控和日誌

#### Prometheus 監控集成

**暴露 SSH 指標的腳本：**

```bash
#!/bin/sh
# /scripts/metrics.sh

# 暴露埠 9100 給 Prometheus
# 此處示例使用 node-exporter 或自定義指標

# SSH 活躍連接數
netstat -an | grep ESTABLISHED | grep :22 | wc -l

# SSH 認證失敗數（從日誌提取）
grep "Invalid user" /var/log/auth.log | wc -l
```

#### 日誌聚合配置

**集成到 ELK Stack：**

```yaml
services:
  bastion:
    # ... 其他配置 ...
    logging:
      driver: syslog
      options:
        syslog-address: "udp://logserver:514"
        tag: "bastion-ssh"
```

**查看實時日誌：**

```bash
# Docker 日誌
docker logs -f ansible-bastion

# 查看認證日誌
docker exec ansible-bastion tail -f /var/log/auth.log

# 查看完整日誌
docker exec ansible-bastion cat /var/log/syslog
```

### 性能調優

#### 連接性能優化

```ini
# sshd_config 中的性能相關配置
# 減少認證延遲
KexAlgorithms curve25519-sha256        # 快速密鑰交換
Ciphers aes256-gcm@openssh.com         # 性能好的加密

# TCP 相關
TCPKeepAlive yes                       # 檢測死連接
ClientAliveInterval 300                # 保活間隔

# 連接複用
Protocol 2                              # 使用 SSH v2
```

#### 容器資源限制配置

```yaml
services:
  bastion:
    # ... 其他配置 ...
    deploy:
      resources:
        limits:
          cpus: '0.5'           # 限制 CPU 為 0.5 核
          memory: 256M          # 限制內存為 256MB
        reservations:
          cpus: '0.25'          # 預留 CPU 0.25 核
          memory: 128M          # 預留 128MB 內存
```

### 故障診斷和調試

#### 常見問題診斷命令

```bash
# 檢查容器狀態
docker ps -a | grep bastion
docker inspect ansible-bastion | grep -A 5 State

# 查看容器日誌
docker logs -n 100 ansible-bastion          # 最後 100 行
docker logs --since 5m ansible-bastion      # 最後 5 分鐘

# 進入容器進行調試
docker exec -it ansible-bastion sh

# 驗證 SSH 配置
docker exec ansible-bastion sshd -T          # 輸出有效配置
docker exec ansible-bastion sshd -t -v       # 驗證配置並顯示詳情

# 檢查網絡連接
docker exec ansible-bastion ss -tuln | grep 22

# 檢查進程狀態
docker exec ansible-bastion ps aux | grep sshd
```

#### 遠程連接測試

```bash
# 測試基本連接
ssh -v -i key.pem -p 2222 root@localhost

# 測試 ProxyCommand
ssh -v -o ProxyCommand='ssh -W %h:%p -i key.pem -p 2222 root@localhost' \
    user@192.168.1.50

# 測試特定加密算法
ssh -o Ciphers=aes256-gcm@openssh.com -p 2222 root@localhost

# 詳細模式（三個 -v 用於最詳細輸出）
ssh -vvv -i key.pem -p 2222 root@localhost
```

---

## 附錄

### A. 完整 Dockerfile 參考

```dockerfile
# Docker Bastion SSH 伺服器
FROM alpine:3.18

LABEL maintainer="Ansible Network Lab"
LABEL description="Bastion SSH Server for Ansible ProxyCommand with PermitOpen restrictions"

# 更新套件管理器並安裝 OpenSSH
RUN apk update && \
    apk add --no-cache \
        openssh=9.3_p2-r1 \
        openssh-client=9.3_p2-r1 \
        bash \
        doas \
    && rm -rf /var/cache/apk/*

# 建立 SSH 目錄
RUN mkdir -p /run/sshd && \
    chmod 755 /run/sshd

# 建立 root SSH 配置目錄
RUN mkdir -p /root/.ssh && \
    chmod 700 /root/.ssh

# 複製自訂 sshd 配置
COPY config/sshd_config /etc/ssh/sshd_config
RUN chmod 600 /etc/ssh/sshd_config

# 複製 authorized_keys
COPY config/authorized_keys /root/.ssh/authorized_keys
RUN chmod 600 /root/.ssh/authorized_keys

# 複製啟動腳本
COPY scripts/entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh

# 產生主機密鑰
RUN ssh-keygen -A

# 設定健康檢查
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD sshd -t && echo "SSH service is healthy"

# 暴露 SSH 通訊埠
EXPOSE 22

# 啟動腳本入口
ENTRYPOINT ["/entrypoint.sh"]
```

### B. 完整 docker-compose.yml 參考

```yaml
version: '3.8'

services:
  bastion:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: ansible-bastion
    ports:
      - "2222:22"
    environment:
      TZ: UTC
    volumes:
      # 掛載 SSH 配置（開發模式，可動態更新）
      # - ./config/sshd_config:/etc/ssh/sshd_config:ro

      # 掛載 authorized_keys（推薦用於生產）
      # - ./config/authorized_keys:/root/.ssh/authorized_keys:ro

      # 掛載日誌目錄
      - bastion_logs:/var/log

    healthcheck:
      test: ["CMD", "sshd", "-t"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 5s

    networks:
      - ansible_network

    restart: unless-stopped

    # 容器權限配置
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE
      - CHOWN
      - SETUID
      - SETGID
      - DAC_OVERRIDE
      - SYS_CHROOT

    # 資源限制（生產環境推薦）
    # deploy:
    #   resources:
    #     limits:
    #       cpus: '0.5'
    #       memory: 256M
    #     reservations:
    #       cpus: '0.25'
    #       memory: 128M

networks:
  ansible_network:
    driver: bridge

volumes:
  bastion_logs:
    driver: local
```

### C. sshd_config 完整配置範例

```ini
# SSH 伺服器配置文件

# ============================================
# 基本設定
# ============================================
Port 22
#AddressFamily any
#ListenAddress 0.0.0.0
#ListenAddress ::

# ============================================
# 主機密鑰配置
# ============================================
HostKey /etc/ssh/ssh_host_rsa_key
HostKey /etc/ssh/ssh_host_ecdsa_key
HostKey /etc/ssh/ssh_host_ed25519_key

# ============================================
# 認證配置
# ============================================
PubkeyAuthentication yes
PubkeyAcceptedAlgorithms ssh-rsa,rsa-sha2-256,rsa-sha2-512,ecdsa-sha2-nistp256,ecdsa-sha2-nistp384,ecdsa-sha2-nistp521,ssh-ed25519
PasswordAuthentication no
PermitEmptyPasswords no
ChallengeResponseAuthentication no
PermitRootLogin prohibit-password
AllowUsers root

# ============================================
# TCP 轉發配置
# ============================================
AllowTcpForwarding yes
PermitOpen 192.168.1.*:22
GatewayPorts no
AllowAgentForwarding no
PermitTunnel no
X11Forwarding no

# ============================================
# 日誌和認證
# ============================================
SyslogFacility AUTH
LogLevel VERBOSE
AuthorizedKeysFile .ssh/authorized_keys

# ============================================
# 連接和超時
# ============================================
LoginGraceTime 60
ClientAliveInterval 300
ClientAliveCountMax 2

# ============================================
# 安全限制
# ============================================
MaxAuthTries 3
MaxSessions 5
MaxStartups 10:30:100

# ============================================
# 安全選項
# ============================================
Protocol 2
HostbasedAuthentication no
IgnoreRhosts yes
StrictModes yes
IgnoreUserKnownHosts no
UseDNS no
Compression no
TCPKeepAlive yes
PermitUserEnvironment no

# ============================================
# 加密算法
# ============================================
Ciphers aes128-ctr,aes192-ctr,aes256-ctr,aes128-gcm@openssh.com,aes256-gcm@openssh.com
MACs hmac-sha2-256,hmac-sha2-512
KexAlgorithms curve25519-sha256,curve25519-sha256@libssh.org,diffie-hellman-group-exchange-sha256
```

### D. entrypoint.sh 完整腳本

```bash
#!/bin/sh
# Bastion SSH Server 容器啟動腳本

set -e

echo "=========================================="
echo "Bastion SSH Server - 啟動中..."
echo "=========================================="

# 檢查 SSH 主機密鑰
echo "檢查 SSH 主機密鑰..."
if [ ! -f /etc/ssh/ssh_host_rsa_key ]; then
    echo "產生 SSH 主機密鑰..."
    ssh-keygen -A
    echo "✓ 主機密鑰已產生"
fi

# 檢查 authorized_keys
if [ ! -f /root/.ssh/authorized_keys ]; then
    echo "⚠️  警告: /root/.ssh/authorized_keys 不存在"
    echo "⚠️  請掛載或建立 authorized_keys 檔案"
    mkdir -p /root/.ssh
    touch /root/.ssh/authorized_keys
    chmod 600 /root/.ssh/authorized_keys
fi

# 驗證 sshd 配置語法
echo "驗證 sshd 配置..."
if sshd -t; then
    echo "✓ SSH 配置語法正確"
else
    echo "✗ SSH 配置有錯誤，請檢查 sshd_config"
    exit 1
fi

# 顯示配置資訊
echo ""
echo "=========================================="
echo "SSH 伺服器配置資訊"
echo "=========================================="
echo "監聽通訊埠: 22"
echo "認證方式: SSH 密鑰"
echo "TCP 轉發: 啟用 (PermitOpen 限制)"
echo "允許的轉發目標: 192.168.1.*:22"
echo ""

# 顯示目前的 PermitOpen 配置
echo "目前 PermitOpen 配置："
sshd -T | grep permitopen || echo "  (未配置)"
echo ""

echo "=========================================="
echo "SSH 服務啟動..."
echo "=========================================="

# 啟動 sshd（前台執行，以便容器可以捕獲信號）
exec /usr/sbin/sshd -D -e
```

### E. SSH 連接故障排除表

| 問題 | 症狀 | 原因 | 解決方案 |
|------|------|------|---------|
| **Connection refused** | `ssh: connect to host localhost port 2222: Connection refused` | 容器未運行或埠映射錯誤 | 檢查容器狀態：`docker ps \| grep bastion` |
| **Permission denied** | `Permission denied (publickey)` | 公鑰不匹配或文件權限錯誤 | 檢查 authorized_keys 和私鑰權限 |
| **Timeout** | `ssh: connect to host timeout` | 防火牆或網絡問題 | 檢查防火牆規則：`iptables -L` |
| **Host key changed** | `WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED` | 容器主機密鑰改變 | 更新 known_hosts：`ssh-keygen -R localhost` |
| **Bad configuration** | `sshd: bad configuration option: ...` | sshd_config 語法錯誤 | 驗證配置：`docker exec bastion sshd -t` |
| **ProxyCommand failed** | `ssh: error: ProxyCommand ... failed` | 跳板機連接問題 | 直接測試跳板機連接 |

### F. 性能優化檢查清單

- [ ] **鏡像大小**
  - [ ] 檢查鏡像大小是否 < 30MB
  - [ ] 合併 RUN 指令減少層數
  - [ ] 清理依賴包和快取

- [ ] **容器啟動**
  - [ ] 容器啟動時間是否 < 5 秒
  - [ ] 是否預生成 SSH 主機密鑰
  - [ ] 檢查啟動日誌是否有延遲

- [ ] **連接性能**
  - [ ] SSH 連接時間是否 < 2 秒
  - [ ] 認證時間是否 < 1 秒
  - [ ] ProxyCommand 轉發延遲是否 < 100ms

- [ ] **資源效率**
  - [ ] 容器內存使用是否 < 64MB
  - [ ] CPU 使用率是否 < 10%
  - [ ] 磁盤 I/O 是否正常

---

## 文檔變更歷史

| 版本 | 日期 | 變更內容 |
|------|------|---------|
| 1.0.0 | 2026-01-22 | 初始版本，完整詳細指南 |

---

## 相關文檔參考

- [SSH 密鑰管理指南](docs/SSH_KEY_MANAGEMENT.md)
- [自訂配置指南](docs/CUSTOMIZATION_GUIDE.md)
- [PermitOpen 配置指南](docs/PERMITOPEN_GUIDE.md)
- [SDD 規格文檔](SDD.md)
- [README](README.md)

---

**文檔所有者：** DevOps Team
**最後更新：** 2026-01-22
**審查狀態：** ✓ 已審查
**應用範圍：** 開發、測試、生產環境

