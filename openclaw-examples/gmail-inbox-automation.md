# Gmail 智能收件箱管理与自动草稿回复 (Gmail Inbox Automation)

**来源/参考**: [25 OpenClaw Use Cases That Are Actually Worth Setting Up [2026]](https://remoteopenclaw.com/blog/openclaw-use-cases-complete-guide)

## 1. 应用场景 (Application Scenario)

**背景与目的**：
在日常工作中，许多专业人士需要处理大量的电子邮件，其中包含紧急事务、需要回复的邮件、资讯类订阅以及垃圾邮件。手动筛选并逐一回复这些邮件不仅耗时，还容易遗漏重要信息。此用例旨在利用 OpenClaw 自动化管理 Gmail 收件箱。

**面临的挑战**：
- 区分邮件优先级（紧急 vs. 资讯 vs. 噪音）。
- 模仿用户的写作风格自动起草回复。
- 在不打扰用户的情况下进行静默处理，仅在真正紧急时发送即时通知。

## 2. 技术方案 (Technical Architecture/Solution)

该方案结合了 OpenClaw 的 Heartbeat 机制与 MCP（Model Context Protocol）工具，实现每 30 分钟一次的后台静默邮件处理。

**核心组件：**
- **Skill/Plugin**: `mcp-gmail`（通过直接 API 或 Zapier 接入）用于读取和操作 Gmail；`telegram-notifier` 用于紧急事件的即时推送。
- **Heartbeat 配置**: 
  利用 OpenClaw 的 `HEARTBEAT.md` 进行周期性检查调度。当系统触发 Heartbeat 轮询时，OpenClaw 会读取任务列表并执行邮件扫描。

**工作流 (Workflow)**：
```mermaid
graph TD
    A[OpenClaw Heartbeat (每30分钟)] --> B[调用 Gmail MCP 获取未读邮件]
    B --> C{邮件分类}
    C -->|紧急| D[生成回复草稿]
    C -->|需要回复| D
    C -->|资讯/噪音| E[标记为已读或归档]
    D --> F[调用 Gmail API 保存草稿]
    F --> G{是否紧急?}
    G -->|是| H[通过 Telegram 发送通知]
    G -->|否| I[静默完成任务]
```

**HEARTBEAT.md 配置示例**：
```markdown
# 周期性任务 (Heartbeat)
- **邮箱扫描**: 扫描我的 Gmail 收件箱。
  - 提取最近 30 分钟内的未读邮件。
  - 将它们分类为：urgent（紧急）、needs-reply（需回复）、informational（资讯）、noise（噪音）。
  - 对于 urgent 和 needs-reply 类别，基于我过往的写作风格生成回复草稿并保存在 Gmail。
  - 如果分类为 urgent，立刻通过 Telegram 频道发送摘要通知我。
```

## 3. 实现效果 (Results/Outcomes)

**优势 (Pros)**：
- **大幅节省时间**：用户登录邮箱时，所有重要邮件已有生成的草稿，只需审阅并点击发送即可。
- **减少注意力分散**：只有真正的紧急邮件才会触发移动端推送通知。
- **风格一致性**：由于 OpenClaw 可以读取并模仿用户的历史上下文，生成的草稿语气自然。

**可改进之处 (Areas for Improvement)**：
- 对于极其复杂、需要跨部门确认的邮件，AI 草稿可能不够准确，仍需大量人工修改。
- 频率固定为每 30 分钟，可能会有最多 30 分钟的紧急邮件延迟。若需要更实时的响应，可考虑结合 Webhook 触发。

## 4. 其他相关信息 (Other Info)

在实际部署中，建议为 OpenClaw 配置专用的应用密码 (App Password) 以保证安全性。同时，初期运行阶段建议将所有自动处理的邮件单独打上一个 `AI-Drafted` 的标签，方便用户集中审查 AI 的处理准确度。