# 游客下载功能配置指南

允许未登录的匿名用户直接下载 Cloudreve 分享的文件。

## 概述

默认情况下，Cloudreve 要求接收方登录后才能下载分享的文件。通过配置 **Anonymous（游客）** 用户组的权限，访客可以直接打开分享链接下载文件，无需任何登录操作。

## 原理

Cloudreve 使用 **Boolset**（基于位掩码的权限编码）来控制用户组的权限。每个位（bit）代表一个权限开关。游客组的权限决定了未登录访客能做什么。

## 权限位说明

**用户组权限**的位定义如下：

| 位 | 说明 | 是否必须 |
|----|------|---------|
| 0  | 是否为管理员 | ❌ |
| 1  | 是否为匿名用户组 | ✅（游客组必须开启） |
| 2  | 是否可以分享文件 | ❌ |
| 3  | 是否可以访问 WebDAV | ❌ |
| 4  | 是否可以服务端打包下载 | 可选 |
| 5  | 是否可以执行归档压缩任务 | 可选 |
| 6  | 是否可以开启 WebDAV 代理 | ❌ |
| 7  | **是否可以下载他人分享** | ✅ **必须开启** |
| 8  | **是否可以免费下载分享** | ✅ **推荐开启**（跳过积分/流量扣费） |
| 9  | 是否可以离线下载 | ❌ |
| 10 | 是否可以转移存储策略 | ❌ |
| 11 | 是否使用重定向直链 | 可选 |

## 数据库配置

执行以下 SQL 为游客组设置合适的权限：

```sql
-- 开启：匿名标识(bit 1) + 下载分享(bit 7) + 免费下载(bit 8)
-- 字节 0: 0x82（位 1 和 7）
-- 字节 1: 0x01（位 8）
UPDATE groups
SET permissions = x'8201',
    updated_at  = datetime('now')
WHERE id = 3;
```

翻译成二进制：
- `0x82` → `10000010` → 位 1（匿名组）+ 位 7（下载分享）已开启
- `0x01` → `00000001` → 位 8（免费下载）已开启

## 站点 URL 配置

Cloudreve 需要知道自己的公网地址，才能生成正确的分享链接。

### 通过数据库修改

```sql
UPDATE settings
SET value = 'http://你的公网IP:5212',
    updated_at = datetime('now')
WHERE name = 'siteURL';
```

### 通过配置文件修改（`data/conf.ini`）

```ini
[System]
SiteURL = http://你的公网IP:5212
```

> ⚠️ 如果 `SiteURL` 设为 `localhost`，生成的分享链接他人无法访问。

## 用户操作步骤

权限和站点 URL 配置完成后：

1. 登录 Cloudreve → 上传文件
2. **右键**文件 → **分享**
3. 在分享对话框中：
   - ✅ 开启 **允许下载**
   - ❌ 关闭 **提取码**（免密码直接访问）
   - 设置过期时间（按需）
4. 复制分享链接

**分享链接示例：**
```
http://8.160.178.239:5212/s/AbCdEf
```
打开这个链接即可直接预览和下载文件，无需登录。

## 验证方法

1. 在**无痕/隐私浏览器窗口**中打开分享链接
2. 应该能看到文件预览页面
3. 点击下载按钮——文件应直接下载，不会弹出登录提示
4. 运行以下 SQL 检查权限：

```sql
SELECT id, name, hex(permissions) FROM groups;
```

预期输出：

```
1|Admin|FD1A01
2|User|8408
3|Anonymous|8201
```

## 安全建议

### 防止链接被滥用

公开分享链接意味着任何知道链接的人都能访问。建议：

1. **设置过期时间** — 分享时设定合理的过期时间（如 24 小时或 7 天）
2. **限制下载次数** — 可设置"剩余下载次数"，达到次数后自动失效
3. **启用提取码** — 如果分享敏感文件，开启提取码，只有知晓密码的人能打开
4. **定期清理** — 不再需要分享时，在 Cloudreve 中删除对应分享记录

### 服务器防护

```bash
# 1. 配置防火墙，只放行必要端口
sudo firewall-cmd --add-port=5212/tcp --permanent
sudo firewall-cmd --reload

# 2. 配置 Nginx 反代 + HTTPS（详见 README）
# 3. 监控访问日志，发现异常 IP 及时封禁
```

## 注意事项

- 修改游客组权限后通常**立即生效**，无需重启容器（但重启更保险）
- **之前生成的分享链接**可能不会自动获得新权限——建议重新生成
- 确保服务器防火墙开放了 Cloudreve 端口（默认 `5212`）
- 如果站点开启了"下载扣积分"功能，必须开启位 8（免费下载），否则游客会报错

## 排错指南

<details>
<summary><b>🔗 分享链接显示 localhost</b></summary>

**原因：** `siteURL` 仍指向 `localhost`。

**解决方法：**
```sql
UPDATE settings
SET value = 'http://你的实际IP:5212',
    updated_at = datetime('now')
WHERE name = 'siteURL';
```

然后重启容器：
```bash
docker compose restart
```
</details>

<details>
<summary><b>🚫 分享页只能预览，没有下载按钮</b></summary>

**可能原因及解决方法：**

1. **分享时没开启下载** — 重新分享文件，勾选 ✅ **允许下载**

2. **数据库权限未生效** — 检查游客组权限：
   ```sql
   SELECT id, name, hex(permissions) FROM groups WHERE id = 3;
   ```
   预期结果：`3|Anonymous|8201`
   
   如果不是，执行修复：
   ```sql
   UPDATE groups SET permissions = x'8201', updated_at = datetime('now') WHERE id = 3;
   ```

3. **需要重启容器**：
   ```bash
   docker compose restart
   ```
</details>

<details>
<summary><b>🔒 访客打开链接要求输入密码</b></summary>

**原因：** 创建分享时开启了**提取码**。

**解决方法：** 重新生成分享链接，关闭提取码开关。
</details>

<details>
<summary><b>❌ 分享链接 404 或"分享不存在"</b></summary>

**可能原因：**

1. **链接已过期** — 重新创建分享，设置更长的有效期或不限时
2. **数据库被重置** — 旧分享链接全部失效，需重新创建
3. **源文件被删除** — 分享的文件已不存在
</details>

<details>
<summary><b>🔑 提示"管理员限制下载"</b></summary>

**原因：** 游客组缺少"免费下载"权限（位 8），而站点开启了积分扣费。

**解决方法：** 为游客组开启位 8：
```sql
UPDATE groups SET permissions = x'8201', updated_at = datetime('now') WHERE id = 3;
```
</details>

<details>
<summary><b>🌐 外网无法访问服务器</b></summary>

**按顺序检查以下项目：**

1. **服务器防火墙**：
   ```bash
   ss -tlnp | grep 5212
   # CentOS/RHEL 放行端口
   sudo firewall-cmd --add-port=5212/tcp --permanent
   sudo firewall-cmd --reload
   ```

2. **云服务商安全组** — 登录阿里云/腾讯云/AWS 等控制台，在安全组入方向放行端口 5212

3. **Docker 端口映射**：
   ```bash
   docker ps | grep cloudreve
   # 应显示：0.0.0.0:5212->5212/tcp
   ```

4. **Cloudreve 监听地址** — 检查 `data/conf.ini`：
   ```ini
   [System]
   Listen = :5212    # 必须是 :5212，不是 127.0.0.1:5212
   ```
</details>
