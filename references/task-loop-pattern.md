# 任務閉環循環模式（n8n → Hermes → Notion）

## 場景

n8n 查詢 Notion 資料庫中「非已完成」的待辦任務，透過 Telegram 推送清單給 Hermes Agent。Agent 完成任務後須更新 Notion 狀態，並觸發 n8n 獲取下一項任務。

## 推薦架構：Webhook 觸發（取代 REST API run）

相較於 buggy 的 `/rest/workflows/{id}/run` API（n8n 2.19.5 的 `startNodes` 參數報 `undefined`），**使用 Webhook 節點作為觸發器更可靠**：

```
建立專屬 Webhook 工作流 →
  1. Webhook 觸發 (POST /webhook/hermes-task-loop)
  2. HTTP Request (查 Notion 未完成任務)
  3. Code Node (格式化 + 按優先級排序)
  4. IF Node (myResult !== 'skip')
  5. Telegram (發送待辦清單)
```

### 觸發方式

```bash
curl -s -X POST 'http://localhost:5678/webhook/hermes-task-loop' \
  -H 'Content-Type: application/json' \
  -d '{"source":"hermes","action":"task-updated"}'
```

回傳 `{"message":"Workflow was started"}` 即成功。

## 完整循環

```
Hermes 完成任務
  │
  ▼  PATCH /v1/pages/{id} (狀態 → 已完成)
  │
  ▼  POST /webhook/hermes-task-loop (觸發 n8n)
  │
  ▼  n8n 工作流：
  │     ├─ HTTP Request → 查 Notion (filter: 狀態 ≠ 已完成)
  │     ├─ Code Node → 格式化 + 排序
  │     ├─ IF Node → 檢查是否有任務
  │     └─ Telegram → 發送待辦清單
  │
  ▼  Hermes 接收 Telegram
  │     ├─ 解析清單，取出第一項
  │     ├─ 執行任務
  │     └─ 回到循環開始
```

## 關鍵細節

### 觸發 n8n 的正確方式

**推薦：使用 Webhook 工作流（POST /webhook/{path}）**

建立一個專用 webhook 工作流，每次任務更新後發送 POST 請求：

```bash
curl -s -X POST 'http://localhost:5678/webhook/hermes-task-loop' \
  -H 'Content-Type: application/json' \
  -d '{"source":"hermes","action":"task-updated"}'
```

優點：
- 不需要 cookie/認證
- 不會有 `/rest/workflows/{id}/run` 的 `undefined` bug
- 工作流激活後立即可用
- 可帶入 payload 給後續節點使用

**替代方案：使用 REST API（當無法用 webhook 時）**

```python
trigger_resp = terminal("""
curl -s -b /tmp/n8n_cookies.txt -X POST \
  http://localhost:5678/rest/workflows/{WF_ID}/run \
  -H 'Content-Type: application/json' \
  -d '{"triggerToStartFrom": {"mode": "trigger", "nodeName": "每小時觸發"}}'
""", timeout=15)
```
```

### 執行狀態檢查

```python
import time
time.sleep(5)  # 等 n8n 跑完
check = terminal(f"curl -s -b /tmp/n8n_cookies.txt \
  'http://localhost:5678/rest/executions/{exec_id}'", timeout=10)
status = json.loads(check["output"])["data"]["status"]
# 可能的值: "new", "running", "success", "error", "waiting", "unknown"
```

### Notion 狀態更新

```bash
curl -s -X PATCH "https://api.notion.com/v1/pages/{page_id}" \
  -H "Authorization: Bearer $NOTION_API_KEY" \
  -H "Notion-Version: 2025-09-03" \
  -H "Content-Type: application/json" \
  -d '{"properties": {"狀態": {"select": {"name": "已完成"}}}}'
```

### 注意事項

- 先更新 Notion，再觸發 n8n — 確保查詢的是最新資料
- 不要在觸發 n8n 後直接去 Notion 查詢下一項 — 等 Telegram 消息送達
- Cookie 可能過期 — 觸發 n8n 前先確認 cookie 有效性