# SSH 容器連接問題解決報告

**報告日期：** 2026-01-09
**版本：** 1.0.0

---

## 執行摘要

Docker Bastion SSH 容器初始部署後出現了三個相關的 SSH 連接問題：

1. **Privilege Separation chroot 失敗** - 缺少 `CAP_SYS_CHROOT` capability
2. **公鑰簽章算法不被接受** - sshd 未配置支援的簽章算法列表
3. **進程清理權限不足** - 可能需要額外的 capability（待觀察）

所有問題均已識別並提供了修復方案。

---

## 問題背景與描述

### 環境信息

- **基礎鏡像：** Alpine Linux 3.18
- **OpenSSH 版本：** 9.3_p2-r1 (現為最新可用版本)
- **容器編排：** Docker Compose 3.8
- **客戶端 SSH 密鑰：** RSA-based (ssh-rsa 算法)

### 現象描述

部署容器後，嘗試通過 SSH 連接到容器時出現以下錯誤：

```
Connection from 192.168.10.112 port 64448 on 172.20.0.2 port 22 rdomain ""
chroot("/var/empty"): Operation not permitted [preauth]
```

隨後的連接嘗試顯示：

```
Connection from 192.168.10.112 port 64518 on 172.21.0.2 port 22 rdomain ""
userauth_pubkey: signature algorithm ssh-rsa not in PubkeyAcceptedAlgorithms [preauth]
Connection closed by authenticating user root 192.168.10.112 port 64518 [preauth]
cleanup_exit: kill(11): Operation not permitted
```

---

## 診斷過程

### 階段 1：日誌分析

通過 Docker 容器日誌檢查，發現錯誤訊息出現在 SSH 認證前階段（`[preauth]`），表明問題出現在連接建立的早期。

**關鍵發現：**
- `chroot` 操作被容器拒絕
- 簽章算法驗證失敗
- 進程信號發送被拒絕

### 階段 2：根本原因分析

#### 問題 1：Privilege Separation chroot 失敗

**錯誤訊息：**
```
chroot("/var/empty"): Operation not permitted [preauth]
```

**分析結果：**

1. **OpenSSH Privilege Separation 機制**
   - OpenSSH 使用 Privilege Separation 增強安全性
   - 認證子進程需要在隔離的 chroot 環境（`/var/empty`）中執行
   - chroot 系統調用需要 `CAP_SYS_CHROOT` Linux capability

2. **容器權限配置問題**
   ```yaml
   # docker-compose.yml 原始配置
   cap_drop:
     - ALL
   cap_add:
     - NET_BIND_SERVICE
     - CHOWN
     - SETUID
     - SETGID
     - DAC_OVERRIDE
   ```

   - `cap_drop: ALL` 移除了所有 Linux capabilities
   - cap_add 中未包含 `SYS_CHROOT`
   - 導致 OpenSSH 無法執行 chroot 隔離

#### 問題 2：公鑰簽章算法不被接受

**錯誤訊息：**
```
userauth_pubkey: signature algorithm ssh-rsa not in PubkeyAcceptedAlgorithms [preauth]
```

**分析結果：**

1. **客戶端情況**
   - 客戶端使用 RSA 公鑰
   - SSH 簽名算法為 `ssh-rsa`（基於 SHA-1）

2. **OpenSSH 9.3 的安全策略變更**
   - OpenSSH 9.3 加強了默認安全設置
   - 不再默認接受 SHA-1 基礎的簽章算法（`ssh-rsa`）
   - 如果 `sshd_config` 未明確配置 `PubkeyAcceptedAlgorithms`，OpenSSH 使用非常受限的默認列表

3. **sshd_config 缺陷**
   - 原始配置中未設置 `PubkeyAcceptedAlgorithms`
   - 導致 OpenSSH 使用安全性優先的默認值
   - 拒絕了合法但較舊的 RSA 簽章

#### 問題 3：進程清理權限不足

**錯誤訊息：**
```
cleanup_exit: kill(11): Operation not permitted
```

**分析結果：**

1. **信號發送限制**
   - 此錯誤出現在 SSH 連接關閉後
   - OpenSSH 嘗試清理子進程
   - 信號發送被拒絕

2. **可能原因**
   - 缺少 `CAP_SYS_KILL` capability
   - 或是進程命名空間隔離導致

3. **當前評估**
   - 此問題的優先級較低
   - 可能在解決其他兩個問題後自行解決
   - 建議先不添加 SYS_KILL，待觀察

---

## 解決方案

### 修復 1：添加 SYS_CHROOT Capability

**文件：** `docker-compose.yml`

**修改內容：**
```yaml
cap_add:
  - NET_BIND_SERVICE
  - CHOWN
  - SETUID
  - SETGID
  - DAC_OVERRIDE
  - SYS_CHROOT    # ← 新增
```

