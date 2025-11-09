# 数据上报统计系统开发总纲

## 1. 总体目标
构建一个可扩展、高可靠的前端数据上报与统计分析系统，支持实时/批量数据采集、清洗与标准化、异常检测、指标统计、可视化报表、告警通知、权限控制与安全审计，并提供完备 API 接口。整体架构采用微服务 + 事件驱动模式，前端使用 React，后端使用 NestJS，数据库主选 PostgreSQL（结合 TimescaleDB 扩展进行时序优化），后期可插拔扩展到 ClickHouse/Redis/Kafka 等。

## 2. 架构概览

### 2.1 技术栈选型
- 前端：React + TypeScript + Zustand/Redux Toolkit + Ant Design/Chakra UI + Recharts/ECharts
- 后端框架：NestJS（模块化、DI、良好的测试支持）
- 数据库：PostgreSQL（核心事务型 + 报表元数据）+ TimescaleDB（时序指标）  
- 消息队列：Kafka（事件解耦，可回溯）/ 初期可用 RabbitMQ 替代
- 缓存：Redis（热数据、告警规则缓存、令牌黑名单）
- 对象存储（可选）：MinIO / AWS S3（存放报表导出文件、归档日志）
- 搜索与日志：Elastic Stack 或 OpenSearch（操作审计、系统日志）
- 容器与编排：Docker + Kubernetes（生产环境）/ 本地 Docker Compose
- CI/CD：GitHub Actions + ArgoCD（或 Tekton）/ SonarQube 代码质量
- 可观测性：Prometheus + Grafana（系统指标），OpenTelemetry（分布式追踪）
- 鉴权与身份：JWT + RBAC，后期可扩展 OAuth2 / SSO

### 2.2 微服务划分（建议初期单仓库 Monorepo：Nx / Yarn Workspaces）
| 服务 | 职责 | 关键依赖 |
|------|------|----------|
| api-gateway | 聚合转发、统一鉴权、限流 | NestJS, RateLimiter |
| auth-service | 用户、角色、权限、令牌颁发 | PostgreSQL, Redis |
| ingestion-service | 数据接收（SDK 上报）、批量导入、格式校验 | Kafka Producer |
| normalization-service | 标准化、数据清洗、字段映射 | Kafka Consumer |
| metrics-service | 指标聚合存储（时序） | TimescaleDB |
| anomaly-service | 异常检测算法执行、模型训练 | Kafka、Redis |
| alert-service | 告警规则管理、触发、通知发送 | Redis、SMTP、SMS 网关 |
| reporting-service | 报表生成、计划任务、导出 | PostgreSQL、队列 |
| dashboard-service | 查询聚合、数据接口优化 (CQRS) | TimescaleDB、Redis |
| audit-log-service | 操作日志、系统事件归档检索 | OpenSearch |
| sdk-build | 维护前端/移动端 SDK | npm 发布 |
| admin-frontend | 管理后台（React） | API Gateway |
| public-frontend | 用户可视化门户（React） | API Gateway |

### 2.3 数据流 (Event Driven)
1. 前端 SDK 上报事件 -> ingestion-service HTTP/批量接口
2. ingestion 写入原始事件表 + 推送 Kafka Topic raw_events
3. normalization-service 监听 raw_events -> 清洗 -> 输出 normalized_events
4. metrics-service 按窗口聚合（滑动/滚动）写入时序表 metrics_timeseries
5. anomaly-service 消费 normalized/metrics 触发算法 -> 推送 anomaly_events
6. alert-service 依据规则订阅 anomaly_events 或周期拉取指标 -> 触发通知
7. reporting-service 调度周期性任务生成报表快照
8. dashboard 前端通过 gateway 查询最新指标与分析结果
9. 所有操作事件写入 audit-log-service 供审计

### 2.4 关键设计原则
- 解耦：写入与分析分离（事件队列）
- 弹性：可水平扩展的消费组
- 幂等：写入侧使用业务唯一键（event_id + source + timestamp）
- 可追溯：事件链路 ID（trace_id）贯穿
- 安全：零信任接口（签名 + 速率限制 + 白/黑名单）
- 配置中心：后期可接入 Consul / Nacos / Apollo

## 3. 数据模型设计（初版）

### 3.1 核心表（PostgreSQL）
```
users(id, username, email, password_hash, status, created_at, updated_at)
roles(id, name, description)
user_roles(user_id, role_id)
permissions(id, code, description)
role_permissions(role_id, permission_id)

projects(id, name, api_key, status, owner_id, created_at)
project_settings(project_id, key, value, updated_at)

event_raw(id, project_id, event_key, payload_json, source_type, received_at)
event_normalized(id, project_id, event_key, event_type, user_id, session_id, attributes_json, occurred_at, normalized_at)

metrics_timeseries(id, project_id, metric_name, dimension_json, bucket_start, bucket_end, value, compute_version)
anomaly_events(id, project_id, metric_name, detector_type, score, expected_range, actual_value, bucket_start, detected_at, severity, status)

alert_rules(id, project_id, name, rule_type, metric_name, condition_expression, threshold_json, window, channels_json, enabled, created_at, updated_at)
alerts(id, rule_id, anomaly_event_id, status, message, sent_channels_json, created_at, acknowledged_at)

report_templates(id, project_id, name, config_json, schedule_cron, enabled, created_at)
report_instances(id, template_id, generated_at, storage_uri, status, parameters_json)

audit_logs(id, actor_user_id, action, resource_type, resource_id, meta_json, ip, user_agent, created_at)
```

