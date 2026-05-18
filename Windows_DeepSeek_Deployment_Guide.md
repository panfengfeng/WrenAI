# Wren AI Windows + DeepSeek 本机部署使用手册

本文档适用于在 Windows 本机使用 Docker Desktop 和 Docker Compose 部署当前 `legacy/v1` 分支，并使用 DeepSeek 作为 LLM。命令默认使用 PowerShell。

## 1. 部署架构

本分支推荐使用 Docker Compose 启动完整服务栈：

| 服务 | 作用 |
| --- | --- |
| `wren-ui` | Web UI 和 GraphQL BFF，默认访问入口 |
| `wren-ai-service` | AI 服务，负责 LLM 调用、RAG、SQL 生成、图表生成 |
| `wren-engine` | 语义查询引擎，负责 MDL 到 SQL/查询执行 |
| `ibis-server` | SQL/数据处理辅助服务 |
| `qdrant` | 向量数据库 |
| `bootstrap` | 初始化共享数据卷 |

默认访问地址：

```text
http://localhost:3000
```

## 2. 前置要求

请先确认 Windows 机器已安装：

- Docker Desktop
- Docker Compose
- PowerShell
- DeepSeek API Key
- 一个 embedding 模型服务的 API Key

> 注意：DeepSeek 负责 LLM 推理，但 Wren AI 还需要 embedding 模型做向量检索。仓库里的 DeepSeek 示例默认使用 OpenAI `text-embedding-3-large` 作为 embedding，因此默认还需要 `OPENAI_API_KEY`。

检查 Docker：

```powershell
docker --version
docker-compose --version
```

如果你的 Docker Desktop 只提供 Docker Compose V2，可以使用：

```powershell
docker compose version
```

本文档默认使用 `docker-compose`。如果你的环境没有 `docker-compose` 命令，可以把所有 `docker-compose` 替换为 `docker compose`。

## 3. 准备配置文件

进入仓库的 `docker` 目录：

```powershell
cd C:\Work\github\WrenAI\docker
```

复制环境变量模板：

```powershell
Copy-Item .env.example .env
```

复制 DeepSeek 配置为 Docker 部署使用的 `config.yaml`：

```powershell
Copy-Item ..\wren-ai-service\docs\config_examples\config.deepseek.yaml config.yaml
```

> 仓库示例文件注释中提到 `~/.wrenai/config.yaml`，但当前 Docker Compose 文件实际挂载的是 `docker/config.yaml`，因此 Docker 部署时应放在 `docker/config.yaml`。

## 4. 配置环境变量

编辑 `docker\.env`：

```powershell
notepad .env
```

至少设置以下内容：

```env
DEEPSEEK_API_KEY=你的_deepseek_api_key

# 如果继续使用默认 OpenAI embedding，需要填写
OPENAI_API_KEY=你的_openai_api_key

USER_UUID=你的_uuid
HOST_PORT=3000
```

生成 `USER_UUID`：

```powershell
[guid]::NewGuid().ToString()
```

如果本机 `3000` 端口已被占用，可以修改：

```env
HOST_PORT=3001
```

修改后访问地址也随之变为：

```text
http://localhost:3001
```

## 5. DeepSeek 配置说明

DeepSeek 示例配置文件位于：

```text
wren-ai-service/docs/config_examples/config.deepseek.yaml
```

复制到 `docker/config.yaml` 后，主要配置如下：

```yaml
type: llm
provider: litellm_llm
models:
  - api_base: https://api.deepseek.com/v1
    model: deepseek/deepseek-reasoner
  - api_base: https://api.deepseek.com/v1
    model: deepseek/deepseek-chat
  - api_base: https://api.deepseek.com/v1
    model: deepseek/deepseek-coder
    alias: default
```

默认 pipeline 中：

- `deepseek/deepseek-coder` 作为默认模型
- `deepseek/deepseek-chat` 用于部分回答类任务
- `deepseek/deepseek-reasoner` 用于 SQL reasoning 相关任务

## 6. Embedding 配置说明

DeepSeek 示例默认 embedding 配置为：

```yaml
type: embedder
provider: litellm_embedder
models:
  - model: text-embedding-3-large
    alias: default
    api_base: https://api.openai.com/v1
    timeout: 120
```

