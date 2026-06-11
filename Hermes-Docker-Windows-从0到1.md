# Hermes Agent — Windows Docker 从 0 到 1 部署指南

本文档基于 Windows + Docker Desktop 环境，使用公司中转 OpenAI API 完成 Hermes Agent 的首次部署。

---

## 一、核心概念（先看懂再动手）

### 1.1 三个东西不要混淆

| 概念 | 是什么 | setup 结束后在哪 |
|------|--------|------------------|
| **镜像（Image）** | `nousresearch/hermes-agent:latest`，程序模板 | Docker 本地缓存，用 `docker images` 查看 |
| **容器（Container）** | 镜像运行时的实例；`setup` 用 `--rm`，结束后容器被删 | `docker ps -a` 一般看不到 setup 容器 |
| **配置数据（Volume）** | `config.yaml`、`.env`、会话、记忆等 | **`C:\Users\<用户名>\.hermes\`** |

### 1.2 配置文件放哪里

| 文件 | Docker 场景下的位置 | 能否放在项目目录 |
|------|---------------------|------------------|
| `config.yaml` | `%USERPROFILE%\.hermes\config.yaml` | ❌ 不会自动读取 |
| `.env` | `%USERPROFILE%\.hermes\.env` | ❌ Docker 不读项目 `.env`（除非挂载为 `/opt/data`） |

容器内路径为 `/opt/data`，通过 `-v` 挂载与宿主机 `%USERPROFILE%\.hermes` 同步。

### 1.3 中转 API 的正确写法

- **API Key** → 写在 `.env`：`OPENAI_API_KEY=sk-xxx`
- **中转地址** → 写在 `config.yaml` 的 `base_url`，**不要**依赖 `.env` 里的 `OPENAI_BASE_URL`（主模型不读该变量）

示例 `config.yaml` 片段：

```yaml
model:
  default: gpt-4o          # 改成中转站支持的模型名
  provider: custom
  base_url: https://newapi.aocloud.aosmith.com.cn/v1
  api_key: ""              # 留空，自动使用 .env 中的 OPENAI_API_KEY
```

---

## 二、前置条件

1. 安装并启动 **Docker Desktop**（Windows）
2. 确认 Docker 可用（在 **CMD** 中执行）：

```cmd
docker --version
docker ps
```

3. 准备：
   - 中转 API 地址：`https://newapi.aocloud.aosmith.com.cn/v1`
   - API Key：`sk-...`
   - 模型名：如 `gpt-4o`、`gpt-5.5` 等（以中转站支持为准）

---

## 三、从 0 到 1：完整步骤

### 步骤 1：创建数据目录

> **注意**：以下命令在 **CMD（命令提示符）** 中执行。`$env:USERPROFILE` 是 PowerShell 语法，CMD 中请用 `%USERPROFILE%`。

```cmd
mkdir %USERPROFILE%\.hermes
```

目录路径示例：`C:\Users\chenmingmin\.hermes`

### 步骤 2：运行 setup 向导（自动生成配置）

```cmd
docker run -it --rm -v "%USERPROFILE%\.hermes:/opt/data" nousresearch/hermes-agent:latest setup
```

说明：

- 首次运行会自动拉取镜像，需等待几分钟
- `-v` 把配置写入本机 `%USERPROFILE%\.hermes`
- `setup` 结束后容器自动删除（`--rm`），**配置留在本机目录**

### 步骤 3：setup 向导各页面怎么选

#### 3.1 模型与端点

| 提示 | 建议操作 |
|------|----------|
| API 基础 URL | `https://newapi.aocloud.aosmith.com.cn/v1` |
| API Key | 输入你的 `sk-...` |
| Select model | 输入模型名，如 `4o` 或 `gpt-4o` |
| Context length | **直接回车留空**（自动检测） |
| Display name | **直接回车**（默认即可）或填 `newapi` |

#### 3.2 Terminal 后端

| 选项 | 含义 |
|------|------|
| **Local**（推荐） | 命令在 Hermes 容器内部执行 |
| Docker | 在容器里再启一个 Docker 沙箱，配置复杂，初次不推荐 |

→ 选 **Local** 或 **Keep current (local)**，回车。

#### 3.3 图像生成（Image Gen）

不需要画图 → 选 **Skip**。

#### 3.4 网页搜索（Search Provider）

| 需求 | 选择 |
|------|------|
| 暂时不用搜索 | **Skip** |
| 免费搜索、无需 API Key | **DuckDuckGo (ddgs)** |
| 更好质量 | **Brave Search (Free)**（需申请 `BRAVE_API_KEY`） |

不要选 **Nous Subscription**（需 Nous 订阅，与公司中转无关）。

#### 3.5 其他项

Gateway（Telegram/Discord 等）首次可跳过，之后需要再配置。

### 步骤 4：确认配置已生成

```cmd
dir %USERPROFILE%\.hermes
type %USERPROFILE%\.hermes\config.yaml
type %USERPROFILE%\.hermes\.env
```

应看到至少包含：

- `config.yaml` — 模型、`base_url`、`provider`
- `.env` — API Key

### 步骤 5：启动 Hermes 服务

进入项目目录，使用 Windows 专用 compose 文件：

