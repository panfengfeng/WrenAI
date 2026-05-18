# Wren AI Linux + DeepSeek 源码部署使用手册

本文档适用于在 Linux 本机尽量不依赖 Docker 的方式部署当前 `legacy/v1` 分支，并使用 DeepSeek 作为 LLM。

相比 Docker Compose 部署，源码部署需要分别启动多个服务，复杂度明显更高。推荐仅在需要二次开发、调试源码或定制服务时使用。

## 1. 部署目标

源码部署需要启动以下服务：

| 服务 | 启动方式 | 默认端口 |
| --- | --- | --- |
| `qdrant` | 本机二进制或源码构建 | `6333`, `6334` |
| `wren-engine` | `wren-engine` 子仓库源码构建 | `8080` |
| `ibis-server` | `wren-engine/ibis-server` 源码运行 | `8000` |
| `wren-ai-service` | 当前仓库源码运行 | `5555` |
| `wren-ui` | 当前仓库源码运行 | `3000` |

访问入口：

```text
http://localhost:3000
```

## 2. 重要限制

当前仓库的 `wren-engine` 是 git submodule：

```text
wren-engine -> git@github.com:Canner/wren-engine.git
```

如果本机没有配置 GitHub SSH key，执行 submodule 初始化时会失败。可以改用 HTTPS 手动克隆：

```bash
cd /path/to/WrenAI
git clone https://github.com/Canner/wren-engine.git wren-engine
cd wren-engine
git checkout 47ca29ebba291100ba5d70ce1790f9887eaed7a0
```

`47ca29ebba291100ba5d70ce1790f9887eaed7a0` 是当前 `legacy/v1` 分支记录的 `wren-engine` submodule commit。

## 3. 前置依赖

建议 Linux 环境安装：

- Git
- Node.js 18
- Yarn 4.5.x
- Python 3.12.x，用于 `wren-ai-service`
- Python 3.11.x，用于 `wren-engine/ibis-server`
- Poetry 1.8.3
- Just 1.36+
- Rust + Cargo
- JDK 21+
- Maven，或使用 `wren-core-legacy` 自带的 `mvnw`
- Qdrant 1.15.0

检查版本：

```bash
git --version
node -v
yarn -v
python3 --version
poetry --version
just --version
rustc --version
cargo --version
java -version
```

> 如果同一台机器需要同时使用 Python 3.12 和 Python 3.11，建议使用 `pyenv` 管理。

## 4. 启动 Qdrant

Wren AI 当前 Docker 配置使用的是 Qdrant `v1.15.0`。源码部署时建议使用同版本。

可以从 Qdrant GitHub Releases 下载 Linux 二进制包，解压后启动：

```bash
mkdir -p ~/wren-runtime/qdrant
cd ~/wren-runtime/qdrant

# 下载 qdrant v1.15.0 的 Linux 二进制包后解压
# 解压后通常会得到 qdrant 可执行文件

./qdrant --storage-dir ./storage
```

启动后检查：

```bash
curl http://localhost:6333/
```

能返回 Qdrant 服务信息即可。

## 5. 启动 Wren Engine

进入 `wren-engine`：

```bash
cd /path/to/WrenAI/wren-engine
```

准备 engine 配置目录：

```bash
mkdir -p wren-core-legacy/docker/etc/mdl
cat > wren-core-legacy/docker/etc/config.properties <<'EOF'
node.environment=production
EOF

cat > wren-core-legacy/docker/etc/mdl/sample.json <<'EOF'
{"catalog":"test_catalog","schema":"test_schema","models":[]}
EOF
```

构建 legacy Java engine：

```bash
cd wren-core-legacy
./mvnw clean install -DskipTests -P exec-jar
```

启动 engine：

```bash
java \
  -Dconfig=docker/etc/config.properties \
  --add-opens=java.base/java.nio=ALL-UNNAMED \
  -jar wren-server/target/wren-server-0.15.2-SNAPSHOT-executable.jar
```

启动后默认监听：

```text
http://localhost:8080
```

## 6. 启动 Ibis Server

`ibis-server` 依赖 Python 3.11、Rust/Cargo、Poetry 和 Just。

另开一个终端：

```bash
cd /path/to/WrenAI/wren-engine/ibis-server
```

创建 `.env`：

```bash
cat > .env <<'EOF'
WREN_ENGINE_ENDPOINT=http://localhost:8080
EOF
```

安装依赖：

```bash
just install
```

启动：

```bash
just run
```

默认监听：

```text
http://localhost:8000
```

如果需要开发热重载：

```bash
just dev
```

## 7. 配置并启动 Wren AI Service

另开一个终端：

```bash
cd /path/to/WrenAI/wren-ai-service
```

安装依赖：

```bash
poetry install
```

初始化配置：

```bash
just init
```

将 DeepSeek 示例覆盖为本地源码部署配置：

```bash
cp docs/config_examples/config.deepseek.yaml config.yaml
```

编辑 `config.yaml`，把 Docker 网络内的服务名改为本机地址：

```yaml
type: engine
provider: wren_ui
endpoint: http://localhost:3000

---
type: engine
provider: wren_ibis
endpoint: http://localhost:8000

---
type: document_store
provider: qdrant
location: http://localhost:6333
embedding_model_dim: 3072
timeout: 120
recreate_index: true
```

