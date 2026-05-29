# weekly-report-extraction Skill

从钉钉日志抓取指定人员周报 → 保存 → 按产品线提取信息 → 生成汇总文档 → 部署预览。完整周报处理流水线。

## 功能

- **抓取周报**：通过钉钉日志 MCP 自动拉取指定人员（王顺泽/汪玉瑶/徐航/葛玉）的周报
- **智能归类**：根据易云簿/财云差异化规则自动区分产品线归属
- **汇总分析**：生成结构化 HTML 汇总文档，支持指标卡片、数据表格、关键发现
- **对比分析**：上周 vs 本周数据对比 + 任务完成对比 + 冲突项标注
- **深度分析**：运营总监视角分析（洞察/风险/行动建议/数据引用）
- **在线预览**：部署到 Nginx 静态目录，手机可直接访问

## 文件结构

```
weekly-report-extraction/
├── SKILL.md                          # 主流程控制
├── README.md                         # 本文件
└── references/
    ├── config.json                   # 集中配置（人员/路径/MCP/模板）
    ├── 钉钉日志MCP使用指南.md         # MCP 工具参数与踩坑记录
    ├── 易云簿-财云差异化说明.md        # 产品线区分规则
    ├── 周报升级规范-v1.1.md           # 升级规范与模板结构
    ├── 运营总监分析提示词.md           # 运营总监角色与分析框架
    ├── weekly-report-template.html    # 原始周报 HTML 模板
    ├── summary-template.html          # 业务汇总 HTML 模板
    ├── comparison-template.html       # 对比汇总 HTML 模板
    ├── director-view-template.html    # 运营总监视角 HTML 模板
    ├── weekly-report-data-schema.json # 原始周报 JSON Schema
    └── summary-data-schema.json       # 业务汇总 JSON Schema
```

## 使用方法

1. 将本目录放入 Hermes Agent / WorkBuddy 的 `skills` 目录
2. 修改 `references/config.json`：
   - 填入真实的钉钉日志 MCP key（`mcp.key`）
   - 填入真实的人员 userId（`team_members[].dingtalk_user_id`）
   - 根据实际环境调整 `paths` 路径
3. 根据需要启用/禁用人员（`team_members[].enabled`）
4. 触发 skill（见 SKILL.md 触发时机章节）

## 外部依赖

| 依赖 | 说明 |
|------|------|
| **钉钉日志 MCP** | 提供 `get_received_report_list`、`get_report_entry_details` 等 7 个工具 |
| **Nginx**（可选） | HTML 在线预览，需配置静态目录 |

## 版本

| 版本 | 日期 | 说明 |
|------|------|------|
| **v4.1** | 2026-05-29 | 补齐缺失的 references 文件（config.json、模板、Schema、指南）；SKILL.md 重构为流程控制。 |
| **v4.0** | 2026-05-18 | 新增对比汇总、运营总监视角分析。 |
| **v3.0** | 2026-05-11 | 配置文件驱动模式、HTML 模板+JSON 分离。 |
