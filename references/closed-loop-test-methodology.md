# 閉環測試方法論

用於驗證 Hermes ↔ Notion ↔ n8n ↔ Telegram 端到端閉環的測試流程。

## 測試目標

證明整個自動化閉環能夠：
1. Hermes 處理任務 → 更新 Notion（已完成）
2. 觸發 n8n 重新查詢
3. n8n 回傳更新版待辦清單到 Telegram
4. 任務數量正確遞減

## 測試步驟

### 1. 建立測試任務

在 Notion DB 中建立 N 個測試任務（建議 3 個），每項使用不同的優先級、分類：

```bash
TASKS_DB="資料庫ID"
NOTION_KEY="$NOTION_API_KEY"

for i in 1 2 3; do
  NAME="🧪 測試任務${I} - 描述"
  curl -s -X POST "https://api.notion.com/v1/pages" \
    -H "Authorization: Bearer $NOTION_KEY" \
    -H "Notion-Version: 2022-06-28" \
    -H "Content-Type: application/json" \
    -d '{
      "parent": {"database_id": "'$TASKS_DB'"},
      "properties": {
        "任務名稱": {"title": [{"text": {"content": "'"$NAME"'"}}]},
        "狀態": {"select": {"name": "待開始"}},
        "優先級": {"select": {"name": "P1 重要"}},
        "專案": {"select": {"name": "其他"}}
      }
    }'
done
```

> ⚠️ 確認 Notion DB 屬性名稱。常見差異：`專案`（不是「分類」）、`優先級`（P0~P3）、`狀態`（select 類型，不是 status 類型）。

### 2. 查詢起始狀態

確認測試任務已正確建立：

```bash
curl -s -X POST "https://api.notion.com/v1/databases/$TASKS_DB/query" \
  -H "Authorization: Bearer $NOTION_KEY" \
  -H "Notion-Version: 2022-06-28" \
  -H "Content-Type: application/json" \
  -d '{"filter":{"property":"狀態","select":{"equals":"待開始"}},"page_size":10}'
```

記錄總數 count。

### 3. 執行閉環循環

對每個測試任務依次執行：

```bash
# Step A: 查詢當前待開始任務，取優先級最高的
curl -s -X POST "https://api.notion.com/v1/databases/$TASKS_DB/query" \
  -H "Authorization: Bearer $NOTION_KEY" \
  -H "Notion-Version: 2022-06-28" \
  -H "Content-Type: application/json" \
  -d '{"filter":{"property":"狀態","select":{"equals":"待開始"}},"page_size":5}'

# Step B: 更新為「已完成」
PAGE_ID="目標頁面ID"
curl -s -X PATCH "https://api.notion.com/v1/pages/$PAGE_ID" \
  -H "Authorization: Bearer $NOTION_KEY" \
  -H "Notion-Version: 2022-06-28" \
  -H "Content-Type: application/json" \
  -d '{"properties":{"狀態":{"select":{"name":"已完成"}}}}'

# Step C: 觸發 n8n（Webhook 方式 — 最可靠）
curl -s -X POST "http://localhost:5678/webhook/hermes-task-loop" \
  -H "Content-Type: application/json" -d '{}'
# 預期回應：{"message":"Workflow was started"}

# Step D: 等待 n8n 執行
sleep 3

# Step E: 重新查詢 Notion 驗證計數
curl -s -X POST "https://api.notion.com/v1/databases/$TASKS_DB/query" \
  -H "Authorization: Bearer $NOTION_KEY" \
  -H "Notion-Version: 2022-06-28" \
  -H "Content-Type: application/json" \
  -d '{"filter":{"property":"狀態","select":{"equals":"待開始"}},"page_size":10}'
# 驗證：待開始數量應比上一輪少 1
```

**循環 N 次**（N = 測試任務數量），每次減少 1 個待開始任務。

### 4. 最終驗證

```bash
# 檢查待開始 — 應回到原始數量（不含測試任務）
curl -s -X POST "https://api.notion.com/v1/databases/$TASKS_DB/query" \
  -H "Authorization: Bearer $NOTION_KEY" \
  -H "Notion-Version: 2022-06-28" \
  -H "Content-Type: application/json" \
  -d '{"filter":{"property":"狀態","select":{"equals":"待開始"}},"page_size":10}' | jq '.results | length'

# 檢查已完成 — 應包含所有測試任務
curl -s -X POST "https://api.notion.com/v1/databases/$TASKS_DB/query" \
  -H "Authorization: Bearer $NOTION_KEY" \
  -H "Notion-Version: 2022-06-28" \
  -H "Content-Type: application/json" \
  -d '{"filter":{"property":"狀態","select":{"equals":"已完成"}},"page_size":10}' | python3 -c "
import sys,json
d=json.load(sys.stdin)
test_count = sum(1 for r in d['results'] if '測試任務' in r['properties'].get('任務名稱',{}).get('title',[{}])[0].get('plain_text',''))
print(f'{len(d[\"results\"])} completed, {test_count} test tasks')
"
```

### 5. 檢查 n8n 執行記錄（可選）

```bash
# 查看閉環工作流的執行歷史
curl -s -b /tmp/n8n_cookies.txt "http://localhost:5678/rest/executions?workflowId=$WF_ID&limit=5" | \
  python3 -c "import sys,json; d=json.load(sys.stdin); [print(f'ID:{e[\"id\"]} Status:{e[\"status\"]}') for e in d.get('data',{}).get('results',d.get('data',[]))[:5]]"
```

## 通過條件（Checklist）

- [ ] 測試任務全部成功建立（Notion API 回 200）
- [ ] 第一輪：完成 1 個 → 待開始 -1
- [ ] 第二輪：完成 1 個 → 待開始 -1
- [ ] ... 循環 N 輪
- [ ] n8n Webhook 每次回 `Workflow was started`
- [ ] 最終待開始 = 原始數量（測試任務全部歸檔）
- [ ] 已完成中包含所有 N 個測試任務
- [ ] 不受影響的任務保持原狀

## 常見陷阱

| 陷阱 | 症狀 | 解法 |
|------|------|------|
| Notion 屬性名不同 | 400 validation_error 或任務建立成功但缺少欄位 | 先 GET 不帶 filter 查詢所有頁，看 `properties` 結構 |
| n8n cookie 過期 | Webhook 方式不受影響 👍 | 使用 Webhook 觸發（`POST /webhook/{path}`），不是 `/rest/workflows/{id}/run` |
| 優先級排序 | Hermes 處理順序和 n8n 可能不同 | Hermes 主動設定優先級排序：P0 > P1 > P2 > P3 > 無優先級 |
| Webhook path 不對 | 404 "not registered for POST" | 確認 webhook node 的 `path` 參數和 `httpMethod: POST` 配置 |

## 為什么用 Webhook 而不是 REST API 觸發

| 特性 | Webhook `POST /webhook/{path}` | REST API `POST /rest/workflows/{id}/run` |
|------|------|------|
| Cookie 需要？ | ❌ 不需要 | ✅ 需要 login + cookie file |
| versionId 需要？ | ❌ 不需要 | ✅ 需要 (且 PATCH 後會變) |
| 可靠度 | ✅ 總是可用 | ⚠️ n8n 2.19.5 有 bug |
| 預期回應 | `{"message":"Workflow was started"}` | `{data: {executionId: ...}}` |