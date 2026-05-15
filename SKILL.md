---
name: hermes-n8n-integration
description: Hermes Agent 透過 REST API 管理 n8n 工作流的完整流程 — 登入、查詢、建立、修復、激活、驗證
---

# Hermes ↔ n8n 整合管理

從 Hermes Agent 直接管理 n8n 工作流的標準流程，**不需要打開瀏覽器**。

## 觸發條件

- 用戶要求「透過 Hermes 管理 n8n」
- 需要修復 n8n 工作流執行錯誤（400 Bad Request、jsonBody 問題、credential 問題）
- 需要建立新的 n8n 工作流
- 需要查詢工作流狀態、執行歷史
- n8n browser session 不穩定時，使用 API 方式管理
- Notion API 回傳 400 validation_error（Select 屬性值不匹配）
- 需要修改 Notion filter 條件或節點名稱

## 前置準備

### 1. 確認 cookie 有效

```python
from hermes_tools import terminal
import json

# 檢查 cookie 有效性
check = terminal("curl -s -b /tmp/n8n_cookies.txt 'http://localhost:5678/rest/workflows?limit=1'", timeout=10)
try:
    result = json.loads(check["output"])
    if result.get("data"):
        print("✅ Cookie valid")
    else:
        print("❌ Need re-login")
except:
    print("❌ Need re-login")
```

### 2. 重新登入（cookie 過期時）

```python
# 登入並存 cookie
login = terminal("""
curl -s -c /tmp/n8n_cookies.txt -X POST http://localhost:5678/rest/login \
  -H "Content-Type: application/json" \
  -d '{"emailOrLdapLoginId":"admin@hermes.local","password":"admin@hermes.local"}'
""", timeout=10)
```

> ⚠️ 字段是 `emailOrLdapLoginId`，不是 `email`！用錯會報 `Required`

## 標準流程

### 列出所有工作流

```python
result = terminal("curl -s -b /tmp/n8n_cookies.txt http://localhost:5678/rest/workflows", timeout=10)
workflows = json.loads(result["output"])["data"]
for w in workflows:
    print(f"{w['id']} | {w['name']} | active={w['active']} | versionId={w.get('versionId','?')[:12]}")
```

### 讀取工作流完整內容

```python
result = terminal(f"curl -s -b /tmp/n8n_cookies.txt http://localhost:5678/rest/workflows/{wf_id}", timeout=10)
wf = json.loads(result["output"])["data"]
nodes = wf.get("nodes", [])
for n in nodes:
    print(f"  {n['name']} ({n['type']})")
```

### 修復 HTTP Request Node（jsonBody + contentType）

這是 **最常見的故障模式** — n8n 2.19.5 的 HTTP Request V3 node (typeVersion 4.2) 有兩個陷阱：

**陷阱 1**：jsonBody 必須是 raw dict，不是字串
**陷阱 2**：即使 jsonBody 是 dict，也必須設 `contentType: "json"` 和 `specifyBody: "json"`

#### 完整的修復流程 (deactivate → patch → activate)

```python
import json
from hermes_tools import terminal
import json as j

WF_ID = "你的工作流ID"

# Step 1: Deactivate
terminal(f"""
curl -s -b /tmp/n8n_cookies.txt -X POST http://localhost:5678/rest/workflows/{WF_ID}/deactivate
""", timeout=10)

# Step 2: Get workflow
result = terminal(f"curl -s -b /tmp/n8n_cookies.txt http://localhost:5678/rest/workflows/{WF_ID}", timeout=10)
wf = j.loads(result["output"])["data"]

# Step 3: Fix all httpRequest nodes
for node in wf["nodes"]:
    if node["type"] != "n8n-nodes-base.httpRequest":
        continue
    p = node.get("parameters", {})
    # Fix jsonBody type (if stored as string)
    if isinstance(p.get("jsonBody"), str):
        p["jsonBody"] = j.loads(p["jsonBody"])
    # Fix contentType — CRITICAL for n8n 2.19.5
    p["contentType"] = "json"
    p["specifyBody"] = "json"
    # Ensure headers are present
    p["sendHeaders"] = True
    if not p.get("headerParameters", {}).get("parameters"):
        p["headerParameters"] = {"parameters": [
            {"name": "Authorization", "value": "Bearer YOUR_TOKEN"},
            {"name": "Notion-Version", "value": "2022-06-28"},
            {"name": "Content-Type", "value": "application/json"}
        ]}

# Step 4: Patch (will produce new versionId)
payload = j.dumps({
    "name": wf["name"],
    "nodes": wf["nodes"],
    "connections": wf.get("connections"),
    "settings": wf.get("settings", {}),
    "staticData": wf.get("staticData"),
    "pinData": wf.get("pinData", {}),
    "versionId": wf["versionId"]
}, ensure_ascii=False)

terminal(f"cat > /tmp/wf_patch.json << 'HERMESEOF'\n{payload}\nHERMESEOF", timeout=5)

patch = terminal(f"""
curl -s -b /tmp/n8n_cookies.txt -X PATCH http://localhost:5678/rest/workflows/{WF_ID} \
  -H 'Content-Type: application/json' \
  -d @/tmp/wf_patch.json
""", timeout=10)
new_vid = j.loads(patch["output"])["data"]["versionId"]

# Step 5: Activate with new versionId
terminal(f"""
curl -s -b /tmp/n8n_cookies.txt -X POST http://localhost:5678/rest/workflows/{WF_ID}/activate \
  -H 'Content-Type: application/json' \
  -d '{{"versionId": "{new_vid}"}}'
""", timeout=15)

print("✅ Workflow fixed and reactivated!")
```

