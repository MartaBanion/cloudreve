# Cloudreve 部署项目

基于 Docker 部署的 [Cloudreve](https://cloudreve.org/) 私有云盘服务。

## 📋 环境要求

- 服务器（Linux / Windows / macOS 均可）
- [Docker](https://docs.docker.com/engine/install/) >= 20.x
- [Docker Compose](https://docs.docker.com/compose/install/) >= 2.x
- 至少 512MB 内存

## 🚀 快速开始

### 1. 克隆项目

```bash
git clone https://github.com/MartaBanion/cloudreve.git
cd cloudreve
```

### 2. 创建必要目录

```bash
mkdir -p conf data uploads avatar
```

### 3. 生成配置文件

创建 `conf/conf.ini`（你也可以直接使用 `data/conf.ini` 中已有的配置）：

```bash
cat > conf/conf.ini << 'EOF'
[System]
; 调试模式
Debug = false
; 运行模式（master / slave）
Mode = master
; 监听端口
Listen = :5212
; Session 密钥（请修改为随机字符串）
SessionSecret = 你的随机字符串密钥
; HashID 盐值（请修改为随机字符串）
HashIDSalt = 你的随机盐值
EOF
```

> ⚠️ **注意：** 请将 `SessionSecret` 和 `HashIDSalt` 替换为随机字符串，可以使用以下命令生成：
> ```bash
> openssl rand -base64 32
> ```

### 4. （可选）导入已有数据

如果你已经有之前的数据库文件，将以下文件放入对应目录：

| 文件 | 放到 |
|------|------|
| `cloudreve.db` | `data/` |
| `conf.ini` | `data/` 或 `conf/` |

### 5. 启动服务

```bash
docker-compose up -d
```

### 6. 访问服务

打开浏览器访问：**http://你的服务器IP:5212**

默认管理员账号（首次启动时在 Docker 日志中查看）：

```bash
docker-compose logs cloudreve | grep "初始管理员"
```

会输出类似这样的信息：
```
初始管理员账号：admin@cloudreve.org
初始管理员密码：xxxxxxxxxx
```

## 🐳 Docker Compose 配置说明

```yaml
services:
  cloudreve:
    image: cloudreve/cloudreve:latest   # 镜像来源
    container_name: cloudreve           # 容器名称
    restart: unless-stopped             # 自动重启策略
    ports:
      - "5212:5212"                     # 端口映射（宿主机:容器）
    volumes:
      - ./uploads:/cloudreve/uploads    # 上传文件目录（未使用）
      - ./conf:/cloudreve/conf          # 配置文件目录（未使用）
      - ./data:/cloudreve/data          # 数据库 + 上传文件 + 配置文件
      - ./avatar:/cloudreve/avatar      # 头像目录
    environment:
      - TZ=Asia/Shanghai                # 时区
```

### 目录结构说明

```
cloudreve/
├── docker-compose.yml     # Docker 编排文件
├── conf/                  # 配置文件目录（可选）
├── data/
│   ├── cloudreve.db       # SQLite 数据库（用户、文件索引等）
│   ├── conf.ini           # 主配置文件
│   └── uploads/           # 上传文件存储目录
│       └── {uid}/         # 按用户 ID 分目录
├── uploads/               # 备用上传目录（未使用）
└── avatar/                # 用户头像
```

## 🔧 常用命令

```bash
# 启动服务
docker-compose up -d

# 停止服务
docker-compose down

# 查看日志
docker-compose logs -f --tail 50

# 重启服务
docker-compose restart

# 更新镜像
docker-compose pull
docker-compose up -d
```

## ⚠️ 注意事项

1. **备份数据库**：`data/cloudreve.db` 包含了所有用户和文件记录，请定期备份
2. **端口冲突**：如果 5212 端口被占用，修改 `docker-compose.yml` 中的端口映射（如 `"5213:5212"`）
3. **文件权限**：如果出现上传失败，检查 `data/uploads/` 目录的写入权限
4. **安全建议**：生产环境建议使用反向代理（如 Nginx）+ HTTPS 部署

## 📦 数据备份

```bash
# 备份整个数据目录
tar -czf cloudreve-backup-$(date +%Y%m%d).tar.gz data/
```

## 🔗 相关链接

- [Cloudreve 官方文档](https://docs.cloudreve.org/)
- [Cloudreve GitHub](https://github.com/cloudreve/Cloudreve)
