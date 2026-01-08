# Ansible 整合範例

本範例展示如何將 Docker Bastion SSH Server 與 Ansible 整合使用。

## 快速開始

### 1. 啟動 Bastion 跳板機

```bash
# 回到專案根目錄
cd ../..

# 啟動跳板機
docker-compose up -d

# 測試連接
make test-connect
```

### 2. 配置 Ansible

#### 方法 1：在 Inventory 中配置 ProxyCommand

```yaml
# inventory.yml
all:
  vars:
    ansible_ssh_common_args: >
      -o ProxyCommand="ssh -W %h:%p -i ~/.ssh/bastion_key -p 2222 root@localhost"
      -o StrictHostKeyChecking=no

  children:
    network_devices:
      hosts:
        router1:
          ansible_host: 192.168.1.11
          ansible_user: admin
        router2:
          ansible_host: 192.168.1.12
          ansible_user: admin
```

#### 方法 2：在 ansible.cfg 中全局配置

```ini
[defaults]
inventory = ./inventory

[ssh_connection]
# 透過跳板機的 SSH 配置
ssh_args = -o ProxyCommand="ssh -W %h:%p -i ~/.ssh/bastion_key -p 2222 root@localhost" -o StrictHostKeyChecking=no
```

### 3. 測試連接

```bash
# 測試單個主機
ansible router1 -m ping

# 測試所有主機
ansible all -m ping

# 執行命令
ansible all -m shell -a "show version"
```

## 配置說明

### ProxyCommand 參數解釋

```bash
-o ProxyCommand="ssh -W %h:%p -i ~/.ssh/bastion_key -p 2222 root@localhost"
```

- `-W %h:%p`: 啟用 stdio 轉發模式，%h 和 %p 會被替換為目標主機和端口
- `-i ~/.ssh/bastion_key`: 指定連接跳板機的私鑰
- `-p 2222`: 跳板機的 SSH 端口
- `root@localhost`: 跳板機的用戶和地址

### ansible_ssh_common_args vs ssh_args

| 配置位置 | 作用範圍 | 優先級 |
|---------|---------|--------|
| `ansible_ssh_common_args` | 可以針對特定主機或組 | 高 |
| `ansible.cfg` 中的 `ssh_args` | 全局所有連接 | 低 |

建議：
- 如果所有設備都使用同一個跳板機 → 使用 `ansible.cfg`
- 如果不同設備使用不同跳板機 → 使用 `ansible_ssh_common_args`

## 進階配置

### 使用環境變數

```yaml
# inventory.yml
all:
  vars:
    bastion_host: "{{ lookup('env', 'BASTION_HOST') | default('localhost') }}"
    bastion_port: "{{ lookup('env', 'BASTION_PORT') | default('2222') }}"
    bastion_key: "{{ lookup('env', 'BASTION_KEY') | default('~/.ssh/bastion_key') }}"
    ansible_ssh_common_args: >
      -o ProxyCommand="ssh -W %h:%p -i {{ bastion_key }} -p {{ bastion_port }} root@{{ bastion_host }}"
```

使用方式：

```bash
export BASTION_HOST=192.168.1.100
export BASTION_PORT=2222
export BASTION_KEY=~/.ssh/bastion_key

ansible-playbook playbook.yml
```

### 多跳板機配置

如果有多個環境需要不同的跳板機：

```yaml
# inventory.yml
all:
  children:
    production:
      vars:
        ansible_ssh_common_args: >
          -o ProxyCommand="ssh -W %h:%p -i ~/.ssh/prod_bastion -p 2222 root@prod-bastion.company.com"
      hosts:
        prod-router1:
          ansible_host: 10.0.1.1

    staging:
      vars:
        ansible_ssh_common_args: >
          -o ProxyCommand="ssh -W %h:%p -i ~/.ssh/stag_bastion -p 2222 root@stag-bastion.company.com"
      hosts:
        stag-router1:
          ansible_host: 10.1.1.1
```

## 故障排除

### 無法連接到目標主機

```bash
# 1. 先測試跳板機連接
ssh -i ~/.ssh/bastion_key -p 2222 root@localhost

# 2. 測試透過跳板機的轉發
ssh -o ProxyCommand='ssh -W %h:%p -i ~/.ssh/bastion_key -p 2222 root@localhost' admin@192.168.1.11

# 3. 使用 Ansible 的詳細模式
ansible router1 -m ping -vvv
```

### PermitOpen 限制問題

如果看到 "administratively prohibited" 錯誤：

```bash
# 檢查跳板機的 PermitOpen 配置
docker exec ansible-bastion grep PermitOpen /etc/ssh/sshd_config

# 確保目標 IP 和端口在允許範圍內
# 例如：PermitOpen 192.168.1.*:22
```

需要修改的話，編輯 `../../config/sshd_config` 並重新構建：

```bash
cd ../..
docker-compose up -d --build
```

### SSH 密鑰權限問題

```bash
# 確保私鑰權限正確
chmod 600 ~/.ssh/bastion_key

# 確保 SSH 目錄權限正確
chmod 700 ~/.ssh
```

## 範例 Playbook

### 簡單的配置收集

```yaml
# gather_facts.yml
---
- name: 收集網路設備資訊
  hosts: all
  gather_facts: no

  tasks:
    - name: 獲取設備版本
      ios_command:
        commands:
          - show version
      register: version_output

    - name: 顯示結果
      debug:
        var: version_output.stdout_lines
```

執行：

```bash
ansible-playbook gather_facts.yml
```

### 配置部署

```yaml
# deploy_config.yml
---
- name: 部署設備配置
  hosts: routers
  gather_facts: no

  tasks:
    - name: 配置介面
      ios_config:
        lines:
          - description Managed by Ansible
          - ip address 192.168.1.1 255.255.255.0
        parents: interface GigabitEthernet0/0
```

## 相關資源

- [Ansible Network Modules](https://docs.ansible.com/ansible/latest/collections/ansible/netcommon/)
- [SSH ProxyCommand](https://en.wikibooks.org/wiki/OpenSSH/Cookbook/Proxies_and_Jump_Hosts)
- [Bastion 配置文檔](../../docs/CUSTOMIZATION_GUIDE.md)