> ⚠️ **activate 需要傳 `versionId`**！n8n 2.19.5 的 activate API 會驗證 versionId，不傳會報 400 `Required`。
> ⚠️ 每次 PATCH 後 versionId 會變，必須用最新的 versionId 才能 activate。
> ⚠️ 如果跳過 deactivate 直接 patch，versionId 可能不會更新。**一定要先 deactivate。**

#### 手動觸發驗證 (含 Telegram 發送確認)

```python
# 手動觸發執行（需要指定 trigger node）
trigger = terminal(f"""
curl -s -b /tmp/n8n_cookies.txt -X POST http://localhost:5678/rest/workflows/{WF_ID}/run \
  -H 'Content-Type: application/json' \
  -d '{{"triggerToStartFrom": {{"mode": "trigger", "nodeName": "每小時觸發"}}}}'
""", timeout=15)
exec_id = json.loads(trigger["output"]).get("data", {}).get("executionId")

# 等幾秒後檢查結果
import time
time.sleep(5)

check = terminal(f"curl -s -b /tmp/n8n_cookies.txt 'http://localhost:5678/rest/executions/{exec_id}'", timeout=10)
status = json.loads(check["output"])["data"]["status"]
print(f"Execution {exec_id}: {status}")
```

> ⚠️ /rest/workflows/{id}/run API 在 n8n 2.19.5 有 bug：
> - 方式 A (`startNodes` + `destinationNode`)：報 `Could not find a node named "undefined"`
> - 方式 B (`triggerToStartFrom`)：✅ **可以成功建立 execution**
> 
> 所以一定要用方式 B。

### 查詢執行歷史與錯誤排查

```python
# 列出最近 5 次執行
result = terminal(f"curl -s -b /tmp/n8n_cookies.txt 'http://localhost:5678/rest/executions?workflowId={WF_ID}&limit=5'", timeout=10)
data = json.loads(result["output"])["data"]
for e in data.get("results", []):
    print(f"ID:{e['id']} Status:{e['status']} Mode:{e['mode']} Started:{e.get('startedAt','')[:19]}")

# 查看錯誤詳情
detail = terminal(f"curl -s -b /tmp/n8n_cookies.txt 'http://localhost:5678/rest/executions/{EXEC_ID}?includeData=true'", timeout=10)
text = detail["output"]

# 常見錯誤模式檢查
if "body failed validation" in text:
    print("❌ jsonBody 格式問題 — 檢查 contentType / jsonBody 型別")
elif "Bad request" in text and "body" in text:
    print("❌ Notion API 拒絕請求 — 檢查 body 格式和 Authorization header")
elif "Found credential with no ID" in text:
    print("❌ Credential 錯誤 — 改用 sendHeaders 手動送 token")
elif "Unknown error" in text and "JsTaskRunnerSandbox" in text:
    print("❌ Code Node task runner 崩潰 — 重啟 n8n 或改用 HTTP Request node")
```

### 修改工作流內容（節點改名陷阱 + 重建 payload 流程）

**核心原則：只改參數時寧可保留舊名稱，別改名。名稱只是 UI 顯示，不影響行為。**

#### 輕量修改（只改 filter 值、參數）

直接在 `nodes` 陣列中找到目標節點修改即可。**不要動 `name` 欄位。**

```python
from hermes_tools import terminal
import json

result = terminal(f"curl -s -b /tmp/n8n_cookies.txt http://localhost:5678/rest/workflows/{WF_ID}", timeout=10)
wf = json.loads(result['output'])['data']

# 找到目標節點，只改參數
for node in wf['nodes']:
    if node['name'] == '查詢待辦任務':
        # 例：將 select.equals 改為 select.does_not_equal
        f = node['parameters']['jsonBody']['filter']['select']
        val = f.pop('equals', None)
        if val:
            f['does_not_equal'] = '已完成'
# 保留 connections 不動

# 用 Python 寫檔，避免 shell 轉義 Unicode
payload = {'name': wf['name'], 'nodes': wf['nodes'], 'connections': wf['connections'],
           'settings': wf.get('settings',{}), 'staticData': wf.get('staticData'),
           'pinData': wf.get('pinData',{}), 'versionId': wf['versionId']}
with open('/tmp/wf_patch.json', 'w', encoding='utf-8') as f:
    json.dump(payload, f, ensure_ascii=False)
```

#### 改名（需要重建整個 payload）

