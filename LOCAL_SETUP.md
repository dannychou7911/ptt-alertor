# 本地開發環境啟動指南

> 這份文件為**接手此專案但不熟 Go／後端**的開發者撰寫。目標：用 Docker 跑起所有依賴服務，讓 `./ptt-alertor` 能在本機完整運作，不需要 AWS 帳號、不需要第三方 bot 設定。

## 概念先行

| 你熟的前端對應 | Ptt-Alertor 的對應 | 用途 |
|---|---|---|
| `npm install` | `go mod download` | 下載依賴 |
| `npm run build` | `go build` | 編譯 |
| `npm start` | `./ptt-alertor` | 執行 |
| `.env.local` | `.env` | 環境變數 |
| MongoDB/PostgreSQL | Redis + DynamoDB | 資料庫 |

**資料存在哪裡**：
- **Redis**：使用者資料、訂閱、看板快取（想像成「動態狀態」）
- **DynamoDB**：文章、看板（想像成「持久性紀錄」）
- AWS 只是 DynamoDB 的原生服務商，**DynamoDB Local** 是 AWS 官方提供、跑在本機的假 DynamoDB，零成本、零帳號。

## 前置需求

- **Go 1.21+**：`brew install go`
- **Docker Desktop**：已安裝
- **AWS CLI**（用來建 DynamoDB table）：`brew install awscli`

## 啟動步驟

### 1. 啟動依賴服務（Redis + DynamoDB Local + GUI）

```bash
docker compose -f docker-compose.dev.yml up -d
```

這會在背景啟動三個容器：

| 服務 | Port | 用途 |
|---|---|---|
| `redis` | 6379 | 快取 |
| `dynamodb` | 8000 | 假 DynamoDB |
| `dynamodb-admin` | 8001 | Web GUI，打開 http://localhost:8001 就能看資料 |

確認服務起來：

```bash
docker compose -f docker-compose.dev.yml ps
```

### 2. 建立 DynamoDB Tables

程式用到兩個 table，第一次必須手動建立。複製貼上即可：

```bash
aws dynamodb create-table \
  --table-name articles \
  --attribute-definitions AttributeName=Code,AttributeType=S \
  --key-schema AttributeName=Code,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --endpoint-url http://localhost:8000

aws dynamodb create-table \
  --table-name boards \
  --attribute-definitions AttributeName=Name,AttributeType=S \
  --key-schema AttributeName=Name,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --endpoint-url http://localhost:8000
```

> 實際 key schema 參考 `models/article/dynamodb.go` 和 `models/board/dynamodb.go`。上面是從 `GetItem` 的 key 欄位推論，若跑起來遇到 `ValidationException` 再對照程式調整。

打開 http://localhost:8001 應該能看到兩個 table。

### 3. 設定環境變數

```bash
cp .env.example .env
```

預設值已經對應 `docker-compose.dev.yml` 的位置，不用改就能跑。

接著把 `.env` 載入 shell：

```bash
# 方法一：手動 source（這個檔案需自行檢查格式；nvm 風格）
set -a; source .env; set +a

# 方法二：用 direnv（推薦，需另外安裝）
brew install direnv
echo 'dotenv' > .envrc
direnv allow
```

### 4. 編譯並執行

```bash
go mod download
go build
./ptt-alertor
```

成功時會看到：

```
INFO[0000] Start Jobs
INFO[0000] Start Ptt Monitor
INFO[0000] Web Server Start on Port 9090
```

### 5. 驗證

```bash
# 首頁（會顯示 public/index.html）
open http://localhost:9090/

# 看板 API（無需認證）
curl http://localhost:9090/boards

# 使用者 API（需要 .env 裡設定的 Basic Auth）
curl -u admin:admin http://localhost:9090/users

# 建立一筆測試使用者
curl -u admin:admin -X POST http://localhost:9090/users \
  -H "Content-Type: application/json" \
  -d '{
    "profile": {"account":"test","email":"test@example.com"},
    "subscribes": [{"board":"Gossiping","keywords":["測試"]}]
  }'
```

## 常用指令

```bash
# 停止依賴服務（保留資料）
docker compose -f docker-compose.dev.yml down

# 停止並清除所有資料（下次啟動等於全新）
docker compose -f docker-compose.dev.yml down -v

# 看某個服務的 log
docker compose -f docker-compose.dev.yml logs -f redis
docker compose -f docker-compose.dev.yml logs -f dynamodb

# 執行測試
go test -race -tags test ./...

# 編譯時熱重載（需另外安裝 air）
go install github.com/air-verse/air@latest
air
```

## 哪些功能本地測不到

| 功能 | 本地可用？ | 說明 |
|---|---|---|
| HTTP API（`/boards`、`/users` 等） | ✅ | 完全可用 |
| PTT 爬蟲 | ✅ | 會連真實 `www.ptt.cc`（請勿頻繁） |
| 背景 Checker 比對訂閱 | ✅ | log 會顯示比對結果 |
| LINE／Telegram／Messenger 推送 | ❌ | 需要對應 bot token 且公開 URL |
| Bot webhook 接收 | ⚠️ | 需搭配 [ngrok](https://ngrok.com) 暴露 port 9090 |

**想測 bot webhook**：

```bash
ngrok http 9090
# 把產生的 https://xxxx.ngrok.io 填進對應 bot 平台的 webhook 設定
```

## 踩雷排查

| 症狀 | 可能原因 | 解法 |
|---|---|---|
| 啟動時 `panic: dial tcp :6379` | Redis 沒起來 | `docker compose -f docker-compose.dev.yml ps` 確認 |
| `ResourceNotFoundException: articles` | Table 沒建 | 回到步驟 2 |
| `ValidationException: key schema` | Table 的 key 欄位與程式不符 | 參考 `models/*/dynamodb.go` 的 `GetItem` 參數 |
| build 失敗 `unknown directive: toolchain` | Go 版本太舊 | 升級 Go 到 1.21 以上 |
| 反覆被 PTT 擋 | 爬太快 | 把 `jobs/checker.go` 的 `duration` 改長一點 |

## 參考

- 架構總覽：[CLAUDE.md](./CLAUDE.md)
- 請求流程圖：[REQUEST_FLOW.md](./REQUEST_FLOW.md)
- DynamoDB Local 官方文件：https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/DynamoDBLocal.html
