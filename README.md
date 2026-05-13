# Hermes ↔ n8n Integration

> Hermes Agent 透過 REST API 管理 n8n 工作流的完整解決方案
> **不需要打開瀏覽器**，全部 CLI / API 完成

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![n8n](https://img.shields.io/badge/n8n-2.19.5+-blue)](https://n8n.io)
[![Hermes Agent](https://img.shields.io/badge/Hermes_Agent-ready-success)](https://hermes-agent.nousresearch.com)

## 📖 簡介

這個 repo 是一個 **Hermes Agent Skill** — 一套讓 AI Agent 直接透過 REST API 管理 n8n 工作流自動化的完整方案。適合不想開瀏覽器、想在終端機或 Telegram 裡遙控 n8n 的人。

### 能做什麼？

- ✅ 列出所有工作流 + 狀態
- ✅ 讀取工作流完整節點內容
- ✅ 修復 HTTP Request Node（n8n 2.19.5 常見 bug）
- ✅ 啟用/停用工作流
- ✅ 手動觸發執行
- ✅ 查詢執行歷史 + 錯誤診斷
- ✅ 自動登入 + cookie 管理

## 🚀 快速開始

### 前置

- n8n 運行在 `localhost:5678`
- Hermes Agent 已安裝

### 登入

```python
from hermes_tools import terminal

terminal("""
curl -s -c /tmp/n8n_cookies.txt -X POST http://localhost:5678/rest/login \
  -H "Content-Type: application/json" \
  -d '{"emailOrLdapLoginId":"admin@hermes.local","password":"your_password"}'
""", timeout=10)
```

> ⚠️ 字段是 `emailOrLdapLoginId`，不是 `email`！

### 列出工作流

```python
import json
from hermes_tools import terminal

result = terminal("curl -s -b /tmp/n8n_cookies.txt http://localhost:5678/rest/workflows", timeout=10)
workflows = json.loads(result["output"])["data"]
for w in workflows:
    print(f"{w['id']} | {w['name']} | active={w['active']}")
```

## 🐛 已知陷阱（全是踩過的坑）

| 陷阱 | 症狀 | 解決方案 |
|------|------|----------|
| `jsonBody` 是字串 | 400 Bad Request | 用 `json.loads()` 轉 dict |
| 缺少 `contentType` | body 送空字串 | 設 `contentType: "json"`, `specifyBody: "json"` |
| activate 沒 `versionId` | 400 Required | 用最新 versionId 調用 activate |
| Cookie 過期 | 401 但不報錯 | 每次操作前驗證 cookie |
| PATCH 前沒 deactivate | versionId 不更新 | 一定要先 deactivate 再 patch |
| HTTP Request node auth | credential 綁定失敗 | 不用 credential，改手動 Header |
| Manual run API | 報 nodeName 錯誤 | 用 `triggerToStartFrom`，不用 `startNodes` |

詳細說明見 [`SKILL.md`](SKILL.md)。

## 📁 目錄結構

```
hermes-n8n-integration/
├── SKILL.md                          # 主要技能文檔（全流程）
├── references/
│   ├── error-pattern-matching.md     # 錯誤模式快速診斷
│   └── manual-trigger-via-api.md     # 手動觸發工作流 API 說明
└── README.md                         # 本文件
```

## 🔗 相關資源

- [n8n REST API 文檔](https://docs.n8n.io/api/)
- [Hermes Agent](https://hermes-agent.nousresearch.com)
- [n8n 本地安裝指南](https://docs.n8n.io/hosting/installation/)

## 📄 License

MIT