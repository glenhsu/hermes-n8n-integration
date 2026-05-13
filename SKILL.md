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

#### 手動觸發驗證

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

> ⚠️ `/rest/workflows/{id}/run` API 在 n8n 2.19.5 有 bug：
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

## ⚠️ 關鍵陷阱彙整

| 陷阱 | 症狀 | 解決方案 |
|------|------|----------|
| jsonBody 是字串 | 400 Bad Request | 用 `json.loads()` 轉成 dict |
| 缺少 contentType | body 送空字串 `""` | 設 `contentType: "json"`, `specifyBody: "json"` |
| activate 沒 versionId | 400 Required | 用最新的 versionId 調用 activate |
| Cookie 過期 | API 回 401 但不被檢查 | 每次操作前先驗證 |
| PATCH 後 versionId 沒更新 | activate 失敗 | 必須先 deactivate 再 patch |
| HTTP Request node auth | Found credential with no ID | **不用 credential**，改手動 Header |
| Manual run API | Cannot read nodeName | 用 `triggerToStartFrom` 方式，不用 `startNodes` |

## 參考

- n8n REST API: `/rest/workflows`, `/rest/executions`, `/rest/login`
- 完整 n8n 技能（含安裝、密碼重置、SQLite 操作）: `skill_view('n8n')`
- Notion API 查詢格式: `POST /v1/databases/{id}/query` with `filter` + `page_size`