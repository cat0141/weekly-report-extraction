---
name: weekly-report-extraction
description: 从钉钉日志抓取指定人员周报、保存本地、提取产品线信息并生成汇总文档。当用户要求抓取周报、提取周报信息、汇总周报、对比周报、或用运营总监视角分析周报时使用此skill。
agent_created: true
version: v4.2
created: 2026-05-03
updated: 2026-06-01
---

# weekly-report-extraction — 周报全流程处理

## 触发时机

| 触发词 | 进入流程 |
|--------|---------|
| "抓取周报" / "拉周报" | 第一阶段：从钉钉日志抓取 |
| "提取周报信息" / "按产品线汇总" | 第二阶段：按产品线提取 |
| "汇总周报" / "生成汇总" | 第二阶段 + 可选第三阶段 |
| "对比周报" / "周报对比" | 第三阶段：生成对比汇总 |
| "运营总监视角" / "深度分析" | 第三阶段：运营总监视角分析 |
| "升级周报" / "不要流水账" | 遵循 v1.1 升级规范 |

---

## 本 Skill 文件清单

| 文件 | 用途 |
|------|------|
| `SKILL.md` | 主流程控制（本文件） |
| `README.md` | 简要说明与使用入口 |
| `references/config.json` | 集中配置：人员/路径/MCP key/模板/产品线/环境 |
| `references/钉钉日志MCP使用指南.md` | 7 个 MCP 工具的参数、示例、错误码、踩坑记录 |
| `references/易云簿-财云差异化说明.md` | 产品线区分规则（提取前必读） |
| `references/周报升级规范-v1.1.md` | 大王要求的升级规范：模板结构、核心原则 |
| `references/运营总监分析提示词.md` | 运营总监角色设定、分析维度、输出格式 |
| `references/weekly-report-template.html` | 原始周报 HTML 渲染模板 |
| `references/summary-template.html` | 业务汇总 HTML 渲染模板 |
| `references/comparison-template.html` | 对比汇总 HTML 模板 |
| `references/director-view-template.html` | 运营总监视角 HTML 模板 |
| `references/weekly-report-data-schema.json` | 原始周报 JSON 数据结构 |
| `references/summary-data-schema.json` | 业务汇总 JSON 数据结构 |

---

## 外部依赖

| 依赖 | 说明 |
|------|------|
| **钉钉日志 MCP** | 提供 `get_received_report_list`、`get_report_entry_details` 等 7 个工具 |
| **tencent-server-ops** (server 模式可选) | SSH 到服务器部署 HTML 到 Nginx |

---

## 核心原则

1. **配置文件驱动**：所有可变信息（人员、路径、模板、环境）从 `references/config.json` 读取，不硬编码。
2. **环境自动适配**：本地电脑写本地文件夹，服务器上跑写服务器路径。由 `env.mode` 控制。
3. **流程与细节分离**：SKILL.md 管流程，`references/` 管细节。
4. **先查再归类**：第二阶段提取前必须先读 `references/易云簿-财云差异化说明.md`。
5. **不确定就标注**：产品线归属不确定时标注"待确认"，不猜测。

---

## 第0步：环境判断

**读取 `references/config.json` 中的 `env.mode` 值：**

| mode | 说明 | 使用路径配置 |
|------|------|-------------|
| `local` | 本地电脑运行 | `paths_local` |
| `server` | 阿里云服务器运行 | `paths_server` |

**路径选择规则：**
```
if env.mode == "local":
    paths = config["paths_local"]       # D:/BaiduSyncdisk/workspace/...
else if env.mode == "server":
    paths = config["paths_server"]      # /root/workspace-openclaw/...
```

> **首次运行前**：确认 `config.json` 中 `env.mode` 值与当前运行环境一致。默认 `local`。

---

## 第一阶段：从钉钉日志抓取周报

> 用户未提供原始周报时执行。MCP 工具详细参数见 `references/钉钉日志MCP使用指南.md`。

### Step 1：读取配置并搜索周报

1. **读取 `references/config.json`**：
   - 根据 `env.mode` 选择 `paths_local` 或 `paths_server` 作为路径配置
   - 获取 `mcp.key`、`team_members` 列表
2. **调用 `get_received_report_list`** 拉取收到的日志列表。
   - ️ `startTime` + `endTime` + `size` + `cursor` **四个参数缺一不可**
   - ⚠️ `size` 最大只能传 **10**（传 20/100 返回 0 条）
   - ⚠️ `cursor` 从 0 开始每次 +1（不是上次返回的值）
3. **遍历结果**按 `creator_user_name` 匹配 `config.json` 中 `enabled: true` 的人员。
4. **时间范围搜索**：如果第一轮未找到某人员，扩大时间范围继续翻页。

### Step 2：获取周报详情

对匹配到的 `report_id`，调用 `get_report_entry_details`：
- **优先使用 `contentV2`**：含 `key`、`value`、`type`、`images`
- `value` 是明文文本（Markdown+HTML 混合），非加密值
- 备用 `report_content.value`（也是明文）
- 不要解析 `richTextValue`（加密格式）

### Step 3：转换并保存原始周报

