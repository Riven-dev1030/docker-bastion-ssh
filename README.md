# Docker Bastion SSH Server

一個輕量級、安全的 SSH 跳板機 (Bastion Host) Docker 鏡像，專為 ProxyCommand 方式的 SSH 轉發設計，支援 PermitOpen 限制。

## 功能特性

✅ **SSH 密鑰認證** - 僅支援密鑰認證，禁用密碼認證
✅ **TCP 轉發限制** - 使用 PermitOpen 限制轉發目標
✅ **ProxyCommand 支援** - 完全支援 SSH -W 參數進行 stdio 轉發
✅ **輕量級** - 基於 Alpine Linux，鏡像大小僅 ~20MB
✅ **容器化** - 易於部署和擴展
✅ **安全** - 遵循 SSH 安全最佳實踐

## 快速開始

### 1. 準備 SSH 公鑰

```bash
# 生成 SSH 密鑰對（如果還沒有）
ssh-keygen -t rsa -b 4096 -f ~/.ssh/bastion_key -N ""

# 複製公鑰到配置檔案
cat ~/.ssh/bastion_key.pub > config/authorized_keys
```

### 2. 啟動跳板機

```bash
# 使用 docker-compose 構建和啟動
docker-compose up -d

# 或使用 Makefile
make build
make up
```

### 3. 驗證連接

```bash
# 測試 SSH 連接
ssh -i ~/.ssh/bastion_key -p 2222 root@localhost

# 或使用 make 命令
make test-connect

# 查看容器日誌
make logs
```

## 使用場景

### 1. Ansible 跳板機

完整範例請參考：[examples/ansible-integration/](examples/ansible-integration/)

```yaml
# inventory.yml
all:
  vars:
    ansible_ssh_common_args: >
      -o ProxyCommand="ssh -W %h:%p -i ~/.ssh/bastion_key -p 2222 root@localhost"
```

### 2. 直接 SSH 轉發

```bash
# 透過跳板機連接到內網設備
ssh -o ProxyCommand='ssh -W %h:%p -i ~/.ssh/bastion_key -p 2222 root@localhost' user@192.168.1.10
```

### 3. SCP 文件傳輸

```bash
# 透過跳板機傳輸文件
scp -o ProxyCommand='ssh -W %h:%p -i ~/.ssh/bastion_key -p 2222 root@localhost' file.txt user@192.168.1.10:/path/
```

### 4. 作為多環境 SSH Gateway

適用於 DevOps、網路管理、雲端環境等場景，提供安全的 SSH 訪問入口。

## 配置說明

### 目錄結構

```
docker-bastion-ssh/
├── Dockerfile                    # Docker 鏡像定義
├── docker-compose.yml           # Docker Compose 配置
├── Makefile                     # 管理工具
├── config/
│   ├── sshd_config             # SSH 伺服器配置
│   └── authorized_keys.example # SSH 公鑰範例
├── scripts/
│   ├── entrypoint.sh           # 容器啟動腳本
│   └── test-bastion.sh         # 測試腳本
├── docs/                        # 文檔目錄
│   ├── SSH_KEY_MANAGEMENT.md   # SSH 密鑰管理指南
│   ├── CUSTOMIZATION_GUIDE.md  # 自訂配置指南
│   └── PERMITOPEN_GUIDE.md     # PermitOpen 配置指南
└── examples/                    # 整合範例
    └── ansible-integration/    # Ansible 整合範例
```

### 主要配置項 (sshd_config)

| 配置項 | 值 | 說明 |
|--------|-----|------|
| `PubkeyAuthentication` | yes | 啟用密鑰認證 |
| `PasswordAuthentication` | no | 禁用密碼認證 |
| `AllowTcpForwarding` | yes | 啟用 TCP 轉發（ProxyCommand 需要） |
| `PermitOpen` | 192.168.1.*:22 | 只允許轉發到指定網段 |
| `PermitRootLogin` | prohibit-password | 允許 root 登入但需要密鑰 |
| `ClientAliveInterval` | 300 | 每 300 秒傳送一個保活訊號 |

修改 `config/sshd_config` 可以自訂這些配置。詳細說明請參考：[docs/CUSTOMIZATION_GUIDE.md](docs/CUSTOMIZATION_GUIDE.md)

## Makefile 命令

```bash
make help           # 顯示所有可用命令
make build          # 構建 Docker 鏡像
make up             # 啟動容器
make down           # 停止容器
make logs           # 查看容器日誌
make shell          # 進入容器 shell
make test-connect   # 測試 SSH 連接
make test-config    # 測試 SSH 配置
make clean          # 清理容器和鏡像
make verify         # 驗證環境和配置
```