如果**必須改名**（如節點名稱已過時），必須同步更新四處：
1. `nodes` 陣列中該節點的 `name`
2. `connections` 中以舊名作為 key 的 output
3. `connections` 中所有引用舊名的 input（在其他節點的 `main` 陣列中）
4. 從 `每小時觸發` 這種上游節點的 connections 中也要更新引用

**建議方式**：用 `execute_code` 工具（`from hermes_tools import terminal, ...`）來操作 JSON，避免：
- Shell 轉義 Unicode/中文失敗
- JSON payload 太長被截斷
- `cat > /tmp/xxx.json << 'EOF'` 遇到特殊字符報錯

```python
# ✅ 推薦：用 execute_code 處理 JSON payload
from hermes_tools import terminal
import json

result = terminal(f"curl -s -b /tmp/n8n_cookies.txt http://localhost:5678/rest/workflows/{WF_ID}", timeout=10)
wf = json.loads(result['output'])['data']

# ... 修改 nodes 和 connections ...

payload = { ... }  # 完整 payload
with open('/tmp/wf_patch.json', 'w', encoding='utf-8') as f:
    json.dump(payload, f, ensure_ascii=False)
```

然後在 terminal 中：
```bash
curl -s -b /tmp/n8n_cookies.txt -X PATCH http://localhost:5678/rest/workflows/{WF_ID} \
  -H 'Content-Type: application/json' \
  -d @/tmp/wf_patch.json
```

> ⚠️ `cat > /tmp/file << 'HERMESEOF'` 在處理含中文的 JSON 時容易出錯（shell 轉義、EOF 衝突），改用 Python `json.dump()` 寫檔更可靠。

### Notion API Select 屬性值不匹配

當 n8n workflow 中的 Notion query filter 使用 `select.equals`，但該值不存在於 Notion DB 的 Select 選項中時，Notion API 會回傳 400 `validation_error`：

```
select option "未開始" not found for property "狀態". Available options: "待開始", "進行中", "已完成", "阻塞".
```

**解決方法：** 將 filter 中的 `equals` 值改為 Notion DB 中實際存在的選項。

#### 改為排除特定值的 filter

如果是要查「不是已完成」的所有任務，Notion API 支援 `select.does_not_equal`：

```python
# 從 select.equals 改為 select.does_not_equal
f = node['parameters']['jsonBody']['filter']['select']
val = f.pop('equals', None)  # 移除等於
if val:
    f['does_not_equal'] = '已完成'  # 改為不等於
```

這會查詢所有狀態不等於「已完成」的任務（包含待開始、進行中、阻塞等）。

**驗證 Notion DB 的 Select 選項：** 可透過 `POST /v1/databases/{DB_ID}`（不含 query 路徑）的 response 中的 `properties.狀態.select.options` 陣列查看所有可用選項。

| 陷阱 | 症狀 | 解決方案 |
|------|------|----------|
| jsonBody 是字串 | 400 Bad Request | 用 `json.loads()` 轉成 dict |
| 缺少 contentType | body 送空字串 `""` | 設 `contentType: "json"`, `specifyBody: "json"` |
| activate 沒 versionId | 400 Required | versionId **必須在 JSON body 中傳遞**，不可作為 query param |
| Cookie 過期 | API 回 401 但不被檢查 | 每次操作前先驗證 |
| deactivate 後 versionId 不變但 activate 會失敗 | 404 Version not found | deactivate 後重新 GET workflow 獲取最新 versionId 再 activate |
| HTTP Request node auth | Found credential with no ID | **不用 credential**，改手動 Header |
| Manual run API | Cannot read nodeName | 用 `triggerToStartFrom` 方式，不用 `startNodes` |
| Notion Select 值不匹配 | 400 validation_error | filter 的 `equals` 值須與 Notion DB 實際選項完全一致 |
| 誤改節點名稱 | connections 引用斷裂，workflow 損壞 | 用 `execute_code` + Python 重構整個 payload（見「節點改名陷阱+重建 payload 流程」） |
| PUT 回 404 | 以為 workflow 不存在 | ⚠️ n8n 2.19.5 **不接受 PUT** `/rest/workflows/{id}`，回 404。**改用 PATCH** |
| activate API 報 `propertyValues[itemName] is not iterable` | 無法透過 REST API 激活任何工作流 | n8n 2.19.5 的 activate 端點有 bug，連最小工作流（schedule + noOp）也無法激活。**解決方案：** 直接寫入 SQLite `UPDATE workflow_entity SET active=1 WHERE id='...'`，然後重啟 n8n。注意：如果 n8n 沒有重啟（舊進程還在跑），DB 的 active=1 仍會被 n8n memory cache 讀到 → 在 `curl -s http://localhost:5678/rest/workflows` 中會顯示 active=true |
| executeCommand 節點內含 localhost:5678/webhook 路徑 | activate 時 `_findConflictingWebhooks` 報錯 | n8n 2.19.5 activation 時會掃描**所有節點參數**中的字串，如果發現 localhost:5678/webhook/XXX 路徑會誤判為 webhook 衝突。解決方案：通知改用 log 檔寫入（`echo "msg" >> /tmp/xxx.log`），或透過 Hermes 自己的 Telegram bot API（不用 local webhook） |