1. **8 步 HTML 转换**（详见 `references/钉钉日志MCP使用指南.md` 第 3 节）：
   ①清 `<span>` → ②图片占位 → ③表格解析 → ④恢复图片 → ⑤标题转换 → ⑥粗体 → ⑦分割线 → ⑧分段
2. **用 `references/weekly-report-template.html` 渲染**（模板 + JSON 数据分离，JSON Schema 见 `references/weekly-report-data-schema.json`）。
3. **保存路径**（根据 env.mode 选择）：`{paths.raw_reports_dir}/人名_YYYYMMDD.html`

---

## 第二阶段：按产品线提取信息

> 用户要求"汇总"或"提取信息"时执行。

### Step 4：先读差异化文档（不能跳过）

**读取 `references/易云簿-财云差异化说明.md`**，明确：
- 易云簿 → 终端企业客户
- 财云 → 代账公司/财务机构、AI会计师、专业版
- 不确定 → 标注"待确认"

### Step 5：逐条提取并归类

从 Step 3 生成的原始周报中逐条提取，按差异化规则判断归属。

### Step 6：写入汇总文档

用 `references/summary-template.html` + `references/summary-data-schema.json` 生成汇总 HTML。

保存路径（根据 env.mode 选择）：`{paths.summary_dir}/{产品线}周报汇总_{YYYYMMDD}.html`

输出格式见 `references/周报升级规范-v1.1.md` 的"汇总输出格式"章节。

### Step 7：部署到 Nginx 静态目录（仅 server 模式）

> **local 模式跳过此步**，生成的 HTML 文件保存在本地文件夹即可。

**server 模式**：HTML 文件**不能**通过文档系统 API 链接访问（会触发下载）。必须部署到 Nginx：

```bash
cp 文件.html {paths.nginx_static_dir}/
chown nginx:nginx {paths.nginx_static_dir}/文件.html
# 访问链接: {paths.nginx_base_url}/文件.html
```

如需 SSH 到服务器操作，加载 `tencent-server-ops` skill 或 `mcp-config` skill 参考 SSH 配置。

---

## 第三阶段：高级分析

### 对比汇总

当用户要求"对比周报"时：
- 使用 `references/comparison-template.html` 渲染
- 必须包含 6 个模块：人员提交状态、核心指标环比、任务完成对比、冲突项、关键思考、下周计划
- 趋势标识：`↑` 增长 / `↓` 下降 / `—` 持平 / `⚠️` 数据缺失
- 数据引用用 `<details class="quote">` 折叠，`target="_blank"` 新窗口打开

### 运营总监视角分析

当用户要求"运营总监视角"或"深度分析"时：
- **先加载 `references/运营总监分析提示词.md`**：获取角色设定、5 维分析框架、输出结构
- 使用 `references/director-view-template.html` 渲染
- 输出 6 个模块：核心洞察 → 风险预警 → 缺失数据 → 行动建议 → 数据引用 → 总监点评

---

## ️ 常见陷阱速查

| # | 陷阱 | 详情 |
|---|------|------|
| 1 | 不要凭感觉归类 | 见 `references/易云簿-财云差异化说明.md` |
| 2 | 不要遗漏产品线 | 周报可能同时涉及多产品线 |
| 3 | AI会计师 ≠ 易云簿 | 是财云升级版 |
| 4 | 专业版 ≠ 易云簿版本 | 是财云版本体系 |
| 5 | 四川一新泽 → 财云 | 不是易云簿客户 |
| 6 | 独立标题不默认归属 | 标注"待确认" |
| 7 | 合计数据不拆分 | 无法拆分则不放入单产品线 |
| 8 | MCP size 最大=10 | 见 `references/钉钉日志MCP使用指南.md` §1 |
| 9 | contentV2 > richTextValue | 见 `references/钉钉日志MCP使用指南.md` §2 |
| 10 | 8 步 HTML 转换 | 见 `references/钉钉日志MCP使用指南.md` §3 |
| 11 | MCP key 在 config.json | 不要用 config.yaml 被截断的 `***` |
| 12 | 原始周报存 .html | 图片需要 HTML 渲染 |
| 13 | 汇总文件名含日期 | 避免覆盖历史 |
| 14 | 模板 + JSON 分离 | 样式一次定义，数据随时替换 |
| 15 | contentV2.value 是明文 | 不是加密值 |
| 16 | env.mode 必须匹配 | local 模式用 local 路径，server 模式用 server 路径 |
| 17 | 本地模式跳过 Nginx | local 环境无 Nginx，Step 7 直接跳过 |
| 18 | 时间范围搜索找人 | 扩大范围翻页 + 按 creator_user_name 过滤 |

---

## 规则

| # | 规则 | 违反后果 |
|---|------|---------|
| R1 | 第二阶段必须先读差异化文档 | 产品线混淆 |
| R2 | 不确定产品线归属时标注"待确认" | 错误归类 |
| R3 | 配置从 config.json 读取，不硬编码 | 换人换路径失效 |
| R4 | 提示词从 references 加载，不内联 | SKILL.md 膨胀不可维护 |
| R5 | MCP 调用前检查 4 参数完整性 | API 返回 40035 |
| R6 | get_received_report_list size ≤ 10 | 返回 0 条 |
| R7 | env.mode 必须与实际运行环境一致 | 文件存到错误路径 |
