# 自动化 AI 行业日报系统 (Automated AI Industry Daily Report)

**Sources**: 
- https://skywork.ai/skypage/en/ultimate-guide-openclaw-ai-agent/2038533037563396096

## 1. 应用场景 (Application Scenario)
**背景与目的**：
在快速发展的 AI 行业中，从业者需要每天获取最新的新闻、模型发布和行业动态。手动搜集和整理信息非常耗时。本用例旨在利用 OpenClaw 构建一个自动化的 AI 行业日报系统，能够定期抓取行业新闻、过滤噪音，并生成结构化的每日报告发送给用户。

**困难与挑战**：
- **信息过载**：AI 领域的信息源繁多，如何有效提取高价值内容。
- **定时与调度**：需要一个可靠的机制在每天特定时间触发任务，而不是依赖用户手动唤醒。
- **Token 消耗**：如果每次检查都读取大量冗余的提示词或无关网页，将导致昂贵的 API 开销。

## 2. 技术方案 (Technical Architecture/Solution)
**工作流与核心组件**：
本方案核心利用 OpenClaw 的 **Heartbeat**（心跳）机制结合 Web 搜索能力来实现自动化工作流。

- **Skills / Plugins**:
  - `web-hybrid-search` / `web-fetch`：用于每日抓取最新的 AI 新闻站点或聚合信息。
  - `cron` 调度：处理定时发布逻辑。

- **Heartbeat 配置解析**：
  配置 `HEARTBEAT.md` 以定义定期检查和日报生成的频率。为避免频繁的 Token 消耗，心跳被设置为较长的间隔（例如 45-60 分钟），或仅在特定时间段内活跃。

  ```markdown
  # HEARTBEAT.md
  
  ## Daily Task: AI Industry Report
  **Interval**: Every 24 hours (run between 08:00 AM - 09:00 AM)
  **Prompt**: 
  1. 使用 web-hybrid-search 搜索过去 24 小时的 "AI industry news", "LLM releases".
  2. 总结出 3-5 条最重要的新闻。
  3. 如果在过去 24 小时内未生成过报告，则将总结输出。
  4. 如果不满足条件或已完成，回复 HEARTBEAT_OK。
  ```
  *OpenClaw 仅在心跳触发时处理 `HEARTBEAT.md`，并在发现无事可做时以静默丢弃的 `HEARTBEAT_OK` 结束回合，从而降低成本。*

## 3. 实现效果 (Results/Outcomes)
**优点**：
- **大幅节省时间**：用户每天清晨自动获取总结报告，节省了手动整理的时间（据统计，此类邮件/信息分发任务每周可节省数小时）。
- **成本可控**：合理配置 Heartbeat 时间间隔（如 45 分钟或限定活跃时间）并将任务精简（小于 200 Tokens），能有效控制 API 计费。
- **始终在线的体验**：OpenClaw 从被动的聊天机器人转变为主动推送价值的个人助理。

**缺点/改进空间**：
- **过度抓取风险**：如果搜索结果较长，网页爬取时可能消耗过多上下文窗口。可以使用 `lightContext: true` 进一步削减成本。
- **状态追踪**：简单的 Heartbeat 需要本地文件（如 `heartbeat-state.json`）记录上一次生成的报告时间戳，避免在时间窗口内重复发送。可以考虑将其迁移到确切时间的 OpenClaw Cron 任务 (`openclaw cron add`) 以获得更精确的时间控制。

## 4. 其他相关信息 (Other Info)
- **Heartbeat vs Cron**：根据 Skywork 的最佳实践，对于需要“确切时间”执行的任务（例如“每天上午 8 点准时发送”），推荐使用 OpenClaw Cron；而对于“周期性监控和条件判断”（例如“有突发重大新闻时随时通知”），则更适合通过优化后的 `HEARTBEAT.md` 结合状态监控来实现。
- 如果 `HEARTBEAT.md` 是空文件，OpenClaw 会跳过心跳运行以节省 API 调用。