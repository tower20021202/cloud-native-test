# TSMC Messenger — 後端 + 資料庫實作計畫

## 目標

基於 HackMD 文件規格，為已完成的 React 前端建置完整的後端服務和資料庫，實現真正的即時通訊系統。

---

## ✅ 已確認讀取 HackMD 文件內容

| 項目 | 內容 |
|------|------|
| PostgreSQL Schema | 6 張表：users, chat_rooms, chat_room_members, messages, notifications, user_presence |
| 開發時程 | 4 週計畫（Week 1~4） |
| 技術建議 | Docker、HTTPS、Keycloak/NATS、Prometheus/Grafana |
| 資安要求 | 密碼 hash、不用 localhost、版本鎖定 |

---

## 技術選型

| 層級 | 技術 | 說明 |
|------|------|------|
| **後端框架** | Python + FastAPI | 現代 async 框架，內建 WebSocket 支援 |
| **資料庫** | PostgreSQL（直接安裝） | 依 HackMD 規格，免 Docker |
| **即時通訊** | FastAPI WebSocket | 內建支援，不需額外套件 |
| **認證** | JWT (PyJWT) + bcrypt (passlib) | Token 認證 + 密碼雜湊 |
| **Google OAuth** | httpx + Google OAuth 2.0 API | 第三方登入 |
| **ORM** | SQLAlchemy + Alembic | Python 最成熟的 ORM + 資料庫遷移 |
| **密碼安全** | passlib[bcrypt] | 業界標準密碼雜湊 |

---

## 資料庫 Schema（依 HackMD 規格）

```sql
-- 1. users (使用者基本資料表)
CREATE TABLE users (
  id              BIGSERIAL PRIMARY KEY,
  username        VARCHAR(50) UNIQUE NOT NULL,
  email           VARCHAR(255) UNIQUE,
  password_hash   TEXT,
  display_name    VARCHAR(100),
  avatar_url      TEXT,
  auth_provider   VARCHAR(30) DEFAULT 'local',
  provider_user_id VARCHAR(255),
  is_active       BOOLEAN DEFAULT TRUE,
  created_at      TIMESTAMP NOT NULL DEFAULT NOW(),
  updated_at      TIMESTAMP NOT NULL DEFAULT NOW()
);

-- 2. chat_rooms (聊天室表)
CREATE TABLE chat_rooms (
  id              BIGSERIAL PRIMARY KEY,
  name            VARCHAR(100),
  room_type       VARCHAR(10) NOT NULL DEFAULT 'direct', -- 'direct' | 'group'
  created_by      BIGINT REFERENCES users(id),
  last_message_at TIMESTAMP DEFAULT NOW(),
  created_at      TIMESTAMP NOT NULL DEFAULT NOW(),
  updated_at      TIMESTAMP NOT NULL DEFAULT NOW()
);

-- 3. chat_room_members (聊天室成員)
CREATE TABLE chat_room_members (
  id              BIGSERIAL PRIMARY KEY,
  room_id         BIGINT NOT NULL REFERENCES chat_rooms(id) ON DELETE CASCADE,
  user_id         BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  is_admin        BOOLEAN DEFAULT FALSE,
  last_read_at    TIMESTAMP DEFAULT NOW(),
  joined_at       TIMESTAMP NOT NULL DEFAULT NOW(),
  UNIQUE(room_id, user_id)
);

-- 4. messages (訊息表)
CREATE TABLE messages (
  id              BIGSERIAL PRIMARY KEY,
  room_id         BIGINT NOT NULL REFERENCES chat_rooms(id) ON DELETE CASCADE,
  sender_id       BIGINT NOT NULL REFERENCES users(id),
  content         TEXT NOT NULL,
  message_type    VARCHAR(10) DEFAULT 'text', -- 'text' | 'image' | 'file'
  created_at      TIMESTAMP NOT NULL DEFAULT NOW()
);

-- 5. notifications (通知表)
CREATE TABLE notifications (
  id              BIGSERIAL PRIMARY KEY,
  user_id         BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  type            VARCHAR(30) NOT NULL,
  content         TEXT,
  is_read         BOOLEAN DEFAULT FALSE,
  reference_id    BIGINT,
  created_at      TIMESTAMP NOT NULL DEFAULT NOW()
);

-- 6. user_presence (在線狀態表)
CREATE TABLE user_presence (
  id              BIGSERIAL PRIMARY KEY,
  user_id         BIGINT UNIQUE NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  status          VARCHAR(10) DEFAULT 'offline', -- 'online' | 'offline' | 'away'
  last_seen_at    TIMESTAMP DEFAULT NOW()
);

-- 效能索引
CREATE INDEX idx_messages_room_created_at ON messages(room_id, created_at);
CREATE INDEX idx_chat_room_members_user_id ON chat_room_members(user_id);
CREATE INDEX idx_chat_room_members_room_id ON chat_room_members(room_id);
CREATE INDEX idx_chat_rooms_last_message_at ON chat_rooms(last_message_at);
CREATE INDEX idx_notifications_user_is_read ON notifications(user_id, is_read);
CREATE INDEX idx_user_presence_status ON user_presence(status);
```

