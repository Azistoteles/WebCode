# WebCodeCli Docker 部署文档

## 概述

本文档详细说明如何从 GitHub 拉取代码、构建 Docker 镜像、并部署运行 WebCodeCli 服务。

---

## 一、环境准备

### 1.1 系统要求
- Docker 已安装
- Git 已安装
- 端口 5000 可用

### 1.2 检查环境
```bash
# 检查 Docker
docker --version

# 检查 Git
git --version
```

---

## 二、拉取代码

### 2.1 克隆仓库
```bash
# 进入工作目录
cd /data/webcode

git clone https://github.com/xuzeyu91/WebCode.git

# 进入项目目录
cd WebCode
```

### 2.2 项目结构
```
WebCode/
├── Dockerfile                  # Docker 镜像构建文件
├── docker-compose.yml          # Docker Compose 配置
├── .env.example                # 环境变量模板
├── docker/
│   ├── docker-entrypoint.sh   # 容器启动脚本
│   └── codex-config.toml       # Codex 配置模板
├── WebCodeCli/                 # 主项目
└── WebCodeCli.Domain/          # 领域项目
```

---

## 三、配置环境变量

### 3.1 创建 .env 文件
```bash
cd /data/webcode/WebCode

# 复制环境变量模板
cp .env.example .env

# 编辑环境变量
vi .env
```

### 3.2 环境变量配置示例
```bash
# ============================================
# 应用端口
# ============================================
APP_PORT=5000

# ============================================
# Claude Code 配置
# ============================================
ANTHROPIC_BASE_URL=https://api.antsk.cn/
ANTHROPIC_AUTH_TOKEN=your_token_here
ANTHROPIC_MODEL=glm-4.7
ANTHROPIC_SMALL_FAST_MODEL=glm-4.7

# ============================================
# Codex 配置
# ============================================
NEW_API_KEY=your_api_key_here
CODEX_MODEL=glm-4.7
CODEX_MODEL_REASONING_EFFORT=medium
CODEX_PROFILE=ipsa
CODEX_BASE_URL=https://api.antsk.cn/v1
CODEX_PROVIDER_NAME=azure codex-mini
CODEX_APPROVAL_POLICY=never
CODEX_SANDBOX_MODE=danger-full-access

# ============================================
# 数据库配置
# ============================================
DB_TYPE=Sqlite
DB_CONNECTION=Data Source=/app/data/webcodecli.db
```

---

## 四、构建 Docker 镜像

### 4.1 执行构建
```bash
cd /data/webcode/WebCode

# 构建镜像（使用 host 网络模式避免网络问题）
docker build --network=host -t webcodecli:latest .
```

### 4.2 构建过程说明
构建过程包含以下阶段：

#### 阶段 1: 构建阶段 (build)
- 基础镜像: `mcr.microsoft.com/dotnet/sdk:10.0`
- 安装 Node.js 20.x
- 还原 NuGet 包
- 构建 TailwindCSS
- 编译 .NET 应用

#### 阶段 2: 运行时镜像 (final)
- 基础镜像: `mcr.microsoft.com/dotnet/aspnet:10.0`
- 安装基础依赖: curl, wget, git, python3 等
- 安装 Node.js 20.x
- 安装 Rust (Codex 需要)
- 安装 Claude Code CLI: `@anthropic-ai/claude-code`
- 安装 Codex CLI: `@openai/codex`
- 配置 Codex
- 创建必要目录
- 复制应用文件

### 4.3 验证镜像
```bash
docker images webcodecli
```

预期输出:
```
REPOSITORY   TAG       IMAGE ID       CREATED          SIZE
webcodecli   latest    d3747c95c2c2   17 seconds ago   2.78GB
```

---

## 五、准备配置文件

### 5.1 复制现有配置（如果有）
```bash
# 如果有旧服务配置，复制过来
cp /webcode/app/appsettings.json /data/webcode/WebCode/appsettings.json
```

