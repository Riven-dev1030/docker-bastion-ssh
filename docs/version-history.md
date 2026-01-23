# 版本歷史

## [1.0.2] - 2026-01-22

### 新增
- docs/troubleshooting/2026-01-22-alpine-openssh-permitopen-wildcard-issue.md：Alpine Linux OpenSSH PermitOpen 通配符功能缺陷分析報告
  * 深入分析 Alpine Linux 上 OpenSSH 9.0p1-r5 和 9.3p2 版本的 PermitOpen 通配符不工作問題
  * 證實問題是 Alpine Linux 編譯/打包方式的系統級缺陷，而非版本或配置問題
  * 包含對照實驗設計、根本原因分析、技術深度探討
  * 提供三個解決方案：短期（使用具體 IP）、中期（切換基礎鏡像）、長期（官方修復）
  * 詳細的驗證方法、故障排除清單、配置示例

- docs/troubleshooting/2026-01-22-bastion-server-dockerization-guide.md：SSH 跳板機伺服器 Docker 化實施原理詳細指南
  * 完整記錄實體伺服器 (192.168.213.136) 的 SSH 跳板機服務 Docker 化過程
  * 詳細對比實體伺服器與 Docker 容器的配置差異（18 項配置對比）
  * 深入解析 Dockerfile 設計原理、啟動流程、容器權限配置
  * 包含 4 個實際使用案例（Ansible 整合、多環境部署、ProxyCommand、SCP 傳輸）
  * 提供生產環境最佳實踐、性能優化、故障排除指南
  * 完整附錄包含所有配置文件和腳本範例

### 文檔
- Alpine Linux OpenSSH 通配符問題現已被完整記錄和分析
- 跨越舊跳板機 (192.168.213.136) 和新跳板機 (192.168.213.31) 的 Alpine Linux 通配符問題已確認為系統級缺陷
- SSH 跳板機 Docker 化的完整技術文檔現已可用，涵蓋從實體伺服器到容器化的全過程
- 建立了實體伺服器與容器化環境的完整配置對比基準

---

## [1.0.1] - 2026-01-09

### 修復
- 修復 SSH 連接認證問題（Privilege Separation chroot 失敗）
  * docker-compose.yml：添加 SYS_CHROOT capability
  * config/sshd_config：添加 PubkeyAcceptedAlgorithms 支援 ssh-rsa 簽章
- 修正 SDD.md 和問題報告的所有者署名

### 新增
- docs/troubleshooting/2026-01-09-ssh-connection-issues.md：SSH 連接問題的完整解決報告
  * 包含三個問題的詳細分析、根本原因和修復方案
  * 安全性評估和驗證步驟
  * 故障排除流程和後續建議

---

## [1.0.0] - 2026-01-09

### 新增
- 建立完整的 SDD (Specification Driven Development) 規格文檔
  - 詳細的產品概述和價值主張
  - 11 個功能需求的完整定義
  - 5 個非功能需求（性能、可靠性、安全、可維護性、兼容性）
  - 系統架構設計（整體架構圖、組件說明、數據流）
  - 4 個主要使用場景和工作流程說明
  - 技術實現細節（Dockerfile、SSH 配置、Docker Compose）
  - API 和接口設計規範
  - 部署架構（本地、生產、Kubernetes）
  - 完整的安全性設計（認證、加密、訪問控制）
  - 6 個級別的測試策略
  - 12 個未來擴展計畫項目（短中長期）

### 文檔位置
- `SDD.md` - 主規格文檔

---

## 首次提交

此專案包含以下文檔和代碼：
- README.md - 快速開始指南
- Dockerfile - Docker 鏡像定義
- docker-compose.yml - 容器編排
- Makefile - 管理工具
- config/ - SSH 配置文件
- scripts/ - 啟動腳本和測試工具
- docs/ - 詳細文檔
- examples/ - 整合範例