---

## API 端點設計

### 認證 API (`/api/auth`)

| Method | Path | 說明 | 資安 |
|--------|------|------|------|
| POST | `/api/auth/register` | 註冊（bcrypt 雜湊密碼） | 密碼至少 6 字元 |
| POST | `/api/auth/login` | 登入（回傳 JWT） | 比對 bcrypt hash |
| GET | `/api/auth/google` | Google OAuth 入口 | Passport.js |
| GET | `/api/auth/google/callback` | Google 回調 | 自動建立/登入帳號 |
| GET | `/api/auth/me` | 取得當前使用者 | 需 JWT |
| POST | `/api/auth/logout` | 登出 | 清除 presence |

### 聊天室 API (`/api/chatrooms`)

| Method | Path | 說明 |
|--------|------|------|
| GET | `/api/chatrooms` | 取得使用者的聊天室列表（含未讀數） |
| POST | `/api/chatrooms` | 建立聊天室（1 對 1 或群組） |
| GET | `/api/chatrooms/:id/messages` | 取得歷史訊息（分頁） |
| POST | `/api/chatrooms/:id/messages` | 發送訊息 |
| PUT | `/api/chatrooms/:id/read` | 標記已讀 |

### 使用者 API (`/api/users`)

| Method | Path | 說明 |
|--------|------|------|
| GET | `/api/users/search?q=` | 搜尋使用者（by username/email） |
| GET | `/api/users/:id` | 取得使用者資料 |

### WebSocket 事件

| 事件 | 方向 | 說明 |
|------|------|------|
| `new_message` | Server → Client | 即時推送新訊息 |
| `user_online` | Server → Client | 使用者上線通知 |
| `user_offline` | Server → Client | 使用者離線通知 |
| `typing` | Client ↔ Server | 正在輸入提示 |

---

## 資安措施

| 項目 | 實作方式 |
|------|---------|
| **密碼儲存** | bcrypt 雜湊，salt rounds = 12，**絕不**存明文 |
| **JWT Token** | 包含 userId + email，1 天過期，使用 HS256 簽章 |
| **API 保護** | 所有 `/api/chatrooms` 和 `/api/users` 需 JWT 驗證 |
| **SQL 注入防護** | 使用 Prisma ORM 的參數化查詢 |
| **XSS 防護** | 訊息內容前端已做 React 自動跳脫 |
| **CORS** | 限定前端 origin |
| **密碼驗證** | 至少 6 字元，註冊時檢查 |

---

## 專案結構