### SQLite 直接刪除工作流

當 REST API 無法刪除工作流（鬼影草稿、active=True 工作流拒絕被刪）時，直接操作 SQLite：

```python
import sqlite3

db = sqlite3.connect("/home/athing/.n8n/database.sqlite")

# 先找出 id
rows = db.execute("SELECT id, name, active FROM workflow_entity").fetchall()
for r in rows:
    print(f"{r[0]} | {r[1]} | active={r[2]}")

# 刪除工作流（必須先清相關表，否則 FK 會報錯）
TABLES_TO_CLEAR = [
    "shared_workflow", "workflow_history", "workflow_dependency",
    "workflow_statistics", "execution_entity", "processed_data",
    "workflow_publish_history", "workflow_published_version",
    "workflow_builder_session",
]
wid = "目標工作流ID"
for t in TABLES_TO_CLEAR:
    try:
        db.execute(f"DELETE FROM {t} WHERE workflowId = ?", (wid,))
    except Exception:
        pass  # 有些表可能不存在
db.execute("DELETE FROM workflow_entity WHERE id = ?", (wid,))
db.commit()
```

> ⚠️ **必做順序：** 先關閉 n8n → 操作 SQLite → 重啟 n8n。中間不要啟動 n8n，否則它會重新註冊 cron。

### CLI 命令

| 操作 | 命令 |
|------|------|
| 列表 | `n8n list:workflow` |
| 發布 | `n8n publish:workflow --id=<id>`（需重啟 n8n 才生效） |
| 匯出 | `n8n export:workflow --id=<id> --output=<file>` |
| 匯入 | `n8n import:workflow --input=<file>` |

> ⚠️ `n8n publish:workflow` 後**必須重啟 n8n**，CLI 不支援 `delete:workflow`。

### 工作流 JSON 結構要點

#### Schedule Trigger 參數格式 (v2.19)

```json
{
  "rule": {
    "interval": [
      {"field": "cronExpression", "expression": "0 * * * *"}
    ]
  }
}
```
- `interval` 必須是**陣列**，不是物件！
- 每小時：`0 * * * *`
- 每晚 21:00：`0 21 * * *`

#### Telegram Node 參數

```json
{
  "resource": "message",
  "operation": "sendMessage",
  "chatId": "8781947120",
  "text": "={{ $json.result }}"
}
```

#### 節點連接格式

```json
{
  "節點A": {
    "main": [[{"node": "節點B", "type": "main", "index": 0}]]
  }
}
```

#### 觸發方式：Webhook vs REST API

有兩種方式觸發 n8n 工作流重新執行，**Webhook 方式更簡單（不用 cookie、不用 auth）：**

| 方式 | 優點 | 缺點 |
|------|------|------|
| ✅ **Webhook** `POST /webhook/{path}` | 不需 cookie/auth，不回傳 401，永遠可用 | Webhook 工作流需先激活 |
| ❌ **REST API** `POST /rest/workflows/{id}/run` | 不需預建 webhook node | 需 cookie + versionId，n8n 2.19.5 有 bug |

**Webhook 方式（推薦）：**
```bash
curl -s -X POST 'http://localhost:5678/webhook/hermes-task-loop' \
  -H 'Content-Type: application/json' \
  -d '{}'
# 回傳：{"message":"Workflow was started"}
```
適用場景：工作流有 Webhook node 作為觸發器。無需 login、無需 cookie、無需 versionId。

**REST API 方式（傳統）：**
```bash
curl -s -b /tmp/n8n_cookies.txt -X POST http://localhost:5678/rest/workflows/{WF_ID}/run \
  -H 'Content-Type: application/json' \
  -d '{"triggerToStartFrom": {"mode": "trigger", "nodeName": "每小時觸發"}}'
```
適用場景：工作流使用 Schedule Trigger，無 Webhook node。

> ⚠️ **Webhook 方式永遠先用** — 如果 n8n 有對應的 webhook endpoint，優先使用 webhook，因為不會遇到 cookie 過期問題。

## 任務閉環循環（n8n → Hermes → Notion）

當 n8n 定時推送待辦任務清單到 Telegram 時，形成一個自動化閉環：

**流程：**
1. 📥 **接收** — n8n 發送「📋 待辦任務 N 項」到 Telegram
2. 🧠 **解析** — 擷取第一項未完成的任務（n8n 會按 Notion 排序順序推送）
3. ✅ **執行** — 完成該任務（程式、回覆、系統操作等）
4. 📝 **更新 Notion** — 將該任務的「狀態」欄位從「待開始」改為「已完成」
5. 🔄 **觸發 n8n** — 手動觸發 `/rest/workflows/{WF_ID}/run` 讓 n8n 重新查詢
6. ⏳ **等待下一輪** — n8n 會再推送剩餘任務，重複步驟 2-5 直到清單為空

