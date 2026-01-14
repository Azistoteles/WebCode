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

# 克隆 feature_docker 分支
git clone -b feature_docker https://github.com/xuzeyu91/WebCode.git

# 进入项目目录
cd WebCode
```

### 2.2 项目结构
```
WebCode/
├── Dockerfile                  # Docker 镜像构建文件
├── docker-compose.yml          # Docker Compose 配置
├── .env.example                # 环境变量模板
├── deploy-docker.sh            # 快速部署脚本
├── DEPLOY_DOCKER.md            # 部署文档（本文件）
├── docker/
│   ├── docker-entrypoint.sh   # 容器启动脚本
│   └── codex-config.toml       # Codex 配置模板
├── skills/                     # CLI 技能文件目录
│   ├── codex/                  # Codex CLI 技能
│   └── claude/                 # Claude CLI 技能
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

## 五、准备配置文件和技能

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

### 5.3 准备技能文件（重要）

技能文件是 Claude 和 Codex CLI 的扩展功能，包含各种预定义的工作流和任务模板。

#### 5.3.1 创建技能目录
```bash
mkdir -p /data/webcode/WebCode/skills/codex
mkdir -p /data/webcode/WebCode/skills/claude
```

#### 5.3.2 复制技能文件
```bash
# 从原始服务复制技能文件
cp -r /data/www/.codex/skills/* /data/webcode/WebCode/skills/codex/
cp -r /data/www/.claude/skills/* /data/webcode/WebCode/skills/claude/
```

#### 5.3.3 技能列表

**Codex 技能** (20个):
```
algorithmic-art           # 算法艺术生成
brand-guidelines         # 品牌指南处理
canvas-design           # Canvas 设计工具
distributed-task-orchestrator  # 分布式任务编排
doc-coauthoring         # 文档协作
docx                    # DOCX 文件处理
frontend-design         # 前端设计
internal-comms          # 内部通信
mcp-builder             # MCP 构建器
ms-agent-framework-rag  # MS Agent Framework RAG
office-to-md            # Office 转 Markdown
pdf                     # PDF 处理
planning-with-files     # 文件规划
pptx                    # PPTX 处理
skill-creator           # 技能创建器
slack-gif-creator       # Slack GIF 创建器
theme-factory           # 主题工厂
web-artifacts-builder   # Web 构建器
webapp-testing          # Web 应用测试
xlsx                    # Excel 处理
```

**Claude 技能** (18个):
```
algorithmic-art         # 算法艺术生成
brand-guidelines       # 品牌指南处理
canvas-design         # Canvas 设计工具
doc-coauthoring       # 文档协作
docx                  # DOCX 文件处理
frontend-design       # 前端设计
internal-comms        # 内部通信
mcp-builder           # MCP 构建器
office-to-md          # Office 转 Markdown
pdf                   # PDF 处理
planning-with-files   # 文件规划
pptx                  # PPTX 处理
skill-creator         # 技能创建器
slack-gif-creator     # Slack GIF 创建器
theme-factory         # 主题工厂
web-artifacts-builder # Web 构建器
webapp-testing        # Web 应用测试
xlsx                  # Excel 处理
```

#### 5.3.4 验证技能文件
```bash
# 检查 Codex 技能
ls /data/webcode/WebCode/skills/codex/

# 检查 Claude 技能
ls /data/webcode/WebCode/skills/claude/

# 应该分别看到 20 个和 18 个技能目录
```

---

## 六、停止原有服务（如有）

### 6.1 停止 systemd 服务
```bash
# 停止服务
systemctl stop webcode.service

# 禁用开机自启
systemctl disable webcode.service

# 验证状态
systemctl status webcode.service
```

### 6.2 停止旧容器（如有）
```bash
# 查看运行中的容器
docker ps -a

# 停止并删除旧容器
docker rm -f webcodecli
```

---

## 七、启动容器

### 7.1 创建工作区目录
```bash
mkdir -p /data/webcode/workspace
```

### 7.2 启动容器（完整挂载，包含技能文件）
```bash
cd /data/webcode/WebCode

docker run -d \
  --name webcodecli \
  --restart unless-stopped \
  --network=host \
  --env-file .env \
  -v /data/webcode/WebCode/appsettings.json:/app/appsettings.json \
  -v /data/webcode/workspace:/webcode/workspace \
  -v /data/webcode/WebCode/skills/codex:/root/.codex/skills \
  -v /data/webcode/WebCode/skills/claude:/root/.claude/skills \
  -v webcodecli-data:/app/data \
  -v webcodecli-workspaces:/app/workspaces \
  -v webcodecli-logs:/app/logs \
  webcodecli:latest
```

### 7.3 启动参数说明

| 参数 | 说明 |
|------|------|
| `-d` | 后台运行 |
| `--name webcodecli` | 容器名称 |
| `--restart unless-stopped` | 重启策略 |
| `--network=host` | 使用主机网络 |
| `--env-file .env` | 环境变量文件 |
| `-v ...:...` | 挂载目录/文件 |

### 7.4 挂载说明