```
cloud native/
├── server/                        # Python 後端
│   ├── app/
│   │   ├── __init__.py
│   │   ├── main.py                # FastAPI 進入點 + WebSocket
│   │   ├── config.py              # 環境設定（DB URL、JWT SECRET）
│   │   ├── database.py            # SQLAlchemy 連線設定
│   │   ├── models.py              # ORM 資料模型（6 張表）
│   │   ├── schemas.py             # Pydantic 請求/回應格式
│   │   ├── auth.py                # JWT + bcrypt 認證邏輯
│   │   ├── routers/
│   │   │   ├── auth_router.py     # 認證路由
│   │   │   ├── chatroom_router.py # 聊天室路由
│   │   │   └── user_router.py     # 使用者路由
│   │   └── ws_manager.py          # WebSocket 連線管理
│   ├── alembic/                   # 資料庫遷移
│   ├── requirements.txt           # Python 依賴
│   └── .env                       # 環境變數
├── src/                           # 前端（已完成，改造 store）
│   ├── store/
│   │   ├── useAuthStore.js        # 改為呼叫 /api/auth
│   │   └── useChatStore.js        # 改為呼叫 /api/chatrooms + WebSocket
│   └── ...
└── ...
```

---

## 前端 Store 改造

### useAuthStore.js
- `login()` → `POST /api/auth/login` → 儲存 JWT 至 localStorage
- `register()` → `POST /api/auth/register`
- `loginWithGoogle()` → 重定向至 `/api/auth/google`
- 初始化時檢查 localStorage 的 JWT，呼叫 `/api/auth/me` 驗證

### useChatStore.js
- 初始化時 `GET /api/chatrooms` 載入聊天室列表
- `sendMessage()` → `POST /api/chatrooms/:id/messages` + WebSocket 發送
- 新增 WebSocket 連線，監聽 `new_message` 即時更新
- `createChat()` → `POST /api/chatrooms`
- 歷史訊息 → `GET /api/chatrooms/:id/messages?page=&limit=`
- 移除所有 mock 資料和 setTimeout 模擬

---

## PostgreSQL 安裝（免 Docker）

1. 前往 https://www.postgresql.org/download/windows/ 下載安裝
2. 安裝時記住設定的密碼
3. 建立資料庫：
```sql
CREATE DATABASE tsmc_messenger;
```
4. `.env` 設定：
```
DATABASE_URL=postgresql://postgres:<你的密碼>@localhost:5432/tsmc_messenger
JWT_SECRET=<隨機字串>
```

---

## 實作順序

### Phase 1：後端基礎（FastAPI + PostgreSQL + SQLAlchemy）
1. 建立 `server/` 目錄、Python venv 虛擬環境
2. 安裝 PostgreSQL（本地）、建立資料庫
3. SQLAlchemy models + Alembic migrate
4. FastAPI 伺服器啟動

### Phase 2：認證系統
1. 註冊 API（passlib bcrypt 密碼雜湊）
2. 登入 API（PyJWT 產生 Token）
3. JWT dependency 驗證
4. Google OAuth（模擬，之後可接真實憑證）

### Phase 3：聊天功能
1. 聊天室 CRUD API
2. 訊息發送/查詢 API
3. FastAPI WebSocket 即時推送
4. 未讀計數邏輯

### Phase 4：前端改造
1. 改造 useAuthStore → 真實 API
2. 改造 useChatStore → 真實 API + WebSocket
3. 刪除 mock/ 資料夾

### Phase 5：進階功能
1. 在線狀態（user_presence）
2. 通知系統
3. 搜尋使用者

---

## Open Questions

> [!IMPORTANT]
> **Docker**：你的電腦有安裝 Docker Desktop 嗎？PostgreSQL 需要透過 Docker 運行。如果沒有 Docker，我可以改用 SQLite 作為開發資料庫（不需要額外安裝），但最終部署建議使用 PostgreSQL。

> [!IMPORTANT]
> **Google OAuth**：真正的 Google OAuth 需要到 Google Cloud Console 建立 OAuth 2.0 憑證（Client ID + Client Secret）。你是否已經有 Google Cloud 專案？還是我先用模擬方式，等你準備好再接入？

> [!NOTE]
> **即時通訊方式**：HackMD 提到可以用 NATS 做高效能訊息，但考量開發效率，我計畫先用原生 WebSocket（ws 套件）實作，效能已足夠應付 demo。未來需要水平擴展時再引入 NATS/Redis Pub-Sub。
