# PTT-Alertor 請求流程圖

## 📍 場景：使用者查詢看板文章
**GET /boards/gossiping/articles**

```mermaid
graph TD
    A["🌐 使用者請求<br/>GET /boards/gossiping/articles"] -->|HTTP Request| B["📨 HTTP Router<br/>main.go:98"]
    B -->|日誌記錄| C["🔍 Request Logger<br/>ServeHTTP"]
    C -->|路由匹配| D["🎯 Controller<br/>BoardArticleIndex"]
    
    D -->|建立實例| E["📦 Models Factory<br/>models.NewBoard"]
    E -->|選擇驅動| F{{"🔀 Driver 抽象層<br/>interface Board"}}
    
    F -->|讀取| G["💾 DynamoDB/Redis<br/>board.go:FetchArticles"]
    
    G -->|查詢看板| H["📄 看板資料<br/>articles: []Article"]
    H -->|序列化| I["🔄 JSON Marshal<br/>article.go"]
    
    I -->|回傳| J["✅ HTTP Response<br/>application/json"]
    J -->|日誌記錄| K["📝 Log with Fields<br/>method, IP, URI"]
    K -->|顯示| L["👤 瀏覽器/客戶端<br/>articles list JSON"]
```

---

## 📍 場景：後台監控新文章並發送通知
**背景工作流程**

```mermaid
graph TD
    Start["🚀 應用啟動<br/>main.go:init()"] -->|啟動工作| J1["Checker"]
    Start -->|啟動工作| J2["PushSumChecker"]
    Start -->|啟動工作| J3["CommentChecker"]
    Start -->|啟動工作| J4["PttMonitor"]
    
    J1 -->|每 250ms| Check1["🔄 檢查高優先看板<br/>checkHighBoardDuration"]
    J1 -->|每 500ms| Check2["🔄 檢查一般看板<br/>checkBoards"]
    
    Check1 -->|發現新文章| Chan1["📢 Board Channel<br/>boardCh"]
    Check2 -->|發現新文章| Chan1
    
    Chan1 -->|關鍵字訂閱者| Sub1["👥 checkKeywordSubscriber"]
    Chan1 -->|作者訂閱者| Sub2["👥 checkAuthorSubscriber"]
    
    Sub1 -->|比對關鍵字| Match1{{"✓ 符合訂閱<br/>MatchKeyword"}}
    Sub2 -->|比對作者| Match2{{"✓ 符合訂閱<br/>Author Match"}}
    
    Match1 -->|YES| Notify["📤 Checker.ch<br/>通知佇列"]
    Match2 -->|YES| Notify
    
    Notify -->|分派| Worker["⚙️ Worker Pool<br/>300 workers"]
    Worker -->|傳遞訊息| Dispatcher["🔀 check.go<br/>分派到對應頻道"]
    
    Dispatcher -->|LINE| Ch1["📱 channels/line<br/>Notify/API"]
    Dispatcher -->|Messenger| Ch2["💬 channels/messenger<br/>Webhook"]
    Dispatcher -->|Telegram| Ch3["✈️ channels/telegram<br/>Bot API"]
    Dispatcher -->|Email| Ch4["📧 channels/mail<br/>Mailgun"]
    
    Ch1 -->|HTTP POST| API1["🌐 LINE Notify API"]
    Ch2 -->|HTTP POST| API2["🌐 Messenger API"]
    Ch3 -->|HTTP POST| API3["🌐 Telegram Bot API"]
    Ch4 -->|HTTP POST| API4["🌐 Mailgun API"]
    
    API1 -->|推送| User1["👤 使用者手機<br/>LINE 通知"]
    API2 -->|推送| User2["👤 Facebook<br/>Messenger"]
    API3 -->|推送| User3["👤 Telegram App"]
    API4 -->|推送| User4["👤 Email"]
```

---

## 📍 場景：使用者透過 LINE Bot 新增訂閱
**POST /line/callback**

