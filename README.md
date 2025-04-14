# 云端开发环境部署流程：Git + VS Code + Docker Compose + SSH Agent

本指南介绍如何利用本地 Mac、远程 Linux 服务器、VS Code、Docker Compose、 Git 和 SSH Agent 搭建一个高效、安全且环境一致的云端开发环境。

**目标:** 在远程 Linux 服务器上运行 Docker Compose 管理的应用栈，同时在本地 Mac 使用 VS Code 进行编码和调试，并通过 SSH Agent 安全处理 Git 认证，实现开发与生产环境的高度统一。

**核心组件:**

*   **本地 Mac:** 开发工作站 (运行 VS Code, 存储 SSH 私钥)。
*   **远程 Linux 服务器:** 实际计算和运行环境 (运行 Docker Engine, Compose, 容器, 代码)。
*   **VS Code + Remote SSH 扩展:** 连接本地与远程的桥梁，提供无缝的远程开发体验。
*   **Docker Compose:** 在远程服务器上编排多容器应用。
*   **SSH Agent (本地):** 安全管理本地 SSH 私钥。
*   **SSH Agent Forwarding:** 允许远程服务器安全地“借用”本地 Agent 的认证能力。

**前提条件:**

*   **本地 Mac:**
    *   已安装 Visual Studio Code (VS Code)。
    *   已安装 VS Code 的 "Remote - SSH" 扩展。
    *   已安装 Git。
    *   已生成 SSH 密钥对 (推荐 `ed25519`)。
*   **远程 Linux 服务器:**
    *   拥有 SSH 访问权限。
    *   已安装并运行 SSH 服务器 (sshd)。
    *   已安装 Docker Engine (并将用户添加到 `docker` 组)。
    *   已安装 Docker Compose (推荐 v2 插件, `docker compose`)。
    *   已安装 Git。

---

## 部署与配置步骤

### 第一步：准备远程 Linux 服务器

1.  **配置 SSH 服务:**
    *   确保 `sshd` 服务正在运行 (`sudo systemctl status ssh` 或 `sudo service ssh status`)。
    *   **强烈推荐:** 配置基于 SSH 密钥的认证，并禁用密码登录。将你 Mac 的公钥 (`~/.ssh/id_*.pub`) 内容添加到服务器用户的 `~/.ssh/authorized_keys` 文件中。
2.  **安装 Docker Engine:** 参考 Docker 官方文档为你的 Linux 发行版安装。
3.  **安装 Docker Compose:** 确保 `docker compose version` 命令可用。
4.  **安装 Git:** `sudo apt update && sudo apt install git` 或 `sudo yum install git` (如果需要)。

### 第二步：准备本地 Mac 开发环境

1.  **生成或确认 SSH 密钥:**
    *   如果还没有，生成密钥：`ssh-keygen -t ed25519 -C "your_email@example.com"`。
    *   **为私钥设置强密码 (passphrase)。**
2.  **配置本地 SSH Agent:**
    *   添加私钥到 Agent：`ssh-add -K ~/.ssh/your_private_key_file` (例如 `id_ed25519`)。`-K` 将密码存入 Keychain (macOS)。
    *   验证：`ssh-add -l`，应能看到密钥指纹。

### 第三步：配置 VS Code 连接与 SSH Agent Forwarding

1.  **编辑本地 SSH 配置文件 (`~/.ssh/config`):**
    *   添加或修改指向远程服务器的配置项，**必须包含 `ForwardAgent yes`**：
        ```
        Host my-remote-server      # 自定义别名
            HostName server_ip_or_hostname
            User your_remote_username
            IdentityFile ~/.ssh/your_private_key_file # 本地私钥路径
            ForwardAgent yes      # 启用 Agent Forwarding
        ```
2.  **使用 VS Code 连接:**
    *   打开 VS Code，使用 "Remote - SSH" 扩展连接到你配置的 `my-remote-server`。
3.  **验证 Agent Forwarding:**
    *   连接成功后，在 VS Code 中打开**远程终端** (`Terminal > New Terminal`)。
    *   在**远程终端**中运行 `ssh-add -l`。输出应与本地一致。
    *   (可选) 测试 GitHub 连接：`ssh -T git@github.com`，应提示认证成功。

### 第四步：在远程服务器上设置项目

1.  **克隆代码库 (使用 SSH URL):**
    *   在 VS Code 的**远程终端**中，导航到项目目录。
    *   运行 `git clone git@github.com:YourUsername/your-repo.git`。由于 Agent Forwarding，认证会自动完成。
2.  **打开远程项目文件夹:**
    *   在 VS Code 中，使用 "File > Open Folder..." 打开**远程服务器**上的项目目录。
3.  **创建 Docker 文件:**
    *   在 VS Code 中编辑 `Dockerfile`、`docker-compose.yml`、`.env` 等文件。
    *   在 `docker-compose.yml` 中：
        *   使用**明确的版本标签**指定基础镜像 (e.g., `nginx:1.26.1-alpine`)。
        *   **配置卷 (Volumes):** 将本地（服务器上的）源代码目录挂载到容器内，如 `- ./app/src:/app/src`。
    *   在 `Dockerfile` 中：
        *   安装所有系统和 Python 依赖。
        *   最好创建并使用非 root 用户运行应用。

### 第五步：开发工作流

1.  **启动环境:** 在 VS Code **远程终端**的项目根目录运行 `docker compose up -d`。
2.  **编码:** 在 VS Code 中直接编辑代码，更改会通过卷挂载实时同步到容器。
3.  **Git 操作:** 在 VS Code **远程终端**中使用 `git pull/push/commit` 等。与 GitHub (使用 SSH URL) 的交互将通过 Agent Forwarding 安全认证。
4.  **执行命令:** 使用 `docker compose exec <service_name> <command>` 在容器内执行测试、迁移、Shell 等。
    ```bash
    # 示例:
    docker compose exec app pytest
    docker compose exec app python manage.py migrate
    docker compose exec app /bin/bash
    ```
5.  **调试:**
    *   在容器中安装调试库 (如 `debugpy`)。
    *   修改应用启动方式以监听调试端口。
    *   在 `docker-compose.yml` 中映射调试端口。
    *   在 VS Code 中配置 `launch.json` (类型为 "Python: Remote Attach") 进行远程调试。
6.  **访问 Web 应用:**
    *   利用 VS Code 的 "端口" (Ports) 面板自动或手动转发端口。
    *   在**本地 Mac 浏览器**中通过 `localhost:<local_port>` 访问。
7.  **停止环境:** 在 VS Code **远程终端**中运行 `docker compose down`。

---

## 安全注意事项

*   **Agent Forwarding 风险:** 仅在你**完全信任**的远程服务器上启用 `ForwardAgent yes`。远程服务器上的 root 用户理论上可以劫持转发的 agent 连接。
*   **保护本地私钥:** 确保本地私钥文件的安全，并设置强密码。
*   **启用 GitHub 2FA:** 增加账户安全层级。
*   **保持更新:** 定期更新 VS Code、扩展、Docker、Compose 和操作系统。