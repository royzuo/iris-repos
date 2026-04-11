# OpenClaw Dashboard: 本地观测与控制中心

## Sources
- https://github.com/mudrii/openclaw-dashboard

## 1. 应用场景

该案例不是简单的自动化脚本，而是把 OpenClaw 运行态信息聚合成一个本地控制台，用于回答三个核心问题：
1. Gateway 是否在线，依赖是否健康。
2. 当前花费、模型消耗、cron 任务、会话和 sub-agent 是否正常。
3. 是否存在告警、资源异常、成本异常或运行漂移。

难点在于，OpenClaw 的状态分散在 CLI、日志、会话、cron、子代理和多渠道消息里，人工拼接成本高，且无法快速定位问题。

## 2. 技术方案

### 角色分类
- **新角色**：`Observer_`
- 说明：OpenClaw 在这里承担的是“观测、汇聚、展示、告警”的系统角色，不是执行任务本身。

### 架构概览
```mermaid
flowchart LR
  U[用户 / 浏览器] --> UI[OpenClaw Dashboard]
  UI --> S[Go HTTP Server]
  S --> R[数据采集器 /refresh]
  S --> G[OpenClaw Gateway /v1/chat/completions]
  S --> H[/api/system
系统探针]
  R --> D[data.json]
  D --> UI
```

### 核心组件
| 组件 | 作用 |
|---|---|
| `cmd/openclaw-dashboard` | CLI 入口 |
| `internal/apprefresh` | 拉取并生成仪表盘数据 |
| `internal/appserver` | HTTP 服务 |
| `internal/appchat` | 通过 Gateway 做自然语言查询 |
| `internal/appsystem` | CPU/RAM/Swap/Disk 与 Gateway 探针 |
| `web/index.html` | 零依赖前端 |

### 关键能力
- 本地运行，不依赖外部云服务
- 自动刷新与预热，打开即看
- `/api/chat` 提供面向运维的自然语言问答
- `/api/system` 暴露运行时健康状态
- 速率限制与超时控制，避免控制台本身成为负担

### Skills / Plugins / Hooks / Heartbeat
- **Skills**：未显式声明业务技能，主要依赖 OpenClaw Gateway 与内置数据采集流程
- **Plugins**：未明确依赖第三方插件，强调 zero-dependency 前端
- **Hooks**：未见显式 hooks
- **Heartbeat**：未配置独立 heartbeat 任务，数据刷新通过页面打开与定时 auto-refresh 完成

### 简化流程
1. 浏览器打开 Dashboard。
2. 前端调用 `/api/refresh`。
3. 后端刷新 `data.json`。
4. 页面渲染成本、会话、cron、模型和子代理信息。
5. 用户可通过 `/api/chat` 进一步追问。

## 3. 实现效果

### 优点
- 把分散信息收敛到单页控制台。
- 便于快速判断健康状态和成本趋势。
- 本地化部署，隐私和延迟都更好。

### 缺点
- 更偏观测层，不直接执行复杂修复。
- 依赖本机 OpenClaw 数据可用性。
- 过多指标时仍可能需要二次筛选。

### 可改进点
- 增加告警分级和事件关联分析。
- 增加历史异常回放。
- 增加与 cron / session 的跳转联动。

## 4. 其他相关信息

该案例体现了 OpenClaw 的新系统角色：从“执行器”升级为“控制平面可视化层”。这类案例适合单独归为 `Observer_`，因为其核心价值是持续观测、解释和决策辅助。