对应的 Qdrant 向量维度为：

```yaml
type: document_store
provider: qdrant
embedding_model_dim: 3072
```

如果继续使用 OpenAI `text-embedding-3-large`，无需修改 `embedding_model_dim`，保持 `3072` 即可。

如果更换为其它 embedding 模型，必须同步修改：

- `model`
- `api_base`
- `.env` 中对应的 API Key
- `embedding_model_dim`

`embedding_model_dim` 必须与 embedding 模型实际输出维度一致，否则索引和检索会失败。

## 7. 启动服务

在 `docker` 目录执行：

```powershell
docker-compose --env-file .env up -d
```

查看服务状态：

```powershell
docker-compose --env-file .env ps
```

查看所有日志：

```powershell
docker-compose --env-file .env logs -f
```

重点查看 AI 服务日志：

```powershell
docker-compose --env-file .env logs -f wren-ai-service
```

启动完成后访问：

```text
http://localhost:3000
```

## 8. 基本使用流程

进入 Web UI 后，通常按以下流程使用：

1. 创建或进入项目
2. 配置数据源连接
3. 导入数据库 schema
4. 在 UI 中建立或调整语义模型 MDL
5. 点击部署，让语义模型同步到 Engine 和 AI Service
6. 在问答界面用自然语言提问
7. 查看生成 SQL、查询结果、图表和 AI 解释

Wren AI 的核心不是直接把数据库 DDL 丢给 LLM，而是通过 MDL 语义层定义表、字段、关系和指标，再让 LLM 基于语义层生成 SQL。

## 9. 修改配置后重启

如果只修改了 `config.yaml` 或 LLM/embedding 配置，重建 AI Service：

```powershell
docker-compose --env-file .env up -d --force-recreate wren-ai-service
```

如果修改了端口、镜像版本或多个服务配置，可以整体重启：

```powershell
docker-compose --env-file .env down
docker-compose --env-file .env up -d
```

## 10. 停止和清理

停止服务：

```powershell
docker-compose --env-file .env down
```

停止并删除数据卷：

```powershell
docker-compose --env-file .env down -v
```

> `down -v` 会删除 Qdrant、SQLite、初始化数据等 Docker volume 内容，等同于重置本地环境，请谨慎使用。

## 11. Windows 常见注意事项

### Docker Desktop 未启动

如果命令提示无法连接 Docker daemon，请先启动 Docker Desktop，并等待状态变为 Running。

### WSL2 后端

Docker Desktop 通常建议使用 WSL2 backend。可以在 Docker Desktop 设置中确认：

```text
Settings -> General -> Use the WSL 2 based engine
```

### 镜像拉取失败

确认网络可以访问：

- `ghcr.io`
- `docker.io`
- `registry-1.docker.io`

如果公司网络或代理限制 Docker 拉镜像，需要先配置 Docker Desktop proxy。

### 端口冲突

修改 `.env`：

```env
HOST_PORT=3001
```

然后重启：

```powershell
docker-compose --env-file .env down
docker-compose --env-file .env up -d
```

### AI 功能不可用

优先检查：

- `.env` 中是否填写 `DEEPSEEK_API_KEY`
- 如果使用默认 embedding，是否填写 `OPENAI_API_KEY`
- `docker\config.yaml` 是否存在
- `wren-ai-service` 是否正常启动

查看日志：

```powershell
docker-compose --env-file .env logs -f wren-ai-service
```

### 修改模型后仍然没有生效

重建 AI Service：

```powershell
docker-compose --env-file .env up -d --force-recreate wren-ai-service
```

### Qdrant embedding 维度报错

通常是 `embedding_model_dim` 与 embedding 模型实际维度不一致。

如果已经用错误维度初始化过 Qdrant，可能需要清理 volume 后重新启动：

```powershell
docker-compose --env-file .env down -v
docker-compose --env-file .env up -d
```

## 12. 推荐最小配置

最小可用组合：

```text
LLM: DeepSeek
Embedding: OpenAI text-embedding-3-large
Vector DB: Qdrant
UI: http://localhost:3000
```

对应需要的 API Key：

```env
DEEPSEEK_API_KEY=你的_deepseek_api_key
OPENAI_API_KEY=你的_openai_api_key
```