```mermaid
sequenceDiagram
    participant User as 👤 使用者
    participant LineME as LINE App
    participant Server as 🖥️ Ptt-Alertor<br/>Port 9090
    participant Command as 🎯 command/
    participant Models as 📦 models/
    participant Redis as 💾 Redis
    participant LineBotAPI as 🌐 LINE Bot API

    User ->> LineME: 傳訊息<br/>"新增 八卦 問卦"
    LineME ->> Server: POST /line/callback<br/>webhook event
    
    Server ->> Server: Log request<br/>method, IP, URI
    Server ->> Command: HandleRequest<br/>line.go
    
    Command ->> Command: 解析訊息<br/>Parse text
    Command ->> Command: 中文指令解析<br/>actionType<br/>board<br/>keyword
    
    Command ->> Models: user.Find(lineID)
    Models ->> Redis: GET user:lineID
    Redis -->> Models: User 物件
    Models -->> Command: 使用者資料
    
    Command ->> Models: subscription.Add<br/>board + keyword
    Models ->> Redis: HSET subscription:lineID<br/>board → keywords
    Redis -->> Models: ✓ saved
    
    Command ->> LineBotAPI: LineNotify()<br/>HTTP POST
    LineBotAPI -->> LineME: "✅ 訂閱成功"
    LineME -->> User: 顯示確認訊息
    
    Note over Server: 後續 Checker Job<br/>會持續監控此看板
```

---

## 📍 詳細：Controller → Models → Driver 流程

```mermaid
graph LR
    A["🎯 Controller<br/>controllers/board.go<br/>BoardArticleIndex"] -->|1. 呼叫| B["📦 Models Factory<br/>models.Board"]
    
    B -->|2. 返回| C["🏭 Board 實例<br/>models/board/board.go"]
    
    C -->|3. 委派操作| D{{"🔀 Driver 介面<br/>interface Board<br/>getArticles<br/>all<br/>update"}}
    
    D -->|4. 具體實作| E1["🗄️ DynamoDB Driver<br/>models/board/dynamodb.go"]
    D -->|4. 具體實作| E2["🗄️ Redis Driver<br/>models/board/redis.go"]
    D -->|4. 具體實作| E3["🗄️ File Driver<br/>models/board/file.go"]
    
    E1 -->|5. 查詢| F1["AWS DynamoDB<br/>Region: us-west-2<br/>Table: boards"]
    E2 -->|5. 查詢| F2["Redis 記憶體<br/>Key pattern:<br/>board:boardName"]
    E3 -->|5. 讀取| F3["本地 JSON 檔案<br/>storage/boards/"]
    
    F1 -->|6. 返回資料| G["📊 Articles 陣列"]
    F2 -->|6. 返回資料| G
    F3 -->|6. 返回資料| G
    
    G -->|7. 序列化| H["🔄 JSON Marshal<br/>encoding/json"]
    H -->|8. 回傳| I["✅ HTTP 200 OK<br/>application/json"]
```

---

## 📍 資料流：Article Model 的 Driver 抽象

```mermaid
graph TD
    A["Article 結構體<br/>article.go<br/>id, code, title, link<br/>author, board, pushSum"] -->|包含| B["🔀 Driver 介面<br/>interface Driver<br/>Find<br/>Save<br/>Delete"]
    
    B -->|實作選項| C1["DynamoDB 驅動<br/>models/article/dynamodb.go<br/>AWS SDK"]
    B -->|實作選項| C2["Redis 驅動<br/>models/article/redis.go<br/>Redigo"]
    
    C1 -->|DynamoDB 特性| D1["✓ 分散式<br/>✓ 高可用<br/>✓ 成本較高<br/>✗ 啟動複雜"]
    
    C2 -->|Redis 特性| D2["✓ 超快速<br/>✓ 無須設定<br/>✗ 記憶體有限<br/>✗ 斷電遺失"]
    
    Note over B: 透過 Factory 模式<br/>models/models.go<br/>自動選擇驅動實作
```

---

## 📍 通知發送佇列系統

