# Cloudreve 私有云盘

基于 Docker 部署的 [Cloudreve](https://cloudreve.org/) 私有云盘服务，包含完整数据。

## 📋 环境要求

### 系统依赖

| 组件 | 最低版本 | 推荐版本 | 本机测试版本 |
|------|---------|---------|------------|
| [Docker](https://docs.docker.com/engine/install/) | `20.x` | `24.x+` | `26.1.3` |
| [Docker Compose](https://docs.docker.com/compose/install/) | `2.x` | `2.20+` | `2.27.0` |
| 操作系统 | Linux x86_64 | Linux x86_64 | Alibaba Cloud Linux 3 |
| 内存 | 512MB | 1GB+ | - |

> **架构支持**：本仓库基于 `linux/amd64`（x86_64）测试。ARM64（如树莓派、Apple Silicon Mac）理论上可用，但未测试。

### Cloudreve 版本

| 组件 | 版本 |
|------|------|
| Cloudreve 镜像 | `cloudreve/cloudreve:latest`（测试于 2026-06-06 构建） |
| 数据库 | SQLite（内置，无需额外安装） |

> 💡 `latest` 标签指向 Cloudreve 最新稳定版。如需固定版本，可在 `docker-compose.yml` 中改为具体版本号（如 `cloudreve/cloudreve:4.0.0`）。

### 安装 Docker 和 Docker Compose

如果新机器尚未安装 Docker，按以下步骤操作。

#### Ubuntu / Debian

```bash
# 卸载旧版本
sudo apt remove docker docker-engine docker.io containerd runc

# 安装依赖
sudo apt update
sudo apt install -y ca-certificates curl gnupg

# 添加 Docker 官方 GPG 密钥和源
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list

# 安装 Docker
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# 启动并设置开机自启
sudo systemctl enable docker --now

# （可选）将当前用户加入 docker 组，避免每次用 sudo
sudo usermod -aG docker $USER
# 注意：重新登录后生效，或执行 newgrp docker
```

#### CentOS / RHEL / Alibaba Cloud Linux

```bash
# 卸载旧版本
sudo yum remove docker docker-client docker-client-latest docker-common docker-latest docker-latest-logrotate docker-logrotate docker-engine

# 安装依赖
sudo yum install -y yum-utils

# 添加 Docker 源
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

# 安装 Docker
sudo yum install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# 启动并设置开机自启
sudo systemctl enable docker --now
```

#### 验证安装

```bash
# 检查 Docker 版本
docker --version
# 输出示例：Docker version 26.1.3, build b72abbb

# 检查 Docker Compose 版本
docker compose version
# 输出示例：Docker Compose version v2.27.0

# 测试容器运行
docker run hello-world
```

> **提示**：如果 `docker compose` 命令不可用，请确认 `docker-compose-plugin` 已正确安装。旧版系统可能需要单独安装 `docker-compose`（Python 版），但建议使用 `docker compose`（v2 插件版）。

### 安装常见问题

<details>
<summary><b>❌ bash: git: command not found</b></summary>

```bash
# Ubuntu/Debian
sudo apt install -y git

# CentOS/RHEL
sudo yum install -y git
```
</details>

<details>
<summary><b>❌ curl: command not found</b></summary>

```bash
# Ubuntu/Debian
sudo apt install -y curl

# CentOS/RHEL
sudo yum install -y curl
```
</details>

<details>
<summary><b>❌ lsb_release: command not found（Ubuntu 添加 Docker 源时）</b></summary>

```bash
sudo apt install -y lsb-release
```

如果仍报错，直接用系统代号替换 `$(lsb_release -cs)`：
```bash
# 查看当前 Ubuntu 版本
cat /etc/os-release
# 例如 Ubuntu 22.04 则直接写 jammy，20.04 写 focal
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu jammy stable" | sudo tee /etc/apt/sources.list.d/docker.list
```
</details>

<details>
<summary><b>❌ yum-config-manager: command not found（CentOS）</b></summary>

```bash
sudo yum install -y yum-utils
```
</details>

<details>
<summary><b>❌ Package docker-ce conflicts with podman（CentOS/RHEL）</b></summary>

部分 CentOS/RHEL 8+ 预装了 `podman` 和 `buildah`，与 Docker 冲突。先移除它们：

```bash
sudo yum remove -y podman buildah runc
# 然后重新安装 Docker
sudo yum install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
```
</details>

<details>
<summary><b>❌ Cannot connect to the Docker daemon</b></summary>

Docker 服务未启动：

```bash
# 查看 Docker 服务状态
sudo systemctl status docker

# 如果未运行，手动启动
sudo systemctl start docker

# 设置开机自启
sudo systemctl enable docker

# 如果启动失败，查看详细日志
sudo journalctl -u docker --no-pager -n 50
```
</details>

<details>
<summary><b>❌ docker: permission denied — 非 root 用户运行 Docker</b></summary>

当前用户不在 `docker` 用户组：

```bash
# 将当前用户加入 docker 组
sudo usermod -aG docker $USER

# 使组变更立即生效（二选一）
newgrp docker          # 当前终端生效
# 或 退出重新登录 SSH

# 验证
docker ps
```
</details>

<details>
<summary><b>❌ docker compose: command not found</b></summary>

Docker Compose v2 插件未安装：

```bash
# Ubuntu/Debian
sudo apt install -y docker-compose-plugin

# CentOS/RHEL
sudo yum install -y docker-compose-plugin
```

如果无法安装插件，可安装独立版：
```bash
# 下载 v2.27.0 （替换为最新版本号）
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

# 验证
docker-compose --version
```
</details>

<details>
<summary><b>❌ docker run hello-world 失败 — 无法拉取镜像</b></summary>

网络问题导致 Docker Hub 无法访问（国内常见）：

```bash
# 配置国内镜像加速器
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": [
    "https://docker.1ms.run",
    "https://docker.xuanyuan.me",
    "https://hub.rat.dev"
  ]
}
EOF

# 重启 Docker
sudo systemctl daemon-reload
sudo systemctl restart docker

# 再次测试
docker run hello-world
```

> 镜像加速器地址可能随政策变化，建议搜索最新的国内可用地址。
</details>

<details>
<summary><b>❌ 容器启动后立即退出（Cloudreve 无法启动）</b></summary>

```bash
# 查看容器日志
docker compose logs

# 常见原因 1：data 目录权限问题
sudo chown -R 1000:1000 data/

# 常见原因 2：数据库文件损坏 — 检查 data/cloudreve.db 是否存在且非空
ls -lh data/cloudreve.db

# 常见原因 3：conf.ini 中 SessionSecret 为空
cat data/conf.ini
```
</details>

<details>
<summary><b>❌ 浏览器无法访问 http://服务器IP:5212</b></summary>

```bash
# 1. 检查容器是否在运行
docker ps | grep cloudreve

# 2. 检查端口是否监听
ss -tlnp | grep 5212

# 3. 检查防火墙（CentOS/RHEL）
sudo systemctl status firewalld
sudo firewall-cmd --list-ports
# 放行 5212 端口
sudo firewall-cmd --add-port=5212/tcp --permanent
sudo firewall-cmd --reload

# 4. 检查云服务商安全组（阿里云/腾讯云/AWS 等）
# ⇒ 登录云控制台，在安全组/防火墙规则中放行 5212 端口（TCP）
```
</details>

<details>
<summary><b>❌ 忘记管理员密码</b></summary>

```bash
# 方法一：查看首次部署时的默认密码
docker compose exec cloudreve /cloudreve --initial-password

# 方法二：通过 SQLite 直接重置密码
# 安装 sqlite3 工具
sudo apt install -y sqlite3   # Ubuntu/Debian
sudo yum install -y sqlite    # CentOS/RHEL

# 重置 admin 密码为 123456
sqlite3 data/cloudreve.db "UPDATE users SET password='\$2a\$10\$lC6cXJ5C5vY5V.PE5oXh/Owj0p5cZ0YqQ0L3Q1F0z0Z0X0Y0Z0X0Y0Z0X0Y0Z' WHERE id=1;"
```

> 重置后请立即登录修改密码。
</details>

## 🚀 快速开始

### 1. 克隆项目

```bash
git clone https://github.com/MartaBanion/cloudreve.git
cd cloudreve
```

### 2. 启动服务

```bash
docker-compose up -d
```

Docker 会自动拉取镜像并启动服务，所有数据（数据库、配置文件、上传文件）已包含在项目中，**无需额外配置**。

### 3. 访问服务

浏览器打开 **http://你的服务器IP:5212**

### 4. 登录

管理员账号：`admin@cloudreve.org`

> 🔑 密码如果是首次部署的默认密码，可以通过以下命令查看：
> ```bash
> docker-compose exec cloudreve /cloudreve --initial-password
> ```
>
> 或者登录后在管理后台自行重置密码。

### 5. （可选）修改站点地址

登录后进入 **管理面板 → 系统设置**，将"站点 URL"改为你的实际访问地址（如 `http://你的公网IP:5212`）。

## 📂 项目结构

```
cloudreve/
├── docker-compose.yml     # Docker 编排文件
├── README.md              # 本文件
├── conf/                  # 配置文件备用目录（可忽略）
├── docs/
│   └── visitor-download.md # 游客下载功能配置指南
├── data/
│   ├── cloudreve.db       # SQLite 数据库（用户、文件索引、设置）
│   ├── conf.ini           # 主配置文件（SessionSecret、HashIDSalt）
│   └── uploads/           # 上传文件存储
│       └── {uid}/         # 按用户 ID 分目录
├── uploads/               # 备用目录（当前未使用）
└── avatar/                # 用户头像
```

## 🐳 Docker Compose 说明

```yaml
services:
  cloudreve:
    image: cloudreve/cloudreve:latest
    container_name: cloudreve
    restart: unless-stopped
    ports:
      - "5212:5212"                     # web 端口映射
    volumes:
      - ./uploads:/cloudreve/uploads    # 上传目录（预留）
      - ./conf:/cloudreve/conf          # 配置文件目录（备用）
      - ./data:/cloudreve/data          # 核心数据：数据库+配置+上传文件
      - ./avatar:/cloudreve/avatar      # 头像目录
    environment:
      - TZ=Asia/Shanghai
```

### 各目录作用

| 宿主机目录 | 容器内挂载点 | 用途 |
|-----------|-------------|------|
| `data/` | `/cloudreve/data/` | **核心数据** — 数据库(`.db`)、配置文件(`conf.ini`)、上传文件(`uploads/`) |
| `conf/` | `/cloudreve/conf/` | 配置文件备用目录（当前配置文件在 `data/` 中） |
| `uploads/` | `/cloudreve/uploads/` | 上传目录备用（当前上传文件存储在 `data/uploads/`） |
| `avatar/` | `/cloudreve/avatar/` | 用户头像 |

## 👤 游客直接下载（匿名访问）

默认情况下 Cloudreve 需要登录才能下载分享的文件。本项目已配置 **Anonymous 游客组** 支持免登录下载。

### 使用方式

1. 登录 Cloudreve → 找到文件 → **右键 → 分享**
2. 在分享设置中：
   - ✅ 开启 **允许下载**
   - ❌ 关闭 **提取码**（可选，免密码更方便）
3. 把生成的链接发给任何人，打开即可 **直接下载**，无需登录

### 技术原理

通过修改游客组的权限位（Boolset），启用了以下权限：

| 权限 | 说明 |
|------|------|
| `bit 1` | 匿名用户组标识 |
| `bit 7` | 允许下载他人分享的文件 |
| `bit 8` | 免费下载（不计入流量/积分） |

详细配置说明见 [docs/visitor-download.md](docs/visitor-download.md)。

## 🔧 常用命令

```bash
# 启动
docker-compose up -d

# 停止
docker-compose down

# 查看日志
docker-compose logs -f --tail 50

# 重启
docker-compose restart

# 更新镜像
docker-compose pull && docker-compose up -d
```

## 🔄 迁移到新机器

本项目的设计目标就是一键迁移：

```bash
# 新机器上执行
git clone https://github.com/MartaBanion/cloudreve.git
cd cloudreve
docker-compose up -d
```

所有数据（用户、文件记录、配置）都在 `data/` 目录中，随仓库一起同步。

## ⚠️ 注意事项

1. **端口修改**：如果 5212 端口被占用，编辑 `docker-compose.yml` 修改端口映射（如 `"5213:5212"`）
2. **站点 URL**：部署后登录管理后台修改"站点 URL"为实际访问地址，否则部分功能可能异常
3. **数据备份**：`data/cloudreve.db` 是 SQLite 数据库，建议定期备份
4. **安全建议**：生产环境建议配置 Nginx 反代 + HTTPS

## 📦 备份

```bash
# 备份整个数据目录
tar -czf cloudreve-backup-$(date +%Y%m%d).tar.gz data/
```

## 🔗 相关链接

- [Cloudreve 官方文档](https://docs.cloudreve.org/)
- [Cloudreve GitHub](https://github.com/cloudreve/Cloudreve)
