# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 專案概述

Ptt-Alertor 是一個 Go 應用程式，用於監控 PTT（台灣最大的 BBS）的新文章，並透過多種管道（LINE、LINE Notify、Facebook Messenger、Telegram、Email）發送通知給使用者。使用者可以訂閱看板的關鍵字、作者或推文數門檻。

## 建置與測試指令

```bash
# 安裝依賴套件
go get

# 執行測試（含競爭條件檢測與覆蓋率）
# `-tags test` 會排除 `models/user/file.go`（該檔使用 `+build !test`），與 CI 一致
# 多數測試透過直接注入 `Mock{}` driver 加上 `alicebob/miniredis` 替代真實 Redis，不需要本機 Redis/DynamoDB
go test -race -tags test -coverprofile=coverage.txt -covermode=atomic ./...

# 執行單一測試
go test -v -tags test ./path/to/package -run TestName

# 檢視覆蓋率
go tool cover -html=coverage.txt

# 建置並執行
go build && ./ptt-alertor

# Docker 建置
docker build -t ptt-alertor .
```

## 架構說明

### 核心元件

**進入點 (`main.go`):**
- Web 伺服器運行於 port 9090，gops agent 運行於 port 6060
- `init()` 在 `main()` 之前執行一次性啟動／資料遷移工作：`PushSumKeyReplacer`、`MigrateBoard`、`Top`、`CacheCleaner`、`Generator`、`Fetcher`、`MigrateDB`、`CategoryCleaner`（新增或調整這些時要留意它們會在每次啟動都跑一次）
- `startJobs()` 啟動長駐背景工作：`Checker`、`PushSumChecker`、`CommentChecker`、`PttMonitor`
- Cron 排程：`Top`（每小時）、`PushSumKeyReplacer`（每 48 小時）

**工作排程 (`jobs/`):**
- `Checker`：主迴圈，監控看板新文章，比對使用者的關鍵字/作者訂閱。以 `boardCh` 分流，高優先看板（`BOARD_HIGH`）每 1 秒掃一次，一般看板間隔 250ms 節拍
- `PushSumChecker`：監控文章是否達到推文／噓文數門檻
- `CommentChecker`：追蹤使用者關注文章的新推文
- `PttMonitor`：每分鐘打 PTT 首頁；連續失敗超過 `retry` 次後，偵測到 PTT 恢復時會重新啟動上述三個 Checker（PTT 掛掉期間的自我修復機制）
- `check.go`：訊息工作池（300 個 worker），接收 `check` interface，依使用者 Profile 分派至對應的頻道（LINE／Messenger／Telegram／Email 可同時多發）

**資料模型 (`models/`):**
- 使用 Driver 模式支援多種儲存後端（Redis 用於使用者/看板快取，DynamoDB 用於文章/看板持久化）
- `models/models.go`：工廠函式 `User()`、`Article()`、`Board()` 決定 production 使用哪些 driver；測試時會有對應的 mock driver（受 `-tags test` 控制）
- 每個模型資料夾含多個 driver 檔（例如 `board/` 有 `dynamodb.go`、`redis.go`、`file.go`），共用同一個 interface
- `user.User`：使用者檔案，包含通知端點（LINE、Messenger、Telegram、Email）
- `subscription.Subscription`：每位使用者的看板 + 關鍵字/作者/推文門檻/文章訂閱

**連線與工具 (`connections/`、`myutil/`、`session/`):**
- `connections/redis.go`、`connections/postgres.go`：集中管理外部連線工廠；driver 實作都透過這裡取得連線
- `myutil/`：共用工具（diff、字串切片、UTF-8、檔案、log runtime info）——新增工具前先看這裡是否已有
- `session/`：聊天機器人對話的狀態管理（`manager.go` + memory provider）

**PTT 爬蟲 (`ptt/`):**
- `ptt/web/crawler.go`：HTML 爬蟲，抓取 www.ptt.cc 的看板頁面與文章
- `ptt/rss/`：替代方案，使用 RSS 抓取
- 處理 R18 cookie 繞過，以存取年齡限制看板

**通知頻道 (`channels/`):**
- LINE（透過 LINE Messaging API 與 LINE Notify）
- Facebook Messenger（webhook 方式）
- Telegram（透過 Bot API）
- Email（透過 Mailgun）

**指令處理 (`command/`):**
- 處理來自聊天機器人的使用者指令（新增/刪除關鍵字、作者、推文門檻）
- 指令為中文：新增/刪除（關鍵字）、新增作者/刪除作者、新增推文數/新增噓文數

### 環境變數

必要：
- `REDIS_ENDPOINT`、`REDIS_PORT`：Redis 連線設定
- `TELEGRAM_TOKEN`：Telegram bot token
- `AUTH_USER`、`AUTH_PW`：管理 API 的 Basic Auth 認證
- `BOARD_HIGH`：高優先看板（以逗號分隔，檢查頻率較高）

AWS（DynamoDB 用）：
- 標準 AWS SDK 環境變數

## Gotchas

- **Dockerfile Go 版本落差**：`Dockerfile` 仍用 `golang:1.15-alpine`，但 `go.mod` 宣告 `go 1.21` / `toolchain 1.22`。修改 Docker build 時需一併升級 base image，否則會編譯失敗
- **啟動時的一次性工作**：`main.go` 的 `init()` 會每次啟動都跑一次 `MigrateBoard`、`MigrateDB`、`Fetcher`、`Generator` 等；改動這些 job 前要確認重複執行是 idempotent
- **`BOARD_HIGH` 空值**：若未設定，`strings.Split` 會得到 `[""]`（長度 1 的空字串切片），需留意 `highBoards` 初始化邏輯
- **Graceful shutdown 僅涵蓋 HTTP Server**：背景 job（Checker 等）並未接 signal，關機時會被硬中斷

## 相關文件與資源

- `REQUEST_FLOW.md`：多個重要場景（看板查詢、背景推播、LINE Bot 新增訂閱）的 Mermaid 流程圖
- `public/`：靜態 Web UI（`index`、`line`、`messenger`、`telegram`、`docs` 等 HTML）——`main.go` 中設為 `NotFound` handler 的預設 file server
- `ecs-deploy`：shell 腳本，用於 AWS ECS 服務部署；`.github/workflows/main.yml` 的 deploy job 會呼叫它
- `.aws/`：本地開發用 AWS config／credentials 範本

## API 端點

- `GET /boards` - 列出所有看板
- `GET /boards/:boardName/articles` - 列出看板文章
- `GET /boards/:boardName/articles/:code` - 取得特定文章
- `GET /keyword/boards`、`/author/boards`、`/pushsum/boards` - 列出有訂閱的看板
- `GET/POST/PUT /users` - 使用者管理（需 basic auth）
- LINE、Messenger、Telegram 的 Webhook 端點
