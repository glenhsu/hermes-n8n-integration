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
| activate 沒 versionId | 400 Required | 用最新的 versionId 調用 activate |
| Cookie 過期 | API 回 401 但不被檢查 | 每次操作前先驗證 |
| deactivate 後 versionId 不變但 activate 會失敗 | 404 Version not found | deactivate 後重新 GET workflow 獲取最新 versionId 再 activate |
| HTTP Request node auth | Found credential with no ID | **不用 credential**，改手動 Header |
| Manual run API | Cannot read nodeName | 用 `triggerToStartFrom` 方式，不用 `startNodes` |
| Notion Select 值不匹配 | 400 validation_error | filter 的 `equals` 值須與 Notion DB 實際選項完全一致 |
| 誤改節點名稱 | connections 引用斷裂，workflow 損壞 | 用 `execute_code` + Python 重構整個 payload（見「節點改名陷阱+重建 payload 流程」） |

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

### 一次性建立 + 啟用

建立時直接在 JSON 中加上 `"active": true` 即可免去後續 activate 步驟：

```json
{
  "name": "工作流名稱",
  "nodes": [...],
  "connections": {...},
  "active": true,
  "settings": {"saveManualExecutions": true}
}
```

## 參考

- n8n REST API: `/rest/workflows`, `/rest/executions`, `/rest/login`
- 完整 n8n 技能（含安裝、密碼重置、SQLite 操作）: `skill_view('n8n')`
- Notion API 查詢格式: `POST /v1/databases/{id}/query` with `filter` + `page_size`