```mermaid
graph TD
    A["Checker Job<br/>發現符合訂閱的新文章"] -->|加入佇列| B["📢 Checker.ch<br/>channel"]
    
    B -->|接收| C["check.go<br/>checkMessage func"]
    
    C -->|分派| D["⚙️ Worker Pool<br/>300 workers"]
    
    D -->|讀取訊息| E["Checker 物件<br/>board<br/>keyword<br/>author<br/>articles"]
    
    E -->|查詢使用者| F["User 物件<br/>Profile<br/>Line/Messenger/Telegram<br/>Mail 端點"]
    
    F -->|判斷通知方式| G{{"多頻道通知<br/>條件判斷"}}
    
    G -->|Line Enable| H1["channels/line<br/>Notify函式"]
    G -->|Messenger Enable| H2["channels/messenger<br/>Send函式"]
    G -->|Telegram Enable| H3["channels/telegram<br/>SendMessage函式"]
    G -->|Mail Enable| H4["channels/mail<br/>Send函式"]
    
    H1 -->|HTTP POST| I1["LINE Notify API<br/>request.go"]
    H2 -->|HTTP POST| I2["Messenger API<br/>http.go"]
    H3 -->|HTTP POST| I3["Telegram Bot API<br/>telegram.go"]
    H4 -->|HTTP POST| I4["Mailgun API<br/>mail.go"]
    
    I1 -->|成功| J["✅ 使用者收到通知"]
    I2 -->|成功| J
    I3 -->|成功| J
    I4 -->|成功| J
```

---

## 📍 HTTP 請求完整生命週期

```mermaid
graph TD
    A["🌐 CLIENT<br/>GET /boards/gossiping/articles"] -->|TCP 連線| B["🖥️ HTTP Server<br/>Port 9090"]
    
    B -->|1. 接收| C["myRouter.ServeHTTP<br/>main.go:30"]
    
    C -->|2. 日誌| D["log.WithFields<br/>method, IP, URI"]
    
    D -->|3. 路由查詢| E["httprouter<br/>julienschmidt/httprouter"]
    
    E -->|4. 找到| F["GET /boards/:boardName/articles<br/>→ BoardArticleIndex"]
    
    F -->|5. 執行| G["func BoardArticleIndex<br/>controllers/board.go"]
    
    G -->|6. 建立| H["models.Board<br/>factory pattern"]
    
    H -->|7. 設定參數| I["bd.Name = GOSSIPING"]
    
    I -->|8. 呼叫| J["bd.FetchArticles"]
    
    J -->|9. 呼叫 Driver| K["DynamoDB / Redis<br/>實作"]
    
    K -->|10. 查詢| L["資料庫<br/>articles table"]
    
    L -->|11. 返回| M["article.Articles<br/>[]Article"]
    
    M -->|12. 序列化| N["json.Marshal<br/>encoding/json"]
    
    N -->|13. 寫入| O["http.ResponseWriter<br/>w"]
    
    O -->|14. 發送| P["HTTP 200 OK<br/>Content-Type: application/json"]
    
    P -->|TCP| Q["🌐 CLIENT<br/>收到 JSON 回應"]
    
    Q -->|渲染| R["瀏覽器/App<br/>展示文章列表"]
```

---

## 📍 時序圖：背景監控工作

```mermaid
sequenceDiagram
    participant Main as main.go
    participant Checker as Checker Job
    participant PttWeb as PTT Web<br/>www.ptt.cc
    participant DB as Redis/DynamoDB
    participant UserModel as User Model
    participant Notifier as Notifier

    Main ->> Checker: go jobs.NewChecker().Run()
    activate Checker
    
    loop 每 250ms (高優先看板)
        Checker ->> Checker: 檢查高優先看板
        Checker ->> PttWeb: 爬蟲抓新文章
        PttWeb -->> Checker: HTML 頁面
    end
    
    loop 每 500ms (一般看板)
        Checker ->> Checker: 檢查一般看板
        Checker ->> PttWeb: 爬蟲抓新文章
        PttWeb -->> Checker: HTML 頁面
    end
    
    Note over Checker: 發現新文章 code=M.2024...
    Checker ->> DB: 檢查文章是否已存在
    DB -->> Checker: false (新文章!)
    
    Checker ->> Checker: 將看板發送至 boardCh
    Checker ->> Checker: checkKeywordSubscriber
    Checker ->> UserModel: 查詢訂閱此看板+關鍵字的使用者
    UserModel ->> DB: HGETALL subscription:board:keyword
    DB -->> UserModel: users list
    UserModel -->> Checker: []User
    
    Note over Checker: 比對每個使用者
    Checker ->> Checker: user.MatchKeyword(article.Title)
    
    alt 符合訂閱條件
        Checker ->> Notifier: 將訊息加入佇列
        Note over Notifier: 由 300 workers 並行處理
        Notifier ->> Notifier: 判斷使用者啟用的通知管道
        Notifier ->> Notifier: 呼叫 LINE/Telegram/等 API
    else 不符合
        Note over Checker: 略過此使用者
    end
    
    deactivate Checker
```