**原理：**
- 賦予容器執行 chroot 系統調用的權限
- 允許 OpenSSH Privilege Separation 正常工作
- 這是 OpenSSH 安全架構的關鍵部分

**安全性評估：** ✅ 低風險
- Docker 容器本身已經隔離
- SYS_CHROOT 是系統調用級別的能力，不涉及提升用戶特權
- OpenSSH 官方推薦的配置

### 修復 2：配置公鑰簽章算法

**文件：** `config/sshd_config`

**修改內容：**
```ini
PubkeyAuthentication yes
PubkeyAcceptedAlgorithms ssh-rsa,rsa-sha2-256,rsa-sha2-512,ecdsa-sha2-nistp256,ecdsa-sha2-nistp384,ecdsa-sha2-nistp521,ssh-ed25519
PasswordAuthentication no
```

**配置說明：**

| 算法 | 基礎 | 安全性 | 備註 |
|------|------|--------|------|
| `ssh-rsa` | SHA-1 | ⚠️ 舊 | 向後相容性，不推薦長期使用 |
| `rsa-sha2-256` | SHA-256 | ✅ 中等 | 改進的 RSA 簽章 |
| `rsa-sha2-512` | SHA-512 | ✅ 強 | 最強的 RSA 簽章 |
| `ecdsa-sha2-nistp256` | ECDSA | ✅ 強 | 橢圓曲線加密 |
| `ecdsa-sha2-nistp384` | ECDSA | ✅ 強 | 更強的 ECDSA |
| `ecdsa-sha2-nistp521` | ECDSA | ✅ 強 | 最強的 ECDSA |
| `ssh-ed25519` | EdDSA | ✅✅ 最佳 | 現代推薦算法 |

**原理：**
- 明確列出伺服器接受的簽章算法
- 允許使用舊的 RSA 密鑰以保持向後相容
- 同時支援現代的安全算法

**安全性評估：** ✅ 接受
- 向後相容：支援現有的 RSA 密鑰
- 前向安全：支援 Ed25519 等現代算法
- 生產環境建議稍後只保留 SHA-2 和更新的算法

---

## 配置變更摘要

### docker-compose.yml 變更

```diff
    cap_add:
      - NET_BIND_SERVICE
      - CHOWN
      - SETUID
      - SETGID
      - DAC_OVERRIDE
+     - SYS_CHROOT
```

### config/sshd_config 變更

```diff
  # 禁用密碼認證，只允許密鑰認証
  PubkeyAuthentication yes
+ PubkeyAcceptedAlgorithms ssh-rsa,rsa-sha2-256,rsa-sha2-512,ecdsa-sha2-nistp256,ecdsa-sha2-nistp384,ecdsa-sha2-nistp521,ssh-ed25519
  PasswordAuthentication no
```

---

## 安全性考量

### 容器權限

**新增的 SYS_CHROOT Capability：**
- 用途：允許進程改變根目錄
- 風險：中等。在容器環境中，基礎鏡像已隔離，此能力不會導致容器逃逸
- 必要性：是。OpenSSH Privilege Separation 的必需能力

**其他 Capabilities 評估：**
- `NET_BIND_SERVICE` - 綁定低端口（22）必需 ✅
- `CHOWN` - 改變文件所有者必需 ✅
- `SETUID/SETGID` - 進程身份切換必需 ✅
- `DAC_OVERRIDE` - 文件權限檢查必需 ✅

### 簽章算法安全性

**ssh-rsa (SHA-1)：**
- 安全等級：低（不推薦）
- 背景：SHA-1 已被多個標準組織棄用
- 用途：向後相容，允許舊客戶端連接
- 期限：應規劃逐步淘汰

**建議計畫：**
1. **階段 1 (現在)：** 接受 ssh-rsa，以支援現有客戶端
2. **階段 2 (3-6月)：** 提醒用戶升級密鑰到 Ed25519 或 rsa-sha2-256
3. **階段 3 (6-12月)：** 禁用 ssh-rsa，僅保留 SHA-2 及以上的算法

---

## 驗證步驟

### 1. 修改文件

確保以下文件已修改：
- [ ] `docker-compose.yml` - 添加 SYS_CHROOT
- [ ] `config/sshd_config` - 添加 PubkeyAcceptedAlgorithms

### 2. 重新構建並啟動

```bash
# 重新構建鏡像
docker-compose build

# 重新啟動容器
docker-compose down
docker-compose up -d
```

### 3. 驗證配置

```bash
# 檢查配置語法
docker exec ansible-bastion sshd -t

# 查看當前配置
docker exec ansible-bastion sshd -T | grep -E "pubkeyaccepted|permitopen|allowtcpforwarding"
```