| 宿主机路径 | 容器路径 | 说明 |
|------------|----------|------|
| `/data/webcode/WebCode/appsettings.json` | `/app/appsettings.json` | 配置文件 |
| `/data/webcode/workspace` | `/webcode/workspace` | 工作区目录 |
| `/data/webcode/WebCode/skills/codex` | `/root/.codex/skills` | Codex 技能 |
| `/data/webcode/WebCode/skills/claude` | `/root/.claude/skills` | Claude 技能 |
| `webcodecli-data` | `/app/data` | 数据卷 |
| `webcodecli-workspaces` | `/app/workspaces` | 工作区卷 |
| `webcodecli-logs` | `/app/logs` | 日志卷 |

---

## 八、验证部署

### 8.1 检查容器状态
```bash
docker ps | grep webcodecli
```

预期输出:
```
e7aba0101017   webcodecli:latest   "/docker-entrypoint.…"   Up X seconds (healthy)   webcodecli
```

### 8.2 查看容器日志
```bash
docker logs --tail 50 webcodecli
```

### 8.3 验证技能挂载
```bash
# 检查 Codex 技能数量
docker exec webcodecli ls /root/.codex/skills/ | wc -l

# 检查 Claude 技能数量
docker exec webcodecli ls /root/.claude/skills/ | wc -l

# 列出所有 Codex 技能
docker exec webcodecli ls /root/.codex/skills/

# 列出所有 Claude 技能
docker exec webcodecli ls /root/.claude/skills/
```

预期输出:
- Codex: 20 个技能
- Claude: 18 个技能

### 8.4 检查健康状态
```bash
# 检查健康端点
curl http://localhost:5000/health

# 查看容器健康检查
docker inspect webcodecli | grep -A 5 Health
```

---

## 九、日常维护

### 9.1 修改配置
```bash
# 1. 编辑配置文件
vi /data/webcode/WebCode/appsettings.json

# 2. 重启容器使配置生效
docker restart webcodecli

# 3. 查看日志确认
docker logs --tail 20 webcodecli
```

### 9.2 管理技能文件

#### 添加新技能
```bash
# 1. 将新技能复制到相应目录
cp -r /path/to/new-skill /data/webcode/WebCode/skills/codex/

# 2. 重启容器
docker restart webcodecli

# 3. 验证技能已加载
docker exec webcodecli ls /root/.codex/skills/ | grep new-skill
```

#### 更新现有技能
```bash
# 1. 编辑技能文件
vi /data/webcode/WebCode/skills/codex/skill-name/SKILL.md

# 2. 重启容器
docker restart webcodecli
```

#### 删除技能
```bash
# 1. 删除技能目录
rm -rf /data/webcode/WebCode/skills/codex/skill-name

# 2. 重启容器
docker restart webcodecli
```

#### 从容器导出技能
```bash
# 如果容器内有新技能，导出到宿主机
docker cp webcodecli:/root/.codex/skills/skill-name /data/webcode/WebCode/skills/codex/
```

### 9.3 查看日志
```bash
# 实时查看日志
docker logs -f webcodecli

# 查看最近 100 行
docker logs --tail 100 webcodecli
```

### 9.4 容器管理
```bash
# 重启容器
docker restart webcodecli

# 停止容器
docker stop webcodecli

# 启动容器
docker start webcodecli
```

### 9.5 更新镜像
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
  -v /data/webcode/WebCode/skills/codex:/root/.codex/skills \
  -v /data/webcode/WebCode/skills/claude:/root/.claude/skills \
  -v webcodecli-data:/app/data \
  -v webcodecli-workspaces:/app/workspaces \
  -v webcodecli-logs:/app/logs \
  webcodecli:latest
```

---

## 十、故障排查

### 10.1 容器无法启动
```bash
# 查看详细日志
docker logs webcodecli

# 检查配置文件
cat /data/webcode/WebCode/appsettings.json

# 检查环境变量
cat /data/webcode/WebCode/.env
```

### 10.2 技能未加载
```bash
# 检查技能目录
ls -la /data/webcode/WebCode/skills/codex/
ls -la /data/webcode/WebCode/skills/claude/

# 检查容器内技能
docker exec webcodecli ls /root/.codex/skills/
docker exec webcodecli ls /root/.claude/skills/

# 确认挂载点
docker inspect webcodecli | grep -A 10 Mounts
```

### 10.3 网络问题
```bash
# 检查端口是否被占用
netstat -tlnp | grep 5000

# 使用 host 网络模式（推荐）
docker run --network=host ...
```

### 10.4 权限问题
```bash
# 检查目录权限
ls -la /data/webcode/

