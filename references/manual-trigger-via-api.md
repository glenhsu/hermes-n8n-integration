# 手動觸發 n8n 工作流（REST API）

> 從 `hermes-n8n-integration` 移入 `n8n` 的 reference

## 問題

n8n 2.19.5 的 `/rest/workflows/{id}/run` API 有 bug：
- `startNodes` + `destinationNode` → 報 `Cannot read nodeName`
- `triggerToStartFrom` → ✅ 可以成功建立 execution

## 正確方式

```bash
curl -s -b /tmp/n8n_cookies.txt -X POST http://localhost:5678/rest/workflows/{WF_ID}/run \
  -H 'Content-Type: application/json' \
  -d '{"triggerToStartFrom": {"mode": "trigger", "nodeName": "每小時觸發"}}'
```

回傳中的 `data.executionId` 可用來查詢結果。

## 查詢執行結果

```bash
curl -s -b /tmp/n8n_cookies.txt "http://localhost:5678/rest/executions/{EXEC_ID}?includeData=true"
```

`status` 字段：`success` / `error` / `running` / `waiting`。

## 最近 5 次執行

```bash
curl -s -b /tmp/n8n_cookies.txt "http://localhost:5678/rest/executions?workflowId={WF_ID}&limit=5"
```