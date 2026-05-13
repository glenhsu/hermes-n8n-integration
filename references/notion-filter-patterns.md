# Notion API Filter 模式速查

用於 `POST /v1/databases/{id}/query` 的 `filter` 參數。

## Select 屬性

```json
// 等於特定值
"filter": {
  "property": "狀態",
  "select": { "equals": "待開始" }
}

// 不等於特定值（NOT 已完成 = 所有未完成）
"filter": {
  "property": "狀態",
  "select": { "does_not_equal": "已完成" }
}
```

## Status 屬性

```json
"filter": {
  "property": "狀態",
  "status": { "equals": "Done" }
}
```

## Checkbox 屬性

```json
"filter": {
  "property": "已核對",
  "checkbox": { "equals": false }
}
```

## Date 屬性

```json
// 今天內
"filter": {
  "property": "截止日",
  "date": { "equals": "{{today}}" }
}

// 已過期（早於今天）
"filter": {
  "property": "截止日",
  "date": { "before": "{{today}}" }
}
```

## 複合條件（AND / OR）

```json
// 多條件 AND
"filter": {
  "and": [
    { "property": "狀態", "select": { "does_not_equal": "已完成" } },
    { "property": "優先級", "select": { "equals": "高" } }
  ]
}

// 多條件 OR
"filter": {
  "or": [
    { "property": "狀態", "select": { "equals": "待開始" } },
    { "property": "狀態", "select": { "equals": "進行中" } }
  ]
}
```

## 驗證 Select 選項

```bash
curl -s -X POST "https://api.notion.com/v1/databases/{DB_ID}" \
  -H "Authorization: Bearer {TOKEN}" \
  -H "Notion-Version: 2022-06-28" \
  -H "Content-Type: application/json" \
  | jq '.properties.狀態.select.options[].name'
```