**核心原則：**
- **一次只做一件事** — 每次閉環只處理一項任務，完成後觸發 n8n 重新計算
- **先更新 Notion 再觸發 n8n** — 確保 n8n 查到的資料已經是最新狀態
- **手動觸發用 triggerToStartFrom** — 見上方「手動觸發驗證」章的 API 方式

**觸發 n8n 的時機：**
```
完成任務 → 更新 Notion (已完成) → 觸發 n8n (/run) → 等待新消息 → 重複
```

> ⚠️ 不要在觸發 n8n 後立即去 Notion 查詢下一項任務，等 n8n 的 Telegram 消息送達後再處理，這樣才能確保執行順序正確且不重複處理同一項。

### Webhook 工作流的建立與激活（n8n 2.19 特定流程）

Webhook-triggered 工作流（使用 `n8n-nodes-base.webhook` 節點）在 n8n 2.19 中需要遵循特定流程才能正常收發請求。

### 建立時的關鍵配置

```json
{
  "id": "n1-webhook",
  "name": "收件端",
  "type": "n8n-nodes-base.webhook",
  "typeVersion": 2,
  "parameters": {
    "path": "my-webhook-path",
    "httpMethod": "POST",
    "responseMode": "onReceived",
    "responseData": "allEntries",
    "options": {
      "httpMethod": "POST"
    }
  },
  "webhookId": "my-webhook-path"
}
```

> ⚠️ `httpMethod` 必須同時設在 `parameters` 頂層和 `options` 內層。預設 webhook node 僅接受 GET。

### 激活流程（deactivate → PATCH → activate）

#### 1. 建好工作流後先 deactivate

```bash
curl -s -b /tmp/n8n_cookies.txt -X POST http://localhost:5678/rest/workflows/{WF_ID}/deactivate
```

#### 2. PATCH 更新節點配置

```python
import json
from hermes_tools import terminal

result = terminal(f"curl -s -b /tmp/n8n_cookies.txt http://localhost:5678/rest/workflows/{WF_ID}", timeout=10)
wf = json.loads(result['output'])['data']

# 修改 webhook node
for n in wf['nodes']:
    if n['type'] == 'n8n-nodes-base.webhook':
        n['parameters']['httpMethod'] = 'POST'
        n['parameters']['options'] = n['parameters'].get('options', {})
        n['parameters']['options']['httpMethod'] = 'POST'

with open('/tmp/wf_patch.json', 'w', encoding='utf-8') as f:
    json.dump({"name": wf['name'], "nodes": wf['nodes'], "connections": wf['connections'],
               "settings": wf.get('settings',{}), "staticData": wf.get('staticData')},
               f, ensure_ascii=False)

patch = terminal(f"""curl -s -b /tmp/n8n_cookies.txt -X PATCH http://localhost:5678/rest/workflows/{WF_ID} \
  -H 'Content-Type: application/json' -d @/tmp/wf_patch.json""", timeout=10)
new_vid = json.loads(patch['output'])['data']['versionId']
```

#### 3. 用 versionId 激活（⚠️ 必須在 JSON body 中傳）

```bash
curl -s -b /tmp/n8n_cookies.txt -X POST http://localhost:5678/rest/workflows/{WF_ID}/activate \
  -H 'Content-Type: application/json' \
  -d '{"versionId": "NEW_VERSION_ID"}'
```

> ⚠️ **versionId 必須在 JSON body 中傳遞**，不能作為 query param。用 query param 會報 `"invalid_type": "Required"`。

#### 4. 觸發 webhook

激活後即可用 POST 觸發：
```bash
curl -s -X POST 'http://localhost:5678/webhook/{path}' \
  -H 'Content-Type: application/json' \
  -d '{"source":"hermes","action":"check-tasks"}'
```

回傳 `{"message":"Workflow was started"}` 表示成功。

> 如果 webhook node 只設了 GET（未設 httpMethod），`POST /webhook/{path}` 會回 404 `"This webhook is not registered for POST requests. Did you mean to make a GET request?"`，但 `GET /webhook/{path}` 會回 200。解決方式：在 node parameters 中加入 `httpMethod: "POST"`。

### 執行輸出解析（壓縮資料格式）

n8n 2.19 的 `/rest/executions/{id}` 回傳的 `data.data` 字段是一個**編號引用壓縮 JSON 陣列**。解析方式：

```python
import json

detail = terminal(f"curl -s -b /tmp/n8n_cookies.txt 'http://localhost:5678/rest/executions/{EXEC_ID}'", timeout=10)
parts = json.loads(json.loads(detail['output'])['data']['data'])

# Telegram 發送成功的標誌：找 {"ok": true, "result": ...}
for i in range(len(parts)):
    if isinstance(parts[i], dict) and parts[i].get('ok') == True:
        print(f"Telegram OK at part {i}: {json.dumps(parts[i], ensure_ascii=False)[:200]}")
        # message_id 在 result 指向的另一個 part 中
        break

# 找 Telegram 發送的訊息內容：通常包含 📋 等 Emoji
for i in range(len(parts)):
    if isinstance(parts[i], str) and '📋' in parts[i]:
        print(f"Message text at part {i}: {parts[i]}")
        break

# 找 Node 執行時序（用 executionStatus 標記）
for i in range(len(parts)):
    if isinstance(parts[i], dict) and 'executionStatus' in parts[i]:
        print(f"Node {i}: time={parts[i].get('executionTime')}ms, status={parts[i].get('executionStatus')}")
```