```cmd
cd /d d:\AO.Code\Lab\AI.Personal\AI.Personal\hermes-agent
docker compose -f docker-compose.windows.yml up -d
```

检查容器状态：

```cmd
docker ps
```

应看到：

- `hermes` — Gateway
- `hermes-dashboard` — Web 界面

### 步骤 6：访问 Web 界面

浏览器打开：

```
http://127.0.0.1:9119
```

发送测试消息，例如：`你好，介绍一下你自己`。

### 步骤 7：查看日志（排错用）

```cmd
docker logs -f hermes
docker logs -f hermes-dashboard
```

按 `Ctrl+C` 退出日志跟踪。

---

## 四、setup 完成后的常用命令

| 操作 | CMD 命令 |
|------|----------|
| 启动服务 | `docker compose -f docker-compose.windows.yml up -d` |
| 停止服务 | `docker compose -f docker-compose.windows.yml down` |
| 重启服务 | `docker compose -f docker-compose.windows.yml restart` |
| 查看容器 | `docker ps` |
| 查看镜像 | `docker images nousresearch/hermes-agent` |
| 命令行聊天（不启后台） | `docker run -it --rm -v "%USERPROFILE%\.hermes:/opt/data" nousresearch/hermes-agent:latest` |
| 重新运行 setup | `docker run -it --rm -v "%USERPROFILE%\.hermes:/opt/data" nousresearch/hermes-agent:latest setup` |
| 编辑配置 | `notepad %USERPROFILE%\.hermes\config.yaml` |
| 编辑密钥 | `notepad %USERPROFILE%\.hermes\.env` |

---

## 五、不用 setup、手动创建配置（备选）

若跳过向导，可手动创建：

**`%USERPROFILE%\.hermes\.env`**

```bash
OPENAI_API_KEY=sk-你的密钥
```

**`%USERPROFILE%\.hermes\config.yaml`**

```yaml
model:
  default: gpt-4o
  provider: custom
  base_url: https://newapi.aocloud.aosmith.com.cn/v1
  api_key: ""
```

然后执行 `docker compose -f docker-compose.windows.yml up -d`。

---

## 六、架构示意

```
Windows 宿主机
  └── Docker Desktop
        ├── 镜像: nousresearch/hermes-agent:latest（留在 Docker 本地）
        │
        ├── 容器 hermes（Gateway，持久运行）
        │     └── 读取 /opt/data ← 挂载自 C:\Users\<你>\.hermes
        │
        └── 容器 hermes-dashboard（Web UI，端口 9119）
              └── 同样挂载 C:\Users\<你>\.hermes
```

**「Hermes 本身跑在 Docker 里」** = Hermes 程序在容器内运行，不是在 Windows 上直接 `pip install hermes`。

**Terminal 选 Local** = Agent 执行 shell 命令时，在 **Hermes 容器内部**运行，而不是在 Windows 上，也不是再套一层 Docker。

---

## 七、常见问题

### Q1：`mkdir $env:USERPROFILE\.hermes` 报「文件名、目录名或卷标语法不正确」

你在 **CMD** 里用了 PowerShell 语法。请改用：

```cmd
mkdir %USERPROFILE%\.hermes
```

### Q2：setup 完成后「镜像在哪里」？

- **配置**：`C:\Users\<你>\.hermes\`
- **镜像**：Docker 本地，`docker images` 查看
- **setup 容器**：已删除（`--rm`）

### Q3：项目目录 `hermes-agent\.env` 会生效吗？

Docker 默认只挂载 `%USERPROFILE%\.hermes`，**不会**读项目里的 `.env` / `config.yaml`。

### Q4：修改配置后如何生效？

```cmd
notepad %USERPROFILE%\.hermes\config.yaml
docker compose -f docker-compose.windows.yml restart
```

### Q5：同一数据目录能跑两个 Gateway 吗？

**不能**。会导致会话文件并发写入损坏。

### Q6：中转站在宿主机本地（如 `localhost:8080`）

容器内访问宿主机用 `host.docker.internal`：

```yaml
base_url: http://host.docker.internal:8080/v1
```

### Q7：API Key 写在 `config.yaml` 还是 `.env`？

**推荐只写在 `.env`**，`config.yaml` 里 `api_key: ""` 留空。不要把密钥提交到 Git。

---

## 八、检查清单

部署完成后逐项确认：

- [ ] `%USERPROFILE%\.hermes\config.yaml` 存在，且 `base_url` 正确
- [ ] `%USERPROFILE%\.hermes\.env` 存在，且 `OPENAI_API_KEY` 已填
- [ ] `docker ps` 显示 `hermes`、`hermes-dashboard` 均为 Up
- [ ] http://127.0.0.1:9119 能打开且能收到模型回复
- [ ] `model.default` 与中转站支持的模型名一致

---

## 九、参考路径速查

| 用途 | 路径 |
|------|------|
| 配置目录（Docker） | `C:\Users\<用户名>\.hermes\` |
| 项目源码 | `d:\AO.Code\Lab\AI.Personal\AI.Personal\hermes-agent\` |
| Compose 文件 | `docker-compose.windows.yml` |
| Dashboard | http://127.0.0.1:9119 |
| 官方 Docker 文档 | `website/i18n/zh-Hans/.../user-guide/docker.md` |

---

*文档生成日期：2026-06-10*