### 5.2 配置文件结构
```json
{
    "Logging": {
        "LogLevel": {
            "Default": "Information",
            "Microsoft.AspNetCore": "Warning"
        }
    },
    "AllowedHosts": "*",
    "urls": "http://*:5000",
    "Authentication": {
        "Enabled": true,
        "Users": [
            {
                "Username": "your_username",
                "Password": "your_password"
            }
        ]
    },
    "DBConnection": {
        "DbType": "Sqlite",
        "ConnectionStrings": "Data Source=WebCodeCli.db",
        "VectorConnection": "WebCodeCliMem.db",
        "VectorSize": 1536
    },
    "CliTools": {
        "MaxConcurrentExecutions": 3,
        "DefaultTimeoutSeconds": 300,
        "EnableCommandWhitelist": true,
        "TempWorkspaceRoot": "/webcode/workspace/",
        "WorkspaceExpirationHours": 24,
        "NpmGlobalPath": "/usr/bin/npm/",
        "Tools": [...]
    }
}
```

---

## 六、启动容器

### 6.1 创建工作区目录
```bash
mkdir -p /data/webcode/workspace
```

### 6.2 启动容器（完整挂载）
```bash
cd /data/webcode/WebCode

docker run -d \
  --name webcodecli \
  --restart unless-stopped \
  --network=host \
  --env-file .env \
  -v /data/webcode/WebCode/appsettings.json:/app/appsettings.json \
  -v /data/webcode/workspace:/webcode/workspace \
  -v webcodecli-data:/app/data \
  -v webcodecli-workspaces:/app/workspaces \
  -v webcodecli-logs:/app/logs \
  webcodecli:latest
```

### 6.3 启动参数说明

| 参数 | 说明 |
|------|------|
| `-d` | 后台运行 |
| `--name webcodecli` | 容器名称 |
| `--restart unless-stopped` | 重启策略 |
| `--network=host` | 使用主机网络 |
| `--env-file .env` | 环境变量文件 |
| `-v ...:...` | 挂载目录/文件 |

### 6.4 挂载说明

| 宿主机路径 | 容器路径 | 说明 |
|------------|----------|------|
| `/data/webcode/WebCode/appsettings.json` | `/app/appsettings.json` | 配置文件 |
| `/data/webcode/workspace` | `/webcode/workspace` | 工作区目录 |
| `webcodecli-data` | `/app/data` | 数据卷 |
| `webcodecli-workspaces` | `/app/workspaces` | 工作区卷 |
| `webcodecli-logs` | `/app/logs` | 日志卷 |

---

## 七、验证部署

### 7.1 检查容器状态
```bash
docker ps | grep webcodecli
```

预期输出:
```
38aad40ecc0b   webcodecli:latest   "/docker-entrypoint.…"   Up X seconds (healthy)   webcodecli
```

### 7.2 查看容器日志
```bash
docker logs --tail 50 webcodecli
```

### 7.3 检查健康状态
```bash
# 检查健康端点
curl http://localhost:5000/health

# 查看容器健康检查
docker inspect webcodecli | grep -A 5 Health
```

---

## 八、日常维护

### 8.1 修改配置
```bash
# 1. 编辑配置文件
vi /data/webcode/WebCode/appsettings.json

# 2. 重启容器使配置生效
docker restart webcodecli

# 3. 查看日志确认
docker logs --tail 20 webcodecli
```

### 8.2 查看日志
```bash
# 实时查看日志
docker logs -f webcodecli

# 查看最近 100 行
docker logs --tail 100 webcodecli
```

### 8.3 容器管理
```bash
# 重启容器
docker restart webcodecli

# 停止容器
docker stop webcodecli

# 启动容器
docker start webcodecli
```

### 8.4 更新镜像
```bash
# 1. 拉取最新代码
cd /data/webcode/WebCode
git pull origin feature_docker

# 2. 重新构建镜像
docker build --network=host -t webcodecli:latest .

# 3. 重启容器
docker rm -f webcodecli
docker run -d \
  --name webcodecli \
  --restart unless-stopped \
  --network=host \
  --env-file .env \
  -v /data/webcode/WebCode/appsettings.json:/app/appsettings.json \
  -v /data/webcode/workspace:/webcode/workspace \
  -v webcodecli-data:/app/data \
  -v webcodecli-workspaces:/app/workspaces \
  -v webcodecli-logs:/app/logs \
  webcodecli:latest
```

---

## 九、故障排查

