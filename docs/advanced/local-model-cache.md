# 本地模型缓存与自定义版本

为了避免每次启动都重新从 HuggingFace 下载模型，可以通过配置 `MODEL_DIR` 环境变量和自定义模型清单，让系统优先使用本地缓存。以下步骤以 `zjunlp/OceanGPT-o-7B-v0.1` 为例。

## 1. 准备模型目录

1. 在宿主机创建一个专门存放模型的目录，例如：
   ```bash
   mkdir -p ~/models/zjunlp/OceanGPT-o-7B-v0.1
   ```
2. 使用 `huggingface-cli` 或手动下载，将模型文件解压到上述目录中，确保 safetensors、tokenizer 等文件完整可用。

> 建议安装 `huggingface_hub[hf_xet]` 提升含 Xet 仓库模型的下载速度：
> ```bash
> pip install "huggingface_hub[hf_xet]"
> ```

## 2. 配置 `MODEL_DIR`

系统会根据 `MODEL_DIR` 环境变量定位本地模型根目录。你可以按照部署方式进行设置：

- **直接运行（非 Docker）**：在 shell 中导出变量后再启动服务。
  ```bash
  export MODEL_DIR=~/models
  uv run python main.py
  ```
- **Docker Compose 开发环境**：在项目根目录复制 `.env.template` 为 `.env`，并添加：
  ```ini
  MODEL_DIR=/absolute/path/to/models
  ```
  随后执行 `docker compose up -d`。Compose 会自动把宿主机的模型目录挂载到容器内 `/models`，并通过 `MODEL_DIR_IN_DOCKER=/models` 通知后端使用该路径。

完成以上步骤后，`src/config/app.py` 会读取 `MODEL_DIR`，并在启动时优先检查本地目录是否已经存在指定模型。若路径正确，管理器就不会触发二次下载。

## 3. 注册自定义模型名称（可选）

默认配置里并没有 `zjunlp/OceanGPT-o-7B-v0.1` 这个条目，可通过在 `saves/config/base.toml` 中登记模型，避免前端或后端回退到上游原始名称：

1. 如果 `saves/config/base.toml` 不存在，可以新建：
   ```bash
   mkdir -p saves/config
   ```
2. 在文件中写入如下内容：
   ```toml
   model_dir = "/models"  # 如果是非 Docker 环境，可替换为宿主机实际路径

   [model_names.local-ocean]
   name = "本地 OceanGPT"
   url = "https://huggingface.co/zjunlp/OceanGPT-o-7B-v0.1"
   base_url = "http://127.0.0.1:8000/v1"  # 若通过 vLLM/Ollama 暴露了兼容的 HTTP 接口
   default = "zjunlp/OceanGPT-o-7B-v0.1"
   env = "NO_API_KEY"
   models = ["zjunlp/OceanGPT-o-7B-v0.1"]
   ```

   - `env = "NO_API_KEY"` 表示该模型不需要额外的 API Key。
   - 若你使用本地 vLLM/Ollama 等服务提供推理能力，请把 `base_url` 调整为实际的 HTTP 地址。

> 保存后重启后端服务，`Config` 类会加载上述模型清单，让前端配置面板直接显示该条目。

## 4. 快速校验

- 宿主机确认模型存在：
  ```bash
  ls ~/models/zjunlp/OceanGPT-o-7B-v0.1
  ```
- 进入后端容器验证挂载：
  ```bash
  docker compose exec api ls /models/zjunlp/OceanGPT-o-7B-v0.1
  ```
- 通过后端日志确认没有触发重复下载：
  ```bash
  docker compose logs api | rg "OceanGPT"
  ```

若仍出现远程下载提示，请检查：

1. `MODEL_DIR` 是否指向了正确的绝对路径；
2. 宿主机目录权限是否允许容器读取；
3. `base.toml` 中模型名称是否与本地目录一致；
4. 推理服务（如 vLLM）是否已经就绪并提供正确的 `base_url`。

通过以上配置，系统会在启动阶段优先匹配本地缓存目录，避免重复从官方仓库获取模型权重。