---

## 📍 環境變數 → 程式碼對應

```mermaid
graph LR
    A["環境變數<br/>.env / shell"] -->|設定| B["os.Getenv"]
    
    B -->|REDIS_ENDPOINT| C["connections/redis.go<br/>Redis 連線"]
    B -->|DB_CONNECTION| D["connections/postgres.go<br/>PostgreSQL / DynamoDB"]
    
    B -->|TELEGRAM_TOKEN| E["main.go:22<br/>telegramToken"]
    B -->|AUTH_USER| F["main.go:23<br/>authUser"]
    B -->|AUTH_PW| G["main.go:24<br/>authPassword"]
    
    B -->|BOARD_HIGH| H["jobs/checker.go:24<br/>highBoardNames"]
    
    B -->|LINE_CLIENT_ID| I["channels/line/notify.go<br/>LINE OAuth"]
    B -->|APP_HOST| J["channels/line/notify.go<br/>Callback URL"]
    
    C -->|初始化| K["models/board/redis.go<br/>快取層"]
    D -->|初始化| L["models/article/dynamodb.go<br/>持久層"]
    E -->|初始化| M["channels/telegram<br/>Bot 連線"]
    H -->|使用| N["Checker<br/>優先檢查"]
```

---

## 🎯 核心設計模式

### 1. **Driver 抽象層 (Strategy Pattern)**
```go
// 介面定義
type Driver interface {
    Find(code string, article *Article)
    Save(a Article) error
    Delete(code string) error
}

// 多個實作
- article.DynamoDB (implements Driver)
- article.Redis (implements Driver)

// Factory 建立
var Article = func() *article.Article {
    return article.NewArticle(new(article.DynamoDB))
}
```

### 2. **工作佇列 (Producer-Consumer Pattern)**
- **Producer**: Checker job 發現新文章
- **Channel**: `boardCh`, `Checker.ch`
- **Consumer**: Worker pool (300 workers)

### 3. **中文指令解析 (Command Pattern)**
- 使用者輸入: `"新增 八卦 問卦"`
- Parser: 解析關鍵字、看板、指令類型
- Executor: 執行訂閱操作

---

## 📊 資料流向摘要

```
Client 請求
  ↓
HTTP Router (main.go)
  ↓
Controller (controllers/)
  ↓
Models Factory (models/models.go)
  ↓
Driver 介面 (interface)
  ↓
具體實作 (DynamoDB/Redis/File)
  ↓
外部服務 (AWS/Redis/Local)
  ↓
HTTP 回應
```

---

## 🔄 背景工作監控流程

```
↓ 應用啟動 ↓
  │
  ├─→ Checker (檢查新文章)
  │    ├─ High Priority Boards (每 250ms)
  │    └─ Normal Boards (每 500ms)
  │
  ├─→ PushSumChecker (監控推文數)
  │
  ├─→ CommentChecker (監控新推文)
  │
  ├─→ PttMonitor (監控看板狀態)
  │
  └─→ Cron Jobs
       ├─ Top (每小時)
       └─ PushSumKeyReplacer (每 48 小時)

      ↓ 發現符合訂閱的文章 ↓
          │
          ├─ 比對關鍵字
          ├─ 比對作者
          └─ 比對推文數門檻
              ↓
          符合 → 加入通知佇列
              ↓
          Worker Pool (300 workers)
              ↓
          選擇通知管道 (LINE/Telegram/Messenger/Mail)
              ↓
          呼叫外部 API
              ↓
          使用者收到通知
```