### 9.1 容器无法启动
```bash
# 查看详细日志
docker logs webcodecli

# 检查配置文件
cat /data/webcode/WebCode/appsettings.json

# 检查环境变量
cat /data/webcode/WebCode/.env
```

### 9.2 网络问题
```bash
# 检查端口是否被占用
netstat -tlnp | grep 5000

# 使用 host 网络模式（推荐）
docker run --network=host ...
```

### 9.3 权限问题
```bash
# 检查目录权限
ls -la /data/webcode/

# 修改权限
chmod -R 755 /data/webcode/workspace
```

---

## 十、快速部署脚本

### 10.1 一键部署脚本
保存为 `/data/webcode/WebCode/deploy-docker.sh`:

```bash
#!/bin/bash
set -e

echo "=========================================="
echo "WebCodeCli Docker 部署脚本"
echo "=========================================="

# 停止旧服务
echo "停止旧服务..."
systemctl stop webcode.service 2>/dev/null || true
systemctl disable webcode.service 2>/dev/null || true
docker rm -f webcodecli 2>/dev/null || true

# 创建目录
echo "创建目录..."
mkdir -p /data/webcode/workspace
mkdir -p /data/webcode/WebCode

# 拉取代码
echo "拉取代码..."
cd /data/webcode
if [ -d "WebCode" ]; then
    cd WebCode
    git pull origin feature_docker
else
    git clone -b feature_docker https://github.com/xuzeyu91/WebCode.git
    cd WebCode
fi

# 配置环境变量
if [ ! -f .env ]; then
    echo "请配置 .env 文件"
    cp .env.example .env
    exit 1
fi

# 构建镜像
echo "构建镜像..."
docker build --network=host -t webcodecli:latest .

# 启动容器
echo "启动容器..."
docker run -d \
  --name webcodecli \
  --restart unless-stopped \
  --network=host \
  --env-file .env \
  -v /data/webcode/WebCode/appsettings.json:/app/appsettings.json \
  -v /data/webcode/workspace:/webcode/workspace \
  -v webcodecli-data:/app/data \
  -v webcodecli-workspaces:/app/workspaces \
  -v webcodecli-logs:/app/logs \
  webcodecli:latest

echo "=========================================="
echo "部署完成！"
echo "=========================================="
docker ps | grep webcodecli
```

### 10.2 使用脚本
```bash
chmod +x /data/webcode/WebCode/deploy-docker.sh
/data/webcode/WebCode/deploy-docker.sh
```

---

## 十一、系统服务配置（可选）

### 11.1 创建 systemd 服务
创建 `/etc/systemd/system/webcode-docker.service`:

```ini
[Unit]
Description=WebCodeCli Docker Container
After=docker.service
Requires=docker.service

[Service]
Type=oneshot
RemainAfterExit=yes
WorkingDirectory=/data/webcode/WebCode
ExecStart=/usr/bin/docker start webcodecli
ExecStop=/usr/bin/docker stop webcodecli
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

### 11.2 启用服务
```bash
# 重载 systemd
systemctl daemon-reload

# 启用服务
systemctl enable webcode-docker.service

# 启动服务
systemctl start webcode-docker.service
```

---

## 附录

### A. Docker Compose 方式

```bash
cd /data/webcode/WebCode

# 启动
docker-compose up -d

# 停止
docker-compose down

# 重启
docker-compose restart
```

### B. 备份与恢复

#### 备份
```bash
# 备份数据卷
docker run --rm \
  -v webcodecli-data:/data \
  -v /backup:/backup \
  alpine tar czf /backup/webcodecli-data-$(date +%Y%m%d).tar.gz /data

# 备份配置
cp /data/webcode/WebCode/appsettings.json /backup/appsettings.json-$(date +%Y%m%d)
```

#### 恢复
```bash
# 恢复数据卷
docker run --rm \
  -v webcodecli-data:/data \
  -v /backup:/backup \
  alpine tar xzf /backup/webcodecli-data-20250114.tar.gz -C /

# 恢复配置
cp /backup/appsettings.json-20250114 /data/webcode/WebCode/appsettings.json
docker restart webcodecli
```

---

**文档版本**: 1.0
**更新日期**: 2026-01-14
**维护者**: WebCode Team
