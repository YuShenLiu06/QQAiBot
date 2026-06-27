# QQAiBot 运维手册

基于 AstrBot + NapCat 的 QQ 群 AI 机器人，Docker 一键部署方案。

## 简介

QQAiBot 是一个运行在 QQ 群中的 AI 机器人，采用分层架构：

- **NapCat**：QQ 协议底层实现，基于 NTQQ 提供 OneBot11 协议接口，负责 QQ 消息收发
- **AstrBot**：AI 机器人平台，提供 Web 管理面板、插件系统、人格设定等功能
- **DeepSeek API**：LLM 模型服务，提供 AI 对话能力

整个系统通过 Docker Compose 编排，两个容器运行在同一桥接网络中，通过反向 WebSocket 通信。本机 macOS 部署无需任何 Python 依赖，所有运行时环境均已容器化。

## 前置要求

### 硬件与操作系统

- **处理器**：Apple Silicon (arm64) Mac
- **操作系统**：macOS 12.0 或更高版本

### 软件依赖

1. **Docker Desktop**（需 Rosetta 2 支持）

```bash
# 安装 Rosetta 2（arm64 Mac 首次使用 Docker 必需）
softwareupdate --install-rosetta --agree-to-license

# 完全退出并重新打开 Docker Desktop（如果显示 "Docker Desktop is unable to start"）
```

2. **验证 Docker 安装**

```bash
docker version
docker compose version
```

预期输出显示 Docker Engine 和 Docker Compose 版本信息。

## 快速开始（本机 macOS）

### 1. 进入项目目录

```bash
cd /Users/yushen/opt/QQAiBot
```

### 2. 创建环境配置文件

```bash
# 获取当前用户 UID 和 GID
id -u  # macOS 首个用户通常为 501
id -g  # macOS 首个用户通常为 20
```

创建 `.env` 文件：

```bash
cat > .env << 'EOF'
NAPCAT_UID=501
NAPCAT_GID=20
EOF
```

> **注意**：请将上述 UID 和 GID 替换为 `id` 命令的实际输出值。

### 3. 启动服务

```bash
# 首次启动会自动拉取镜像
docker compose up -d

# 查看服务状态
docker compose ps
```

预期输出显示两个服务均为 `Up` 状态：

```
NAME                IMAGE                          STATUS             PORTS
astrbot   soulter/astrbot:latest        Up                 0.0.0.0:6185->6185/tcp, 0.0.0.0:6199->6199/tcp
napcat    mlikiowa/napcat-docker:latest Up                 0.0.0.0:6099->6099/tcp
```

## 配置 NapCat（登录 QQ）

### 1. 获取 WebUI 登录 Token

```bash
docker compose logs napcat | grep -i token
```

在日志中查找类似 `WebUi Token: 7c2ffbd4bba3` 的字符串（12 位十六进制，每次部署不同）。

### 2. 打开 NapCat WebUI

在浏览器中访问：http://127.0.0.1:6099

### 3. 登录并扫码

1. 在登录页面输入上一步获取的 Token
2. 点击登录后，页面会显示 QQ 二维码
3. 使用手机 QQ 扫码登录机器人账号

> **重要**：`MODE=astrbot` 环境变量会自动配置反向 WebSocket 连接到 `ws://astrbot:6199/ws`。如果日志未显示自动连接成功，需在 NapCat WebUI 的"网络配置"中手动添加 WebSocket Client，目标 URL 为 `ws://astrbot:6199/ws`。

### 4. 验证连接

```bash
docker compose logs astrbot | grep -i "onebot"
```

成功连接后，日志会显示类似 `aiocqhttp(OneBot v11) 适配器已连接` 的信息。

## 配置 AstrBot（面板/模型/系统提示词）

### 1. 打开 AstrBot 管理面板

在浏览器中访问：http://127.0.0.1:6185

### 2. 首次登录

- 用户名：`astrbot`
- 密码：**随机生成**（不是 `astrbot`），从启动日志获取：

```bash
docker compose logs astrbot | grep -i "Initial password"
```

