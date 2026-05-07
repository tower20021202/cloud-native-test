# Cloud Native Test

本專案包含 React/Vite 前端與 FastAPI 後端。以下步驟可用來在本地端啟動並測試完整流程。

## 環境需求

- Python 3.11
- Node.js / npm

## 1. 建立並啟用 Python 虛擬環境

在專案根目錄執行：

```bash
python3 -m venv .venv
source .venv/bin/activate
```

安裝後端套件：

```bash
pip install -r server/requirements.txt
```

## 2. 建立本地測試資料

後端預設使用 SQLite，不需要另外啟動 PostgreSQL。建議從 `server` 目錄執行 seed，讓 SQLite 資料庫建立在 `server/tsmc_messenger.db`。

```bash
cd server
python seed.py
```

預設測試帳號：

| Email | Password |
| --- | --- |
| admin@tsmc.com | password123 |
| alice@tsmc.com | password123 |
| bob@tsmc.com | password123 |

## 3. 啟動後端 API

仍在 `server` 目錄中執行：

```bash
uvicorn app.main:app --host 0.0.0.0 --port 3001 --reload
```

後端服務：

- API base URL: `http://localhost:3001/api`
- Health check: `http://localhost:3001/api/health`
- API docs: `http://localhost:3001/docs`
- WebSocket: `ws://localhost:3001/ws`

可以用以下指令確認後端是否正常：

```bash
curl http://localhost:3001/api/health
```

成功時會回傳：

```json
{"status":"ok","service":"TSMC Messenger API"}
```

## 4. 啟動前端

開另一個 terminal，回到專案根目錄：

```bash
npm install
npm run dev
```

前端預設會啟動在：

```text
http://localhost:5173
```

## 5. 本地端測試流程

1. 打開 `http://localhost:5173`
2. 使用預設帳號登入，例如：

```text
alice@tsmc.com
password123
```

3. 確認前端可以正常呼叫 API。
4. 若要測試即時通訊或在線狀態，可以開兩個瀏覽器視窗，分別登入不同帳號，例如 `alice@tsmc.com` 與 `bob@tsmc.com`。

## 常見問題

### `uvicorn` 找不到指令

請確認已啟用虛擬環境：

```bash
source .venv/bin/activate
```

或直接使用 venv 裡的 uvicorn：

```bash
.venv/bin/uvicorn app.main:app --host 0.0.0.0 --port 3001 --reload
```

如果目前在專案根目錄，請改成：

```bash
cd server
../.venv/bin/uvicorn app.main:app --host 0.0.0.0 --port 3001 --reload
```

### 前端連不到後端

請確認後端跑在 `3001`：

```bash
curl http://localhost:3001/api/health
```

前端預設會連到：

- `http://localhost:3001/api`
- `ws://localhost:3001/ws`

### 想重新建立測試資料庫

停止後端後刪除 SQLite 檔案，再重新執行 seed：

```bash
rm server/tsmc_messenger.db
cd server
python seed.py
```