### 3.2 索引与性能
- event_raw(event_key, project_id, received_at) BTREE
- event_normalized(project_id, event_type, occurred_at) 分区 + 索引
- metrics_timeseries(project_id, metric_name, bucket_start) Timescale hypertable
- anomaly_events(project_id, detected_at) 时间索引
- audit_logs(resource_type, resource_id, created_at) 组合索引

### 3.3 扩展性
- attributes_json / dimension_json 使用 JSONB + GIN 索引
- compute_version 字段支持重算与版本回滚
- 报表存储可切换对象存储 + CDN

## 4. 异常检测算法路线
| 阶段 | 算法 | 说明 |
|------|------|------|
| MVP | 固定阈值、移动平均 (SMA)、环比/同比对比 | 快速上线 |
| Phase 2 | EWMA、Z-Score、IQR、Percentile-based | 适应波动 |
| Phase 3 | CUSUM、Change Point (Pruned Exact Linear) | 突变检测 |
| Phase 4 | 基线建模（Prophet 或 ARIMA） | 复杂季节趋势 |
| Phase 5 | 半监督/异常聚类（Isolation Forest） | 自动学习模式 |

算法输出统一结构：{score, severity, expected_range, confidence}，驱动告警规则统一处理。

## 5. 告警系统设计
- 触发来源：实时异常事件 / 定时阈值扫描 / 报表对比
- 条件表达式 DSL：如 `error_rate > 0.05 AND p95_response_time > 800`
- 预聚合：Redis 缓存最近窗口统计避免重复计算
- 通知渠道：Email(SES/SMTP)、短信(第三方)、企业微信/钉钉 Webhook、Slack
- 告警去重：同一规则同一窗口内仅一次（fingerprint）
- 升级策略：持续超阈值 N 次 -> severity escalate
- 确认与抑制：acknowledge 后进入静默期 (suppression window)

## 6. API 设计原则

### 6.1 认证与授权
- JWT（短期）+ Refresh Token（长生命周期）
- API Key 用于项目事件上报（header: X-Project-Key + 请求签名 HMAC）
- RBAC：后端基于 NestJS Guard + Casl / 自研权限映射

### 6.2 版本与规范
- Base URL: /api/v1
- 响应包装：{ code, message, data, traceId }
- 错误码规范：APP模块前缀 + 数字（如 AUTH_40101）

### 6.3 示例接口（部分）
```
POST /ingest/events               // 单事件上报
POST /ingest/events/batch         // 批量
GET  /metrics/query               // 指标查询
GET  /anomalies                   // 异常列表
POST /alert-rules                 // 创建告警规则
PATCH /alert-rules/:id            // 更新
GET  /reports/templates           // 报表模板查询
POST /reports/templates           // 新增模板
POST /reports/templates/:id/run   // 立即生成
GET  /audit/logs                  // 审计日志
```

### 6.4 SDK 上报规范
```
{
  eventKey: "page.view",
  timestamp: 1731123456789,
  sessionId: "...",
  userId: "...",
  attributes: {
    url: "/home",
    referrer: "/",
    duration: 1200
  },
  context: {
    ua: "...",
    locale: "zh-CN"
  }
}
```

## 7. 前端功能规划（React）

### 7.1 模块
- 登录与权限路由
- 项目管理（API Key 管理、设置）
- 实时事件监控视图
- 指标看板（折线、柱状、饼图、热力图）
- 异常与告警中心（列表、过滤、确认）
- 告警规则配置（可视化条件编辑器）
- 报表模板设计器（拖拽 + 预览）
- 审计日志查看
- 用户与角色管理
- 系统设置（通知渠道、阈值全局策略）

### 7.2 状态管理
- 全局身份与权限：Auth Store
- 查询缓存：React Query
- 组件级图表数据：按维度缓存 + 自动失效机制

### 7.3 可视化
- ECharts：大量图表类型
- 统一主题与组件库（AntD 样式定制）
- 图表数据转换层（Adapter：将查询结果转换为 series）

## 8. 安全与合规

| 领域 | 措施 |
|------|------|
| 传输安全 | HTTPS/TLS 1.2+，HSTS，前端 SDK 支持降级策略 |
| 数据隔离 | project_id 全局过滤，防止越权读取 |
| 输入校验 | Joi / Zod Schema + 防注入（ORM 参数化） |
| 速率限制 | 基于用户/IP/项目三维，Redis Token Bucket |
| 重放防护 | 上报事件签名 + timestamp + nonce |
| 敏感字段 | pgcrypto 加密（如用户邮箱） |
| 日志脱敏 | 统一字段脱敏策略（手机号、邮箱） |
| 审计 | 关键操作写入 audit_logs，不可变性（追加写） |
| 灰度发布 | 按项目或用户分组启用新功能 |
| 备份恢复 | PostgreSQL WAL 归档 + 定期快照；对象存储多副本 |

