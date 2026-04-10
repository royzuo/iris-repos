# 自动竞品情报追踪与分析 (Automated Competitive Intelligence Tracker)

**Sources**: 
- https://remoteopenclaw.com/blog/openclaw-use-cases-complete-guide
- https://forwardfuture.ai/p/what-people-are-actually-doing-with-openclaw-25-use-cases

## 1. 应用场景 (Application Scenario)
**背景与目的**：
在瞬息万变的市场环境中，产品团队和市场团队需要实时监控竞争对手的动态，包括新功能发布、定价策略调整、社交媒体风向以及新闻报道。人工检索这些信息耗时耗力，且容易遗漏关键变化。
本方案旨在利用 OpenClaw 自动化每日竞品监测，从多个来源提取关键信息，进行语义去重和倾向性分析，最终为团队提供简洁明了的行动建议简报。

**挑战与痛点**：
- 信息源分散（官网、推特、PR 稿件等），难以统一。
- 噪声极大，很多更新不具备战略价值。
- 高频度的定时抓取任务容易因为触发机制和鉴权问题而变得不稳定。

## 2. 技术方案 (Technical Architecture/Solution)
本方案充分利用了 OpenClaw 的生态系统，特别是通过 **Heartbeat** 进行周期性调度。

### 核心组件配置
* **Skills (技能)**:
  - `web-hybrid-search`: 结合 SearchCans 与 Linkup，执行针对特定竞品品牌词的多维网页搜索。
  - `web_fetch`: 提取最新产品发布页面和官方博客文章的具体内容转为 Markdown。
* **Plugins (插件)**:
  - `openclaw-slack`: 将最终生成的竞品简报格式化并推送到团队专属的 `#competitive-intel` Slack 频道。
* **Hooks (钩子)**:
  - 利用 `post-message-hook` 对输出的简报进行合规性扫描，防止将内部机密误当成查询关键词发送给外部搜索引擎。

### Heartbeat 配置解析
通过定制 `HEARTBEAT.md` 和结合系统 Cron，实现智能的周期性任务流：
```markdown
# HEARTBEAT.md
在每次 Heartbeat 触发时检查以下状态：
1. 读取 `memory/competitive-state.json` 查看上次抓取时间。
2. 如果距离上次抓取已超过 12 小时：
   - 触发 `web-hybrid-search` 搜索特定竞品关键字列表。
   - 分析提取到的信息并与昨日摘要进行对比（差异分析）。
   - 将变更整理输出并调用 Slack 插件发送。
   - 更新 `competitive-state.json` 中的时间戳。
3. 如果未超过，回复 HEARTBEAT_OK。
```
*备注：通过 Heartbeat 结合状态文件（State File）的方式，取代了单纯的僵化 Cron 任务，使助手能够根据上次执行情况和当前上下文智能决定是否启动重负载的信息汇总流。*

### 工作流 (Workflow Diagram)
```mermaid
graph TD
    A[Heartbeat Trigger] --> B{Check State: >12h?}
    B -- Yes --> C[web-hybrid-search (Keywords)]
    B -- No --> D[Reply HEARTBEAT_OK]
    C --> E[web_fetch (Extract Details)]
    E --> F[LLM: Diff & Analysis]
    F --> G[Generate Markdown Report]
    G --> H[Push to Slack Channel]
    H --> I[Update competitive-state.json]
```

## 3. 实现效果 (Results/Outcomes)
**优点 (Pros)**：
- **降噪能力强**：LLM 可以精准识别出“小版本迭代”与“战略级功能发布”的区别，过滤掉无效的营销废话。
- **自动化闭环**：结合 Heartbeat 和 Slack 插件，团队成员无需主动拉取，随时可以在协作平台接收到整理好的情报。
- **上下文感知**：由于 OpenClaw 会在内部记忆 (Memory) 中保留之前的竞品状态，它可以提供趋势分析（例如：“竞争对手连续三周在强调安全性”）。

**缺点 (Cons) 与 改进空间**：
- **反爬限制**：一些竞品官网有严格的 Cloudflare 防护，`web_fetch` 偶尔会失败。未来考虑引入专门的浏览器自动化 Agent（如 Playwright 节点）来绕过。
- **Token 消耗**：全量对比多个竞品的网页历史内容极其消耗 Token。可以改进为先让轻量级模型（如 GPT-4o-mini 或 Gemini Flash）做一次粗筛。

## 4. 其他相关信息 (Other Info)
为了提高搜索精度，建议在提示词中明确排除股市财报等非产品向新闻（除非有特定需求）：
> *Prompt 建议：仅关注产品功能更新、API 变更、定价页面调整和重要的人事变动，忽略常规的公关通稿和股市波动。*