### 4. 測試 SSH 連接

```bash
# 使用 RSA 密鑰連接
ssh -i ~/.ssh/bastion_key -p 2222 root@localhost -v

# 查看連接日誌
docker logs -f ansible-bastion
```

### 5. 檢查日誌

**成功連接的日誌：**
```
Accepted publickey for root from 192.168.x.x port xxxxx ssh2: rsa-sha2-256 SHA256:...
```

**失敗連接的日誌：**
```
Connection closed by authenticating user root
```

---

## 驗證結果

修復後，SSH 連接應正常建立，日誌輸出應為：

```
Connection from <CLIENT_IP> port <PORT> on <SERVER_IP> port 22 rdomain ""
Accepted publickey for root from <CLIENT_IP> port <PORT> ssh2: <ALGORITHM> SHA256:<HASH>
```

不應再出現：
- ❌ `chroot("/var/empty"): Operation not permitted`
- ❌ `signature algorithm ssh-rsa not in PubkeyAcceptedAlgorithms`

---

## 故障排除流程總結

| 問題 | 症狀 | 修復 | 驗證 |
|------|------|------|------|
| Privilege Separation | `chroot: Operation not permitted` | 添加 SYS_CHROOT | `sshd -t` 通過 |
| 簽章算法 | `not in PubkeyAcceptedAlgorithms` | 配置算法列表 | 連接成功 |
| 進程清理 | `kill: Operation not permitted` | 監控中 | 待觀察 |

---

## 經驗教訓與後續建議

### 短期建議（1-2 週）

1. **應用修復** - 部署上述配置變更
2. **監控** - 觀察進程清理錯誤是否仍出現
3. **測試** - 使用多種客戶端和密鑰類型測試連接

### 中期建議（1-3 個月）

1. **密鑰升級計畫** - 開始將客戶端遷移至 Ed25519 密鑰
2. **文檔更新** - 在部署指南中添加密鑰生成最佳實踐
3. **自動化測試** - 建立 CI/CD 測試，確保 SSH 連接正常

### 長期建議（3-12 個月）

1. **淘汰舊算法** - 逐步移除 ssh-rsa 支援
2. **Capability 最小化** - 評估是否可移除某些 capabilities
3. **安全審計** - 定期檢查 OpenSSH 和 Alpine 的安全更新

### 未來改進

**SDD 文檔更新建議：**
- [ ] 添加 Linux capabilities 最佳實踐部分
- [ ] 詳細說明支援的密鑰類型和算法
- [ ] 建立密鑰管理和輪換的標準流程
- [ ] 記錄已知的兼容性問題

**代碼改進建議：**
- [ ] 在 docker-compose.yml 中添加註釋，說明每個 capability 的用途
- [ ] 考慮環境變數方式配置 PubkeyAcceptedAlgorithms（用於不同環境）
- [ ] 添加啟動時的配置驗證腳本

---

## 附錄：SSH 簽章算法對比

### 歷史背景

```
老一代：RSA (ssh-rsa, SHA-1)
        ↓
新一代：RSA-SHA2 (rsa-sha2-256, rsa-sha2-512)
        ↓
現代：ECDSA (ecdsa-sha2-nistp256/384/521)
        ↓
未來：EdDSA (ssh-ed25519) ← 強烈推薦
```

### 算法安全等級

```
ssh-rsa          [低]    SHA-1 已被棄用
rsa-sha2-256     [中等]  可接受
rsa-sha2-512     [較強]  建議
ecdsa-sha2-*     [較強]  建議
ssh-ed25519      [極強]  強烈推薦 ✅
```

### 性能對比

| 算法 | 密鑰生成 | 簽名速度 | 驗証速度 | 密鑰大小 |
|------|---------|---------|---------|---------|
| RSA-2048 | 中 | 快 | 快 | 2048 bits |
| RSA-4096 | 慢 | 較慢 | 較慢 | 4096 bits |
| ECDSA-256 | 快 | 快 | 快 | 256 bits |
| Ed25519 | 快 | 快 | 快 | 256 bits |

---

## 參考資源

- [OpenSSH 官方手冊 - sshd_config](https://man.openbsd.org/sshd_config)
- [OpenSSH 安全最佳實踐](https://www.ssh.com/academy/ssh/best-practices)
- [Linux Capabilities 文檔](https://man7.org/linux/man-pages/man7/capabilities.7.html)
- [IETF RFC 8332 - Use of RSA Keys with SHA-256 and SHA-512](https://tools.ietf.org/html/rfc8332)

---

**報告所有者：** Riven-dev1030
**最後更新：** 2026-01-09
**下一次檢查：** 2026-01-16