## 9. 可观测性与运维
- Metrics: 服务 QPS、延迟、错误率、消费滞后、告警触发频次
- Tracing: 上报事件 -> 清洗 -> 聚合 -> 告警整链追踪
- Logging 分级：INFO（业务流程）、WARN（潜在风险）、ERROR（失败）
- 健康检查：/health（DB/Cache/Queue 子检查）
- 自动扩缩：Kafka lag 与 CPU 利用率触发 HPA
- 灾难演练：队列堆积、数据库主从切换、告警服务降级

## 10. CI/CD 流程
1. 代码提交 -> Pre-commit（ESLint/Prettier/Husky）
2. GitHub Actions：
   - 单元测试 (Jest)
   - Lint & Type Check
   - 安全扫描（Dependabot + Snyk）
   - 构建镜像（Docker） -> 推送 Registry
3. 自动部署：
   - Dev 环境：每次合并主分支自动部署
   - Staging：通过 Tag 或 Release
   - Prod：手动审批 Gate（多签或 ChatOps）
4. 回滚策略：保留最近 N 版本镜像，配置回滚命令

## 11. 测试策略

| 类型 | 内容 |
|------|------|
| 单元测试 | NestJS Modules / Service 逻辑；SDK 数据格式 |
| 集成测试 | API + DB 交互（Testcontainers） |
| 合约测试 | 各微服务之间（Pact） |
| 负载测试 | 上报接口、聚合查询（k6） |
| 异常检测验证 | 回放历史数据集，评估误报/漏报率 |
| 安全测试 | SQL 注入、XSS、权限绕过 |
| Chaos | 队列延迟、部分服务宕机，验证降级策略 |
| 回归测试 | 报表生成、告警规则生效链路 |

## 12. 迭代阶段划分（建议 24 周）

| 阶段 | 周数 | 目标 |
|------|------|------|
| 需求与原型 | 1-2 | PRD、原型、数据字典 |
| 架构与基础设施 | 3-4 | Monorepo、CI、基础服务脚手架 |
| 数据上报 MVP | 5-7 | SDK、ingestion、raw 存储 |
| 清洗与标准化 | 8-9 | normalization、初步查询 |
| 指标聚合 & 时序 | 10-12 | metrics、看板初版 |
| 异常检测 V1 | 13-15 | 简单阈值 + 移动平均 |
| 告警系统 V1 | 16-17 | 规则引擎 + 邮件通知 |
| 报表与模板 | 18-19 | 报表生成与下载 |
| 权限与审计 | 20-21 | RBAC、操作日志查询 |
| 优化与扩展 | 22-23 | 性能优化、灰度机制 |
| 发布与验收 | 24 | 文档、培训、上线 |

## 13. 风险与缓解
| 风险 | 描述 | 缓解 |
|------|------|------|
| 数据量爆发 | 指标聚合压力 | 早期接入 Timescale + 分区策略 |
| 异常误报率高 | 简单阈值不稳定 | 多算法融合、置信度分层 |
| 队列阻塞 | 消费滞后影响实时性 | 监控 lag + 自动扩容 |
| 报表生成耗时 | 大查询慢 | 物化视图 / 预计算缓存 |
| 权限复杂度上升 | 多层角色扩展困难 | RBAC + 细粒度权限表设计 |
| 安全风险 | API Key 泄露 | 轮换机制 + 使用范围限制 |
| 开发协同效率 | 多服务分工混乱 | 统一规范、Monorepo、代码 Owners |

## 14. 文档体系
- 架构白皮书：整体设计说明
- 接口文档：OpenAPI 自动生成 + 示例
- SDK 使用指南：集成、初始化、事件类型约定
- 运维手册：部署、扩缩容、监控指标说明
- 异常检测手册：算法说明与调参策略
- 权限模型说明：角色矩阵与权限点列表

## 15. 后续扩展方向
- 引入 ClickHouse 提升复杂查询性能
- 多租户隔离（schema per tenant 或 row-level policy）
- 机器学习在线模型（自适应阈值）
- 自助数据探索（DSL/SQL 沙箱）
- 插件系统（第三方自定义事件处理器）

---

## 16. 行动优先级（首批实现必选项）
1. Monorepo 搭建 + 基础 CI
2. Auth 模块（用户/项目/API Key）
3. Ingestion + Raw 存储 + 简易 SDK
4. Normalization + 指标聚合（基础：PV、UV、错误率）
5. 看板前端 + 指标查询 API
6. 简单异常检测（阈值 + 移动平均）
7. 告警规则创建与邮件通知
8. 报表模板与定时生成 MVP
9. 审计日志与权限细化
10. 性能与稳定性优化

---

如需我继续为各微服务生成 NestJS 模块结构草稿、数据库迁移脚本示例、或 SDK 设计规范，请继续说明需求。
