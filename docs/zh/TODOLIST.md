# OpenClaw Docker Compose 部署 TodoList

## 基礎建設
- [x] 確認 Docker / Docker Compose 環境已就緒 (Docker 29.2.1 + Compose v2.40.2)
- [x] 確認系統規格（12C/7.6G x86_64 WSL2，符合需求）
- [x] 確認使用者已在 docker 群組

## 檔案建立
- [x] 建立 `todolist.md`（本檔案）
- [x] 建立 `docker-compose.yml`（含安全強化配置）
- [x] 建立 `.env.example`（範例環境變數）
- [x] 建立 `.env`（實際環境變數，含隨機 token）
- [x] 建立 `.gitignore`（保護 .env 等敏感檔案）
- [x] 建立 `Caddyfile`（HTTPS 反向代理，可選）
- [x] 建立 `setup.sh`（一鍵部署腳本）

## 安全強化
- [x] Gateway token 使用隨機生成值 (`openssl rand -hex 32`)
- [x] 設定 no-new-privileges + cap_drop ALL
- [x] 設定 read_only + tmpfs（/tmp, /home/node/.cache）
- [x] 設定 OPENCLAW_SANDBOX_MODE=non-main
- [x] 設定 TOOLS_FS_WORKSPACEONLY=true
- [x] 設定 AGENTS_DEFAULTS_SANDBOX_PIDS_LIMIT=256（防 fork bomb）
- [x] Gateway 預設綁定 127.0.0.1（Docker port mapping 層面）

## 驗證與測試
- [x] `docker compose config` 語法驗證 — 通過
- [x] `docker compose pull` 拉取映像 — 成功 (2026.3.8)
- [x] `docker compose up -d` 啟動測試 — Gateway 正常運行
- [x] 健康檢查 healthcheck — `{"ok":true,"status":"live"}`
- [x] Gateway 可正常存取 http://127.0.0.1:18789

## 實作中發現的問題與修正
- [x] **問題 1**：首次啟動 gateway 報 "Missing config"
  - **原因**：需要 `openclaw.json` config 檔才能啟動
  - **修正**：gateway command 加入 `--allow-unconfigured`，setup.sh 自動建立最小 config
- [x] **問題 2**：`--bind lan` 報 "non-loopback Control UI requires allowedOrigins"
  - **原因**：非 loopback 綁定需要設定 CORS origin
  - **修正**：config 中設定 `gateway.controlUi.dangerouslyAllowHostHeaderOriginFallback=true`
- [x] **問題 3**：Docker volume 權限為 root，container 跑 node 使用者無法寫入
  - **原因**：Docker 預設以 root 建立 volume
  - **修正**：setup.sh 加入 `chown -R node:node` 步驟
- [x] **問題 4**：CLI 容器無子指令時顯示 help 後退出
  - **原因**：CLI 是按需工具，不是常駐服務
  - **修正**：加入 `profiles: ["cli"]` 避免 `docker compose up -d` 自動啟動

- [x] **問題 5**：`--profile cli` 旗標位置錯誤導致 CLI 執行失敗
  - **原因**：`--profile` 是 `docker compose` 的旗標，必須在子指令之前
  - **修正**：`docker compose --profile cli run --rm` 而非 `docker compose run --rm --profile cli`
- [x] **問題 6**：直接存取 `http://localhost:18789` 顯示 `unauthorized: gateway token missing`
  - **原因**：Gateway 需要 Token 認證才能存取 Control UI
  - **修正**：使用帶 Token 的 URL `http://localhost:18789/#token=YOUR_TOKEN` 直接登入

- [x] **問題 7**：帶 Token URL 登入後出現 `pairing required`（WebSocket 1008）
  - **原因**：Gateway 有裝置配對機制，瀏覽器首次連線需配對
  - **修正**：用 `devices list` 查看 Pending 請求，再用 `devices approve <request-id>` 批准
  - **指令**：`docker compose exec openclaw-gateway node dist/index.js devices approve <REQUEST_ID>`
- [x] **問題 8**：`pairingRequired` 不是有效的 config key，嘗試寫入會導致 gateway 啟動失敗
  - **原因**：Config schema 嚴格驗證，不接受未知 key
  - **教訓**：修改 config 前先用 `config validate` 測試

## 使用方式
```bash
# 首次部署
chmod +x setup.sh && ./setup.sh

# 日常管理
docker compose up -d                    # 啟動
docker compose down                     # 停止
docker compose logs -f                  # 查看日誌
docker compose ps                       # 查看狀態
docker compose pull && docker compose up -d  # 更新

# CLI 工具（按需使用，注意 --profile 位置）
docker compose --profile cli run --rm openclaw-cli onboard
docker compose --profile cli run --rm openclaw-cli channels add telegram --token YOUR_TOKEN
docker compose --profile cli run --rm openclaw-cli configure
docker compose --profile cli run --rm openclaw-cli dashboard --no-open  # 取得帶 Token 的登入 URL
```
