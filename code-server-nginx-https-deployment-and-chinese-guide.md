### 推荐的 GitHub 仓库名称
*   `code-server-deployment-guide`
*   `vscode-in-browser-setup`
*   `my-code-server`

---

### 文档内容 (README.md)

```markdown
# 🚀 code-server + Nginx + 中文插件 完整部署指南

本项目记录了在 Linux 服务器上，通过 `systemd` 管理 `code-server`，配合 Nginx 反向代理与基础认证，最终实现通过域名访问全中文 VS Code 环境的完整过程。

> **适用场景**：云服务器 (ECS/VPS)、Docker 宿主机、远程开发环境搭建。

---

## 📝 目录

1.  [环境准备](#-环境准备)
2.  [安装 code-server](#-安装-code-server)
3.  [配置 systemd 服务](#-配置-systemd-服务)
4.  [Nginx 反向代理配置](#-nginx-反向代理配置)
5.  [双重认证优化 (Basic Auth + code-server)](#-双重认证优化-basic-auth--code-server)
6.  [界面汉化 (中文插件)](#-界面汉化-中文插件)
7.  [FAQ / 常见问题](#-faq--常见问题)

---

## 🛠️ 环境准备

*   **操作系统**: CentOS 7 / Ubuntu 20.04+ (文中以 CentOS 7 为例)
*   **软件依赖**: Nginx, systemd, code-server
*   **域名**: 已解析到服务器 IP (例如 `vscode.yourdomain.com`)

---

## 🔧 安装 code-server

1.  **安装 code-server** (请参考官方文档安装最新版，此处为命令记录):
    ```bash
    # 下载并安装 code-server (示例)
    curl -fsSL https://code-server.dev/install.sh | sh
    ```

---

## ⚙️ 配置 systemd 服务

为了让 code-server 在后台稳定运行并开机自启，我们使用 systemd 进行管理。

### 1. 创建服务文件
```bash
sudo vim /usr/lib/systemd/system/code-server@.service
```

### 2. 写入以下配置
```ini
[Unit]
Description=code-server
After=network.target

[Service]
Type=exec
ExecStart=/usr/bin/code-server
Restart=always
User=root

[Install]
WantedBy=default.target
```

### 3. 启动服务
```bash
sudo systemctl daemon-reload
sudo systemctl enable code-server@root
sudo systemctl start code-server@root
```

---

## 🌐 Nginx 反向代理配置

为了让用户通过 HTTPS 访问，并增加一层基础防护。

### 1. 配置 Nginx (含 SSL 证书)
```nginx
# HTTP 自动跳转 HTTPS
server {
    listen 80;
    server_name vscode.yourdomain.com; # 修改为你的域名
    return 301 https://$host$request_uri;
}

# HTTPS 主配置
server {
    listen 443 ssl;
    server_name vscode.yourdomain.com;

    # SSL 证书配置 (请确保路径正确)
    ssl_certificate /path/to/your/cert.crt;
    ssl_certificate_key /path/to/your/cert.key;

    # 基础认证 (htpasswd)
    auth_basic "Restricted Access";
    auth_basic_user_file /etc/nginx/conf.d/auth/.htpasswd;

    location / {
        proxy_pass http://127.0.0.1:1641; # 转发到 code-server
        proxy_set_header Host $host;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_buffering off;
    }
}
```

> **注意**: 请确保云服务器的安全组开放了 `80` 和 `443` 端口。

---

## 🔐 双重认证优化 (Basic Auth + code-server)

在配置过程中，你可能会遇到 **“输入了 Nginx 密码后，还要再输 code-server 密码”** 的双重验证问题。

### 解决方案：取消 code-server 自身的密码验证

既然 Nginx 已经有一层 `htpasswd` 验证，我们可以关闭 code-server 自带的密码验证，实现单点登录体验。

#### 1. 修改 code-server 配置
编辑配置文件 `~/.config/code-server/config.yaml`：

```yaml
bind-addr: 127.0.0.1:1641
auth: none       # 关键步骤：将 password 改为 none
cert: false
```

#### 2. 重启服务
```bash
sudo systemctl restart code-server@root
```

**效果**: 浏览器弹出 Nginx 登录框 -> 输入账号密码 -> 直接进入 VS Code，无需二次验证。

---

## 🇨🇳 界面汉化 (中文插件)

为了让界面更友好，安装官方中文语言包。

### 1. 安装插件
*   打开 code-server 左侧扩展栏 (Extensions)。
*   搜索 `Chinese`。
*   找到 **"Chinese (Simplified) Language Pack for Visual Studio Code"** (发布者: Microsoft)，点击安装。

### 2. 切换语言
*   按 `Ctrl + Shift + P` 打开命令面板。
*   输入 `Configure Display Language`。
*   选择 `zh-cn`。
*   重启编辑器。

---

## 🛡️ FAQ / 常见问题

### 1. 网页打不开 / 连接超时
*   **原因**: 通常是云服务器防火墙（安全组）未开放端口。
*   **解决**: 去云控制台检查，确保 **80** 和 **443** 端口对公网开放。

### 2. Nginx 502 Bad Gateway
*   **原因**: code-server 服务未启动，或端口不匹配。
*   **解决**:
    *   检查 `systemctl status code-server@root` 状态。
    *   检查 Nginx `proxy_pass` 的端口是否与 code-server 监听端口一致（默认 8080 或配置文件中的端口）。

### 3. SELinux 导致无法连接
*   **原因**: CentOS 7 默认开启 SELinux，可能阻止 Nginx 连接本地端口。
*   **解决**:
    ```bash
    # 允许 Nginx 发起网络连接
    sudo setsebool -P httpd_can_network_connect 1
    sudo systemctl restart nginx
    ```

---

## 📬 参考资料
*   code-server 官网: https://github.com/coder/code-server
*   VS Code 中文包: https://marketplace.visualstudio.com/items?itemName=MS-CEINTL.vscode-language-pack-zh-hans
