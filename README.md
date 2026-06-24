# Cloudreve 私有云盘

基于 Docker 部署的 [Cloudreve](https://cloudreve.org/) 私有云盘服务，包含完整数据。

## 📋 环境要求

- 已安装 [Docker](https://docs.docker.com/engine/install/) >= 20.x
- 已安装 [Docker Compose](https://docs.docker.com/compose/install/) >= 2.x
- 至少 512MB 内存

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

| 账号 | 密码 |
|------|------|
| `admin@cloudreve.org` | 见下方说明 |
| `hmei79180@gmail.com` | 原密码 |

> 🔑 管理员密码如果是首次部署的默认密码，可以通过以下命令查看：
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