建议在 `settings` 中显式加上 AI Service 监听端口：

```yaml
settings:
  host: 127.0.0.1
  port: 5555
```

如果原 `settings` 已存在，只需要把 `host` 和 `port` 合并进去，不要创建第二个 `settings` 区块。

编辑 `.env.dev`：

```bash
vim .env.dev
```

至少填写：

```env
DEEPSEEK_API_KEY=你的_deepseek_api_key

# 如果继续使用默认 OpenAI embedding，需要填写
OPENAI_API_KEY=你的_openai_api_key
```

> DeepSeek 示例默认使用 OpenAI `text-embedding-3-large` 做 embedding，因此默认还需要 `OPENAI_API_KEY`。如果更换 embedding 模型，需要同步修改 `config.yaml` 中的 `model`、`api_base` 和 `embedding_model_dim`。

启动 AI Service：

```bash
just start
```

默认监听：

```text
http://localhost:5555
```

## 8. 配置并启动 Wren UI

另开一个终端：

```bash
cd /path/to/WrenAI/wren-ui
```

安装依赖：

```bash
yarn
```

设置环境变量：

```bash
export DB_TYPE=sqlite
export SQLITE_FILE=./db.sqlite3
export WREN_ENGINE_ENDPOINT=http://localhost:8080
export WREN_AI_ENDPOINT=http://localhost:5555
export IBIS_SERVER_ENDPOINT=http://localhost:8000
export EXPERIMENTAL_ENGINE_RUST_VERSION=false
export TELEMETRY_ENABLED=false
export USER_UUID=$(uuidgen)
export GENERATION_MODEL=deepseek/deepseek-coder
```

执行数据库迁移：

```bash
yarn migrate
```

启动 UI：

```bash
yarn dev
```

访问：

```text
http://localhost:3000
```

## 9. 推荐启动顺序

建议按以下顺序启动：

1. Qdrant
2. Wren Engine
3. Ibis Server
4. Wren AI Service
5. Wren UI

每个服务建议单独开一个终端，便于查看日志。

## 10. 基本使用流程

进入 Web UI 后：

1. 创建或进入项目
2. 配置数据源连接
3. 导入数据库 schema
4. 建立或调整语义模型 MDL
5. 点击部署，让语义模型同步到 Engine 和 AI Service
6. 在问答界面用自然语言提问
7. 查看生成 SQL、查询结果、图表和 AI 解释

## 11. 常见问题

### submodule 拉取失败

如果报错类似：

```text
Host key verification failed.
Could not read from remote repository.
```

说明当前 submodule 使用 SSH 地址，而机器没有配置 GitHub SSH key。可以改用 HTTPS：

```bash
cd /path/to/WrenAI
rm -rf wren-engine
git clone https://github.com/Canner/wren-engine.git wren-engine
cd wren-engine
git checkout 47ca29ebba291100ba5d70ce1790f9887eaed7a0
```

### AI Service 连不上 Qdrant

检查 Qdrant：

```bash
curl http://localhost:6333/
```

确认 `wren-ai-service/config.yaml`：

```yaml
location: http://localhost:6333
```

### AI Service 连不上 UI 或 Ibis Server

确认 `wren-ai-service/config.yaml` 中是本机地址，而不是 Docker 服务名：

```yaml
endpoint: http://localhost:3000
endpoint: http://localhost:8000
```

### UI 连不上 AI Service

确认启动 UI 前设置了：

```bash
export WREN_AI_ENDPOINT=http://localhost:5555
```

并确认 AI Service 正在运行：

```bash
curl http://localhost:5555
```

### Qdrant embedding 维度报错

通常是 `embedding_model_dim` 与 embedding 模型实际输出维度不一致。

如果使用 OpenAI `text-embedding-3-large`，保持：

```yaml
embedding_model_dim: 3072
```

如果换成其它 embedding 模型，需要改成对应维度，并清理 Qdrant 存储目录后重启。

### 不建议混用 Docker 网络地址

源码部署时不要使用：

```text
http://wren-ui:3000
http://ibis-server:8000
http://qdrant:6333
```

这些地址只在 Docker Compose 网络中有效。源码部署应使用：

```text
http://localhost:3000
http://localhost:8000
http://localhost:6333
```

## 12. 最小可用配置

```text
LLM: DeepSeek
Embedding: OpenAI text-embedding-3-large
Vector DB: Qdrant 本机二进制
Engine: wren-engine 源码构建
AI Service: wren-ai-service 源码运行
UI: wren-ui 源码运行
```

需要的 API Key：

```env
DEEPSEEK_API_KEY=你的_deepseek_api_key
OPENAI_API_KEY=你的_openai_api_key
```

## 13. 参考

- Wren AI 当前仓库：`README.md`
- Wren UI 源码启动：`wren-ui/README.md`
- Wren AI Service 源码启动：`wren-ai-service/README.md`
- Wren Engine legacy Java engine：`wren-engine/wren-core-legacy/README.md`
- Wren Engine ibis-server：`wren-engine/ibis-server/README.md`