**常見查找模式：**
- Telegram 響應：找 `"ok": true` 的 dict
- 訊息文本：找包含 `📋` 或 `待辦任務` 的長字串
- Notion 查詢結果：找包含 `"object": "list"` 的 dict（通常在 83-87 範圍的 part）
- Code Node 輸出：找 `"myResult": "..."` 的 dict

### 改進的 Code Node（附優先級/分類/截止日）

這是一個可複用的 Code Node 範本，從 Notion query 結果中提取完整的任務資訊並按優先級排序：

```javascript
// 從 $input.first().json 取得 Notion API response
const data = $input.first().json;
const results = data.results || [];

if (results.length === 0) {
  return [{ json: { myResult: 'skip', count: 0 } }];
}

const taskList = [];
for (const item of results) {
  const props = item.properties || {};
  let title = '', priority = '', category = '', dueDate = '', status = '';

  // 提取 title (第一個 type=title 的屬性)
  for (const [k, v] of Object.entries(props)) {
    if (v.type === 'title' && v.title?.length > 0) {
      title = v.title[0].plain_text || '';
      break;
    }
  }
  // 提取其他屬性
  for (const [k, v] of Object.entries(props)) {
    if (v.type === 'select') {
      const val = v.select?.name || '';
      if (k === '優先級') priority = val;
      else if (k === '分類') category = val;
      else if (k === '狀態') status = val;
    }
    if (v.type === 'date' && v.date) dueDate = v.date.start || '';
  }

  const prioIcon = { '高': '🔴', '中': '🟡', '低': '🟢' }[priority] || '⚪';
  taskList.push({ title, priority, category, dueDate, prioIcon });
}

// 按優先級排序：高 → 中 → 低
const order = { '高': 0, '中': 1, '低': 2 };
taskList.sort((a, b) => (order[a.priority] || 9) - (order[b.priority] || 9));

// 格式化輸出
const lines = [`📋 待辦任務 (${taskList.length} 項)：`];
lines.push('━━━━━━━━━━━━━━━━');
for (const t of taskList) {
  let line = `${t.prioIcon} ${t.title}`;
  if (t.category) line += ` [${t.category}]`;
  if (t.dueDate) line += ` 📅${t.dueDate}`;
  lines.push(line);
}
lines.push('━━━━━━━━━━━━━━━━');
lines.push('💡 打開 Notion：https://www.notion.so/');

return [{ json: { myResult: lines.join('\\n'), count: taskList.length } }];
```

### SQLite 強制激活（activate API 故障時的後備方案）

當 n8n 2.19.5 的 activate REST API 報 `propertyValues[itemName] is not iterable` 時（連最小工作流也無法激活），可以直接操作 SQLite：

```python
import sqlite3

conn = sqlite3.connect("/home/athing/.n8n/database.sqlite")
c = conn.cursor()
c.execute("UPDATE workflow_entity SET active=1 WHERE id='目標工作流ID'")
conn.commit()
conn.close()
```

**注意事項：**
1. 設完後**必須重啟 n8n** 讓 schedule trigger 生效（`npx n8n restart` 或 kill + 啟動）
2. 重啟後確認 active 狀態：`curl -s http://localhost:5678/rest/workflows | python3 -c "print(json.load(sys.stdin)...)"`
3. 如果 n8n 沒有重啟（舊進程還在），新設的 active=1 可能不會被載入 — 但 n8n 的 memory 中如果已有該 workflow 的 cache，可能會自動識別到 DB 變化
4. 此法僅適用於 n8n 2.19.5 的已知 bug，後續版本可能修復

**適用場景：** 當 REST API activate 一直報錯，但 workflow 內容完全正常時。

## 一次性建立 + 啟用

建立時直接在 JSON 中加上 `"active": true` 即可免去後續 activate 步驟。**這是推薦方式** — 避免 deactivate/activate 的 versionId 陷阱。

如果工作流已建立但未 active，可以用 PATCH **補啟用**：

```bash
curl -s -b /tmp/n8n_cookies.txt -X PATCH http://localhost:5678/rest/workflows/{WF_ID} \
  -H 'Content-Type: application/json' \
  -d '{"active": true}'
```

> ⚠️ 「補啟用」依賴工作流狀態。如果 workflow 曾被 deactivate，則單純 PATCH active=true 可能不夠，需使用標準的 deactivate → patch → activate 流程（含 versionId）。

```json
{
  "name": "工作流名稱",
  "nodes": [...],
  "connections": {...},
  "active": true,
  "settings": {"saveManualExecutions": true}
}
```

## 同步技能更新到 GitHub（踩坑記錄）

當使用者說「上傳github」時，將最新的 SKILL.md 及 references 同步到 `glenhsu/hermes-n8n-integration` repo。