## 進階配置

### 自訂 PermitOpen 規則

編輯 `config/sshd_config`：

```ini
# 支援多個轉發目標
PermitOpen 192.168.1.*:22      # SSH
PermitOpen 192.168.1.*:23      # Telnet
PermitOpen 192.168.1.*:443     # HTTPS
PermitOpen 10.0.0.*:22         # 其他網段
```

重新構建並啟動：

```bash
docker-compose up -d --build
```

詳細說明請參考：[docs/PERMITOPEN_GUIDE.md](docs/PERMITOPEN_GUIDE.md)

### 使用 Volume 動態更新配置

在 `docker-compose.yml` 中取消註解：

```yaml
volumes:
  - ./config/sshd_config:/etc/ssh/sshd_config:ro
  - ./config/authorized_keys:/root/.ssh/authorized_keys:ro
```

這樣可以在不重新構建鏡像的情況下更新配置。

## 故障排除

### 無法連接到容器

```bash
# 1. 檢查容器狀態
docker ps | grep bastion

# 2. 檢查日誌
docker logs ansible-bastion

# 3. 驗證通訊埠映射
docker port ansible-bastion

# 4. 測試連接（使用詳細模式）
ssh -vvv -i ~/.ssh/bastion_key -p 2222 root@localhost
```

### 權限被拒絕 (Permission denied)

```bash
# 1. 檢查 authorized_keys 是否正確配置
docker exec ansible-bastion cat /root/.ssh/authorized_keys

# 2. 驗證本地私鑰權限（應該是 600）
ls -la ~/.ssh/bastion_key

# 3. 修正權限
chmod 600 ~/.ssh/bastion_key
```

### ProxyCommand 連接失敗

```bash
# 測試跳板機連接
ssh -i ~/.ssh/bastion_key -p 2222 root@localhost

# 測試 TCP 轉發
ssh -v -o ProxyCommand='ssh -W %h:%p -i ~/.ssh/bastion_key -p 2222 root@localhost' user@target-host
```

更多故障排除，請參考 [docs/](docs/) 目錄下的文檔。

## 生產環境建議

### 1. 密鑰管理

- 不要在 Dockerfile 中包含 authorized_keys
- 使用 Docker Secrets 或 Volume 掛載
- 定期輪換 SSH 密鑰

### 2. 日誌持久化

```yaml
volumes:
  - bastion_logs:/var/log
```

### 3. 資源限制

```yaml
deploy:
  resources:
    limits:
      cpus: '0.5'
      memory: 256M
    reservations:
      cpus: '0.25'
      memory: 128M
```

### 4. 健康檢查和重啟策略

已在 `docker-compose.yml` 中預設配置：

```yaml
healthcheck:
  test: ["CMD", "sshd", "-t"]
  interval: 30s
  timeout: 10s
  retries: 3

restart: unless-stopped
```

### 5. 安全加固

- 使用最新的 Alpine 基礎鏡像
- 啟用容器掃描（如 Trivy）
- 限制容器權限（cap_drop）
- 定期審計日誌

## 文檔

- [SSH 密鑰管理指南](docs/SSH_KEY_MANAGEMENT.md)
- [自訂配置指南](docs/CUSTOMIZATION_GUIDE.md)
- [PermitOpen 配置指南](docs/PERMITOPEN_GUIDE.md)
- [Ansible 整合範例](examples/ansible-integration/)

## 整合範例

查看 [examples/](examples/) 目錄了解如何與不同工具整合：

- Ansible
- Terraform（待補充）
- Kubernetes（待補充）

## 清理資源

```bash
# 使用 docker-compose
docker-compose down

# 完整清理（包括鏡像和 volumes）
make clean

# 手動清理
docker stop ansible-bastion
docker rm ansible-bastion
docker rmi ansible-bastion:latest
docker volume rm bastion_logs
```

## 參考資源

- [OpenSSH 官方文檔](https://man.openbsd.org/sshd_config)
- [Docker 官方文檔](https://docs.docker.com/)
- [SSH ProxyCommand 指南](https://en.wikibooks.org/wiki/OpenSSH/Cookbook/Proxies_and_Jump_Hosts)

## 授權

MIT License

## 貢獻

歡迎提交 Issue 和 Pull Request！

---

**注意**：本專案從 [ansible-network-lab](https://github.com/Riven-dev1030/ansible-network-lab) 專案分離而來，專注於提供通用的 SSH 跳板機解決方案。
