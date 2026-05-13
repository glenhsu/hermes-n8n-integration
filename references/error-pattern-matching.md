# n8n 執行錯誤模式比對

從 execution detail 文字中快速診斷：

```python
text = detail["output"]  # from GET /rest/executions/{id}?includeData=true

if "body failed validation" in text:
    print("❌ jsonBody 格式問題 — 檢查 contentType / jsonBody 型別")
elif "Bad request" in text and "body" in text:
    print("❌ Notion API 拒絕請求 — 檢查 body 格式和 Authorization header")
elif "Found credential with no ID" in text:
    print("❌ Credential 錯誤 — 改用 sendHeaders 手動送 token")
elif "Unknown error" in text and "JsTaskRunnerSandbox" in text:
    print("❌ Code Node task runner 崩潰 — 重啟 n8n 或改用 HTTP Request node")
elif "Could not find the workflow" in text:
    print("❌ Workflow ID 不屬於目前 user — 檢查 shared_workflow 表")
elif "Unauthorized" in text:
    print("❌ Cookie 過期 — 重新登入")
elif "invalid_type" in text or "Required" in text:
    print("❌ API 參數格式錯誤 — 檢查字段名/型別")
```