# 修改权限
chmod -R 755 /data/webcode/workspace
chmod -R 755 /data/webcode/WebCode/skills
```

---

## 十一、快速部署脚本

### 11.1 一键部署脚本
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
mkdir -p /data/webcode/WebCode/skills/codex
mkdir -p /data/webcode/WebCode/skills/claude

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

# 复制技能文件（如果存在）
if [ -d "/data/www/.codex/skills" ]; then
    echo "复制 Codex 技能文件..."
    cp -r /data/www/.codex/skills/* /data/webcode/WebCode/skills/codex/
fi

if [ -d "/data/www/.claude/skills" ]; then
    echo "复制 Claude 技能文件..."
    cp -r /data/www/.claude/skills/* /data/webcode/WebCode/skills/claude/
fi

# 构建镜像
echo "构建镜像..."
docker build --network=host -t webcodecli:latest .

# 复制配置文件
if [ -f "/webcode/app/appsettings.json" ] && [ ! -f appsettings.json ]; then
    echo "复制配置文件..."
    cp /webcode/app/appsettings.json /data/webcode/WebCode/appsettings.json
fi

# 启动容器
echo "启动容器..."
docker run -d \
  --name webcodecli \
  --restart unless-stopped \
  --network=host \
  --env-file .env \
  -v /data/webcode/WebCode/appsettings.json:/app/appsettings.json \
  -v /data/webcode/workspace:/webcode/workspace \
  -v /data/webcode/WebCode/skills/codex:/root/.codex/skills \
  -v /data/webcode/WebCode/skills/claude:/root/.claude/skills \
  -v webcodecli-data:/app/data \
  -v webcodecli-workspaces:/app/workspaces \
  -v webcodecli-logs:/app/logs \
  webcodecli:latest

echo "=========================================="
echo "部署完成！"
echo "=========================================="
docker ps | grep webcodecli

echo ""
echo "验证技能挂载:"
echo "Codex 技能: $(docker exec webcodecli ls /root/.codex/skills/ 2>/dev/null | wc -l) 个"
echo "Claude 技能: $(docker exec webcodecli ls /root/.claude/skills/ 2>/dev/null | wc -l) 个"
```

### 11.2 使用脚本
```bash
chmod +x /data/webcode/WebCode/deploy-docker.sh
/data/webcode/WebCode/deploy-docker.sh
```

---

## 十二、系统服务配置（可选）

### 12.1 创建 systemd 服务
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

### 12.2 启用服务
```bash
# 重载 systemd
systemctl daemon-reload

# 启用服务
systemctl enable webcode-docker.service

# 启动服务
systemctl start webcode-docker.service
```

---

## 十三、推送镜像到阿里云

### 13.1 登录阿里云容器镜像服务
```bash
docker login --username=your_alias registry.cn-hangzhou.aliyuncs.com
```

### 13.2 打标签并推送
```bash
# 获取镜像 ID
docker images | grep webcodecli

# 打标签
docker tag [ImageId] registry.cn-hangzhou.aliyuncs.com/tree456/webcode:[镜像版本号]

# 示例
docker tag d3747c95c2c2 registry.cn-hangzhou.aliyuncs.com/tree456/webcode:1.0.0

# 推送镜像
docker push registry.cn-hangzhou.aliyuncs.com/tree456/webcode:1.0.0
```

### 13.3 使用阿里云镜像部署
```bash
# 拉取镜像
docker pull registry.cn-hangzhou.aliyuncs.com/tree456/webcode:1.0.0

# 运行容器
docker run -d \
  --name webcodecli \
  --restart unless-stopped \
  --network=host \
  --env-file .env \
  -v /data/webcode/WebCode/appsettings.json:/app/appsettings.json \
  -v /data/webcode/workspace:/webcode/workspace \
  -v /data/webcode/WebCode/skills/codex:/root/.codex/skills \
  -v /data/webcode/WebCode/skills/claude:/root/.claude/skills \
  -v webcodecli-data:/app/data \
  -v webcodecli-workspaces:/app/workspaces \
  -v webcodecli-logs:/app/logs \
  registry.cn-hangzhou.aliyuncs.com/tree456/webcode:1.0.0
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

# 备份技能文件
tar czf /backup/webcodecli-skills-$(date +%Y%m%d).tar.gz -C /data/webcode/WebCode skills/
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

# 恢复技能文件
tar xzf /backup/webcodecli-skills-20250114.tar.gz -C /data/webcode/WebCode

# 重启容器
docker restart webcodecli
```

### C. 技能文件结构示例

```
skills/
├── codex/
│   ├── algorithmic-art/
│   │   ├── SKILL.md
│   │   ├── examples.md
│   │   └── reference.md
│   ├── pdf/
│   │   ├── SKILL.md
│   │   ├── examples.md
│   │   └── reference.md
│   └── ...
└── claude/
    ├── planning-with-files/
    │   ├── SKILL.md
    │   ├── examples.md
    │   └── reference.md
    └── ...
```

### D. 常用命令速查

```bash
# 查看容器状态
docker ps | grep webcodecli

# 查看容器日志
docker logs -f webcodecli

# 进入容器
docker exec -it webcodecli bash

# 查看技能列表
docker exec webcodecli ls /root/.codex/skills/
docker exec webcodecli ls /root/.claude/skills/

# 重启容器
docker restart webcodecli

# 查看容器详细信息
docker inspect webcodecli

# 查看容器资源使用
docker stats webcodecli
```

---

**文档版本**: 2.0
**更新日期**: 2026-01-14
**维护者**: WebCode Team

### 更新日志
- v2.0 (2026-01-14): 添加技能文件挂载说明，更新部署脚本
- v1.0 (2026-01-14): 初始版本