### 流程

1. **Clone 最新版 repo 到暫存目錄**

```bash
cd /tmp && rm -rf hermes-n8n-integration
git clone https://github.com/glenhsu/hermes-n8n-integration.git
```

2. **複製本地最新技能內容**

```bash
cp ~/.hermes/skills/devops/hermes-n8n-integration/SKILL.md /tmp/hermes-n8n-integration/
cp ~/.hermes/skills/devops/hermes-n8n-integration/references/*.md /tmp/hermes-n8n-integration/references/
```

3. **設定 git identity（臨時 clone 沒有 config）**

```bash
cd /tmp/hermes-n8n-integration
git config user.name "Hsu Glen"
git config user.email "hsupk1986@gmail.com"
```

4. **提交並推送**

```bash
git add -A
git commit -m "🔧 更新 n8n 整合技能"
git push origin main
```

### ⚠️ 踩坑：Push 失敗（No such device or address）

**症狀：** `fatal: could not read Username for 'https://github.com': No such device or address`

**原因：** 在 WSL 的 terminal 中執行 `git push`，git 需要交互式輸入帳密，但 WSL 非互動 shell 無法處理 credential prompt。

**解決方案（任選一）：**

1. **將 token 嵌入 remote URL（推薦）**
   ```bash
   git remote set-url origin https://<username>:<token>@github.com/glenhsu/hermes-n8n-integration.git
   ```
   優點：一次設定永久有效，WSL 非互動 shell 也可 push。

2. **用 `execute_code` + Python wrapper push**
   在 `execute_code` 中用 `os.environ["GITHUB_TOKEN"]` 環境變數傳遞 token，避免 shell 交互。

3. **先問使用者提供 token**
   如果都未設定，直接問使用者要 GitHub Personal Access Token，然後用方式 1 設定。

4. **改用 SSH remote**
   ```bash
   git remote set-url origin git@github.com:glenhsu/hermes-n8n-integration.git
   ```
   前提是使用者已將 SSH public key 加入 GitHub（見 `github-auth` skill）。

### ⚠️ 踩坑：Python requests.Session 無法用於 n8n API

**症狀：** `requests.Session()` 登入成功（HTTP 200），但後續請求仍回 401 Unauthorized。

**原因：** n8n 在非瀏覽器環境下使用 cookie-based auth，但 `requests.Session` 對 n8n 的 `HttpOnly` cookie 處理不一致。

**解決方案：** 每次操作都用 `curl` + cookie file，不要用 Python requests Session。
```python
# ✅ 正確做法
import subprocess
subprocess.run(["curl", "-s", "-c", "/tmp/n8n_cookies.txt",
  "-X", "POST", "http://localhost:5678/rest/login",
  "-H", "Content-Type: application/json",
  "-d", '{"emailOrLdapLoginId":"admin@hermes.local","password":"admin@hermes.local"}'],
  capture_output=True, timeout=10)

# 後續請求都用 -b /tmp/n8n_cookies.txt
subprocess.run(["curl", "-s", "-b", "/tmp/n8n_cookies.txt",
  "http://localhost:5678/rest/workflows?limit=5"],
  capture_output=True, timeout=10)
```

> ⚠️ 即使在同一個 `execute_code` 中先登入再用 session.get() 也不行。curl cookie file 方式始終可靠。

### Notion API：狀態欄位類型確認

Notion DB 的「狀態」欄位可能是 `select` 類型（不是 `status`）。查詢過濾時須確認實際類型，否則會報 `validation_error`：

```python
# ❌ 錯的 — 如果欄位實際是 select 類型
"filter": {"property": "狀態", "status": {"equals": "待開始"}}

# ✅ 對的
"filter": {"property": "狀態", "select": {"equals": "待開始"}}
```

**快速辨別方式：** 先不帶 filter 查詢全部，看回傳的 `properties` 中各欄位的 `type` 值。

### 記憶輔助

- 這個 repo 只包含 `SKILL.md` + `references/` + `README.md` + `LICENSE`
- 不需要 push 整個 `.hermes/skills/` 目錄，只 push repo 內的檔案
- 所有 n8n 踩坑經驗都同步到這個 repo

## 參考

- n8n REST API: `/rest/workflows`, `/rest/executions`, `/rest/login`
- 完整 n8n 技能（含安裝、密碼重置、SQLite 操作）: `skill_view('n8n')`
- Notion API 查詢格式: `POST /v1/databases/{id}/query` with `filter` + `page_size`

---

## 自動化閉環測試驗證方法

完整的端到端閉環測試流程（Hermes → Notion → n8n → Telegram），含測試任務建立、多輪循環執行、最終驗證。

參考文件：`references/closed-loop-test-methodology.md`

## 自動化閉環：Cronjob + Webhook Listener（無須 Hermes Gateway Webhook）

當無法啟用 Hermes gateway webhook platform（重啟會中斷當前連線），使用 cronjob + Python listener 的雙軌方案實現端到端閉環。

### 架構

