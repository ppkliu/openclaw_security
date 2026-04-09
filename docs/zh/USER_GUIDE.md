# OpenClaw Docker Compose 部署方案

完全容器化的 OpenClaw 部署方案，包含安全強化、一鍵部署腳本、可選 HTTPS 反向代理。

## 目錄

- [快速開始](#快速開始)
- [檔案說明](#檔案說明)
- [確認版本](#確認-openclaw-版本)
- [確認服務狀態](#確認服務是否正常啟動)
- [使用 Token 登入](#使用-token-登入-control-ui)
- [常用管理指令](#常用管理指令)
- [更新 OpenClaw](#更新-openclaw)
- [設定 Local 端 LLM](#設定-local-端-llmollama--localai--lm-studio)
- [安全設定說明](#安全設定說明)
- [串接不同系統的安全性判斷](#串接不同系統時的安全性判斷)
- [HTTPS 反向代理（可選）](#https-反向代理caddy可選)
- [疑難排解](#疑難排解)

---

## 快速開始

### 前置需求

- Docker Engine 24+ 與 Docker Compose V2
- 建議至少 2C4G（本方案在 12C/7.6G WSL2 環境驗證通過）

若尚未安裝 Docker：

```bash
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker $USER
# 重新登入或執行 newgrp docker
```

### 一鍵部署

```bash
chmod +x setup.sh
./setup.sh
```

腳本會自動：
1. 檢查 Docker 環境
2. 從 `.env.example` 生成 `.env`（含隨機 Gateway Token）
3. 拉取官方最新映像
4. 初始化設定檔與修正 volume 權限
5. 啟動 Gateway 服務
6. 等待健康檢查通過

### 手動部署

```bash
# 1. 複製環境變數並填入 token
cp .env.example .env
# 編輯 .env，設定 OPENCLAW_GATEWAY_TOKEN（執行 openssl rand -hex 32 生成）

# 2. 拉取映像
docker compose pull

# 3. 啟動
docker compose up -d
```

---

## 檔案說明

| 檔案 | 用途 |
|------|------|
| `docker-compose.yml` | Docker Compose 主設定（含安全強化） |
| `.env.example` | 環境變數範例（可提交到版本控制） |
| `.env` | 實際環境變數（已被 .gitignore 排除） |
| `setup.sh` | 一鍵部署腳本 |
| `Caddyfile` | HTTPS 反向代理設定（可選） |
| `todolist.md` | 部署任務追蹤與踩坑記錄 |
| `.gitignore` | 排除 .env 等敏感檔案 |

---

## 確認 OpenClaw 版本

### 方法 1：透過 CLI 指令

```bash
docker compose exec openclaw-gateway node dist/index.js --version
```

輸出範例：
```
OpenClaw 2026.3.8
```

### 方法 2：查看 Docker 映像標籤

```bash
docker compose images
```

輸出範例：
```
CONTAINER          REPOSITORY                      TAG       IMAGE ID       SIZE
openclaw-gateway   ghcr.io/openclaw/openclaw       latest    abc123def456   1.2GB
```

### 方法 3：查看容器內完整版本資訊

```bash
docker compose exec openclaw-gateway node dist/index.js --version
```

### 方法 4：透過 Control UI

登入 Control UI 後，頁面底部或 Settings 頁面會顯示目前版本號。

---

## 確認服務是否正常啟動

### 方法 1：查看容器狀態

```bash
docker compose ps
```

正常輸出（注意 STATUS 欄位應顯示 `Up ... (healthy)`）：
```
NAME               IMAGE                              SERVICE            STATUS                    PORTS
openclaw-gateway   ghcr.io/openclaw/openclaw:latest   openclaw-gateway   Up 5 minutes (healthy)    127.0.0.1:18789-18790->18789-18790/tcp
```

**關鍵判斷**：
- `Up` = 容器正在運行
- `(healthy)` = 健康檢查通過，服務正常
- `(unhealthy)` = 健康檢查失敗，需查看日誌
- `Exited` = 容器已停止

### 方法 2：健康檢查 API

```bash
curl http://127.0.0.1:18789/healthz
```

正常回應：
```json
{"ok":true,"status":"live"}
```

若無回應或連線拒絕，表示服務未正常啟動。

### 方法 3：查看即時日誌

```bash
docker compose logs -f openclaw-gateway
```

正常啟動會看到類似訊息：
```
[gateway] listening on ws://0.0.0.0:18789 (PID 7)
[gateway] agent model: anthropic/claude-opus-4-6
[heartbeat] started
[health-monitor] started
```

若看到錯誤訊息（如 `Missing config` 或 `failed to start`），請參考 [疑難排解](#疑難排解)。

### 方法 4：瀏覽器存取

直接在瀏覽器開啟：
```
http://localhost:18789
```

若能看到 OpenClaw Control UI 登入頁面，表示服務正常。

---

## 使用 Token 登入 Control UI

### 步驟 1：取得 Gateway Token

Token 在首次執行 `setup.sh` 時自動生成並存放在 `.env` 檔案中。

```bash
# 查看 token
grep OPENCLAW_GATEWAY_TOKEN .env
```

輸出範例：
```
OPENCLAW_GATEWAY_TOKEN=3d82b9ac595a4ad12a0667e5b73ef912ec847d58334d8cec869e4b64cffa442c
```

`=` 後方的整段十六進位字串就是您的 Token。

### 步驟 2：登入 Control UI

**方法 A：帶 Token 的 URL（推薦，一步完成）**

直接在瀏覽器開啟帶 Token 的完整 URL，自動登入：
```
http://localhost:18789/#token=YOUR_TOKEN_HERE
```

例如：
```
http://localhost:18789/#token=3d82b9ac595a4ad12a0667e5b73ef912ec847d58334d8cec869e4b64cffa442c
```

也可用 CLI 自動產生此 URL：
```bash
docker compose --profile cli run --rm openclaw-cli dashboard --no-open
```

**方法 B：手動輸入 Token**

1. 瀏覽器開啟 `http://localhost:18789`
2. 頁面會顯示 Token 輸入欄位（或出現 `unauthorized: gateway token missing` 提示）
3. 貼上步驟 1 取得的 Token（`=` 後方的完整字串）
4. 點擊登入 / 按 Enter
5. 成功後進入 Control UI 主頁面

> **WSL2 使用者**：在 Windows 瀏覽器中開啟 `http://localhost:18789` 即可（WSL2 會自動轉發）。

> **注意**：如果看到 `unauthorized: gateway token missing`，這代表 Gateway 正在正常運行，只是需要提供 Token。使用方法 A 的帶 Token URL 最簡單。

### 步驟 3：裝置配對（首次登入必要）

首次用瀏覽器連線時，Gateway 會要求 **裝置配對 (Device Pairing)**。您會看到 `pairing required` 提示或頁面無法載入。

**在終端機執行以下指令完成配對：**

```bash
# 1. 查看 Pending 的配對請求
docker compose exec openclaw-gateway node dist/index.js devices list

# 輸出範例：
# Pending (1)
# │ Request                              │ Device    │ Role     │ IP         │
# │ adf22455-c904-4a65-ba0c-17756706bbfe │ a579622c… │ operator │ 172.22.0.1 │

# 2. 批准配對（使用上方 Request 欄位的 ID）
docker compose exec openclaw-gateway node dist/index.js devices approve <REQUEST_ID>

# 例如：
docker compose exec openclaw-gateway node dist/index.js devices approve adf22455-c904-4a65-ba0c-17756706bbfe
```

批准後，**重新整理瀏覽器頁面**即可進入 Control UI。

> **提示**：每個新瀏覽器/裝置首次連線都需要配對一次。已配對的裝置可用 `devices list` 的 Paired 區段查看。

### 步驟 4：完成設定

登入後前往 **Settings** 頁面，依需求設定：
- **LLM Provider**：加入 MiniMax、Anthropic 等 API Key
- **Telegram Bot**：輸入 Bot Token 連接 Telegram
- **WhatsApp**：掃描 QR Code 連接
- **其他頻道**：Discord、Slack 等

也可透過 CLI 設定頻道：
```bash
# 互動式設定精靈
docker compose --profile cli run --rm openclaw-cli configure

# 直接加入 Telegram
docker compose --profile cli run --rm openclaw-cli channels add telegram --token YOUR_BOT_TOKEN
```

### Token 安全注意事項

- Token 等同管理員密碼，**請勿外洩**
- Token 存放在 `.env` 中，已被 `.gitignore` 排除
- 如需更換 Token，編輯 `.env` 中的 `OPENCLAW_GATEWAY_TOKEN` 值，然後重啟：
  ```bash
  # 生成新 token
  openssl rand -hex 32
  # 編輯 .env 替換 OPENCLAW_GATEWAY_TOKEN 的值
  # 重啟服務
  docker compose down && docker compose up -d
  ```

---

## 常用管理指令

```bash
# 啟動服務
docker compose up -d

# 停止服務
docker compose down

# 查看服務狀態
docker compose ps

# 查看即時日誌
docker compose logs -f

# 查看 Gateway 日誌（僅 gateway）
docker compose logs -f openclaw-gateway

# 重啟服務
docker compose restart

# 執行 CLI 工具（按需使用）
docker compose --profile cli run --rm openclaw-cli onboard
docker compose --profile cli run --rm openclaw-cli configure
docker compose --profile cli run --rm openclaw-cli channels add telegram --token YOUR_TOKEN
docker compose --profile cli run --rm openclaw-cli dashboard --no-open
```

---

## 更新 OpenClaw

```bash
# 拉取最新映像並重新啟動
docker compose pull
docker compose down
docker compose up -d

# 確認新版本
docker compose exec openclaw-gateway node dist/index.js --version
```

---

## 設定 Local 端 LLM（Ollama / LocalAI / LM Studio）

OpenClaw 支援透過 OpenAI 相容 API 連接本地 LLM，最常見搭配 **Ollama**。

### 方式 1：搭配 Ollama（推薦）

#### 步驟 1：安裝並啟動 Ollama

```bash
# 安裝 Ollama
curl -fsSL https://ollama.com/install.sh | sh

# 拉取模型（以 llama3.1 為例）
ollama pull llama3.1

# 確認 Ollama 運行中
curl http://localhost:11434/api/tags
```

#### 步驟 2：在 OpenClaw 中設定

透過 Control UI：
1. 登入 `http://localhost:18789`
2. 前往 **Settings** > **LLM Providers**
3. 新增 Provider：
   - **Type**: `openai-compatible`
   - **Base URL**: `http://host.docker.internal:11434/v1`（Docker 容器存取 host 端 Ollama）
   - **API Key**: `ollama`（Ollama 不驗證，任意填寫）
   - **Model**: `llama3.1`（或您拉取的模型名稱）

或透過 CLI：
```bash
docker compose --profile cli run --rm openclaw-cli configure
# 在 LLM Provider 步驟選擇 openai-compatible
# Base URL 填入 http://host.docker.internal:11434/v1
```

#### 重要：Docker 網路設定

容器內無法直接存取 host 的 `localhost`，需使用特殊 DNS：

| 環境 | Host URL |
|------|----------|
| Docker Desktop (Windows/macOS) | `http://host.docker.internal:11434/v1` |
| Linux Docker Engine | `http://172.17.0.1:11434/v1`（或使用 `--add-host`） |
| WSL2 + Docker Desktop | `http://host.docker.internal:11434/v1` |

若使用 Linux Docker Engine，需在 `docker-compose.yml` 的 `openclaw-gateway` 服務加入：
```yaml
extra_hosts:
  - "host.docker.internal:host-gateway"
```

#### 步驟 3：驗證連接

```bash
# 從容器內測試 Ollama 連接
docker compose exec openclaw-gateway \
  node -e "fetch('http://host.docker.internal:11434/api/tags').then(r=>r.json()).then(console.log).catch(console.error)"
```

### 方式 2：Ollama 整合到 Docker Compose

若想將 Ollama 一起容器化管理，在 `docker-compose.yml` 加入：

```yaml
services:
  ollama:
    image: ollama/ollama:latest
    container_name: ollama
    volumes:
      - ollama-data:/root/.ollama
    ports:
      - "127.0.0.1:11434:11434"
    restart: unless-stopped
    # GPU 支援（NVIDIA）：取消下方註解
    # deploy:
    #   resources:
    #     reservations:
    #       devices:
    #         - driver: nvidia
    #           count: all
    #           capabilities: [gpu]

volumes:
  ollama-data:
```

此時 OpenClaw 的 Base URL 改為：`http://ollama:11434/v1`（使用 Docker 內部 DNS）。

> **注意**：若 Ollama 使用 `network_mode: service:openclaw-gateway`，則 URL 為 `http://localhost:11434/v1`。

### 方式 3：其他本地 LLM 服務

| 服務 | 預設 Base URL | 說明 |
|------|---------------|------|
| **LM Studio** | `http://host.docker.internal:1234/v1` | 圖形介面，適合 Windows/macOS |
| **LocalAI** | `http://host.docker.internal:8080/v1` | 支援多種模型格式 |
| **llama.cpp server** | `http://host.docker.internal:8080/v1` | 輕量級，適合低資源環境 |
| **vLLM** | `http://host.docker.internal:8000/v1` | 高效能推理，適合 GPU 伺服器 |

設定方式相同：在 OpenClaw Settings 中選 `openai-compatible`，填入對應 Base URL。

### 方式 4：搭配 vLLM（高效能 GPU 推理，推薦生產環境）

**vLLM** 是高效能 LLM 推理引擎，支援 PagedAttention、連續批次處理、Tensor Parallelism，吞吐量遠超 Ollama，適合有 GPU 的伺服器。

#### 前置需求

- NVIDIA GPU（至少 8GB VRAM，推薦 24GB+）
- NVIDIA Driver 525+
- [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html)

```bash
# 安裝 NVIDIA Container Toolkit（Ubuntu）
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg
curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
  sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
  sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
sudo apt update && sudo apt install -y nvidia-container-toolkit
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker

# 驗證 GPU 可用
docker run --rm --gpus all nvidia/cuda:12.6.3-base-ubuntu24.04 nvidia-smi
```

#### 步驟 1：啟動 vLLM

**方法 A：獨立 Docker 容器**

```bash
docker run -d \
  --name vllm \
  --gpus all \
  -v ~/.cache/huggingface:/root/.cache/huggingface \
  -p 127.0.0.1:8000:8000 \
  --restart unless-stopped \
  vllm/vllm-openai:latest \
  --model meta-llama/Llama-3.1-8B-Instruct \
  --max-model-len 8192 \
  --gpu-memory-utilization 0.9
```

**方法 B：整合到 Docker Compose**

在 `docker-compose.yml` 加入 vLLM 服務：

```yaml
services:
  vllm:
    image: vllm/vllm-openai:latest
    container_name: vllm
    runtime: nvidia
    environment:
      - NVIDIA_VISIBLE_DEVICES=all
      - HUGGING_FACE_HUB_TOKEN=${HF_TOKEN:-}
    volumes:
      - vllm-cache:/root/.cache/huggingface
    ports:
      - "127.0.0.1:8000:8000"
    restart: unless-stopped
    command:
      - --model
      - ${VLLM_MODEL:-meta-llama/Llama-3.1-8B-Instruct}
      - --max-model-len
      - "${VLLM_MAX_MODEL_LEN:-8192}"
      - --gpu-memory-utilization
      - "${VLLM_GPU_MEM:-0.9}"
      - --host
      - "0.0.0.0"
      - --port
      - "8000"
    healthcheck:
      test: ["CMD", "curl", "-sf", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 120s  # 模型載入需要時間
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]

volumes:
  vllm-cache:
```

在 `.env` 加入：
```env
# vLLM 設定
VLLM_MODEL=meta-llama/Llama-3.1-8B-Instruct
VLLM_MAX_MODEL_LEN=8192
VLLM_GPU_MEM=0.9
HF_TOKEN=hf_your_huggingface_token_here
```

> **注意**：部分模型（如 Llama 3.1）需要在 [Hugging Face](https://huggingface.co/) 申請存取權限並提供 `HF_TOKEN`。

#### 步驟 2：驗證 vLLM 啟動

```bash
# 等待模型載入（首次可能需要數分鐘下載）
curl http://localhost:8000/health

# 查看可用模型
curl http://localhost:8000/v1/models | python3 -m json.tool

# 測試推理
curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "meta-llama/Llama-3.1-8B-Instruct",
    "messages": [{"role": "user", "content": "Hello!"}],
    "max_tokens": 50
  }'
```

#### 步驟 3：在 OpenClaw 中設定 vLLM

透過 Control UI：
1. 登入 `http://localhost:18789`
2. 前往 **Settings** > **LLM Providers**
3. 新增 Provider：
   - **Type**: `openai-compatible`
   - **Base URL**: 根據部署方式選擇（見下表）
   - **API Key**: `vllm`（vLLM 預設不驗證，任意填寫）
   - **Model**: `meta-llama/Llama-3.1-8B-Instruct`（需與 vLLM 啟動時一致）

| vLLM 部署方式 | OpenClaw 填入的 Base URL |
|--------------|--------------------------|
| 獨立容器（host 端） | `http://host.docker.internal:8000/v1` |
| Docker Compose 整合 | `http://vllm:8000/v1` |
| 遠端伺服器 | `http://<SERVER_IP>:8000/v1` |

或透過 CLI：
```bash
docker compose --profile cli run --rm openclaw-cli configure
# 選擇 openai-compatible
# Base URL: http://vllm:8000/v1（若同一 compose）
# 或 http://host.docker.internal:8000/v1（若獨立容器）
```

#### vLLM 進階設定

**多 GPU Tensor Parallelism**（多張 GPU 跑大模型）：
```bash
# 2 張 GPU 跑 70B 模型
--model meta-llama/Llama-3.1-70B-Instruct \
--tensor-parallel-size 2 \
--max-model-len 4096
```

**量化推理**（降低 VRAM 需求）：
```bash
# AWQ 量化（約減少 50% VRAM）
--model TheBloke/Llama-3.1-8B-Instruct-AWQ \
--quantization awq

# GPTQ 量化
--model TheBloke/Llama-3.1-8B-Instruct-GPTQ \
--quantization gptq
```

**啟用 API Key 認證**（安全加固）：
```bash
--api-key your_secret_key_here
```
此時 OpenClaw 的 API Key 需填入 `your_secret_key_here`。

#### vLLM vs Ollama 比較

| 特性 | vLLM | Ollama |
|------|------|--------|
| **安裝難度** | 需 NVIDIA GPU + 驅動 | 一鍵安裝 |
| **推理速度** | 極快（PagedAttention + 連續批次） | 一般 |
| **並行請求** | 優秀（生產級） | 有限 |
| **VRAM 效率** | 高（PagedAttention 動態分配） | 一般 |
| **支援模型** | HuggingFace 上所有 Transformers 模型 | Ollama 模型庫 |
| **量化支援** | AWQ, GPTQ, SqueezeLLM | GGUF (llama.cpp) |
| **CPU 推理** | 不支援 | 支援 |
| **適用場景** | 有 GPU 的生產環境 | 個人開發 / 無 GPU |

**建議**：
- 有 NVIDIA GPU 24GB+ → 用 **vLLM**
- 無 GPU 或 GPU 記憶體 < 8GB → 用 **Ollama**
- 需要同時服務多人 → 用 **vLLM**

### 常用 Ollama 模型建議

| 模型 | 大小 | 記憶體需求 | 適用場景 |
|------|------|-----------|----------|
| `llama3.1:8b` | 4.7GB | 8GB+ | 一般對話、程式碼 |
| `llama3.1:70b` | 40GB | 48GB+ | 複雜推理 |
| `codellama:13b` | 7.4GB | 16GB+ | 程式碼專用 |
| `mistral:7b` | 4.1GB | 8GB+ | 快速回應 |
| `qwen2.5:14b` | 9GB | 16GB+ | 中文優化 |
| `deepseek-coder-v2:16b` | 9GB | 16GB+ | 程式碼專用（中文友好） |

```bash
# 拉取模型
ollama pull llama3.1:8b
ollama pull qwen2.5:14b

# 列出已安裝模型
ollama list
```

---

## 安全設定說明

本方案已內建多層安全防護：

| 安全措施 | 設定 | 說明 |
|----------|------|------|
| 權限限制 | `cap_drop: ALL` | 移除所有 Linux capabilities |
| 權限提升防護 | `no-new-privileges:true` | 禁止容器內提升權限 |
| 唯讀檔案系統 | `read_only: true` | 容器根檔案系統唯讀 |
| 臨時目錄限制 | `tmpfs: /tmp (noexec, 100m)` | /tmp 不可執行且限制大小 |
| Sandbox 隔離 | `OPENCLAW_SANDBOX_MODE=non-main` | 非主會話強制 sandbox |
| 檔案存取限制 | `TOOLS_FS_WORKSPACEONLY=true` | Agent 只能存取 workspace |
| Fork bomb 防護 | `AGENTS_DEFAULTS_SANDBOX_PIDS_LIMIT=256` | 限制 PID 數量 |
| 網路綁定 | `127.0.0.1` (Docker port mapping) | 預設僅本機可存取 |

### 進階安全建議

- **絕不直接暴露 18789 到公網**，使用反向代理 + HTTPS
- 定期更新到最新版（漏洞修補頻繁）
- 不要在 workspace 放置敏感資料
- 關閉不需要的工具（browser、exec 等）
- 考慮使用 Tailscale / Cloudflare Tunnel 零信任存取

---

## 串接不同系統時的安全性判斷

OpenClaw 可串接多種外部系統（Telegram、WhatsApp、Discord、n8n、API 等），每種串接都引入不同的安全風險。以下提供系統性的評估框架。

### 安全風險等級評估矩陣

| 串接系統 | 風險等級 | 主要風險 | 建議措施 |
|----------|----------|----------|----------|
| **Telegram Bot** | 中 | Bot Token 外洩、Prompt Injection | Token 存 .env、啟用 pairing 審核、限制群組 |
| **WhatsApp** | 高 | Session 劫持、訊息注入、帳號封鎖 | 使用獨立號碼、限制自動回覆、監控異常 |
| **Discord Bot** | 中 | Bot Token 外洩、頻道權限過大 | 最小權限原則、限定頻道、審核指令 |
| **Slack** | 中 | OAuth Token 外洩、Workspace 資料存取 | 限制 scope、使用 Socket Mode |
| **n8n / Make** | 高 | Webhook 無認證、工作流程注入 | Webhook 加密碼驗證、限制 IP |
| **外部 API 呼叫** | 高 | API Key 外洩、SSRF、資料洩露 | Key 存環境變數、限制呼叫範圍 |
| **本地 Ollama** | 低 | 預設無認證、本機存取 | 綁定 127.0.0.1、不暴露到公網 |
| **雲端 LLM API** | 中 | API Key 外洩、費用失控 | 設定用量上限、定期輪換 Key |
| **瀏覽器工具** | 極高 | RCE、Cookie 竊取、Bridge 無認證 | 非必要不啟用、限制存取網站 |

### 串接前的安全檢查清單

在串接任何系統之前，逐項確認：

#### 1. 認證與授權
- [ ] 所有 Token / API Key 是否存放在 `.env` 而非明文寫在 config？
- [ ] Token 是否使用最小權限原則？（例如 Telegram Bot 不需要 admin 權限）
- [ ] 是否啟用 DM pairing 審核？（`dmPolicy: "pairing"`）
- [ ] Gateway Token 是否為高強度隨機值？

#### 2. 網路隔離
- [ ] OpenClaw 是否僅綁定 `127.0.0.1`（不直接暴露公網）？
- [ ] 外部存取是否經過 HTTPS 反向代理？
- [ ] Webhook 端點是否有 IP 白名單或簽名驗證？
- [ ] 本地 LLM（Ollama）是否僅綁定本機？

#### 3. 資料保護
- [ ] Workspace 中是否存放了敏感資料（密碼、私鑰、客戶資料）？
- [ ] `TOOLS_FS_WORKSPACEONLY=true` 是否已啟用？
- [ ] Agent 是否有權限讀取不應存取的檔案？
- [ ] 聊天記錄是否包含機密資訊？

#### 4. Prompt Injection 防護
- [ ] 是否限制了 Agent 可使用的工具？（關閉 exec、browser 等高危工具）
- [ ] 是否所有外部輸入都經過清理？
- [ ] 是否測試過惡意 prompt 的影響？
- [ ] ClawHub 技能是否僅使用 verified skills？

#### 5. 監控與回應
- [ ] 是否有日誌監控？（`docker compose logs -f`）
- [ ] 是否設定了用量告警？（LLM API 費用、訊息量）
- [ ] 是否有異常行為偵測機制？
- [ ] 是否有 Token 輪換計劃？

### 各系統串接安全指南

#### Telegram Bot 串接

```bash
# 安全地加入 Telegram Bot
docker compose --profile cli run --rm openclaw-cli channels add telegram --token YOUR_BOT_TOKEN
```

安全要點：
- **BotFather 設定**：關閉「Allow Groups」除非確實需要
- **Pairing**：首次 DM 時需要 pairing code 認證
- **群組限制**：只加入信任的群組
- **指令限制**：關閉不需要的 Agent 工具

#### Webhook 串接（n8n / Make / 自訂）

安全要點：
- **永遠加認證**：Webhook URL 加入 secret 參數
  ```
  http://localhost:18789/webhook/YOUR_SECRET_HERE
  ```
- **IP 白名單**：若可能，限制來源 IP
- **HTTPS**：Webhook 必須走 HTTPS（透過 Caddy）
- **限速**：設定 rate limit 防止濫用

#### 本地 LLM 串接

安全要點：
- Ollama 預設無認證，確保只綁定 `127.0.0.1`
- 不要將 Ollama API port 暴露到公網
- 若多人共用，考慮加入反向代理認證

### 已知安全漏洞與持續追蹤

OpenClaw 歷史上有多個高危漏洞（截至 2026.3）：

| CVE / GHSA | 風險 | 影響 | 修復版本 |
|------------|------|------|----------|
| CVE-2026-25253 | CVSS 8.8 | WebSocket 跨站劫持 + 1-click RCE | 2026.1.29 |
| CVE-2026-28457 | 高 | Sandbox 路徑穿越寫入 | 2026.2.14 |
| GHSA-q6qf-4p5j-r25g | 高 | Image tool sandbox bypass | 2026.2.23 |
| Browser Bridge 無認證 | 高 | 本機攻擊者可執行 JS | 已修復 |
| Fork bomb (無 pidsLimit) | 中 | 耗盡 host 資源 | 本方案已設定 256 |

**建議**：
- 訂閱 [OpenClaw Security Advisories](https://github.com/openclaw/openclaw/security/advisories)
- 每週執行一次 `docker compose pull && docker compose up -d`
- 定期執行 `docker compose --profile cli run --rm openclaw-cli doctor` 檢查設定

---

## HTTPS 反向代理（Caddy，可選）

如需從外部安全存取，建議使用 Caddy 反向代理：

```bash
# 1. 安裝 Caddy
sudo apt install -y caddy

# 2. 修改 Caddyfile 中的域名
#    將 openclaw.yourdomain.com 改為您的實際域名

# 3. 複製設定
sudo cp Caddyfile /etc/caddy/Caddyfile

# 4. 重啟 Caddy
sudo systemctl restart caddy
```

前提：域名 DNS 已指向此伺服器 IP。Caddy 會自動申請 Let's Encrypt 憑證。

---

## 疑難排解

### 登入後出現 pairing required

這是 Gateway 的裝置配對安全機制，每個新瀏覽器首次連線需配對。

```bash
# 查看 Pending 請求
docker compose exec openclaw-gateway node dist/index.js devices list

# 批准配對（替換 REQUEST_ID）
docker compose exec openclaw-gateway node dist/index.js devices approve REQUEST_ID
```

批准後重新整理瀏覽器即可。

### Gateway 啟動失敗：Missing config

```
Missing config. Run `openclaw setup` or set gateway.mode=local
```

**解決**：執行 `setup.sh` 或手動建立 config：
```bash
docker run --rm --user root \
  -v openclaw_docker_openclaw-config:/mnt/config \
  ghcr.io/openclaw/openclaw:latest \
  sh -c 'cat > /mnt/config/openclaw.json << EOF
{"gateway":{"mode":"local","controlUi":{"dangerouslyAllowHostHeaderOriginFallback":true}}}
EOF
chown node:node /mnt/config/openclaw.json'
```

### Gateway 啟動失敗：non-loopback Control UI requires allowedOrigins

```
non-loopback Control UI requires gateway.controlUi.allowedOrigins
```

**解決**：確認 `openclaw.json` 中包含：
```json
{
  "gateway": {
    "controlUi": {
      "dangerouslyAllowHostHeaderOriginFallback": true
    }
  }
}
```

### 健康檢查一直 unhealthy

```bash
# 查看詳細日誌
docker compose logs openclaw-gateway

# 檢查 port 是否被佔用
ss -tlnp | grep 18789
```

### Volume 權限錯誤

```bash
# 修正權限
docker run --rm --user root \
  -v openclaw_docker_openclaw-config:/mnt/config \
  ghcr.io/openclaw/openclaw:latest \
  sh -c 'chown -R node:node /mnt/config'
```

### WSL2 無法從 Windows 瀏覽器存取

確認 `.env` 中 `OPENCLAW_GATEWAY_BIND_ADDR=127.0.0.1`。WSL2 會自動將 localhost 轉發到 Windows。若仍無法存取：
```bash
# 查看 WSL2 IP
hostname -I | awk '{print $1}'
# 用該 IP 存取：http://<WSL2_IP>:18789
```

### Gateway 啟動迴圈：`plugins.entries.hosts: Unrecognized key: "http"`

症狀：`docker compose logs openclaw-gateway` 一直刷：
```
Config invalid
File: ~/.openclaw/openclaw.json
Problem:
  - plugins.entries.hosts: Unrecognized key: "http"
```

原因：`openclaw.json` 被手動（或舊版工具）寫入了錯誤的 MCP 設定,把 MCP server 放在 `plugins.entries.hosts.<xxx>` 下。OpenClaw 的 schema 嚴格驗證,這個路徑不合法,Gateway 每次啟動都 fail 退出、重啟,形成迴圈,且 `docker compose exec` 也進不去容器,無法用 CLI 修復。

**正確位置**:MCP server 應該放在 `mcp.servers.<name>`,用 `openclaw mcp set` 管理,不是手動塞進 `plugins`。

**緊急復原步驟**(Gateway 開不起來時,只能直接編輯 JSON):
```bash
# 1. 備份壞掉的 config
cp ./openclaw/openclaw.json ./bak/openclaw.json.$(date +%Y%m%d-%H%M%S)

# 2. 移除 plugins.entries 下非法的 hosts 區塊
#    保留其他合法 entries（例如 vllm）
#    可用任何編輯器,把整段 "hosts": { ... } 刪除

# 3. 重啟 gateway
docker compose restart openclaw-gateway
curl -sf http://127.0.0.1:18789/healthz && echo OK

# 4. Gateway 活過來後,用正確方式重新加回 MCP server
docker compose exec openclaw-gateway node dist/index.js mcp set <name> '{"url":"..."}'
```

### MCP server 連不上:`SSE error: Non-200 status code (404)`

症狀:log 出現
```
[bundle-mcp] failed to start server "xxx" (http://host.docker.internal:30003): Error: SSE error: Non-200 status code (404)
```

原因:OpenClaw **預設用舊的 SSE transport**,但近期許多 MCP server 改用新的 **Streamable HTTP transport**(MCP spec 2024-11 起),端點通常在 `/mcp` 而不是 SSE 的 `/sse` + `/messages`。OpenClaw 打根路徑或 `/sse` 都得到 404。

**診斷**:在 host 上分別 curl 各個可能路徑,看哪個回應才是 MCP server:
```bash
curl -v http://localhost:30003/
curl -v http://localhost:30003/sse
curl -v http://localhost:30003/mcp
```
- 若 `/mcp` 回 `400 {"error":"Invalid or missing session ID"}` → 是 Streamable HTTP
- 若 `/sse` 回 `text/event-stream` → 是 SSE
- 若根路徑就是 MCP endpoint,也看看 header

**修法**:用 `openclaw mcp set` 指定正確的 `transport` 欄位:

```bash
# Streamable HTTP(新 spec,常見於最新的 MCP server)
docker compose exec openclaw-gateway node dist/index.js mcp set rga-mcp-server \
  '{"url":"http://host.docker.internal:30003/mcp","transport":"streamable-http","connectionTimeoutMs":10000}'

# 傳統 SSE(transport 可省略)
docker compose exec openclaw-gateway node dist/index.js mcp set my-sse-server \
  '{"url":"http://host.docker.internal:4000/sse"}'

# 需要 Authorization header
docker compose exec openclaw-gateway node dist/index.js mcp set secure-mcp \
  '{"url":"https://mcp.example.com/mcp","transport":"streamable-http","headers":{"Authorization":"Bearer <token>"}}'

# 套用設定
docker compose restart openclaw-gateway
```

**注意**:`mcp set` 只寫 config,不會實際連線驗證。真正的連線是 lazy 建立的,agent 第一次呼叫該 MCP tool 時才會嘗試握手,屆時看 gateway log 才會知道是否成功。

另外容器裡要用 `host.docker.internal` 解析到宿主機 IP,需要 `docker-compose.yml` 設定 `extra_hosts: ["host.docker.internal:host-gateway"]`(本專案預設已設)。

### vLLM 對話失敗:`Context overflow: prompt too large for the model`

症狀:agent 呼叫模型時得到
```
[agent] embedded run agent end: ... isError=true
error=Context overflow: prompt too large for the model.
rawError=400 This model's maximum context length is 32768 tokens.
However, you requested 32000 output tokens and your prompt contains at least 769 input tokens,
for a total of at least 32769 tokens.
```

原因:`.env` 裡的 `VLLM_CONTEXT_WINDOW` / `VLLM_MAX_TOKENS` **大於 vLLM 後端實際的 `--max-model-len`**。OpenClaw 把這兩個值寫進 `openclaw.json`,對每個 request 都 request 到接近上限的 output tokens,加上 prompt 就超過後端真正能處理的長度,被 vLLM 直接 400 掉。

**檢查實際後端上限**:
```bash
# 查 vLLM 啟動參數裡的 --max-model-len(在 vLLM 那台機器上)
# 或從錯誤訊息直接讀:maximum context length is N tokens

# 也可從 /v1/models endpoint 看(部分 vLLM 版本會回 max_model_len)
curl -s http://<vllm-host>:<port>/v1/models | jq
```

**修法**:把 `.env` 的值調到**不超過後端上限**,並預留輸入空間:
```bash
# 假設後端 --max-model-len=32768
VLLM_CONTEXT_WINDOW=32768   # 必須 ≤ 後端實際上限
VLLM_MAX_TOKENS=8192        # output 上限,建議 ≤ context 的 1/4,留空間給 prompt
```

公式:`prompt_tokens + VLLM_MAX_TOKENS ≤ VLLM_CONTEXT_WINDOW`。

改完後**必須強制重寫 `openclaw.json`**(否則 runclaw.sh 會因為 model 名稱相同而跳過重新設定):
```bash
# 方法 A:用 runclaw.sh update 強制重跑所有步驟
./runclaw.sh update

# 方法 B:直接用 config set 更新(不用重跑整個 pipeline)
docker compose exec openclaw-gateway node dist/index.js config set models.providers.vllm \
  '{"baseUrl":"http://<host>:<port>/v1","api":"openai-completions","apiKey":"EMPTY","models":[{"id":"<model-id>","name":"<model-id>","contextWindow":32768,"maxTokens":8192,"reasoning":false,"input":["text"],"cost":{"input":0,"output":0}}]}'
docker compose restart openclaw-gateway
```

### Embedded ACPX runtime(codex-acp)啟動失敗:ENOSPC + Permission denied

症狀:
```
[plugins] embedded acpx runtime backend probe failed:
  embedded ACP runtime probe failed (agent=codex; command=npx @zed-industries/codex-acp@^0.11.1;
  cwd=/home/node/.openclaw/workspace;
  ACP agent exited before initialize completed (exit=126, signal=null):
  npm warn tar TAR_ENTRY_ERROR ENOSPC: no space left on device, write
  sh: 1: codex-acp: Permission denied)
```

**影響範圍**:只影響「OpenClaw 用 embedded ACPX runtime 跑本地 codex coding agent」的功能。對下列功能**沒有影響**:
- vLLM / Anthropic / OpenAI 等外部 provider 的對話
- Control UI、WebSocket、channels
- MCP server registry 與 MCP client 連線

**根本原因**:
1. **ENOSPC**:`docker-compose.yml` 把 `.npm` 和 `.cache` 掛成 tmpfs(大小預設很小,約 64M),不夠放 `@zed-industries/codex-acp` 及其依賴(通常 200MB+)
2. **Permission denied**:就算裝起來了,tmpfs 掛載可能帶 `noexec` flag,npm 安裝出來的 shim 腳本無法執行
3. 容器 `read_only: true` 限制了可寫路徑,所以必須透過 tmpfs 或 volume 提供寫入空間

**不使用 codex 時**:直接忽略這個 warning 即可,它不會阻止 gateway 啟動。

**需要啟用 codex runtime 時**,選一條路:

**方案 A:擴大 tmpfs 並確保可執行**(改 `docker-compose.yml`)
```yaml
services:
  openclaw-gateway:
    # ...
    tmpfs:
      - /tmp:size=256M,mode=1777
      - /home/node/.cache:size=512M,mode=0755,exec
      - /home/node/.npm:size=512M,mode=0755,exec   # 關鍵:exec,不能是 noexec;容量 ≥ 512M
      - /home/node/.openclaw/skills:size=256M,mode=0755,exec
```
注意 `exec` 必須明確指定,許多 Docker 預設 tmpfs 是 `noexec`。改完後 `docker compose down && docker compose up -d`(restart 不會重建 tmpfs)。

**方案 B:改掛 named volume 代替 tmpfs**(資料會持久化但更穩定)
```yaml
services:
  openclaw-gateway:
    # ...
    volumes:
      - openclaw_npm:/home/node/.npm
      - openclaw_cache:/home/node/.cache
volumes:
  openclaw_npm:
  openclaw_cache:
```
需確認 volume owner 是 uid 1000(`node` user),不是就 `docker run --rm --user root -v openclaw_npm:/mnt ghcr.io/openclaw/openclaw:latest chown -R node:node /mnt`。

**方案 C:build 預先裝好 codex-acp 的自訂 image**(最穩)
```dockerfile
FROM ghcr.io/openclaw/openclaw:latest
USER root
RUN npm install -g @zed-industries/codex-acp@0.11.1 \
 && chmod +x $(npm bin -g)/codex-acp
USER node
```
然後在 `docker-compose.yml` 把 `image:` 改成自己 build 的 tag。這條路不受 tmpfs noexec 影響,升級也容易。

### 完全重置

```bash
docker compose down -v   # 停止並刪除 volumes（資料會遺失！）
rm .env
./setup.sh               # 重新部署
```