> 首次登录后立即修改密码。

> **安全提醒**：登录后立即修改密码（右上角头像 → 修改密码）。

### 3. 配置 QQ 机器人连接

1. 左侧菜单进入 **机器人** → **创建机器人**
2. 选择 **OneBot v11** 协议
3. 配置参数：
   - 启用机器人：开启
   - 反向 WebSocket 主机：`0.0.0.0`
   - 反向 WebSocket 端口：`6199`
4. 点击 **保存**

5. 返回 **控制台**，查看日志显示 `aiocqhttp(OneBot v11) 适配器已连接` 即表示成功。

### 4. 接入 DeepSeek 模型

1. 左侧菜单进入 **模型提供商** → **新增**
2. 选择 **DeepSeek**
3. 粘贴 DeepSeek API Key（从 [platform.deepseek.com](https://platform.deepseek.com) 获取）
4. 点击 **保存** → **测试**

测试通过后，模型即可正常使用。

### 5. 配置系统提示词（人格设定）

1. 左侧菜单进入 **更多功能** → **人格设定**
2. 点击 **创建人格**
3. 填写参数：
   - **人格 ID**：如 `猫娘`、`助手`、`程序员` 等（用于命令切换）
   - **系统提示词**：复制自 `persona/示例人设.md` 或自定义

示例提示词：

```
你是一只可爱的猫娘，喜欢在句尾加上"喵"。
你的性格活泼、友好，偶尔会调皮一下。
请用自然、口语化的方式回复。
```

4. 点击 **保存**

#### **关键步骤：设置默认人格**

AstrBot 存在已知问题（issue #7741），创建人格后必须手动绑定才会生效，否则机器人仍使用默认提示词。

**方法一：设置为默认人格**

1. 进入 **模型提供商** → 选择 DeepSeek → **provider_settings**
2. 找到 `default_personality` 字段，设置为人格 ID（如 `"猫娘"`）
3. 保存配置

**方法二：在聊天中动态切换**

在群聊或私聊中使用命令：

```
/persona 猫娘
/reset
```

然后 @机器人 进行测试，观察回复是否符合设定的人格特征。

> **验证方法**：@机器人说"自我介绍一下"，如果回复包含"喵"等特征词，说明人格已生效。

## 测试

### 1. 私聊测试

使用另一个 QQ 号添加机器人为好友，发送任意消息，期待 AI 回复。

### 2. 群聊测试

在任意 QQ 群中 @机器人，发送消息，期待 AI 回复。

### 3. 人格切换测试

在群中发送以下命令：

```
/persona 猫娘
/reset
```

然后 @机器人说"你好"，观察是否使用猫娘人格回复（如句尾带"喵"）。

### 4. 验证失败排查

如果机器人未回复或人格未生效：

1. **检查服务状态**：`docker compose ps`
2. **查看日志**：`docker compose logs -f astrbot`
3. **验证连接**：确保 AstrBot 控制台显示 OneBot 适配器已连接
4. **检查人格绑定**：确认 `default_personality` 已设置或使用了 `/persona` 命令

## Linux 服务器部署

### 1. 服务器环境准备

```bash
# 安装 Docker（Ubuntu/Debian）
curl -fsSL https://get.docker.com | bash

# 安装 Docker Compose（如果未包含）
sudo apt install docker-compose-plugin

# 验证安装
docker version
docker compose version
```

### 2. 防火墙配置

**安全策略**：仅对外暴露 AstrBot 管理面板端口 6185，**切勿**暴露 NapCat WebUI (6099) 和 OneBot 反向 WS 端口 (6199)。

```bash
# Ubuntu UFW 示例
sudo ufw allow 6185/tcp    # AstrBot WebUI
sudo ufw deny 6099/tcp     # NapCat WebUI（拒绝公网访问）
sudo ufw deny 6199/tcp     # OneBot 反向 WS（拒绝公网访问）
sudo ufw enable
```

### 3. 部署项目文件

```bash
# 在本地打包项目
tar czf qqaibot.tar.gz .env docker-compose.yml .gitignore persona/

# 上传到服务器
scp qqaibot.tar.gz user@your-server:/opt/QQAiBot/

# 在服务器上解压
ssh user@your-server
cd /opt/QQAiBot
tar xzf qqaibot.tar.gz
```

### 4. 调整环境配置

```bash
# 修改 .env 文件
cat > .env << 'EOF'
NAPCAT_UID=0    # 服务器通常使用 root 运行 Docker，或改为 1000
NAPCAT_GID=0    # 同上
EOF
```

### 5. 启动服务

```bash
docker compose up -d
docker compose ps
```

### 6. 配置服务

重复"配置 NapCat"和"配置 AstrBot"步骤，但需将 `127.0.0.1` 替换为服务器 IP 地址：

- NapCat WebUI：`http://服务器IP:6099`
- AstrBot 管理面板：`http://服务器IP:6185`

### 7. 公网访问安全加固（可选）

如需通过公网访问 AstrBot 管理面板，必须配置反向代理和双因素认证：

#### Nginx 反向代理配置

```nginx
server {
    listen 443 ssl http2;
    server_name your-domain.com;

    ssl_certificate /path/to/cert.pem;
    ssl_certificate_key /path/to/key.pem;

    location / {
        proxy_pass http://127.0.0.1:6185;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

#### 启用 AstrBot TOTP 双因素认证

1. 登录 AstrBot 管理面板
2. 进入 **WebUI 配置文件** → **系统配置**
3. 找到 **启用WebUI TOTP双因素认证**
4. 设置为 `true` 并保存
5. 重新登录时，扫描二维码绑定 TOTP 应用（Google Authenticator、Authy 等）

> **重要**：面板密码必须足够复杂，建议 16 位以上包含大小写字母、数字和符号。

## 常用运维命令

| 操作 | 命令 | 说明 |
|------|------|------|
| 启动服务 | `docker compose up -d` | 后台启动所有容器 |
| 查看状态 | `docker compose ps` | 检查容器运行状态 |
| 查看日志 | `docker compose logs -f` | 实时查看所有服务日志 |
| 查看单个日志 | `docker compose logs -f astrbot` | 实时查看 AstrBot 日志 |
| 停止服务 | `docker compose down` | 停止并删除容器 |
| 重启服务 | `docker compose restart` | 重启所有容器 |
| 更新镜像 | `docker compose pull && docker compose up -d` | 拉取最新镜像并重启 |
| 进入容器 | `docker compose exec astrbot sh` | 进入 AstrBot 容器 Shell |
| 查看资源占用 | `docker stats` | 查看容器 CPU/内存使用 |

## 可选：自定义插件

AstrBot 支持自定义 Python 插件。开发插件时需要本地 Python 环境。

### 1. 安装 Python（macOS）

```bash
# 使用 Homebrew 安装 Python 3.12
brew install python@3.12

# 验证安装
python3.12 --version
```

### 2. 创建虚拟环境

```bash
cd /Users/yushen/opt/QQAiBot
python3.12 -m venv venv
source venv/bin/activate
```

### 3. 安装 AstrBot 开发依赖

```bash
pip install astrbot
```

### 4. 开发并上传插件

1. 参考 [AstrBot 插件开发文档](https://docs.astrbot.app) 开发插件
2. 在 AstrBot 管理面板中进入 **插件** → **+**
3. 上传插件 ZIP 文件或粘贴代码

> **注意**：运行机器人本身不需要本地 Python，仅开发插件时需要。

## 故障排查

| 症状 | 可能原因 | 解决方案 |
|------|----------|----------|
| Docker Desktop 无法启动 | 未安装 Rosetta 2 | 运行 `softwareupdate --install-rosetta --agree-to-license`，完全退出并重新打开 Docker Desktop |
| NapCat WebUI 403 错误 | Token 错误或过期 | 运行 `docker compose logs napcat \| grep -i token` 获取新 Token |
| AstrBot 控制台显示"连不上" | OneBot 连接配置错误 | 检查反向 WebSocket URL 是否为 `ws://astrbot:6199/ws`，确保两个容器在同一网络 |
| 人格设定不生效 | 未绑定默认人格 | 在 `provider_settings` 中设置 `default_personality`，或在聊天中使用 `/persona <ID> && /reset` |
| DeepSeek 测试失败 | API Key 错误或余额不足 | 检查 [platform.deepseek.com](https://platform.deepseek.com) 余额，重新生成 API Key |
| 镜像拉取缓慢 | 未使用国内镜像源 | 检查 `~/.docker/daemon.json` 中是否配置 `registry-mirrors` |
| 容器频繁重启 | 端口冲突或权限问题 | 检查端口 6099/6185/6199 是否被占用，验证 `.env` 中 UID/GID 正确 |
| amd64 镜像无法运行 | Rosetta 2 未启用或失败 | 重新安装 Rosetta 2，确保 Docker Desktop 已完全重启 |
| 机器人不回复消息 | 连接断开或模型未配置 | 检查 AstrBot 控制台日志，验证 OneBot 适配器连接状态，确认 DeepSeek 模型已测试通过 |
| 日志中出现大量错误 | 内存不足或磁盘满 | 运行 `docker stats` 检查资源占用，清理磁盘空间 |

### 获取详细日志

```bash
# 查看最近 100 行 AstrBot 日志
docker compose logs --tail=100 astrbot

# 持续监控日志
docker compose logs -f astrbot napcat

# 导出日志到文件
docker compose logs > logs.txt 2>&1
```

### 重置配置（数据迁移/重置）

```bash
# 停止服务
docker compose down

# 备份数据（可选）
cp -r data data.backup

# 清空数据（谨慎操作）
rm -rf data/*

# 重新启动
docker compose up -d
```

## 架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                         Docker Compose                            │
│  ┌───────────────────────────桥接网络 astrbot_network─────────────┐│
│  │                                                              ││
│  │  ┌────────────────────┐                ┌─────────────────┐   ││
│  │  │    NapCat 容器     │                │   AstrBot 容器   │   ││
│  │  │  mlikiowa/napcat   │◄──反向WS───────│ soulter/astrbot │   ││
│  │  │                    │ ws://astrbot:  │                 │   ││
│  │  │ • OneBot11 协议    │    6199/ws     │ • WebUI :6185   │   ││
│  │  │ • WebUI :6099      │                │ • 插件系统      │   ││
│  │  │ • QQ 消息收发      │                │ • 人格设定      │   ││
│  │  └────────────────────┘                │ • LLM 接入      │   ││
│  │                                        └────────┬────────┘   ││
│  └─────────────────────────────────────────────────┼─────────────┘│
│                                                   │               │
└───────────────────────────────────────────────────┼───────────────┘
                                                    │
                                                    ▼
                                        ┌─────────────────────┐
                                        │  DeepSeek API       │
                                        │  (LLM 模型服务)     │
                                        └─────────────────────┘
                                                    │
                                                    ▼
                                        ┌─────────────────────┐
                                        │   QQ 群/私聊        │
                                        │   (用户交互)         │
                                        └─────────────────────┘
```

## 端口说明

| 端口 | 服务 | 暴露范围 | 说明 |
|------|------|----------|------|
| 6099 | NapCat WebUI | 本地/内网 | QQ 登录、协议配置，**严禁公网暴露** |
| 6185 | AstrBot 管理面板 | 可选公网 | 机器人配置、插件管理，公网需 HTTPS+TOTP |
| 6199 | OneBot 反向 WS | Docker 内部 | NapCat 与 AstrBot 通信，**严禁公网暴露** |

## 参考资源

- [AstrBot 官方文档](https://docs.astrbot.app)
- [NapCat 使用文档](https://napneko.github.io/)
- [DeepSeek API 文档](https://platform.deepseek.com/docs)
- [OneBot11 标准](https://github.com/botuniverse/onebot-11)

---

**版本**：v1.0  
**更新日期**：2026-06-27  
**维护者**：QQAiBot Team