```
n8n Schedule Trigger（每小時）
  ↓
n8n 工作流執行 → 查 Notion → 發 Telegram → POST /webhook/hermes-task-ready（通知）
                                              ↓
                                    Python Listener（port 8765）
                                      ├─ log 確認收到
                                      └─ 非同步觸發 hermes CLI 或 cronjob
                                              ↓
Hermes Cronjob（每 10 分鐘）
  ├─ 查 Notion「待開始」任務
  ├─ 更新第一項為「已完成」
  ├─ 觸發 n8n 重新執行
  └─ 發 Telegram 報告結果
```

### 為什麼不用純 Listener？

- Hermes 需要**使用者互動**才能執行任務（除非用 cronjob）
- Listener 只能通知，不能自主處理
- **Cronjob 是閉環的主力**，listener 是即時通知的補充

### 建立 Cronjob（推薦主方案）

```bash
hermes cron create n8n-closed-loop \
  --schedule "10m" \
  --prompt "檢查 Notion 待辦任務並處理閉環..." \
  --skills notion \
  --deliver telegram
```

Cronjob 每次執行會自動：
1. 載入 `notion` skill
2. 查詢 Notion DB 中「狀態=待開始」的任務
3. 處理第一項（更新為已完成）
4. 觸發 n8n 工作流重新執行
5. 將結果送回 Telegram

### 建立 Python Webhook Listener（即時通知補充）

當 n8n 工作流需要即時通知時，用獨立的 Python HTTP server：

```python
# 核心邏輯：收到 n8n POST 後觸發 cronjob 或直接處理
def trigger_via_script():
    """無須 hermes CLI 的 fallback：Python 直接處理 Notion 任務"""
    # 1. 查詢 Notion「待開始」任務
    # 2. 更新第一項為「已完成」
    # 3. 觸發 n8n 工作流（GET workflow → POST /run with workflowData）
    
def trigger_n8n_via_api():
    """
    正確觸發 n8n 工作流 REST API
    
    ⚠️ 需要 GET workflow 取得完整 data，然後在 POST /run 時包含：
    - workflowData: 從 GET 取得的完整工作流數據
    - triggerToStartFrom: {"mode": "manual"}
    
    不能只用 triggerToStartFrom 不帶 workflowData，n8n 會報：
    "Cannot read properties of undefined (reading 'nodeName')"
    """
    # 先 GET 工作流
    wf_resp = GET /rest/workflows/{WF_ID}
    wf_data = wf_resp.data
    
    # 再 POST run
    POST /rest/workflows/{WF_ID}/run
    {
        "workflowData": wf_data,
        "triggerToStartFrom": {"mode": "manual"}
    }
```

### Cookie 在 Python 中的正確解析方式

`/tmp/n8n_cookies.txt` 使用 Netscape cookie 格式（含 `#` 註解行），Python `http.cookiejar.MozillaCookieJar` 解析會失敗。

**正確的解析方式**（逐行手動解析）：

```python
with open('/tmp/n8n_cookies.txt') as f:
    raw = f.read()

cookie_val = None
for line in raw.splitlines():
    line = line.strip()
    if not line or line.startswith('#'):
        continue
    parts = line.split('\t')
    if len(parts) >= 7:
        # parts[5] = cookie name, parts[6] = cookie value
        cookie_val = f"{parts[5]}={parts[6]}"
        break

# 然後用 cookie_val 作為 HTTP Header
headers = {'Cookie': cookie_val}
req = urllib.request.Request(url, headers=headers)
```

> ⚠️ **不要在 `execute_code` 中用 Python urllib + cookie 查詢 n8n** — 即使手動解析成功也可能遇到 HTTP 400。**推薦在 `terminal` 中用 curl**（`--cookie /tmp/n8n_cookies.txt` 自動正確解析 Netscape 格式）。

### 完整的 n8n 工作流閉環測試流程

```bash
# 1. 建立測試任務（Notion API）
POST /v1/pages ... 狀態=待開始

# 2. 手動觸發 n8n（POST /run）
curl -s -b /tmp/n8n_cookies.txt -X POST \
  http://localhost:5678/rest/workflows/{WF_ID}/run \
  -H 'Content-Type: application/json' \
  -d '{"workflowData": {...}, "triggerToStartFrom": {"mode": "manual"}}'

# 3. 檢查執行結果
curl -s -b /tmp/n8n_cookies.txt \
  "http://localhost:5678/rest/executions/{EXEC_ID}?includeData=true"

# 4. 等 cronjob 執行（10 分鐘內）
#    或手動觸發 cronjob
hermes cron run {JOB_ID}

# 5. 驗證 Notion 任務狀態
POST /v1/databases/{DB_ID}/query (filter: 狀態=已完成)
# 應看到測試任務已被更新

# 6. 再次觸發 n8n → 應看到剩餘任務減少
```

> ⚠️ **驗證閉環的唯一標準：** n8n 兩次執行之間的任務數量遞減。第一次查到 N 個，第二次查到 N-1 個（因為 cronjob 已處理掉一個），最終歸零。