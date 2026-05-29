# 钉钉日志 MCP 使用指南

> 基于 2026-05-18 实测确认。包含 7 个工具的全部参数说明、示例代码、错误码对照表。

---

## 工具总览

| 工具 | 用途 | 关键限制 |
|------|------|---------|
| `get_received_report_list` | 拉取收到的日志列表 | size 最大 10 |
| `get_send_report_list` | 拉取自己发送的日志列表 | 只能查自己的 |
| `get_report_entry_details` | 获取单条日志详情 | 需要 report_id |
| `get_report_templates` | 获取可用的日志模板 | - |
| `get_report_statistics` | 获取日志统计信息 | - |
| `create_report_entry` | 创建日志条目 | - |
| `update_report_entry` | 更新日志条目 | - |

---

## 1. get_received_report_list — 拉取收到的日志

### 参数

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `startTime` | number | ✅ | 开始时间，毫秒级 Unix 时间戳（如 `1776000000000`） |
| `endTime` | number | ✅ | 结束时间，毫秒级 Unix 时间戳 |
| `size` | number | ✅ | 每页条数，**最大只能传 10** |
| `cursor` | number | ✅ | 翻页游标，**从 0 开始，每次 +1** |

### ⚠️ 关键陷阱

1. **四个参数缺一不可**：少任何一个返回 `40035 不合法的参数`
2. **size 上限 = 10**：传 20、50、100 都返回 **0 条**！必须用 size=10
3. **cursor 是数字增量**：从 0 开始每次 +1，**不是**上次返回的 cursor 值
4. **时间戳是毫秒级**：不是秒级！

### 示例

```javascript
// 拉取 5 月 12 日至 5 月 18 日的周报
const params = {
  startTime: 1776000000000,  // 2026-05-12 00:00:00 毫秒
  endTime: 1776600000000,    // 2026-05-18 23:59:59 毫秒
  size: 10,
  cursor: 0
};

// 翻页：cursor 从 0 → 1 → 2 → ...
// 当返回的 report_list 为空或长度 < 10 时，说明已翻完
```

### 返回结构

```json
{
  "report_list": [
    {
      "report_id": "19e2fce8d3a22e15c9f74cf4d26b1080",
      "creator_user_name": "徐航",
      "creator_user_id": "xxx",
      "report_template_name": "总部客成中心周报",
      "create_time": 1776500000000,
      "modify_time": 1776500000000
    }
  ],
  "has_more": true,
  "next_cursor": null
}
```

### 搜索策略

1. 拉取所有收到的日志（分页翻页，size=10，cursor 从 0 递增）
2. 遍历 `report_list` 按 `creator_user_name` 匹配目标人员
3. 匹配到后提取 `report_id`
4. 如需更多历史周报，扩大时间范围继续翻页

**不要使用 `get_send_report_list` 找人**：该工具只返回当前用户自己发送的日志。

### 按模板名过滤

也可使用 `get_send_report_list` 的 `report_template_name` 参数按模板名过滤：
- 王顺泽 → 模板「产品部周报」
- 汪玉瑶 → 模板「中小微事业群-小微团队汇总周报」
- 徐航 → 模板「总部客成中心周报」
- 葛玉 → 模板「葛玉的周报模板」

---

## 2. get_report_entry_details — 获取日志详情

### 参数

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `report_id` | string | ✅ | 日志 ID（从 `get_received_report_list` 返回中提取） |

### ⚠️ 字段选择优先级

```
contentV2 > report_content.value > report_content.richTextValue
```

- **优先使用 `contentV2`**：每项有 `key`（字段名）、`value`（明文内容）、`type`（字段类型）、`images`（附件列表）
- **`value` 是明文文本**（含 Markdown 格式表格），不是加密值
- **`richTextValue` 是加密格式**，不要直接解析
- 备用 `report_content.value`：也是明文，可直接使用

### 返回结构示例

```json
{
  "report_id": "xxx",
  "contentV2": [
    {
      "key": "本周工作内容",
      "value": "1. 完成了XX功能开发\n2. 对接了XX客户",
      "type": "text",
      "images": ["https://static.dingtalk.com/xxx.png"]
    },
    {
      "key": "下周计划",
      "value": "1. 继续跟进XX项目\n2. 准备XX方案",
      "type": "text",
      "images": []
    }
  ]
}
```

---

## 3. HTML 标签转换流程（8步）

`contentV2.value` 是 Markdown + HTML 混合内容，必须按以下顺序转换为纯 HTML：

```
① 移除 <span> 标签（保留内容）
② ![alt](url) → <img src="url">（用占位符暂存，避免后续处理干扰）
③ |...|...| 表格 → <table> 解析（跳过分隔行 |---|---|）
④ 恢复图片占位符
⑤ ## 标题 → <h2>，### 标题 → <h3>
⑥ **粗体** → <strong>
⑦ --- 分割线 → <hr>
⑧ 按 \n\n 分段，每段转 <p>，已有 HTML 标签的段直接保留
```

---

## 4. 错误码对照表

| 错误码 | 含义 | 解决方案 |
|--------|------|---------|
| `40035` | 不合法的参数 | 检查 startTime/endTime/size/cursor 四个参数是否全部传入 |
| `PARAM_ERROR` | 参数错误 | 检查 MCP key 是否完整（不要使用 config.yaml 中被截断的 `***`） |
| 返回 0 条 | size 超限 | 确认 size ≤ 10 |

---

## 5. 时间戳转换工具

```javascript
// 日期 → 毫秒时间戳
const start = new Date('2026-05-12').getTime();  // → 1776000000000

// 毫秒时间戳 → 日期
const date = new Date(1776000000000);  // → Mon May 12 2026
```

---

## 6. MCP 连接配置

### 钉钉日志 MCP

```
Server ID: 7e6ce70465b4b041880e47a5160e338454c8b43f9284ca6f8a2e44ccf130d433
Key:      4d2866b973935a2d0bf5b2eb6a47a495
URL:      https://mcp-gw.dingtalk.com/server/7e6ce70465b4b041880e47a5160e338454c8b43f9284ca6f8a2e44ccf130d433
```

配置文件路径（LightClaw 服务器）：`~/.hermes/skills/openclaw-imports/` 对应子目录下的 `config.json`

### 获取 MCP URL 方法

1. 访问钉钉开放平台 (open.dingtalk.com)
2. 进入应用 → MCP 服务
3. 复制完整 URL（含 key 参数）

---

## 7. 排错记录

### API Key 截断问题（2026-04-28）

- **症状**：MCP 工具调用返回 `PARAM_ERROR` 或 `not found the specified tool`
- **根因**：配置文件中的 `mcp.url` 以 `key=***` 结尾，Key 被截断
- **修复**：重新从钉钉开放平台获取完整 MCP URL，写入配置文件。skill 中 key 保存在 `references/config.json` 的 `mcp.key` 字段，避免被 config.yaml 截断。

### mcporter --args JSON 转义 Bug

- mcporter CLI 的 `--args` 参数存在双层转义问题
- **推荐**：直接用 `curl` 调用 MCP 工具代替 mcporter

### PARAM_ERROR 排查步骤

1. 首先检查 `references/config.json` → `mcp.key` 是否完整
2. 检查 startTime/endTime/size/cursor 四个参数是否全部传入
3. 确认 size ≤ 10
4. 确认时间戳是毫秒级（不是秒级）
