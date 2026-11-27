---
title: "人才智库系统 执行任务拆解 (tasks.md)"
description: "基于 spec.md v2.24.0（confirmed）与 plan.md v3.21.0 的原子化、有依赖、TDD 顺序、可直接执行的任务清单"
author: "TDD/SDD 架构师（AI）起草"
created: "2026-11-26"
version: "1.4.0"
status: "草案（待评审）"
category: "tasks"
tags: ["人才智库", "任务拆解", "TDD", "微服务", "追溯"]
---

# 人才智库系统 执行任务拆解 (tasks.md)

> 依据：`spec.md` v2.24.0（confirmed）> `plan.md` v3.21.0 > `constitution.md` v3.2 > `AGENTS.md` v3.2。
> v1.4.0 变更（2026-11-27）：**按标准 AI 提示词对齐**——导入映射采用标准 15 字段字典与标准输出适配；必填字段改为公司名称/岗位名称/工作地点/发布日期，行业文件优先/缺省企业确认，技能由 JD 抽取；建议类型统一“课程内容”，AI 输出采用 recommendedType/level/title/desc/accordingTo；同步 T062/T063、T074~T077、T082/T083、T092/T093、T100/T101、T108、T207/T209/T210、T213、T215/T216。
> v1.3.0 变更（2026-11-27）：**冗余表清理与口径收敛**——删除 field_mapping_cache（映射缓存走 Redis）、account_data_scope（取消“指定专业数据”范围）、permission/role_permission（权限点与角色授权代码常量）；T029/T030/T031 同步；并消除 `global/search` 重复职责（T055/T056 仅通知/数据同步状态，搜索统一归 T274/T275）；任务数与 AC 数不变。
> v1.2.1 变更（2026-11-27）：**导入来源字段口径**——必填字段=公司名称/岗位名称/工作城市/发布日期；行业分类不由文件提供（按企业行业匹配/人工确认并回写）；技能要求可选（AI 从岗位描述抽取）；同步 T062/T063、T074/T075、T076/T077、T082/T083、T092/T093、T100/T101、T108。
> v1.2 变更（2026-11-26）：按 spec v2.22 / plan v3.19 收尾口径冻结同步——①质量评分分母=正式表全部记录（T172/T173）；②**参数配置=Nacos OpenAPI（不建 sys_param）**，新增 T300~T303；③模型 Key 复核通过后由 user-service 编排 ai-service 生效、失败日志+告警不回退（T109/T110/T284/T285）；④REQ-72 统计时序预测细化，新增 T304~T309 与 AC-98（AC 总数 91）；⑤OPEN-03 冻结关闭；⑥目标企业清单全量替换（T164/T165）；⑦分析页 3 Tab→4 Tab（T239/T240）；⑧手工补录批次按日聚合（OPEN-DB-05 关闭，T172）；⑨任务调度/备份恢复为占位权限点（17 项不变、无功能入口，T030/T031/T051）；⑩NFR-01 宣传口径保留待复核、不阻塞。
> v1.1 变更（2026-11-26）：按 tasks-prompt-web 与 plan v3.18 / spec v2.21 对齐——XXL-Job 6 类（补采集）、软数智云企业标记回写、AI 协同结构/岗位需求明细/目标企业清单任务同步、AC-92~97 映射补齐、前端路径统一 `frontend/src/`、报告导出陈旧口径修正。
> 本文件是执行拆解，不得反向改写 `spec.md` / `plan.md`（constitution §1.2）。实现中发现口径需变 → 熔断上报 → 先改文档再改代码（AGENTS §2）。

---

## 0. 预检报告 (Pre-flight Check)

### 0.1 一致性检查

- **无阻塞级冲突**，可生成正式任务清单。`spec.md` v2.24.0（91 条 AC）与 `plan.md` v3.21.0 已对齐；早期 v2.3~v2.6 裁决（ImportTask 五态、账号接口 `/api/v1/admin/accounts`、页面模块归并入系统管理页、统一 `/api/v1` 前缀）均已同步并保留。
- 非阻塞差异：①spec.md 头部"plan/tasks 尚未编写"过时描述——**已修订（2026-11-26）**；②plan.md 验收标准口径——**已修订为 91 条（AC-01~98 含空洞，以 spec §7 为准）**；③plan.md 状态为"草案（待评审）"，本 tasks.md 按现版 v3.21.0 拆解，评审后如变更需回改。
- plan.md GAP-01~GAP-08 已于 2026-11-26 用户交互裁决**全部关闭**（plan §1.2）；任务上的 `[GAP-*]` 标记保留作追溯，裁决结果见 §8 销项记录。**OPEN-03（NFR-05）已于 v2.22 冻结关闭**（INV-07 显式失败与重试、不承诺 SLA；审计 90 天清理、其余不自动清理）；**REQ-72 已于 v2.22 细化**（统计时序模型，AC-98），任务见 T304~T309。

### 0.2 技术栈确认（以 plan.md §1.1 / §6.2 为准）

- 后端：Java 17（jakarta.\*，禁 javax/Java 8 写法）+ Spring Boot 3.2.x + Spring Cloud 2023.x + Spring Cloud Alibaba（Nacos 注册+配置）+ Spring Cloud Gateway + OpenFeign（内部同步）+ RestTemplate（仅外部 AI 大模型）+ MyBatis-Plus + MapStruct + EasyExcel + OpenPDF（报告导出 PDF，v3.0）+ jjwt + Knife4j + Lombok（仅 DTO/VO/Entity）；传统 MVC 分层（controller/service/mapper/model/config-client），不用 DDD。
- 中间件（**全部本机 Docker 容器**，v3.3 用户裁决 2026-11-26）：Nacos（注册+配置）、RocketMQ（5 类 Topic，5.x）、**Redis Stack（黑名单/映射缓存/限流/热点缓存/配额计数/串行锁/幂等键/向量索引 RediSearch）**、MySQL 8.0 单实例×4 库（user_db/ai_db/data_db/curriculum_db）、**Elasticsearch（全文检索，只读索引）**、**ELK（Logstash/Filebeat + Kibana 集中日志）**、**Prometheus + Grafana（指标监控）**、**XXL-Job（6 类调度任务，含采集定时执行，2.4.x）**、本地磁盘文件存储（AD-07 v2.7 裁决，取代对象存储）；**GitLab CI/CD** 流水线。
- 前端：Vue 3 + TypeScript + Vite SPA + Vue Router + Pinia + Element Plus + Axios + ECharts + Vitest + Vue Test Utils；Nginx 静态资源与 `/api/v1` 反代。
- 测试：JUnit 5 + @ParameterizedTest（表格驱动，核心逻辑强制）+ Mockito + MockMvc + Testcontainers(MySQL) + Spring Cloud Contract/Mock WebServer（Feign 契约）+ Vitest + Playwright + ArchUnit。
- 参数配置（v2.22 裁决）：系统内页面经 **Nacos OpenAPI** 读写指定 namespace/group/DataId，发布即生效；**不建 sys_param 通用表**（AD-17）；参数变更属高危操作，先建复核单、通过后发布。任务调度本期预留（不建调度管理页面）；备份恢复仅运维执行（系统内无发起入口）。

### 0.3 工作目录确认

- **工程布局（已确认，用户裁决 2026-11-26）**：前后端同一仓库；后端 Maven 多模块集中于同一文件夹（模块化，仓库根聚合 `pom.xml` + 六模块）；前端根目录 **`frontend/`**；Java 基包名 **`com.xjy`**（任务中 `<pkg>` 即 `com/xjy`，各服务包路径 `com.xjy.<service>`）：
  - 后端 Maven 模块（plan §2.3）：`gateway-service/`、`user-service/`、`ai-service/`、`data-service/`、`curriculum-service/`、`common/`、聚合 `pom.xml`。
  - 各服务内部分层（plan §2.3）：`<service>/src/main/java/<pkg>/<service>/controller|service|mapper|model|config|client`。
  - 前端（plan §2.3）：`frontend/src/api|stores|views|components|composables|types`。
- 表结构参考 `db-design.md` v1.13.0（已确认，四库分域）：Entity/Mapper 对齐该文档表名与字段，但它不是业务规则来源（constitution §1.2）。

### 0.4 前后端边界确认

| 边界 | 位置 | 依据 |
| --- | --- | --- |
| 后端业务服务 | `user-service/`（user_db）、`ai-service/`（ai_db）、`data-service/`（data_db）、`curriculum-service/`（curriculum_db） | plan §2.1/AD-01 |
| 网关横切 | `gateway-service/`（路由/JWT/限流/traceId/错误码归一/审计拦截→MQ） | plan AD-02 |
| 公共件 | `common/`（Result/ErrorCode/traceId 过滤器等，禁业务模型） | plan §2.3 |
| 前端 HTTP 出口 | `frontend/src/api/`（唯一 HTTP 出口，统一 `/api/v1`） | plan §2.3/§3.3 |
| 共享类型（契约） | 前端 `frontend/src/types/` 与后端 VO 一一对应；契约源=plan §3.2 DTO 骨架 + Knife4j | plan §3.3 |
| Mock | 前端测试桩掉 axios（Vitest）；服务间契约用 Mock WebServer/契约测试，不建独立 Mock 服务 | plan §5.1 |

### 0.5 追溯标签约定

- `[SCN-*]/[REQ-*]/[AC-*]` → spec.md；`[AD-*]`（架构决策）、`[UH-*]`（异常/边界）、`[NFR-*]`、`[GAP-*]` → plan.md；`[INV-*]` → spec §5。
- **测试类任务（单元/表格驱动/契约/集成/E2E）必带 `[AC-*]`**；纯基础设施守护类测试无对应 AC 时，显式标注"（基础设施守护类，无直接 AC）"并追溯 `[AD-*]`/`[INV-*]`。
- 架构类任务使用 `[ARCH-AD-xx]` 追溯 plan.md §1.4 决策；`[ARCH-3.3]` 追溯 plan.md §3.3 前后端联动验证约束。

### 0.6 阶段映射（承接 plan.md §6.1 里程碑）

| 本清单阶段 | plan.md 里程碑 | 内容 |
| --- | --- | --- |
| Phase 1 | M1 | 基础设施 + 网关 + 用户服务 + 前端骨架 |
| Phase 2 | M2 | 导入主链路（上传→映射→校验入库→结果面板） |
| Phase 3 | M3 | 异步 AI 打标 + 标签治理 + 数据质量 |
| Phase 4 | M4 | 产业数据看板（10 模块 + 性能验收） |
| Phase 5 | M5 | 课程服务 |
| Phase 6 | M6 | 专业调整建议 + 岗位与技能分析（含需求预测） |
| Phase 7 | M7 | 报告成果 + 模型配置完整版 + 可观测性与收尾 |

- **TDD 配对规则**：每组功能 = 测试任务（先）→ 实现任务（后）→ 前端测试（先）→ 前端实现（后）→ 里程碑 E2E 冒烟（最后）。严禁实现先于测试。
- **阶段组织**：按 plan §6.1 里程碑划分（tasks-prompt 允许阶段不限于层序）；每阶段内部严格 TDD 顺序，跨阶段依赖只向后引用，不制造循环。
- **[P] = 与其前置依赖无耦合、可与其他 [P] 任务并行**。无 [P] 标记的任务按序执行或以其依赖为准。
- 任务粒度均为 [Size: S] 或 [Size: M]（无 L）。

---

## 1. Phase 1 —— M1 基础设施 + 网关 + 用户服务 + 前端骨架

### 1.1 基础骨架与公共约束（纯配置/骨架类豁免 TDD 的任务已注明）

- [ ] T001 [ARCH-AD-01] [ARCH-AD-03] [Phase 1] [Size: M] 搭建 Maven 多模块骨架：聚合 `pom.xml` + `common`/`gateway-service`/`user-service`/`ai-service`/`data-service`/`curriculum-service` 七模块，Java 17 + jakarta.\* + Spring Boot 3.2.x/Spring Cloud 2023.x 依赖锁定，各服务建立 controller/service/mapper/model/config/client 分层包。 依赖：无。 文件：`pom.xml`、`common/pom.xml`、`gateway-service/pom.xml`、`user-service/pom.xml`、`ai-service/pom.xml`、`data-service/pom.xml`、`curriculum-service/pom.xml`。 验证：`mvn -q verify` 构建成功；全局搜索无 `javax.` 引用。 完成判据：7 模块构建通过，依赖组合与 plan §6.2 兼容矩阵一致，无 javax/Java 8 写法。 来源：`plan.md#1.4 AD-01/AD-03`、`plan.md#2.3`、`plan.md#6.2`。
- [ ] T002 [ARCH-AD-03] [Phase 1] [Size: S] [P] 编写 ArchUnit 架构守护测试（Controller 不写业务规则、service 无 web 依赖、common 无业务模型、跨服务禁直连 DB）。 依赖：T001。 文件：`common/src/test/java/<pkg>/arch/ArchitectureTest.java`。 验证：ArchUnit 测试运行并全绿。 完成判据：4 条守护规则断言生效，故意破坏任一规则时测试变红。（基础设施守护类，无直接 AC） 来源：`plan.md#5.1 架构守护`；`constitution.md#4`。
- [ ] T003 [INV-07] [ARCH-AD-02] [Phase 1] [Size: S] [P] 编写统一响应与全局异常处理契约测试：`Result{code,message,data,traceId}` 序列化、业务失败→语义化 code、系统故障→500+traceId 且不泄堆栈、HTTP 状态码映射（400/401/403/404/409/413/429/500/503）。 依赖：T001。 文件：`common/src/test/java/<pkg>/web/GlobalExceptionHandlerTest.java`。 验证：测试全绿，逐档断言 plan §3.2 状态码表。 完成判据：每档状态码均有断言，traceId 在成功与失败响应中均存在。（基础设施守护类，无直接 AC） 来源：`plan.md#3.2 统一响应与状态码策略`、`plan.md#4.1`；`spec.md#INV-07`。
- [ ] T004 [INV-07] [ARCH-AD-02] [Phase 1] [Size: M] 实现 common 公共件（通过 T003）：Result/ErrorCode、traceId 过滤器、全局异常处理器、错误码归一基础。 依赖：T003。 文件：`common/src/main/java/<pkg>/web/*.java`、`common/src/main/java/<pkg>/trace/*.java`。 验证：T003 全绿。 完成判据：所有业务服务可复用该公共件且断言通过；无业务模型混入 common（T002 守护通过）。 来源：同 T003。
- [ ] T005 [ARCH-AD-02] [Phase 1] [Size: S] 编写网关路由与横切契约测试：`/api/v1` 路由表按 plan §3.2 前缀正确转发、traceId 生成/透传、下游错误归一、未知路径归一 404。 依赖：T004。 文件：`gateway-service/src/test/java/<pkg>/GatewayRouteTest.java`。 验证：测试全绿，逐一断言 §3.2 路由表四组前缀。 完成判据：路由表与 plan §3.2 完全一致（含 `/api/v1/imports/upload/**` 放开限流标记）。 来源：`plan.md#3.2 网关路由与横切策略`。
- [ ] T006 [ARCH-AD-02] [Phase 1] [Size: M] 实现 gateway-service 路由与过滤器链（通过 T005）：路由配置、traceId 过滤器、错误码归一过滤器。 依赖：T005。 文件：`gateway-service/src/main/java/<pkg>/config/RouteConfig.java`、`gateway-service/src/main/java/<pkg>/filter/*.java`。 验证：T005 全绿。 完成判据：四组路由前缀可达，响应含 traceId，下游错误码归一格式稳定。 来源：`plan.md#3.2`、`plan.md#1.4 AD-02`。
- [ ] T007 [REQ-01] [AC-01] [AC-65] [ARCH-AD-11] [Phase 1] [Size: S] 编写 JWT 机制测试：签发 Claims（角色/权限点/数据范围）、网关校验放行与 401、Redis 黑名单命中即刻拒绝、"记住我"长效 token。 依赖：T006。 文件：`user-service/src/test/java/<pkg>/auth/JwtServiceTest.java`、`gateway-service/src/test/java/<pkg>/filter/AuthFilterTest.java`。 验证：测试全绿（含黑名单后立即 401 断言）。 完成判据：覆盖 AC-65"停用立即禁止访问"的 token 层机制断言。 来源：`plan.md#1.4 AD-11`；`spec.md#REQ-01, AC-01, AC-65`。
- [ ] T008 [REQ-01] [AC-01] [AC-65] [ARCH-AD-11] [Phase 1] [Size: M] 实现 JWT 签发/校验与 TokenBlacklist（通过 T007）：user-service 签发、网关校验、停用/强制下线入黑名单即刻生效、网关注入 X-User-Id/X-User-Roles/X-Perm-Points/X-Data-Scope/X-Trace-Id 头（覆盖写入，防前端伪造）。 依赖：T007。 文件：`user-service/src/main/java/<pkg>/auth/JwtService.java`、`gateway-service/src/main/java/<pkg>/filter/AuthFilter.java`。 验证：T007 全绿。 完成判据：Claims 与 plan AD-11 一致；前端伪造用户头被网关覆盖。 来源：`plan.md#1.4 AD-11`、`plan.md#3.2 网关注入头`；`spec.md#REQ-01/02`。
- [ ] T009 [NFR-02] [ARCH-AD-02] [Phase 1] [Size: S] 编写网关限流测试：超阈值返回 429、`/api/v1/imports/upload/**` 放开阈值不被限。 依赖：T006。 文件：`gateway-service/src/test/java/<pkg>/filter/RateLimitFilterTest.java`。 验证：测试全绿。 完成判据：429 响应含 traceId 与友好文案。（基础设施守护类，无直接 AC） 来源：`plan.md#1.4 AD-06`、`plan.md#3.2`；`spec.md#NFR-02`。
- [ ] T010 [NFR-02] [Phase 1] [Size: S] 实现 Redis 限流计数过滤器（通过 T009）。 依赖：T009。 文件：`gateway-service/src/main/java/<pkg>/filter/RateLimitFilter.java`。 验证：T009 全绿。 完成判据：限流规则走 Nacos 配置，上传路径豁免生效。 来源：同 T009。
- [ ] T011 [REQ-64] [AC-66] [ARCH-AD-02] [Phase 1] [Size: S] 编写网关审计拦截测试：按请求路径映射操作内容（路径映射文案+关键参数摘要）、非审计路径不误产生、生成 audit-log MQ 消息（账号/时间/内容）。 依赖：T006、T013。 文件：`gateway-service/src/test/java/<pkg>/filter/AuditFilterTest.java`。 验证：测试全绿。 完成判据：路径→操作映射表可审计扩展，消息体字段与 plan §4.3 一致。 来源：`plan.md#4.3 审计`、`plan.md#1.4 AD-02`；`spec.md#REQ-64, AC-66`。
- [ ] T012 [REQ-64] [AC-66] [Phase 1] [Size: M] 实现审计拦截与 audit-log 生产（通过 T011）：路径映射配置、MQ 异步推送 user-service（不采用各服务自产审计）。 依赖：T011。 文件：`gateway-service/src/main/java/<pkg>/filter/AuditFilter.java`、`gateway-service/src/main/java/<pkg>/config/AuditPathMapping.java`。 验证：T011 全绿。 完成判据：审计产生点唯一在网关，消息进 `audit-log` Topic。 来源：`plan.md#2.1`、`plan.md#4.3`（用户裁决 2026-11-24）。
- [ ] T013 [ARCH-AD-04] [UH-18] [AC-57] [Phase 1] [Size: S] 编写 MQ 基础件测试：5 类 Topic（ai-tagging-task/result、suggestion-task/result、audit-log、tag-changed、死信）收发封装、消费幂等（业务键 Redis setnx/DB 唯一）重复投递仅生效 1 次、重试上限进死信。 依赖：T001。 文件：`common/src/test/java/<pkg>/mq/IdempotentConsumerTest.java`、`common/src/test/java/<pkg>/mq/MqTemplateTest.java`。 验证：测试全绿（含重复投递断言）。 完成判据：幂等断言覆盖 UH-18，死信可观测。 来源：`plan.md#1.4 AD-04`、`plan.md#4.4`、`plan.md#5.2 ⑥`；`spec.md#AC-57`。
- [ ] T014 [ARCH-AD-04] [Phase 1] [Size: M] 实现 RocketMQ 收发封装、IdempotentConsumer 与重试/死信配置（通过 T013）。 依赖：T013。 文件：`common/src/main/java/<pkg>/mq/*.java`。 验证：T013 全绿。 完成判据：仅 5 类 Topic（§4.4 清单外零新增），消费者可复用幂等注解/切面。 来源：`plan.md#4.4`。
- [ ] T015 [ARCH-AD-05] [Phase 1] [Size: S] 搭建 XXL-Job 接入骨架：注册 6 类 Job 清单占位（导入超时清理/审计 90 天清理/打标失败重试扫描/配额月度统计/大文件分批编排+对账/**采集任务定时执行**）与 java.time.Clock Bean（骨架配置类，豁免 TDD）。 依赖：T001。 文件：`common/src/main/java/<pkg>/time/ClockConfig.java`、各服务 `config/XxlJobConfig.java`。 验证：应用启动注册成功，Job 控制台可见占位。 完成判据：6 类 Job 注册点就绪（plan AD-05 v3.3 恢复 XXL-Job，v3.7 增采集第 6 类），业务逻辑由后续任务填充；时间一律走 Clock 注入。 来源：`plan.md#1.4 AD-05`、`plan.md#4.4`。
- [ ] T016 [ARCH-AD-07] [Phase 1] [Size: S] [P] 编写本地磁盘文件存储客户端测试：分片上传/合并/删除、过期文件清理接口、失败显式。 依赖：T001。 文件：`common/src/test/java/<pkg>/storage/FileStorageClientTest.java`。 验证：测试全绿（Mock 存储端）。 完成判据：接口覆盖 NFR-07/08 所需分片语义。 来源：`plan.md#1.4 AD-07`（v2.7 本地磁盘裁决）、`plan.md#4.4`。
- [ ] T017 [ARCH-AD-07] [Phase 1] [Size: M] 实现本地磁盘文件存储客户端封装（通过 T016）：分片上传/合并/断点查询/生命周期清理。 依赖：T016。 文件：`common/src/main/java/<pkg>/storage/FileStorageClient.java`。 验证：T016 全绿。 完成判据：本地磁盘分片落盘/合并/清理可用（AD-07 v2.7 用户裁决 2026-11-26，取代 MinIO/对象存储）。 来源：`plan.md#1.4 AD-07`。
- [ ] T018 [NFR-07] [NFR-09] [NFR-11] [Phase 1] [Size: S] [P] 编写配置项启动期校验测试：文件上限 3GB、分批阈值 500MB、超时 30 分钟、串行锁等配置非法值（负数/0/超界）拒绝启动。 依赖：T001。 文件：`data-service/src/test/java/<pkg>/config/ImportConfigValidationTest.java`。 验证：测试全绿。 完成判据：非法配置抛出明确错误并拒绝启动（失败显式，INV-07）。 来源：`plan.md#4.2 配置与环境`；`spec.md#NFR-07/09/11`。
- [ ] T019 [NFR-07] [NFR-09] [NFR-11] [Phase 1] [Size: S] 实现 Nacos 配置项与启动期校验（通过 T018）。 依赖：T018。 文件：`data-service/src/main/java/<pkg>/config/ImportProperties.java`。 验证：T018 全绿。 完成判据：阈值全部可配置且非法值拒绝启动。 来源：`plan.md#4.2`。

### 1.1.1 检索 / 日志 / 监控 / 流水线（AD-13，v3.3 裁决 2026-11-26）

- [ ] T274 [ARCH-AD-13] [REQ-40] [Phase 1] [Size: S] [P] 编写 ES 全文检索测试：索引结构（岗位/企业/技能标签三类）、导入完成后索引同步（事件触发+XXL-Job 补偿）、`GET /api/v1/global/search` 契约与命中后回查 MySQL 权威数据、ES 不可用时显式降级提示（不静默失败，INV-07）。 依赖：T001。 文件：`data-service/src/test/java/<pkg>/search/JobSearchIndexTest.java`、`data-service/src/test/java/<pkg>/search/GlobalSearchServiceTest.java`、`data-service/src/test/java/<pkg>/controller/GlobalSearchControllerTest.java`。 验证：测试全绿（Mock ES 客户端+降级断言）。 完成判据：REQ-40 全局搜索行为断言齐备，ES 只读索引不承载业务写入；搜索接口的唯一实现归属本任务组。 来源：`spec.md#REQ-40`；`plan.md#1.4 AD-13`、`plan.md#4.4 ES`。
- [ ] T275 [ARCH-AD-13] [REQ-40] [Phase 1] [Size: M] 实现 ES 索引与全局搜索（通过 T274）：ES 客户端封装、索引映射与初始化、导入完成事件/定时补偿同步、`GET /api/v1/global/search`（命中回查 MySQL；唯一搜索实现，不另建重复 Controller）。 依赖：T274、T061。 文件：`data-service/src/main/java/<pkg>/search/JobSearchIndexService.java`、`data-service/src/main/java/<pkg>/search/GlobalSearchService.java`、`data-service/src/main/java/<pkg>/controller/GlobalSearchController.java`。 验证：T274 全绿。 完成判据：索引同步幂等、ES 故障显式降级（返回明确提示而非空结果）；通知与同步状态仍由 T056 `GlobalController` 负责。 来源：同 T274；`plan.md#3.2 /global/search`。
- [ ] T276 [ARCH-AD-13] [Phase 1] [Size: S] [P] 编写 ELK 日志接入核对单：Logback JSON（traceId/taskCode/userId/level/service）→ Filebeat → ES → Kibana 按 traceId 检索；日志不落业务库；索引保留策略生效。 依赖：T004。 文件：`docs/verify/elk-log-checklist.md`、各服务 `logback-spring.xml`。 验证：检查单逐项核对通过（样例日志可在 Kibana 检出）。 完成判据：INV-07 诊断可经 Kibana 追溯。（基础设施守护类，无直接 AC） 来源：`plan.md#4.3 集中日志`、`plan.md#1.4 AD-13`。
- [ ] T277 [ARCH-AD-13] [Phase 1] [Size: S] 实现 ELK 日志接入配置（通过 T276）：Filebeat/Logstash 配置、ES 索引模板与 ILM 保留策略、Kibana 索引模式。 依赖：T276。 文件：`deploy/elk/filebeat.yml`、`deploy/elk/logstash.conf`、`deploy/elk/index-template.json`。 验证：T276 检查单全部通过。 完成判据：日志集中可检索，保留策略可配置。 来源：同 T276。
- [ ] T278 [ARCH-AD-13] [Phase 1] [Size: S] [P] 编写 Prometheus 指标暴露测试：各服务 `/actuator/prometheus` 可抓取，含 JVM/HTTP 时延/连接池/业务关键指标（导入任务数、MQ 堆积、配额用量）。 依赖：T001。 文件：`common/src/test/java/<pkg>/metrics/MetricsExposureTest.java`。 验证：测试全绿（关键指标名断言）。 完成判据：关键指标可被 Prometheus 抓取。（基础设施守护类，无直接 AC） 来源：`plan.md#4.3 指标监控`、`plan.md#1.4 AD-13`。
- [ ] T279 [ARCH-AD-13] [Phase 1] [Size: S] 实现 Prometheus+Grafana 监控配置（通过 T278）：抓取配置、Grafana 面板 JSON（服务健康/接口时延/MQ 堆积/DB 连接/调度失败）、告警规则（死信/堆积/调度失败/配额阈值）。 依赖：T278。 文件：`deploy/monitor/prometheus.yml`、`deploy/monitor/grafana-dashboard.json`、`deploy/monitor/alerts.yml`。 验证：T278 全绿；Grafana 面板可加载。 完成判据：告警规则覆盖 plan §4.3 列出的四项。 来源：同 T278。
- [ ] T280 [ARCH-AD-13] [Phase 1] [Size: S] 配置 GitLab CI/CD 流水线：构建（`mvn -q verify` 含 ArchUnit/单测）+ 前端（Vitest/build）+ 镜像打包 + 失败阻断；内网无 Runner 时提供等价本地脚本。 依赖：T001、T002。 文件：`.gitlab-ci.yml`、`scripts/ci-local.*`。 验证：流水线 dry-run 或本地脚本执行通过。 完成判据：任一阶段失败阻断交付（plan §6.2 处置）。 来源：`plan.md#1.4 AD-13`、`plan.md#6.2`。
- [ ] T281 [ARCH-AD-13] [Phase 1] [Size: S] 更新部署编排：docker-compose 纳入 ES/Kibana/Filebeat（或 Logstash）/Prometheus/Grafana/XXL-Job 容器，端口仅内网暴露，附资源配额说明。 依赖：T017、T276、T278。 文件：`deploy/docker-compose.yml`、`deploy/README.md`。 验证：`docker compose config` 校验通过；容器清单与 plan §4.2.1 一致。 完成判据：全部中间件本机容器可一键启动，对外仅 Nginx 端口。 来源：`plan.md#4.2.1`、`plan.md#4.4`。

- [ ] T282 [ARCH-AD-13] [REQ-07] [Phase 1] [Size: S] [P] 编写 Redis 向量检索测试：Redis Stack 向量索引创建（技能标签 embedding）、相似度 Top-K 检索、embedding 来源为大模型 API、索引失败/模型不可用时显式降级（回退关键词匹配并提示，不静默）。 依赖：T001。 文件：`ai-service/src/test/java/<pkg>/vector/VectorSearchClientTest.java`。 验证：测试全绿（Mock 向量+降级断言）。 完成判据：向量检索复用 Redis（不引入独立向量库），供同义词合并建议等语义场景使用。 来源：`spec.md#REQ-07`；`plan.md#1.1 技术选型`、`plan.md#4.4 Redis`。
- [ ] T283 [ARCH-AD-13] [REQ-07] [Phase 1] [Size: M] 实现 Redis 向量检索封装（通过 T282）：`VectorSearchClient`（索引创建/写入/查询/删除）、embedding 生成调用（大模型 API）、降级回退关键词匹配。 依赖：T282、T120。 文件：`ai-service/src/main/java/<pkg>/vector/VectorSearchClient.java`、`ai-service/src/main/java/<pkg>/vector/EmbeddingService.java`。 验证：T282 全绿。 完成判据：同义词/关联分析可调用语义相似检索，降级路径显式。 来源：同 T282。

### 1.2 前端骨架

- [ ] T020 [ARCH-3.3] [Phase 1] [Size: S] 编写前端 API 基座测试：axios 响应拦截统一转 Result（含网关归一错误码）、4xx/5xx/429/503 转用户级文案+traceId、useAsyncState 四态（loading/data/empty/error）吸收异常不向 UI 抛未处理异常。 依赖：无。 文件：`frontend/src/api/http.test.ts`、`frontend/src/composables/useAsyncState.test.ts`。 验证：Vitest 全绿。 完成判据：断言覆盖 plan §3.3 异常拦截与错误反馈两条约束。 来源：`plan.md#3.3 前后端联动验证约束`、`plan.md#3.1`。
- [ ] T021 [ARCH-3.3] [Phase 1] [Size: M] 实现前端 API 基座（通过 T020）：`frontend/src/api/http.ts`（baseURL 走 `VITE_API_BASE`，统一 `/api/v1`）、错误拦截、`useAsyncState`、启动脚本检查 `.env` 与 Nginx `/api/v1` 反代一致性。 依赖：T020。 文件：`frontend/src/api/http.ts`、`frontend/src/composables/useAsyncState.ts`、`scripts/check-api-base.*`。 验证：T020 全绿；脚本对不一致配置报错退出。 完成判据：api/ 为唯一 HTTP 出口，错误态可被页面直接消费。 来源：`plan.md#2.3 前端结构`、`plan.md#3.3`。
- [ ] T022 [ARCH-3.1] [Phase 1] [Size: M] 搭建前端布局与路由骨架：左右分栏后台布局（深蓝科技风、固定左侧导航侧边栏可折叠、顶部全局栏、主内容区、**桌面端自适应**，不含手机/平板独立适配）、路由表（login/dashboard/analysis/import/data-quality/admin/courses/suggestions/reports）与登录守卫骨架（含组件测试）。 依赖：T021。 文件：`frontend/src/layouts/MainLayout.vue`、`frontend/src/router/index.ts`、`frontend/src/layouts/MainLayout.test.ts`。 验证：Vitest 组件测试全绿；本地启动路由可达。 完成判据：布局三段结构与工程约定一致，未登录访问受保护路由跳转登录页。 来源：`plan.md#3.1 路由表`；工程约定（左右分栏深蓝科技风布局）。

### 1.3 用户域（登录/账号/RBAC/审计）

- [ ] T023 [REQ-01] [AC-01] [AC-02] [Phase 1] [Size: M] 编写登录 Service 表格驱动测试：凭据正确签发 JWT 进角色默认功能（AC-01）、凭据错误显式提示且不建会话（AC-02）、停用账号拒绝并显式提示（UH-01）、记住我长效 token。 依赖：T008。 文件：`user-service/src/test/java/<pkg>/service/AuthServiceTest.java`。 验证：JUnit5 表格驱动全绿。 完成判据：测试方法名引用 AC 编号（如 `ac02_wrongCredential_noSession`），4 类输入全覆盖。 来源：`spec.md#REQ-01, AC-01, AC-02`；`plan.md#1.2 UH-01`。
- [ ] T024 [REQ-01] [AC-01] [AC-02] [Phase 1] [Size: S] 实现登录 Service（通过 T023）：凭据校验、登录审计记录、JWT 签发。 依赖：T023。 文件：`user-service/src/main/java/<pkg>/service/AuthService.java`。 验证：T023 全绿。 完成判据：失败不签发 token，登录行为落 login_audit（含未知账号尝试）。 来源：同 T023；`db-design.md`（sys_account/login_audit）。
- [ ] T025 [REQ-01] [AC-01] [AC-02] [Phase 1] [Size: S] [P] 编写 auth API 契约测试：`POST /api/v1/auth/login`（400/401）、`POST /api/v1/auth/logout`（401）、`GET /api/v1/auth/me`（user+permissionPoints）。 依赖：T024。 文件：`user-service/src/test/java/<pkg>/controller/AuthControllerTest.java`。 验证：MockMvc 全绿，断言与 plan §3.2 Response 字段一致。 完成判据：登录失败响应显式 message，不建立会话。 来源：`plan.md#3.2 user-service 接口表`；`spec.md#REQ-01`。
- [ ] T026 [REQ-01] [AC-01] [AC-02] [Phase 1] [Size: S] 实现 AuthController（通过 T025）。 依赖：T025。 文件：`user-service/src/main/java/<pkg>/controller/AuthController.java`。 验证：T025 全绿。 完成判据：接口路径/入参/响应与 plan §3.2 一致，无业务逻辑下沉 Controller。 来源：同 T025。
- [ ] T027 [REQ-02] [AC-65] [AC-73] [Phase 1] [Size: M] 编写账号管理测试（Service 表格驱动 + API 契约）：创建字段=姓名、登录账号(account_name)、初始密码、工号/统一认证账号(仅展示)、所属部门、所在城市、角色、数据范围、权限说明（AccountCreateCmd v2.7，CF-05 同步；密码创建入口=系统设置→新建系统账户，用户裁决 2026-11-26）；新增即刻生效启用；密码规则 8 位以上含字母数字；停用立即禁止访问并清会话；强制下线；所在城市为看板【本地】判定依据（AC-73）。 依赖：T024。 文件：`user-service/src/test/java/<pkg>/service/AccountServiceTest.java`、`user-service/src/test/java/<pkg>/controller/AccountControllerTest.java`。 验证：测试全绿，AC-65/AC-73 用例名直引编号。 完成判据：创建/维护/停用/重置/强制下线全链路断言齐备，停用即刻 401 有断言。 来源：`spec.md#REQ-02, AC-65, AC-73`（v2.5）；`plan.md#3.2 AccountCreateCmd`（v2.7 用户裁决）。
- [ ] T028 [REQ-02] [AC-65] [AC-73] [REQ-74] [Phase 1] [Size: M] 实现账号管理（通过 T027）：AccountService + `/api/v1/admin/accounts` 系列接口（GET/POST、PUT {id}、status、password-reset、force-logout），MapStruct 仅字段搬运；**账号创建/停用/密码重置等高风险操作接入双人复核（T285）**。 依赖：T027。 文件：`user-service/src/main/java/<pkg>/service/AccountService.java`、`user-service/src/main/java/<pkg>/controller/AccountController.java`、`user-service/src/main/java/<pkg>/model/**`。 验证：T027 全绿。 完成判据：接口路径与 plan v2.5 裁决一致，停用/强制下线同步写 JWT 黑名单；高风险操作未经复核不生效。 来源：同 T027；`spec.md#REQ-74`。
- [ ] T029 [REQ-63] [AC-03] [AC-74] [UH-14] [Phase 1] [Size: M] 编写 RBAC 权限点与数据范围测试：六类角色功能隔离（AC-03/AC-74：教师不可见系统管理与数据导入）、审计员写操作 403 且无修改入口（UH-14）、数据范围（**全校/只读汇总/全系统**，v2.23 取消“指定专业”）、**账号角色与数据范围变更**立即生效并留痕。 依赖：T024。 文件：`user-service/src/test/java/<pkg>/service/RbacServiceTest.java`、`user-service/src/test/java/<pkg>/security/PermissionCheckTest.java`。 验证：测试全绿（六角色×权限点组合断言）。 完成判据：用例名直引 AC-03/AC-74；只读角色任何写入口均被拒绝。 来源：`spec.md#REQ-63, AC-03, AC-74`（v2.23）；`plan.md#1.2 UH-14`、`plan.md#3.1`。
- [ ] T030 [REQ-63] [AC-03] [AC-74] [Phase 1] [Size: M] 实现 RBAC（通过 T029）：权限点常量清单（**17 项**，spec REQ-63：含关键操作复核，裁剪课程更新审批/建议提交；**任务调度、备份恢复为占位权限点——保留于矩阵、本期无功能入口**）、**角色授权内置常量（不建 permission/role_permission 表）**、UserContext 读取网关注入头、服务内细粒度权限校验、变更留痕。 依赖：T029。 文件：`user-service/src/main/java/<pkg>/security/*.java`、`common/src/main/java/<pkg>/context/UserContext.java`。 验证：T029 全绿。 完成判据：服务层校验独立于网关（双重裁决），权限点=细粒度功能开关。 来源：`spec.md#REQ-63`（v2.23）；`plan.md#1.2 GAP-05`、`plan.md#2.4 UserContext`。
- [ ] T031 [REQ-63] [AC-75] [GAP-05] [Phase 1] [Size: S] 编写权限矩阵只读接口测试：`GET /api/v1/admin/permission-matrix` 返回 **17 项权限点 × 六类角色**授权勾选视图，只读（无写接口）；**矩阵由代码常量计算、不读权限表**；**任务调度、备份恢复为占位权限点（展示但无功能入口）**。 依赖：T030。 文件：`user-service/src/test/java/<pkg>/controller/PermissionMatrixControllerTest.java`。 验证：测试全绿。 完成判据：矩阵结构与 AC-75 一致；权限点枚举以常量清单呈现（17 项，spec REQ-63）。 来源：`spec.md#REQ-63, AC-75`（v2.23）；`plan.md#1.2 GAP-05`。
- [ ] T032 [REQ-63] [AC-75] [Phase 1] [Size: S] 实现权限矩阵只读接口（通过 T031）。 依赖：T031。 文件：`user-service/src/main/java/<pkg>/controller/PermissionMatrixController.java`。 验证：T031 全绿。 完成判据：仅 GET，RBAC 最小授权原则在视图中可核对。 来源：同 T031。
- [ ] T033 [REQ-64] [AC-76] [Phase 1] [Size: S] 编写账号概览统计测试：`GET /api/v1/admin/accounts/overview` 返回账号总数/今日在线数、各账号最近登录时间、各角色人数分布且合计=账号总数。 依赖：T028。 文件：`user-service/src/test/java/<pkg>/controller/AccountOverviewTest.java`。 验证：测试全绿（含分布合计断言）。 完成判据：AC-76 三项断言齐备。 来源：`spec.md#REQ-64, AC-76`。
- [ ] T034 [REQ-64] [AC-76] [Phase 1] [Size: S] 实现账号概览统计（通过 T033）。 依赖：T033。 文件：`user-service/src/main/java/<pkg>/service/AccountStatsService.java`。 验证：T033 全绿。 完成判据：统计口径与实际数据一致。 来源：同 T033。
- [ ] T035 [REQ-64] [AC-66] [Phase 1] [Size: M] 编写审计日志消费落库与查询测试：消费 audit-log MQ（幂等，重复投递仅落 1 条）、按时间倒序分页、仅保留 90 天、无编辑/删除入口。 依赖：T013、T012。 文件：`user-service/src/test/java/<pkg>/mq/AuditLogConsumerTest.java`、`user-service/src/test/java/<pkg>/service/AuditQueryServiceTest.java`。 验证：测试全绿（幂等/倒序/90 天边界断言）。 完成判据：AC-66 用例名直引，审计行不可编辑不可删除有接口层断言。 来源：`spec.md#REQ-64, AC-66`；`plan.md#4.3`。
- [ ] T036 [REQ-64] [AC-66] [Phase 1] [Size: S] 实现审计日志消费与查询（通过 T035）：AuditLogConsumer、审计查询 Service、登录审计查询。 依赖：T035。 文件：`user-service/src/main/java/<pkg>/mq/AuditLogConsumer.java`、`user-service/src/main/java/<pkg>/service/AuditQueryService.java`。 验证：T035 全绿。 完成判据：网关产生→MQ→user-service 落库链路贯通，字段=账号/时间/内容。 来源：同 T035。
- [ ] T037 [REQ-64] [Phase 1] [Size: S] [P] 编写审计日志 90 天清理 Job 测试：Clock 注入模拟 90 天边界，仅删除超期记录。 依赖：T015。 文件：`user-service/src/test/java/<pkg>/job/AuditCleanupJobTest.java`。 验证：测试全绿。 完成判据：90 天整数边界断言正确（保留第 90 天内记录）。 来源：`spec.md#REQ-64`；`plan.md#4.4 定时任务 ②`。
- [ ] T038 [REQ-64] [Phase 1] [Size: S] 实现审计清理 Job（通过 T037）。 依赖：T037。 文件：`user-service/src/main/java/<pkg>/job/AuditCleanupJob.java`。 验证：T037 全绿。 完成判据：XXL-Job 注册可调度，删除量记日志。 来源：同 T037。
- [ ] T039 [REQ-24] [REQ-25] [AC-30] [Phase 1] [Size: M] 编写审计查询与审计报表导出契约测试：审计员可查看/导出操作日志与登录审计，全程只读，任何修改入口返回 403。 依赖：T036。 文件：`user-service/src/test/java/<pkg>/controller/AuditControllerTest.java`。 验证：测试全绿（含只读角色写操作 403 断言）。 完成判据：AC-30 用例名直引，导出文件可获得且无修改入口。 来源：`spec.md#REQ-24, REQ-25, AC-30`；`plan.md#1.2 UH-14`。
- [ ] T040 [REQ-24] [REQ-25] [AC-30] [Phase 1] [Size: S] 实现审计查询/导出接口（通过 T039）：`/api/v1/audit/logs`、`/api/v1/audit/login-records`、`/api/v1/audit/reports/export`。 依赖：T039。 文件：`user-service/src/main/java/<pkg>/controller/AuditController.java`。 验证：T039 全绿。 完成判据：与 plan §3.2 接口表一致，导出为文件流。 来源：同 T039。
- [ ] T041 [REQ-28] [AC-33] [Phase 1] [Size: S] 编写登录页品牌统计 API 测试：`GET /api/v1/brand/stats` 实时联动（导入/判重后三指标随之变化），去重口径（企业按公司名称去重），返回 jobDemand/skillTags/enterprises/cities/records。 依赖：无（data-service 基础）。 文件：`data-service/src/test/java/<pkg>/controller/BrandStatsControllerTest.java`。 验证：测试全绿（含判重后数值变化断言）。 完成判据：AC-33 用例名直引，口径与看板一致（INV-06）。 来源：`spec.md#REQ-28, AC-33`。
- [ ] T042 [REQ-28] [AC-33] [Phase 1] [Size: S] 实现品牌统计 API（通过 T041）：去重口径实时计算。 依赖：T041。 文件：`data-service/src/main/java/<pkg>/service/BrandStatsService.java`、`data-service/src/main/java/<pkg>/controller/BrandStatsController.java`。 验证：T041 全绿。 完成判据：统计 SQL 附 INV-06/INV-01 口径注释。 来源：同 T041。

### 1.3.1 三员分立与关键操作双人复核（REQ-74，v3.6 裁决 2026-11-26）

- [ ] T284 [REQ-74] [AC-92] [Phase 1] [Size: M] 编写关键操作双人复核测试：复核单状态机（待复核→已复核/已驳回）、复核人=提交人本人被拒绝、高风险清单覆盖（账号创建/停用/密码重置、角色与权限变更、模型 Key 配置、参数配置、备份恢复）、复核通过后动作生效、驳回不生效、复核与驳回均留痕入操作日志；**模型 Key 复核通过后由 user-service 编排调用 ai-service `/internal/ai/model-config/apply` 生效；生效失败显式记录操作日志并告警、复核单保持"已复核"不回退**；**参数配置复核通过后经 Nacos OpenAPI 发布（不写 sys_param）**。 依赖：T028、T030。 文件：`user-service/src/test/java/<pkg>/service/ApprovalServiceTest.java`、`user-service/src/test/java/<pkg>/controller/ApprovalControllerTest.java`。 验证：测试全绿（六类角色×复核权限断言）。 完成判据：AC-92 用例名直引，三员分立（系统管理员提交→安全管理员复核）断言齐备，生效失败路径断言齐备。 来源：`spec.md#REQ-74, AC-92`（v2.22）；`plan.md#1.4 AD-14/AD-17`。
- [ ] T285 [REQ-74] [AC-92] [Phase 1] [Size: M] 实现关键操作双人复核（通过 T284）：复核单表与状态机、ApprovalService、`/api/v1/admin/approvals`（**POST 提交复核**/列表/通过/驳回）、高危操作改造为"提交复核→通过后生效"、复核权限点校验（复核人≠提交人）、通过后编排生效（模型 Key→ai-service apply；参数→Nacos 发布），生效失败记操作日志+告警且不回退复核单。 依赖：T284。 文件：`user-service/src/main/java/<pkg>/service/ApprovalService.java`、`user-service/src/main/java/<pkg>/controller/ApprovalController.java`、`user-service/src/main/java/<pkg>/model/Approval.java`。 验证：T284 全绿。 完成判据：高危操作未经复核不生效（失败显式），复核留痕可查，生效失败告警可观测。 来源：同 T284；`plan.md#3.2 /admin/approvals`、`plan.md#2.5`。
- [ ] T286 [REQ-74] [AC-92] [Phase 1] [Size: S] [P] 编写前端复核中心测试：系统管理页复核单列表（待复核/已复核/已驳回）、通过/驳回交互与备注、非复核权限角色不可见复核入口、提交人本人不能复核自己。 依赖：T285。 文件：`frontend/src/views/admin/ApprovalPanel.test.ts`。 验证：Vitest 全绿。 完成判据：AC-92 交互断言齐备。 来源：`spec.md#REQ-74`；`plan.md#3.1 /admin`。
- [ ] T287 [REQ-74] [AC-92] [Phase 1] [Size: S] 实现前端复核中心（通过 T286）：`frontend/src/views/admin/ApprovalPanel.vue`、`frontend/src/api/approvals.ts`，高风操作提交后提示"待复核"。 依赖：T286、T044。 文件：`frontend/src/views/admin/ApprovalPanel.vue`、`frontend/src/api/approvals.ts`。 验证：T286 全绿。 完成判据：复核中心在系统管理页按权限点显隐。 来源：同 T286。

### 1.3.2 参数配置（Nacos，AD-17，v2.22 裁决 2026-11-26）

- [ ] T300 [REQ-63] [AC-75] [AC-92] [ARCH-AD-17] [Phase 1] [Size: S] 编写参数配置测试：`GET /api/v1/admin/params` 返回 Nacos 配置项（key/value/说明，读取当前值）；`PUT /api/v1/admin/params/{key}` 提交变更→建复核单（待复核）→复核通过后经 Nacos OpenAPI 发布到指定 namespace/group/DataId 并即时生效；**断言不写 sys_param 表（无该表读写）**；无权限 403；发布失败显式报错+告警、复核单保持"已复核"不回退。 依赖：T285。 文件：`user-service/src/test/java/<pkg>/service/ParamConfigServiceTest.java`、`user-service/src/test/java/<pkg>/controller/ParamConfigControllerTest.java`。 验证：测试全绿（Mock Nacos OpenAPI + 发布失败断言）。 完成判据：AC-92 参数配置复核链路断言齐备，Nacos 读写目标 namespace/group/DataId 可配置。 来源：`spec.md#REQ-63, REQ-74, AC-75, AC-92`（v2.22）；`plan.md#1.4 AD-17`、`plan.md#3.2 /admin/params`。
- [ ] T301 [REQ-63] [AC-75] [AC-92] [ARCH-AD-17] [Phase 1] [Size: M] 实现参数配置（通过 T300）：`ParamConfigService`（Nacos OpenAPI 客户端封装）+ `/api/v1/admin/params` GET/PUT，变更接入复核编排与发布，发布失败日志+告警。 依赖：T300。 文件：`user-service/src/main/java/<pkg>/service/ParamConfigService.java`、`user-service/src/main/java/<pkg>/controller/ParamConfigController.java`、`user-service/src/main/java/<pkg>/client/NacosConfigClient.java`。 验证：T300 全绿。 完成判据：配置发布即生效；不建 sys_param、不做双写；失败显式（INV-07）。 来源：同 T300；`plan.md#2.1 参数配置`。
- [ ] T302 [REQ-63] [AC-75] [Phase 1] [Size: S] [P] 编写前端参数配置页测试：系统管理页参数配置功能区（列表/编辑表单/提交复核提示"待复核"/复核后生效提示/发布失败提示），无参数配置权限点不可见。 依赖：T301、T048。 文件：`frontend/src/views/admin/ParamPanel.test.ts`。 验证：Vitest 全绿。 完成判据：AC-75 权限显隐与 AC-92 复核交互断言齐备。 来源：`spec.md#REQ-63, AC-75, AC-92`；`plan.md#3.1 /admin`。
- [ ] T303 [REQ-63] [AC-75] [Phase 1] [Size: S] 实现前端参数配置页（通过 T302）：`frontend/src/views/admin/ParamPanel.vue`、`frontend/src/api/params.ts`。 依赖：T302、T287。 文件：`frontend/src/views/admin/ParamPanel.vue`、`frontend/src/api/params.ts`。 验证：T302 全绿。 完成判据：参数配置归入系统管理页功能区，按权限点显隐。 来源：同 T302。

### 1.4 前端用户域与系统管理页

- [ ] T043 [REQ-01] [AC-01] [AC-02] [ARCH-3.3] [Phase 1] [Size: S] [P] 编写前端 auth api/store 测试：登录成功进默认页、失败错误提示展示（凭据错误文案）、401 拦截跳转、登出清态。 依赖：T020。 文件：`frontend/src/api/auth.test.ts`、`frontend/src/stores/auth.test.ts`。 验证：Vitest 全绿（axios 桩）。 完成判据：错误映射与后端 Result/错误码一致。 来源：`spec.md#REQ-01, AC-01, AC-02`；`plan.md#3.3`。
- [ ] T044 [REQ-01] [AC-01] [AC-02] [Phase 1] [Size: S] 实现前端 auth api/store（通过 T043）。 依赖：T043。 文件：`frontend/src/api/auth.ts`、`frontend/src/stores/auth.ts`。 验证：T043 全绿。 完成判据：token 与用户权限态入 Pinia，角色默认页路由正确。 来源：同 T043。
- [ ] T045 [REQ-28] [AC-33] [Phase 1] [Size: S] 编写登录页测试：品牌信息（标题/项目图标/定位文案/建设单位署名/"2026 产业人才智库"提示）+ 三统计指标与定位文案数字实时联动。 依赖：T041。 文件：`frontend/src/views/LoginView.test.ts`。 验证：Vitest 全绿。 完成判据：AC-33 断言品牌文案与指标绑定。 来源：`spec.md#REQ-28, AC-33`。
- [ ] T046 [REQ-28] [AC-33] [Phase 1] [Size: S] 实现登录页（通过 T045）。 依赖：T045、T044。 文件：`frontend/src/views/LoginView.vue`。 验证：T045 全绿。 完成判据：品牌与三指标与 `/api/v1/brand/stats` 联动。 来源：同 T045。
- [ ] T047 [REQ-63] [AC-74] [Phase 1] [Size: S] 编写前端权限守卫与菜单显隐测试：usePermission 按角色/权限点显隐功能区（课程教师不见系统管理与数据导入），路由守卫拒绝越权访问。 依赖：T030。 文件：`frontend/src/composables/usePermission.test.ts`、`frontend/src/router/guard.test.ts`。 验证：Vitest 全绿。 完成判据：AC-74 用例名直引，六角色菜单断言齐备。 来源：`spec.md#REQ-63, AC-74`；`plan.md#3.1`。
- [ ] T048 [REQ-63] [AC-74] [Phase 1] [Size: S] 实现 usePermission 与菜单显隐（通过 T047）。 依赖：T047。 文件：`frontend/src/composables/usePermission.ts`、`frontend/src/router/guard.ts`。 验证：T047 全绿。 完成判据：系统管理页功能区按权限点显隐（plan v2.4 页面裁决）。 来源：同 T047；`plan.md#3.1 /admin`。
- [ ] T049 [REQ-02] [AC-65] [AC-73] [Phase 1] [Size: M] 编写账号管理功能区测试：账号列表、创建表单（九字段：姓名/登录账号/初始密码/工号或统一认证账号(仅展示)/所属部门/所在城市/角色/数据范围/权限说明）、维护（含状态启停）、密码重置、强制下线交互。 依赖：T028。 文件：`frontend/src/views/admin/AccountPanel.test.ts`。 验证：Vitest 全绿。 完成判据：AC-65/AC-73 交互断言（创建即生效、停用立即禁访提示）齐备。 来源：`spec.md#REQ-02, AC-65, AC-73`；`plan.md#3.2`。
- [ ] T050 [REQ-02] [AC-65] [AC-73] [Phase 1] [Size: M] 实现账号管理功能区（通过 T049）：`/admin` 账号管理面板。 依赖：T049、T044。 文件：`frontend/src/views/admin/AccountPanel.vue`、`frontend/src/api/accounts.ts`、`frontend/src/stores/admin.ts`。 验证：T049 全绿。 完成判据：字段与 AccountCreateCmd 一致（不含被裁撤字段）。 来源：同 T049。
- [ ] T051 [REQ-63] [AC-75] [REQ-64] [AC-76] [Phase 1] [Size: S] 编写权限矩阵视图与账号概览测试：只读勾选视图（**17×6**，含任务调度/备份恢复占位项）、账号概览卡（总数/今日在线/角色分布）。 依赖：T032、T034。 文件：`frontend/src/views/admin/PermissionMatrix.test.ts`。 验证：Vitest 全绿。 完成判据：AC-75/AC-76 断言齐备，矩阵无编辑交互、占位权限点无功能入口。 来源：`spec.md#AC-75, AC-76`。
- [ ] T052 [REQ-63] [AC-75] [REQ-64] [AC-76] [Phase 1] [Size: S] 实现权限矩阵视图与账号概览（通过 T051）。 依赖：T051。 文件：`frontend/src/views/admin/PermissionMatrix.vue`。 验证：T051 全绿。 完成判据：与后端 VO 字段一致。 来源：同 T051。
- [ ] T053 [REQ-24] [REQ-64] [AC-30] [AC-66] [Phase 1] [Size: S] 编写审计日志功能区测试：系统管理页"审计日志"按钮点击展开日志列表（操作日志/登录审计/审计报表导出），只读无修改入口。 依赖：T040。 文件：`frontend/src/views/admin/AuditPanel.test.ts`。 验证：Vitest 全绿。 完成判据：AC-30/AC-66 交互断言齐备（按钮展开式，无独立标签页）。 来源：`spec.md#REQ-24/25/64, AC-30, AC-66`；`plan.md#2.4 v2.4 页面裁决`。
- [ ] T054 [REQ-24] [REQ-64] [AC-30] [AC-66] [Phase 1] [Size: S] 实现审计日志功能区（通过 T053）。 依赖：T053。 文件：`frontend/src/views/admin/AuditPanel.vue`、`frontend/src/api/audit.ts`。 验证：T053 全绿。 完成判据：列表倒序、导出可用、无修改入口。 来源：同 T053。
- [ ] T055 [REQ-40] [Phase 1] [Size: S] [P] 编写全局功能 API 测试（非搜索部分）：`/api/v1/global/notifications`（数据更新提醒）、`/api/v1/global/data-sync-status`（最近更新时间+季度更新标签）。 依赖：无。 文件：`data-service/src/test/java/<pkg>/controller/GlobalControllerTest.java`。 验证：测试全绿。 完成判据：两接口契约与 plan §3.2 一致；搜索接口契约由 T274 覆盖。（功能类无独立 AC，追溯 REQ-40） 来源：`spec.md#REQ-40`；`plan.md#3.2`。
- [ ] T056 [REQ-40] [Phase 1] [Size: S] 实现全局功能 API（非搜索部分，通过 T055）：通知提醒、数据同步状态。 依赖：T055。 文件：`data-service/src/main/java/<pkg>/controller/GlobalController.java`。 验证：T055 全绿。 完成判据：同步状态含"季度更新"标签；搜索统一由 T275 的 `GlobalSearchController` 提供（不重复实现）。 来源：同 T055。
- [ ] T057 [REQ-40] [Phase 1] [Size: S] 编写前端全局功能接入测试：顶部全局栏搜索、通知提醒、数据同步状态展示。 依赖：T056、T275。 文件：`frontend/src/components/GlobalBar.test.ts`。 验证：Vitest 全绿。 完成判据：三功能交互断言齐备。 来源：`spec.md#REQ-40`。
- [ ] T058 [REQ-40] [Phase 1] [Size: S] 实现顶部全局栏（通过 T057）。 依赖：T057。 文件：`frontend/src/components/GlobalBar.vue`。 验证：T057 全绿。 完成判据：全局栏并入主布局顶部。 来源：同 T057。
- [ ] T059 [AC-01, AC-02, AC-03, AC-30, AC-31, AC-33, AC-65, AC-66, AC-73, AC-74, AC-75, AC-76, AC-92] [Phase 1] [Size: M] 执行 M1 端到端冒烟与 AC 回归记录（Playwright）：登录→角色默认页→错误/停用拒绝→账号创建停用→权限矩阵→**关键操作提交→复核通过/驳回**→**参数配置提交复核→通过后发布 Nacos 生效**→审计留痕→品牌指标联动→审计员/只读角色无修改入口。 依赖：T024、T026、T028、T030、T032、T034、T036、T040、T042、T044、T046、T048、T050、T052、T054、T056、T058、T287、T303。 文件：`e2e/m1-smoke.spec.ts`、`docs/verify/M1-AC-regression.md`。 验证：Playwright 全绿并产出 AC 回归记录。 完成判据：M1 范围 13 条 AC 逐条有结论（通过/失败证据），AC-31（无业务审批入口）人工核对记录在案。 来源：`plan.md#6.1 M1 端到端验收`；`spec.md` 对应 AC。

---

## 2. Phase 2 —— M2 导入主链路

### 2.1 上传与导入任务

- [ ] T060 [NFR-07] [NFR-08] [AC-86] [UH-07] [UH-08] [Phase 2] [Size: M] 编写分片上传契约测试：`POST /imports/upload/init`（>3GB 显式拒绝 413 并提示超限）、`PUT .../chunk`、`GET .../{uploadId}/status`（断点续传查询）、`POST .../{uploadId}/complete`（合并分片），上传中断可断点续传或自动重试。 依赖：T017。 文件：`data-service/src/test/java/<pkg>/controller/ImportUploadControllerTest.java`。 验证：测试全绿（超限/断点/合并三类断言）。 完成判据：AC-86 用例名直引（ac86_overLimit_rejected / ac86_resumeUpload），阈值走配置。 来源：`spec.md#REQ-03, NFR-07, NFR-08, AC-86`；`plan.md#1.2 UH-07/UH-08`、`plan.md#3.2`。
- [ ] T061 [NFR-07] [NFR-08] [AC-86] [Phase 2] [Size: M] 实现分片上传（通过 T060）：本地磁盘文件存储分片/合并/断点查询，超限校验。 依赖：T060。 文件：`data-service/src/main/java/<pkg>/controller/ImportUploadController.java`、`data-service/src/main/java/<pkg>/service/ImportUploadService.java`。 验证：T060 全绿。 完成判据：3GB 上限可配置且非法值拒绝启动（T019），断点续传真实可用。 来源：同 T060。
- [ ] T062 [REQ-03] [REQ-04] [AC-04] [Phase 2] [Size: M] 编写临时任务生成测试：上传完成生成临时任务记录（任务编码 IMP-*，状态"待处理"）并返回数据给前端人工确认；文件格式校验（.xlsx/CSV-UTF-8，本期拒 JSON）；按标准字段字典 15 项脱敏保留字段（不含个人联系方式/简历原文）；按批次去重；**行业处理：文件已提供行业则优先使用，未提供则按公司名称匹配企业已有行业，输出未匹配/待确认企业列表**。 依赖：T061。 文件：`data-service/src/test/java/<pkg>/service/ImportTaskServiceTest.java`。 验证：测试全绿（编码格式/状态/字段保留/行业优先级断言）。 完成判据：AC-04 前置链路断言齐备；行业文件优先、缺省匹配结果随任务返回。 来源：`spec.md#REQ-03/04, AC-04`（v2.24）、`spec.md#SCN-02 业务规则`。
- [ ] T063 [REQ-03] [REQ-04] [AC-04] [Phase 2] [Size: S] 实现临时任务生成（通过 T062）：complete 后建任务、按标准字段脱敏入库临时表、行业文件优先/缺省匹配 `company.industry`、返回 ImportTaskVO（含待确认企业行业）。 依赖：T062。 文件：`data-service/src/main/java/<pkg>/service/ImportTaskService.java`。 验证：T062 全绿。 完成判据：任务编码唯一，状态=待处理；行业匹配只读，不提前写企业。 来源：同 T062。
- [ ] T064 [REQ-06] [AC-55] [AC-79] [ARCH-AD-04] [Phase 2] [Size: M] 编写 ImportTask 五态状态机表格驱动测试：待处理→已处理（人工确认）→分析中→已完成/失败全流转（AC-55/AC-79）；非法流转拒绝；解析失败/放弃匹配→失败终止且不产生部分入库（UH-02）；AI 失败→失败可重试→分析中；超时→删除（终态）。 依赖：T063。 文件：`data-service/src/test/java/<pkg>/service/ImportTaskStateMachineTest.java`。 验证：表格驱动全绿（合法/非法转移矩阵逐格断言）。 完成判据：状态名与 spec v2.3 五态完全一致，测试命名直引 AC-55/AC-79。 来源：`spec.md#REQ-06, AC-55, AC-79`；`plan.md#2.2 状态机`（用户裁决 v2.3）。
- [ ] T065 [REQ-06] [AC-55] [AC-79] [Phase 2] [Size: S] 实现五态状态机（通过 T064）：状态迁移守卫 + 诊断记录（INV-07）。 依赖：T064。 文件：`data-service/src/main/java/<pkg>/service/ImportTaskStateMachine.java`。 验证：T064 全绿。 完成判据：非法迁移显式抛业务错误，失败态保留诊断可重试。 来源：同 T064。
- [ ] T066 [NFR-11] [AC-88] [UH-16] [Phase 2] [Size: S] 编写导入串行锁测试：已有任务处理中时第二个导入任务进入排队，同一时刻仅 1 个任务被处理（Redis 锁+MQ 消费并发=1）。 依赖：T063。 文件：`data-service/src/test/java/<pkg>/service/ImportSerialLockTest.java`。 验证：测试全绿（并发两任务断言）。 完成判据：AC-88 用例名直引，排队状态对用户可见。 来源：`spec.md#NFR-11, AC-88`；`plan.md#1.2 UH-16`。
- [ ] T067 [NFR-11] [AC-88] [Phase 2] [Size: S] 实现导入串行锁（通过 T066）：RedisLock + 消费并发=1。 依赖：T066。 文件：`data-service/src/main/java/<pkg>/service/ImportSerialExecutor.java`。 验证：T066 全绿。 完成判据：串行语义跨实例有效。 来源：同 T066。
- [ ] T068 [UH-03] [REQ-04] [Phase 2] [Size: S] 编写超时清理 Job 测试："待处理"超 30 分钟无人确认→直接删除临时任务与本地磁盘临时文件（Clock 注入模拟边界），已处理后的任务不受影响。 依赖：T015。 文件：`data-service/src/test/java/<pkg>/job/ImportTimeoutCleanupJobTest.java`。 验证：测试全绿（29/30/31 分钟边界断言）。 完成判据：仅"待处理"可被超时删除，删除留日志。 来源：`spec.md#REQ-04, OPEN-05（已关闭=30 分钟）`；`plan.md#1.2 UH-03`。
- [ ] T069 [UH-03] [Phase 2] [Size: S] 实现导入超时清理 Job（通过 T068）。 依赖：T068。 文件：`data-service/src/main/java/<pkg>/job/ImportTimeoutCleanupJob.java`。 验证：T068 全绿。 完成判据：XXL-Job 可调度，临时文件同步清理。 来源：同 T068。
- [ ] T070 [NFR-09] [AC-87] [ARCH-AD-10] [Phase 2] [Size: M] 编写大文件分批处理测试：文件超 500MB（可配置）后台切割分批读取/解析/处理/入库，批次内本地事务，全程任务状态可见，系统级批次失败可断点续跑。 依赖：T065。 文件：`data-service/src/test/java/<pkg>/service/ImportBatchProcessorTest.java`。 验证：测试全绿（分批边界/断点续跑/状态可见断言）。 完成判据：AC-87 用例名直引，内存恒定（流式）不全量载入。 来源：`spec.md#NFR-09, AC-87`；`plan.md#1.4 AD-10`。
- [ ] T071 [NFR-09] [AC-87] [Phase 2] [Size: M] 实现流式分批解析处理（通过 T070）：EasyExcel SAX/OpenCSV + 批次事务 + 续跑游标。 依赖：T070。 文件：`data-service/src/main/java/<pkg>/service/ImportBatchProcessor.java`。 验证：T070 全绿。 完成判据：600MB 文件处理全程状态可见。 来源：同 T070。

### 2.2 字段映射

- [ ] T072 [REQ-04] [AC-04] [AC-77] [Phase 2] [Size: M] 编写字段映射缓存优先 Service 表格驱动测试：有缓存字段直接复用缓存映射（source=CACHE）、无缓存字段才 AI 预映射（source=AI 带置信度）（AC-77）；缓存错误复用风险场景显式人工确认兜底（R-03）。 依赖：T063。 文件：`data-service/src/test/java/<pkg>/service/FieldMappingServiceTest.java`。 验证：表格驱动全绿（plan §5.2 ⑤ 用例）。 完成判据：AC-77 用例名直引，缓存读写走 Redis 且有过期策略。 来源：`spec.md#REQ-04, AC-04, AC-77`；`plan.md#5.2 ⑤`。
- [ ] T073 [REQ-04] [AC-04] [AC-77] [Phase 2] [Size: S] 实现映射缓存与预映射编排（通过 T072）：Redis 映射缓存 + 缓存优先编排。 依赖：T072。 文件：`data-service/src/main/java/<pkg>/service/FieldMappingService.java`。 验证：T072 全绿。 完成判据：缓存命中/未命中路径分流正确。 来源：同 T072。
- [ ] T074 [REQ-04] [AC-78] [AC-05] [Phase 2] [Size: M] 编写字段映射视图与适配契约测试：`GET /imports/tasks/{id}/mapping` 返回 MappingViewVO{taskCode、headers、firstRow、rules[sourceCol/targetField/confidence/source/suggestion/needConfirm]、**companyIndustries[companyName/matchedIndustry/needConfirm]**}；**ai-service 标准输出 MappingSuggestionVO{mappingField/databseFieldKey/databseFieldValue/matchType/level/status} 经适配层转换**（level%→confidence 数值、status待映射→needConfirm、matchType→suggestion）；置信度 89% 等值展示、低置信/待确认字段显式提示（AC-78）；**仅文件未提供行业时展示待确认企业**；`POST .../mapping/automatch` 自动匹配剩余字段。 依赖：T073、T063。 文件：`data-service/src/test/java/<pkg>/controller/FieldMappingControllerTest.java`、`data-service/src/test/java/<pkg>/service/MappingSuggestionAdapterTest.java`。 验证：测试全绿。 完成判据：AC-78/AC-05 用例名直引，VO 与适配后字段与 plan §3.2 一致。 来源：`spec.md#REQ-04, AC-05, AC-78`（v2.24）；`plan.md#3.2 MappingViewVO/MappingSuggestionVO`。
- [ ] T075 [REQ-04] [AC-78] [AC-05] [Phase 2] [Size: S] 实现映射视图与 automatch 接口（通过 T074）：标准 AI 输出适配 + 企业行业结果与待确认列表。 依赖：T074。 文件：`data-service/src/main/java/<pkg>/controller/FieldMappingController.java`、`data-service/src/main/java/<pkg>/service/MappingSuggestionAdapter.java`。 验证：T074 全绿。 完成判据：接口与契约一致；行业匹配只读。 来源：同 T074。
- [ ] T076 [REQ-04] [AC-05] [AC-77] [UH-02] [UH-04] [Phase 2] [Size: M] 编写人工确认完成测试：提交映射规则（含**企业行业确认**）→状态转"已处理"→系统自动执行校验入库（无独立校验按钮）；**文件已提供行业时直接采用；仅未提供行业的企业需确认，未确认时保存被拒绝；确认后回写 company.industry**；用户放弃匹配→导入终止不产生部分入库（AC-05/UH-02）；AI 预映射失败→转人工全量映射并显式暴露诊断（UH-04）；保存映射写入缓存。 依赖：T073、T065。 文件：`data-service/src/test/java/<pkg>/service/MappingConfirmServiceTest.java`。 验证：测试全绿（含放弃路径"无部分入库"与行业回写断言）。 完成判据：AC-05 用例名直引，确认完成即触发校验入库链路。 来源：`spec.md#REQ-04, AC-05, AC-77`（v2.24）；`plan.md#1.2 UH-02/UH-04`。
- [ ] T077 [REQ-04] [AC-05] [Phase 2] [Size: S] 实现映射确认链路（通过 T076）：`PUT .../mapping` 确认→已处理→自动校验；文件行业优先，缺省行业确认与回写。 依赖：T076。 文件：`data-service/src/main/java/<pkg>/service/MappingConfirmService.java`。 验证：T076 全绿。 完成判据：状态迁移符合五态状态机（T065）；行业确认失败不进入校验。 来源：同 T076。
- [ ] T078 [REQ-04] [AC-77] [UH-04] [Phase 2] [Size: S] 编写 ai-service 预映射 OpenFeign 契约测试：`/internal/ai/pre-mapping` 入参 headers/firstRow、返回 MappingSuggestionVO[]（标准字段：mappingField/databseFieldKey/databseFieldValue/matchType/level/status）、ai-service 不可用返回 503 显式失败（不空集合蒙混）。 依赖：T008。 文件：`ai-service/src/test/java/<pkg>/controller/InternalAiControllerTest.java`、`data-service/src/test/java/<pkg>/client/AiClientContractTest.java`。 验证：Mock WebServer/契约测试全绿。 完成判据：双端契约断言一致（含 `databse` 固定拼写），降级行为显式（UH-04/UH-17）。 来源：`plan.md#3.2 ai-service 内部接口`、`plan.md#5.1 OpenFeign 契约`；`spec.md#REQ-04`。
- [ ] T079 [REQ-04] [AC-77] [Phase 2] [Size: S] 实现预映射内部接口与 Feign 客户端（通过 T078）：ai-service `/internal/ai/pre-mapping` 按标准提示词输出、data-service AiClient + 适配层。 依赖：T078。 文件：`ai-service/src/main/java/<pkg>/controller/InternalAiController.java`、`data-service/src/main/java/<pkg>/client/AiClient.java`。 验证：T078 全绿。 完成判据：`/internal/**` 不经网关（plan v2.6 约定）。 来源：同 T078。

### 2.3 校验入库与导入结果

- [ ] T080 [REQ-05] [AC-06] [INV-01] [Phase 2] [Size: M] 编写重复判定表格驱动测试：按「公司+岗位名称」文本匹配（trim+空白折叠+小写归一，v2.7 裁决 2026-11-26）、批内判重、命中记录入库且标重复不进统计（AC-06/INV-01）。 依赖：T065。 文件：`data-service/src/test/java/<pkg>/service/DuplicateCheckServiceTest.java`。 验证：表格驱动全绿（plan §5.2 ① 五组用例）。 完成判据：AC-06 用例名直引，归一规则显式配置化。 来源：`spec.md#REQ-05, AC-06, INV-01`；`plan.md#5.2 ①`。
- [ ] T081 [REQ-05] [AC-06] [Phase 2] [Size: S] 实现 DuplicateCheckService（通过 T080）。 依赖：T080。 文件：`data-service/src/main/java/<pkg>/service/DuplicateCheckService.java`。 验证：T080 全绿。 完成判据：重复标记为行级数据，任务级"废弃"仅行级标记（INV-08）。 来源：同 T080。
- [ ] T082 [REQ-05] [AC-56] [INV-08] [Phase 2] [Size: M] 编写校验入库 Service 测试：完整性/重复/缺失/错误范围/异常状态一体化检查；**完整性按必填字段（公司名称、岗位名称、工作地点、发布日期）校验；行业文件优先，缺省取 company.industry 回填 job_record.industry_category；技能要求允许为空（AI 从 job_text 抽取）；年龄/语言/职位类型/专业/性别/招聘需求按原文保留、可空**；通过入正式表、不通过留临时表标记废弃（AC-56/INV-08）；系统级失败不产生部分入库；校验统计正确。 依赖：T081、T077。 文件：`data-service/src/test/java/<pkg>/service/ImportValidateServiceTest.java`。 验证：测试全绿（含"无部分入库"、行业回填、技能空值、附加字段保留断言）。 完成判据：AC-56 用例名直引，废弃数据不出现在正式表查询。 来源：`spec.md#REQ-05, AC-56, INV-08`（v2.24）。
- [ ] T083 [REQ-05] [AC-56] [Phase 2] [Size: M] 实现校验入库（通过 T082）：批次事务内逐条判废、正式表/临时表分流、行业回填、技能可空、附加字段原文保留。 依赖：T082。 文件：`data-service/src/main/java/<pkg>/service/ImportValidateService.java`、`data-service/src/main/resources/mapper/ImportValidateMapper.xml`。 验证：T082 全绿。 完成判据：Mapper XML 附 INV-01/INV-08 口径注释；行业缺失且未确认不得入库。 来源：同 T082。
- [ ] T084 [REQ-05] [Phase 2] [Size: S] [P] 编写异常记录抽样查看/导出契约测试：`GET /imports/tasks/{id}/abnormal-records`（抽样查看 + 导出文件流）。 依赖：T083。 文件：`data-service/src/test/java/<pkg>/controller/AbnormalRecordControllerTest.java`。 验证：测试全绿。 完成判据：抽样参数生效，导出可获得文件。（功能类无独立 AC，追溯 REQ-05） 来源：`spec.md#REQ-05`；`plan.md#3.2`。
- [ ] T085 [REQ-05] [Phase 2] [Size: S] 实现异常记录接口（通过 T084）。 依赖：T084。 文件：`data-service/src/main/java/<pkg>/controller/AbnormalRecordController.java`。 验证：T084 全绿。 完成判据：仅展示临时表废弃/异常数据。 来源：同 T084。
- [ ] T086 [REQ-06] [AC-80] [AC-55] [UH-09] [Phase 2] [Size: M] 编写导入结果面板 API 测试：`GET /imports/tasks/{id}/result` 返回①顶部四统计标签（总记录数/有效记录数/重复记录数/缺失字段数量）、②岗位分页列表（岗位名称/城市/月薪/原始技能标签/校验是否通过）、③右侧四质量指标（必填字段完整率/重复职位识别数量/异常薪资范围数量/技能标签归一化完整度）；口径=本次导入批次（ASM-09）；异步打标未完成时"技能标签归一化完整度"显式"标注进行中"不展示数值（AC-80/UH-09）。 依赖：T083。 文件：`data-service/src/test/java/<pkg>/controller/ImportResultControllerTest.java`。 验证：测试全绿（含"标注进行中"断言）。 完成判据：AC-80 用例名直引，ImportResultVO 字段与 plan §3.2 骨架一致。 来源：`spec.md#REQ-06, AC-80, ASM-09`；`plan.md#3.2 ImportResultVO`。
- [ ] T087 [REQ-06] [AC-80] [Phase 2] [Size: M] 实现导入结果面板 API（通过 T086）。 依赖：T086。 文件：`data-service/src/main/java/<pkg>/controller/ImportResultController.java`、`data-service/src/main/java/<pkg>/service/ImportResultService.java`。 验证：T086 全绿。 完成判据：指标计算口径（按记录计缺失、去重标签数分母分子）注释可审计。 来源：同 T086。
- [ ] T088 [REQ-06] [AC-55] [Phase 2] [Size: S] [P] 编写任务记录审计留痕测试：数据版本记入审计日志（AC-55 断言项）。 依赖：T012。 文件：`data-service/src/test/java/<pkg>/service/ImportAuditServiceTest.java`。 验证：测试全绿。 完成判据：AC-55 审计断言齐备。 来源：`spec.md#REQ-06, AC-55`。
- [ ] T089 [REQ-06] [AC-55] [Phase 2] [Size: S] 实现导入审计留痕（通过 T088）。 依赖：T088。 文件：`data-service/src/main/java/<pkg>/service/ImportAuditService.java`。 验证：T088 全绿。 完成判据：审计经 audit-log Topic 汇聚 user-service。 来源：同 T088。

### 2.4 导入管理与规范

- [ ] T090 [REQ-55] [AC-58] [AC-83] [Phase 2] [Size: M] 编写导入概览与任务管理 API 测试：`GET /imports/overview`（本周任务按状态分组已完成/分析中/待处理、今日处理记录含去重后有效记录数、自动分析耗时含模型版本与队列优先级）（AC-83）；`GET /imports/tasks` 任务列表与搜索（文件/时间/记录数/状态）（AC-58）。 依赖：T065。 文件：`data-service/src/test/java/<pkg>/controller/ImportOverviewControllerTest.java`。 验证：测试全绿。 完成判据：AC-58/AC-83 用例名直引，统计与实际任务数据一致。 来源：`spec.md#REQ-55, AC-58, AC-83`。
- [ ] T091 [REQ-55] [AC-58] [AC-83] [Phase 2] [Size: S] 实现导入概览与任务管理 API（通过 T090）。 依赖：T090。 文件：`data-service/src/main/java/<pkg>/controller/ImportOverviewController.java`。 验证：T090 全绿。 完成判据：与 plan §3.2 接口一致。 来源：同 T090。
- [ ] T092 [REQ-70] [AC-81] [Phase 2] [Size: S] [P] 编写导入规范 API 测试：`GET /imports/spec` 返回格式与编码（Excel/CSV UTF-8，JSON 不支持）、必填字段清单（**公司名称/岗位名称/工作地点/发布日期**）、**标准 15 字段字典**、**行业分类说明（文件有则用，无则企业匹配/人工确认）**、**技能字段说明（不作为映射字段，由 AI 从岗位描述抽取）**、数据合规提醒（AC-81，v2.24）。 依赖：无。 文件：`data-service/src/test/java/<pkg>/controller/ImportSpecControllerTest.java`。 验证：测试全绿。 完成判据：AC-81 内容断言齐备。 来源：`spec.md#REQ-70, AC-81`（v2.24）。
- [ ] T093 [REQ-70] [AC-81] [Phase 2] [Size: S] 实现导入规范 API（通过 T092）。 依赖：T092。 文件：`data-service/src/main/java/<pkg>/controller/ImportSpecController.java`。 验证：T092 全绿。 完成判据：规范文案与 spec REQ-70 一致。 来源：同 T092。
- [ ] T094 [REQ-71] [AC-82] [Phase 2] [Size: S] [P] 编写字段识别说明 API 测试：`GET /imports/field-recognition` 返回岗位名称识别/技能实体抽取/城市标准化三识别率指标，高风险字段显式提示人工确认（ASM-17）（AC-82）。 依赖：无。 文件：`data-service/src/test/java/<pkg>/controller/FieldRecognitionControllerTest.java`。 验证：测试全绿。 完成判据：AC-82 用例名直引。 来源：`spec.md#REQ-71, AC-82`。
- [ ] T095 [REQ-71] [AC-82] [Phase 2] [Size: S] 实现字段识别说明 API（通过 T094）。 依赖：T094。 文件：`data-service/src/main/java/<pkg>/controller/FieldRecognitionController.java`。 验证：T094 全绿。 完成判据：识别率统计口径可追溯。 来源：同 T094。
- [ ] T096 [REQ-56] [AC-57] [Phase 2] [Size: S] 编写打标任务投递测试：校验入库完成→状态"分析中"→投递 ai-tagging-task MQ（含批次业务键）；投递失败显式暴露可重试。 依赖：T083、T013。 文件：`data-service/src/test/java/<pkg>/service/TaggingDispatchServiceTest.java`。 验证：测试全绿。 完成判据：AC-57 前置链路断言（状态=分析中、消息含 taskId/批次键）。 来源：`spec.md#REQ-56, AC-57, INV-09`。
- [ ] T097 [REQ-56] [AC-57] [Phase 2] [Size: S] 实现打标任务投递（通过 T096）。 依赖：T096。 文件：`data-service/src/main/java/<pkg>/service/TaggingDispatchService.java`。 验证：T096 全绿。 完成判据：异步解耦（INV-09），入库不等待打标。 来源：同 T096。

### 2.5 前端导入链路

- [ ] T098 [REQ-03] [AC-86] [AC-87] [ARCH-3.3] [Phase 2] [Size: M] 编写前端 import api/store 与分片上传组件测试：分片上传进度、断点续传、超限 413 显式文案、任务创建成功回显。 依赖：T020、T061。 文件：`frontend/src/api/imports.test.ts`、`frontend/src/stores/import.test.ts`、`frontend/src/components/ChunkUploader.test.ts`。 验证：Vitest 全绿。 完成判据：AC-86/AC-87 前端交互断言齐备。 来源：`spec.md#REQ-03, AC-86, AC-87`；`plan.md#3.3`。
- [ ] T099 [REQ-03] [AC-86] [Phase 2] [Size: M] 实现前端上传与 import api/store（通过 T098）。 依赖：T098。 文件：`frontend/src/api/imports.ts`、`frontend/src/stores/import.ts`、`frontend/src/components/ChunkUploader.vue`。 验证：T098 全绿。 完成判据：上传链路与后端接口一致。 来源：同 T098。
- [ ] T100 [REQ-04] [AC-05] [AC-77] [AC-78] [Phase 2] [Size: M] 编写字段映射界面测试：展示任务编码/文件表头/第一行数据/标准 15 字段字典的预匹配结果、各字段匹配置信度（%）与来源（缓存/AI）、低置信与待确认字段显式提示+AI 建议文案、**待确认企业行业列表与人工确认交互（仅文件未提供行业时展示，未确认不可保存）**、"自动匹配剩余字段"、保存映射=人工确认完成；放弃匹配终止导入提示。 依赖：T075。 文件：`frontend/src/views/import/MappingPanel.test.ts`。 验证：Vitest 全绿。 完成判据：AC-78/AC-05 交互断言齐备。 来源：`spec.md#REQ-04, AC-05, AC-77, AC-78`（v2.24）。
- [ ] T101 [REQ-04] [AC-05] [AC-78] [Phase 2] [Size: M] 实现字段映射界面（通过 T100）：含企业行业确认区（仅缺省时），确认后随保存提交。 依赖：T100、T099。 文件：`frontend/src/views/import/MappingPanel.vue`。 验证：T100 全绿。 完成判据：五步流程"字段映射"步骤可用，行业确认完成后方可进入自动校验。 来源：同 T100。
- [ ] T102 [REQ-06] [AC-55] [AC-79] [Phase 2] [Size: S] 编写任务列表与状态展示测试：五态状态可视化、失败任务显式诊断与重试入口、超时删除提示、任务搜索。 依赖：T065。 文件：`frontend/src/views/import/TaskList.test.ts`。 验证：Vitest 全绿。 完成判据：AC-79 状态展示断言（分析中状态正确呈现）。 来源：`spec.md#REQ-06, AC-55, AC-79`。
- [ ] T103 [REQ-06] [AC-55] [AC-79] [Phase 2] [Size: S] 实现任务列表与状态展示（通过 T102）。 依赖：T102。 文件：`frontend/src/views/import/TaskList.vue`。 验证：T102 全绿。 完成判据：状态流转展示与状态机一致。 来源：同 T102。
- [ ] T104 [REQ-06] [AC-80] [Phase 2] [Size: M] 编写导入结果面板测试：顶部四统计标签、岗位分页列表（五字段）、右侧四质量指标竖排、"标注进行中"占位（INV-09/UH-09）。 依赖：T087。 文件：`frontend/src/views/import/ImportResultPanel.test.ts`。 验证：Vitest 全绿。 完成判据：AC-80 面板断言齐备（含打标未完成不展示数值）。 来源：`spec.md#REQ-06, AC-80`。
- [ ] T105 [REQ-06] [AC-80] [Phase 2] [Size: S] 实现导入结果面板（通过 T104）。 依赖：T104。 文件：`frontend/src/views/import/ImportResultPanel.vue`。 验证：T104 全绿。 完成判据：口径标注"本次导入批次"（ASM-09）。 来源：同 T104。
- [ ] T106 [REQ-55] [REQ-70] [REQ-71] [AC-81] [AC-82] [AC-83] [Phase 2] [Size: M] 编写导入概览区与规范/字段识别入口测试：概览统计标签（本周任务分组/今日处理/分析耗时含模型版本与队列优先级）、导入规范展示与"查看字段映射说明"入口、字段识别率三指标与高风险字段提示。 依赖：T091、T093、T095。 文件：`frontend/src/views/import/ImportOverview.test.ts`。 验证：Vitest 全绿。 完成判据：AC-81/AC-82/AC-83 交互断言齐备。 来源：`spec.md#REQ-55/70/71, AC-81~83`。
- [ ] T107 [REQ-55] [REQ-70] [REQ-71] [AC-81] [AC-82] [AC-83] [Phase 2] [Size: S] 实现导入概览区与规范/识别入口（通过 T106）。 依赖：T106。 文件：`frontend/src/views/import/ImportOverview.vue`。 验证：T106 全绿。 完成判据：页面五步流程（上传→映射→校验→分析→完成）完整贯通。 来源：同 T106。
- [ ] T108 [AC-04, AC-05, AC-06, AC-07, AC-55, AC-56, AC-77, AC-78, AC-79, AC-80, AC-81, AC-82, AC-83, AC-86, AC-87, AC-88] [Phase 2] [Size: M] 执行 M2 端到端冒烟与 AC 回归记录（Playwright）：完整导入五步流程（上传→映射确认→**行业文件优先/缺省企业确认**→自动校验入库→分析中→结果面板）、**标准 15 字段字典映射**、**技能要求缺失时 AI 从岗位描述抽取**、判重不进统计、超限拒绝、断点续传、500MB 分批、串行排队、超时清理、放弃匹配无部分入库。 依赖：T061、T063、T065、T067、T069、T071、T075、T077、T079、T081、T083、T085、T087、T089、T091、T093、T095、T097、T099、T101、T103、T105、T107。 文件：`e2e/m2-import.spec.ts`、`docs/verify/M2-AC-regression.md`。 验证：Playwright 全绿并产出 AC 回归记录。 完成判据：M2 核心导入范围 17 条 AC 逐条有结论；采集/清洗链路（AC-93~95）由 T288~T299 的组件/契约测试与 T264 全量回归覆盖；异常路径（UH-02/03/04/07/08/16）均有证据。 来源：`plan.md#6.1 M2`；`spec.md` 对应 AC。

### 2.6 采集任务、手工补录与质量规则（REQ-75/76，v3.7 裁决 2026-11-26）

- [ ] T288 [REQ-75] [AC-93] [Phase 2] [Size: S] [P] 编写采集任务配置测试：新增/修改/删除采集任务（名称、来源渠道、来源类型=模板/文件拉取、目标地址、频率每日/每周/每月、增量策略、启用）、非法配置显式拒绝、执行记录倒序、失败可重试。 依赖：T081。 文件：`data-service/src/test/java/<pkg>/collect/CollectTaskServiceTest.java`、`data-service/src/test/java/<pkg>/collect/CollectTaskControllerTest.java`。 验证：测试全绿。 完成判据：AC-93 配置与执行记录断言齐备。 来源：`spec.md#REQ-75, AC-93`；`plan.md#1.4 AD-15`。
- [ ] T289 [REQ-75] [AC-93] [Phase 2] [Size: M] 实现采集任务配置（通过 T288）：`/api/v1/collect-tasks` CRUD + 执行记录查询，任务表落 data_db。 依赖：T288。 文件：`data-service/src/main/java/<pkg>/collect/CollectTaskService.java`、`data-service/src/main/java/<pkg>/controller/CollectTaskController.java`、`data-service/src/main/java/<pkg>/model/CollectTask.java`。 验证：T288 全绿。 完成判据：配置变更即时生效于下次调度。 来源：同 T288。
- [ ] T290 [REQ-75] [AC-93] [Phase 2] [Size: M] 编写定时采集执行测试：XXL-Job 触发→下载文件（代理/白名单，Mock HTTP）→创建导入任务→走映射/校验链路；增量按唯一键判重不重复入库；下载失败显式记录并可重试。 依赖：T069、T289。 文件：`data-service/src/test/java/<pkg>/collect/CollectExecutorTest.java`。 验证：测试全绿（Mock 下载+增量断言）。 完成判据：AC-93 定时与增量断言齐备。 来源：`spec.md#REQ-75, AC-93`。
- [ ] T291 [REQ-75] [AC-93] [Phase 2] [Size: M] 实现定时采集执行（通过 T290）：`CollectExecutor`（XXL-Job 第 6 类任务）+ 文件下载客户端（超时/代理/白名单）+ 复用导入链路 + 执行记录落库。 依赖：T290。 文件：`data-service/src/main/java/<pkg>/collect/CollectExecutor.java`、`data-service/src/main/java/<pkg>/collect/FileFetchClient.java`。 验证：T290 全绿。 完成判据：失败显式（INV-07），不静默跳过。 来源：同 T290。
- [ ] T292 [REQ-75] [REQ-04] [AC-93] [AC-95] [Phase 2] [Size: S] [P] 编写手工补录测试：单条录入户/岗位/招聘需求数据、强制来源渠道+原始链接、必填校验（公司名称/岗位名称/工作地点/发布日期）、**行业按企业匹配、未匹配需人工确认**、**技能要求可空（AI 从 JD 抽取）**、标准附加字段可空、入库后进入打标链路。 依赖：T081。 文件：`data-service/src/test/java/<pkg>/service/ManualRecordServiceTest.java`。 验证：测试全绿（必填缺失拒绝、行业确认断言）。 完成判据：AC-95 溯源字段断言齐备。 来源：`spec.md#REQ-75, REQ-04/05, AC-93, AC-95`（v2.24）。
- [ ] T293 [REQ-75] [AC-93] [AC-95] [Phase 2] [Size: S] 实现手工补录（通过 T292）：`POST /api/v1/records/manual`，来源渠道与原始链接必填，行业匹配/确认，技能可空，标准附加字段可空。 依赖：T292。 文件：`data-service/src/main/java/<pkg>/service/ManualRecordService.java`、`data-service/src/main/java/<pkg>/controller/ManualRecordController.java`。 验证：T292 全绿。 完成判据：补录数据与导入数据同链路打标。 来源：同 T292。
- [ ] T294 [REQ-03] [REQ-75] [AC-95] [Phase 2] [Size: S] [P] 编写溯源字段与查询测试：岗位记录含来源渠道/采集导入时间/原始链接（source_url 可空）、导入映射支持该列、溯源核对按记录查询返回三要素。 依赖：T081。 文件：`data-service/src/test/java/<pkg>/service/TraceabilityServiceTest.java`。 验证：测试全绿。 完成判据：AC-95 三要素断言齐备。 来源：`spec.md#REQ-03, AC-95`。
- [ ] T295 [REQ-03] [REQ-75] [AC-95] [Phase 2] [Size: S] 实现溯源字段与查询（通过 T294）：JobRecord 增 source_channel/collected_at/source_url，导入映射支持"原始链接"列，溯源查询接口。 依赖：T294。 文件：`data-service/src/main/java/<pkg>/model/JobRecord.java`、`data-service/src/main/java/<pkg>/service/TraceabilityService.java`。 验证：T294 全绿。 完成判据：历史数据缺失该字段时显式展示"未提供"。 来源：同 T294。
- [ ] T296 [REQ-76] [AC-94] [Phase 2] [Size: S] [P] 编写清洗规则与处理台账测试：规则列表（去空/去重/异常值/格式归一）启停与参数维护、规则执行接入导入链路、台账记录问题类型/数据量/处理动作/处理人/时间、按批次查询、规则变更留痕。 依赖：T081。 文件：`data-service/src/test/java/<pkg>/quality/CleanRuleServiceTest.java`、`data-service/src/test/java/<pkg>/quality/QualityLedgerTest.java`。 验证：测试全绿（启停生效断言）。 完成判据：AC-94 用例名直引。 来源：`spec.md#REQ-76, AC-94`；`plan.md#1.4 AD-16`。
- [ ] T297 [REQ-76] [AC-94] [Phase 2] [Size: M] 实现清洗规则与质量台账（通过 T296）：CleanRuleService（规则表+启停）、QualityLedgerService（台账写入/查询）、规则执行挂入校验入库阶段。 依赖：T296。 文件：`data-service/src/main/java/<pkg>/quality/CleanRuleService.java`、`data-service/src/main/java/<pkg>/quality/QualityLedgerService.java`、`data-service/src/main/java/<pkg>/controller/QualityController.java`。 验证：T296 全绿。 完成判据：规则参数非法拒绝保存（失败显式）。 来源：同 T296。
- [ ] T298 [REQ-75] [REQ-76] [AC-93] [AC-94] [Phase 2] [Size: M] 编写前端采集任务与质量规则页测试：采集任务列表/新增编辑/执行记录/手动触发、手工补录表单（来源+链接必填）、清洗规则启停与台账查询。 依赖：T289、T297。 文件：`frontend/src/views/import/CollectPanel.test.ts`、`frontend/src/views/import/QualityRules.test.ts`。 验证：Vitest 全绿。 完成判据：AC-93/AC-94 交互断言齐备。 来源：`spec.md#REQ-75/76`。
- [ ] T299 [REQ-75] [REQ-76] [AC-93] [AC-94] [Phase 2] [Size: M] 实现前端采集任务与质量规则页（通过 T298）：`frontend/src/views/import/CollectPanel.vue`、`frontend/src/views/import/QualityRules.vue`、`frontend/src/api/collect.ts`、`frontend/src/api/quality.ts`。 依赖：T298。 文件：`frontend/src/views/import/CollectPanel.vue`、`frontend/src/views/import/QualityRules.vue`、`frontend/src/api/collect.ts`、`frontend/src/api/quality.ts`。 验证：T298 全绿。 完成判据：采集与质量功能在数据导入页按权限点显隐（数据分析员）。 来源：同 T298。

---

## 3. Phase 3 —— M3 异步 AI 打标 + 标签治理 + 数据质量

### 3.1 AI 模型配置（最小版）与 AI 调用

- [ ] T109 [REQ-67] [AC-69] [AC-92] [Phase 3] [Size: M] 编写模型接入管理测试：新增模型（名称/服务商/接入方式/BaseUrl/APIKey/备注）保存后进入灰度流量验证；API Key 密文存储、保存后不可再查看（回显不含 Key）（AC-69）；模型启停、调用量统计字段；**模型 Key 配置提交复核，复核通过后由 user-service 编排调用 ai-service `/internal/ai/model-config/apply` 生效；生效失败显式记录操作日志并告警、复核单保持"已复核"不回退（AC-92）**。 依赖：T004、T285。 文件：`ai-service/src/test/java/<pkg>/service/ModelAdminServiceTest.java`、`ai-service/src/test/java/<pkg>/controller/ModelAdminControllerTest.java`。 验证：测试全绿（Key 不回显/生效失败告警断言）。 完成判据：AC-69/AC-92 用例名直引，Key 以 AES 密文存储（密钥走环境变量）。 来源：`spec.md#REQ-67, AC-69, REQ-74, AC-92`（v2.22）；`plan.md#4.2`、`plan.md#2.5`。
- [ ] T110 [REQ-67] [AC-69] [AC-92] [Phase 3] [Size: M] 实现模型接入管理（通过 T109）：`/api/v1/admin/models` 系列（ai_db，最小版支撑打标链路）+ 内部接口 `/internal/ai/model-config/apply`（仅 user-service 调用，复核通过后应用配置）。 依赖：T109。 文件：`ai-service/src/main/java/<pkg>/service/ModelAdminService.java`、`ai-service/src/main/java/<pkg>/controller/ModelAdminController.java`、`ai-service/src/main/java/<pkg>/controller/InternalAiController.java`。 验证：T109 全绿。 完成判据：接口与 plan §3.2 ai-service 表一致；内部接口不经网关（`/internal/**`）。 来源：同 T109；`plan.md#3.2 ai-service 内部接口`。
- [ ] T111 [REQ-68] [AC-70] [UH-11] [Phase 3] [Size: M] 编写模型路由策略测试：主备配置、触发条件（连续失败 N 次/超时/错误率）、主模型连续失败达阈值自动回落备用模型、恢复后自动切回、切换在操作日志留痕（AC-70）。 依赖：T110。 文件：`ai-service/src/test/java/<pkg>/service/ModelRouteServiceTest.java`。 验证：表格驱动全绿（失败次数边界断言）。 完成判据：AC-70 用例名直引，切换留痕可查。 来源：`spec.md#REQ-68, AC-70`；`plan.md#1.2 UH-11`。
- [ ] T112 [REQ-68] [AC-70] [Phase 3] [Size: S] 实现路由策略（通过 T111）：`/api/v1/admin/models/route-policy` + 路由执行器。 依赖：T111。 文件：`ai-service/src/main/java/<pkg>/service/ModelRouteService.java`。 验证：T111 全绿。 完成判据：策略可启停，触发条件可配置。 来源：同 T111。
- [ ] T113 [REQ-69] [AC-71] [INV-10] [UH-10] [Phase 3] [Size: M] 编写用量配额与拦截测试：月度 token 配额/预警阈值（80%）/通知方式配置；达 100% QuotaGuard 拒绝非核心任务调用并显式提示、核心任务（导入校验等人工交互链路）不受影响（AC-71/INV-10）；达阈值按所选方式提醒管理员。 依赖：T110。 文件：`ai-service/src/test/java/<pkg>/service/QuotaGuardTest.java`。 验证：测试全绿（核心/非核心任务区分断言）。 完成判据：AC-71 用例名直引，暂停有显式提示不静默。 来源：`spec.md#REQ-69, AC-71, INV-10, ASM-12`；`plan.md#1.2 UH-10`。
- [ ] T114 [REQ-69] [AC-71] [Phase 3] [Size: S] 实现配额计数与 QuotaGuard（通过 T113）：Redis 计数 + 拦截 + 预警。 依赖：T113。 文件：`ai-service/src/main/java/<pkg>/service/QuotaGuard.java`。 验证：T113 全绿。 完成判据：非核心任务清单=技能打标/同义词分析/建议报告生成（ASM-12）。 来源：同 T113。

### 3.2 异步打标链路（ai-service ↔ data-service）

- [ ] T115 [REQ-56] [AC-57] [AC-84] [UH-05] [UH-18] [Phase 3] [Size: M] 编写四类打标消费者测试：消费 ai-tagging-task 执行技能抽取映射/AI 协同岗位打标（**含子类输出：AI 应用开发[含 AI 算法研发岗]/AI 辅助测试·运维**）/**软数智云企业打标（企业级标记，入参 {id: company.id, companyName}，输出 aiFlag 0/1，回写 company.soft_digital_cloud_flag）**/岗位族归类打标；消费幂等（同批回执重复投递仅生效 1 次）；AI 调用失败显式暴露、保留诊断、可重试、不伪造成功（AC-84）；成功结果经 ai-tagging-result 回发。 依赖：T013、T110。 文件：`ai-service/src/test/java/<pkg>/mq/TaggingConsumerTest.java`。 验证：测试全绿（幂等/失败不伪造/子类与企业标记断言）。 完成判据：AC-57/AC-84 用例名直引，诊断信息（请求摘要/耗时/模型/错误码）落 ai_db。 来源：`spec.md#REQ-56, AC-57, AC-84`（v2.19）；`plan.md#1.2 UH-05/UH-18`。
- [ ] T116 [REQ-56] [AC-57] [Phase 3] [Size: M] 实现四类打标消费者与结果回发（通过 T115）：AI_COLLAB 打标同时输出子类（AI_APP_DEV/AI_TEST_OPS，非协同为空）；**SOFT_CLOUD 打标按标准提示词输入 {id: company.id, companyName}、输出 aiFlag，回写企业级标记（company）**。 依赖：T115。 文件：`ai-service/src/main/java/<pkg>/mq/TaggingConsumer.java`。 验证：T115 全绿。 完成判据：提示词在代码层配置（spec §9），无界面维护；子类与软数智云企业标记枚举与 plan/spec 一致。 来源：同 T115。
- [ ] T117 [REQ-09] [AC-11] [Phase 3] [Size: S] 编写 AI 技能抽取测试：岗位文本抽取技能/工具/工程能力三类抽取标签并保留（AC-11）。 依赖：T116。 文件：`ai-service/src/test/java/<pkg>/service/SkillExtractServiceTest.java`。 验证：测试全绿（Mock 模型输出三类标签断言）。 完成判据：AC-11 用例名直引，三类标签分别落库。 来源：`spec.md#REQ-09, AC-11`。
- [ ] T118 [REQ-09] [AC-11] [Phase 3] [Size: S] 实现技能抽取（通过 T117）。 依赖：T117。 文件：`ai-service/src/main/java/<pkg>/service/SkillExtractService.java`。 验证：T117 全绿。 完成判据：抽取结果经 MQ 回写 data_db（INV-03 只增保留）。 来源：同 T117。
- [ ] T119 [REQ-10] [AC-12] [INV-07] [Phase 3] [Size: M] 编写 AI 调用失败处理测试：超时/服务不可用→显式失败、可重新发起、保留诊断信息、不产出不完整结果（AC-12/INV-07）；RestTemplate 外部调用封装超时配置。 依赖：T116。 文件：`ai-service/src/test/java/<pkg>/client/ModelHttpClientTest.java`。 验证：测试全绿（超时/不可用/部分结果三类断言）。 完成判据：AC-12 用例名直引，禁止"空结果即成功"。 来源：`spec.md#REQ-10, AC-12, INV-07`；`plan.md#4.1`。
- [ ] T120 [REQ-10] [AC-12] [Phase 3] [Size: S] 实现 AiClient（RestTemplate）与失败处理（通过 T119）。 依赖：T119。 文件：`ai-service/src/main/java/<pkg>/client/ModelHttpClient.java`。 验证：T119 全绿。 完成判据：失败结果显式可辨识，诊断可查。 来源：同 T119。
- [ ] T121 [REQ-11] [AC-13] [INV-02] [Phase 3] [Size: M] 编写岗位族归类打标测试：固定 3 类归类打标自动生效；不符合 3 类的不打岗位族标签且不计入有效职位（AC-13/INV-02）；人工校正后以人工结果为准。 依赖：T116。 文件：`ai-service/src/test/java/<pkg>/service/JobFamilyServiceTest.java`。 验证：表格驱动全绿（3 类+无法归类+人工校正）。 完成判据：AC-13 用例名直引。 来源：`spec.md#REQ-11, AC-13, INV-02`。
- [ ] T122 [REQ-11] [AC-13] [Phase 3] [Size: S] 实现岗位族归类打标（通过 T121）。 依赖：T121。 文件：`ai-service/src/main/java/<pkg>/service/JobFamilyService.java`。 验证：T121 全绿。 完成判据：归类结果回发 data_db。 来源：同 T121。
- [ ] T123 [REQ-56] [AC-57] [AC-85] [INV-09] [Phase 3] [Size: M] 编写打标回执消费测试（data-service）：消费 ai-tagging-result 更新 tagging_status/标签映射/**AI 协同子类 ai_collab_type（AI_APP_DEV/AI_TEST_OPS，非协同 NULL，CK-15）**/**软数智云企业标记 company.soft_digital_cloud_flag**/分析指标（识别技能标签及新增、岗位类别及新增、分析置信度）；任务状态分析中→已完成/失败；统计口径边界（AC-85：未完成计入岗位需求总量但不计有效职位/岗位族/AI 协同占比）。 依赖：T096、T013。 文件：`data-service/src/test/java/<pkg>/mq/TaggingResultConsumerTest.java`。 验证：表格驱动全绿（plan §5.2 ③ 口径矩阵 + 子类/企业标记回写断言）。 完成判据：AC-57/AC-85 用例名直引，消费幂等断言齐备。 来源：`spec.md#REQ-56, AC-57, AC-85, INV-09`（v2.19/2026-11-26）；`plan.md#5.2 ③`。
- [ ] T124 [REQ-56] [AC-57] [AC-85] [Phase 3] [Size: M] 实现打标回执消费与口径落位（通过 T123）：tagging_status 更新、**ai_collab_type 子类回写**、**软数智云企业标记回写（company）**、映射自动生效（INV-03）。 依赖：T123。 文件：`data-service/src/main/java/<pkg>/mq/TaggingResultConsumer.java`。 验证：T123 全绿。 完成判据：完成前数据不计依赖标注的统计（INV-09）；子类与企业标记同受 INV-04（无人工写入口）。 来源：同 T123。

### 3.3 标签治理（data-service）

- [ ] T125 [REQ-07] [AC-08] [Phase 3] [Size: M] 编写标准技能标签维护测试：标签列表（编码/名称/分类/关联岗位数/热度指数/同比/状态）搜索筛选；新增/修改即刻生效无审核（AC-08）；两维度（类型 技能/工具/工程能力 × 独立分类字段）；启用/停用。 依赖：T004。 文件：`data-service/src/test/java/<pkg>/service/SkillTagServiceTest.java`、`data-service/src/test/java/<pkg>/controller/SkillTagControllerTest.java`。 验证：测试全绿。 完成判据：AC-08 用例名直引，热度指数与看板同口径（ASM-10）。 来源：`spec.md#REQ-07, AC-08, ASM-10`。
- [ ] T126 [REQ-07] [AC-08] [Phase 3] [Size: M] 实现标签维护（通过 T125）：`/api/v1/skill-tags` 系列。 依赖：T125。 文件：`data-service/src/main/java/<pkg>/service/SkillTagService.java`、`data-service/src/main/java/<pkg>/controller/SkillTagController.java`。 验证：T125 全绿。 完成判据：新增即生效，无"待审"状态（spec §4.2 裁剪）。 来源：同 T125。
- [ ] T127 [REQ-07] [AC-67] [ASM-03] [Phase 3] [Size: M] 编写标签合并测试：多选语义相同标签→指定保留标签执行合并→被合并标签映射关系统一指向保留标签；记录版本；发布 tag-changed 事件；岗位匹配/热度计算/课程对齐统计口径随之反映（AC-67/ASM-03）。 依赖：T126。 文件：`data-service/src/test/java/<pkg>/service/TagMergeServiceTest.java`。 验证：测试全绿（映射指向/事件/版本断言）。 完成判据：AC-67 用例名直引，合并不可逆需显式确认（前端任务 T143）。 来源：`spec.md#REQ-07, AC-67, ASM-03`。
- [ ] T128 [REQ-07] [AC-67] [Phase 3] [Size: S] 实现标签合并（通过 T127）：`POST /skill-tags/merge` + 事件发布。 依赖：T127。 文件：`data-service/src/main/java/<pkg>/service/TagMergeService.java`。 验证：T127 全绿。 完成判据：合并结果 MergeResultVO 返回受影响映射数。 来源：同 T127。
- [ ] T129 [REQ-07] [Phase 3] [Size: M] [P] 编写待处理事项测试：AI 同义词合并建议（异步分析产出、采纳后进入合并流程）、低热度标签清理提醒（连续 3 季度热度指数低于配置阈值）。 依赖：T126。 文件：`data-service/src/test/java/<pkg>/service/TagPendingServiceTest.java`。 验证：测试全绿（季度边界断言）。 完成判据：阈值走 Nacos 配置；采纳仅进入合并流程不直接删。（功能类无独立 AC，追溯 REQ-07） 来源：`spec.md#REQ-07`。
- [ ] T130 [REQ-07] [Phase 3] [Size: S] 实现待处理事项（通过 T129）：`/skill-tags/pending-items` 系列。 依赖：T129。 文件：`data-service/src/main/java/<pkg>/service/TagPendingService.java`。 验证：T129 全绿。 完成判据：同义词分析走 ai-service 异步能力。 来源：同 T129。
- [ ] T131 [REQ-08] [AC-09] [AC-10] [INV-03] [Phase 3] [Size: M] 编写标签映射与校正测试：原始标签（导入自带）与抽取标签（AI 产出）只增保留、自动映射到标准技能标签生效（AC-09/INV-03）；未映射显式标记"未映射"；数据分析员校正结果为准并保留校正记录（AC-10）。 依赖：T126。 文件：`data-service/src/test/java/<pkg>/service/TagMappingServiceTest.java`。 验证：测试全绿。 完成判据：AC-09/AC-10 用例名直引，原始标签不可被覆盖。 来源：`spec.md#REQ-08, AC-09, AC-10, INV-03`。
- [ ] T132 [REQ-08] [AC-09] [AC-10] [Phase 3] [Size: S] 实现标签映射与校正（通过 T131）：`/skill-tags/unmapped`、`/skill-tags/mappings/correct`。 依赖：T131。 文件：`data-service/src/main/java/<pkg>/service/TagMappingService.java`。 验证：T131 全绿。 完成判据：校正留痕（版本记录）。 来源：同 T131。
- [ ] T133 [REQ-07] [Phase 3] [Size: S] [P] 编写标签体系概览与版本记录 API 测试：`/skill-tags/overview`（各分类标签项数与占比）、`/skill-tags/{id}/versions`（变更版本记录）。 依赖：T126。 文件：`data-service/src/test/java/<pkg>/controller/TagOverviewControllerTest.java`。 验证：测试全绿。 完成判据：概览统计与列表口径一致。（功能类无独立 AC，追溯 REQ-07） 来源：`spec.md#REQ-07`。
- [ ] T134 [REQ-07] [Phase 3] [Size: S] 实现概览与版本记录 API（通过 T133）。 依赖：T133。 文件：`data-service/src/main/java/<pkg>/controller/TagOverviewController.java`。 验证：T133 全绿。 完成判据：与 plan §3.2 接口一致。 来源：同 T133。
- [ ] T135 [REQ-53] [ASM-03] [Phase 3] [Size: S] 编写 tag-changed 事件契约测试（生产端）：合并/修改标签时发布事件（变更类型/保留标签/被合并标签列表），重复发布消费幂等。 依赖：T128。 文件：`data-service/src/test/java/<pkg>/mq/TagChangedEventTest.java`。 验证：测试全绿。 完成判据：事件契约与 curriculum-service 消费端（T196）对齐，双端字段一致。 来源：`plan.md#4.4 ④`；`spec.md#REQ-53, ASM-03`。
- [ ] T136 [REQ-53] [ASM-03] [Phase 3] [Size: S] 实现 tag-changed 事件发布（通过 T135）。 依赖：T135。 文件：`data-service/src/main/java/<pkg>/mq/TagChangedPublisher.java`。 验证：T135 全绿。 完成判据：事件进 `tag-changed` Topic。 来源：同 T135。

### 3.4 数据质量管理

- [ ] T137 [REQ-57] [AC-59] [Phase 3] [Size: M] 编写数据质量管理测试：编辑记录、删除记录、取消/重新标记重复（取消后重新进入统计，AC-59）、维护操作记入操作日志。 依赖：T083。 文件：`data-service/src/test/java/<pkg>/service/DataQualityServiceTest.java`、`data-service/src/test/java/<pkg>/controller/DataQualityControllerTest.java`。 验证：测试全绿（"重入统计"断言）。 完成判据：AC-59 用例名直引，审计留痕齐备。 来源：`spec.md#REQ-57, AC-59`。
- [ ] T138 [REQ-57] [AC-59] [Phase 3] [Size: S] 实现数据质量管理接口（通过 T137）：`/api/v1/data/records/**`。 依赖：T137。 文件：`data-service/src/main/java/<pkg>/service/DataQualityService.java`、`data-service/src/main/java/<pkg>/controller/DataQualityController.java`。 验证：T137 全绿。 完成判据：重复标记维护与统计口径（INV-01）联动正确。 来源：同 T137。
- [ ] T139 [REQ-11] [AC-13] [INV-04] [Phase 3] [Size: S] 编写岗位族校正与 INV-04 写入口隔离断言测试：`PUT /data/records/{id}/job-family` 人工校正生效（以人工为准）；AI 协同标记/软数智云标记不存在任何写 API/Service 方法（请求返回 404/403）。 依赖：T138。 文件：`data-service/src/test/java/<pkg>/controller/JobFamilyCorrectionTest.java`。 验证：测试全绿（无写入口断言）。 完成判据：AC-13 用例名直引，INV-04 双重断言（网关+服务）。 来源：`spec.md#REQ-11, AC-13, INV-04, ASM-01`；`plan.md#2.2`。
- [ ] T140 [REQ-11] [AC-13] [INV-04] [Phase 3] [Size: S] 实现岗位族校正接口（通过 T139）：仅开放岗位族校正，AI 协同/软数智云无写入口。 依赖：T139。 文件：`data-service/src/main/java/<pkg>/controller/JobFamilyController.java`。 验证：T139 全绿。 完成判据：代码层无对应写方法（T002 ArchUnit 辅助守护）。 来源：同 T139。
- [ ] T141 [REQ-36] [AC-38] [Phase 3] [Size: S] [P] 编写数据质量报告生成下载测试：`GET /api/v1/data/reports/data-quality` 返回文件流（C-06 数据质量报告）。 依赖：T087。 文件：`data-service/src/test/java/<pkg>/controller/DataQualityReportControllerTest.java`。 验证：测试全绿。 完成判据：AC-38 下载断言（文件可获得）。 来源：`spec.md#REQ-36, AC-38`。
- [ ] T142 [REQ-36] [AC-38] [Phase 3] [Size: S] 实现数据质量报告（通过 T141）：报告文件存本地磁盘文件存储。 依赖：T141。 文件：`data-service/src/main/java/<pkg>/service/DataQualityReportService.java`。 验证：T141 全绿。 完成判据：报告内容含 REQ-06 质量指标口径说明。 来源：同 T141。

### 3.5 前端治理页面

- [ ] T143 [REQ-07] [AC-08] [AC-67] [Phase 3] [Size: M] 编写标签体系维护 UI 测试（系统管理页功能区）：标签列表搜索/筛选/统计列、新增/修改表单（名称/编码/分类/状态）、合并标签流程（多选→指定保留标签→确认）、待处理事项（同义词建议/低热度提醒）、标签体系概览、变更版本记录。 依赖：T126、T128、T130、T134。 文件：`frontend/src/views/admin/TagAdminPanel.test.ts`。 验证：Vitest 全绿。 完成判据：AC-08/AC-67 交互断言齐备，合并需显式指定保留标签。 来源：`spec.md#REQ-07, AC-08, AC-67`；`plan.md#3.1 /admin`。
- [ ] T144 [REQ-07] [AC-08] [AC-67] [Phase 3] [Size: M] 实现标签体系维护 UI（通过 T143）：技能标签维护归入系统管理页（plan v2.4 裁决）。 依赖：T143。 文件：`frontend/src/views/admin/TagAdminPanel.vue`、`frontend/src/api/skillTags.ts`、`frontend/src/stores/label.ts`。 验证：T143 全绿。 完成判据：按权限点显隐（数据分析员可见）。 来源：同 T143。
- [ ] T145 [REQ-08] [AC-10] [Phase 3] [Size: S] [P] 编写标签映射校正 UI 测试：未映射标签显式列表、校正交互（选择标准标签）、校正记录展示。 依赖：T132。 文件：`frontend/src/views/admin/TagMappingPanel.test.ts`。 验证：Vitest 全绿。 完成判据：AC-10 交互断言齐备。 来源：`spec.md#REQ-08, AC-10`。
- [ ] T146 [REQ-08] [AC-10] [Phase 3] [Size: S] 实现标签映射校正 UI（通过 T145）。 依赖：T145。 文件：`frontend/src/views/admin/TagMappingPanel.vue`。 验证：T145 全绿。 完成判据：与后端接口一致。 来源：同 T145。
- [ ] T147 [REQ-57] [AC-59] [REQ-11] [AC-13] [Phase 3] [Size: M] 编写数据质量管理页测试（/data-quality）：记录编辑/删除、重复标记取消/重新标记、岗位族人工校正；AI 协同/软数智云标记不可修改的界面表现（无修改入口）。 依赖：T138、T140。 文件：`frontend/src/views/DataQualityView.test.ts`。 验证：Vitest 全绿。 完成判据：AC-59/AC-13 交互断言齐备。 来源：`spec.md#REQ-57/11, AC-59, AC-13`。
- [ ] T148 [REQ-57] [AC-59] [Phase 3] [Size: M] 实现数据质量管理页（通过 T147）。 依赖：T147。 文件：`frontend/src/views/DataQualityView.vue`、`frontend/src/api/dataQuality.ts`。 验证：T147 全绿。 完成判据：维护操作触发审计留痕提示。 来源：同 T147。
- [ ] T149 [AC-08, AC-09, AC-10, AC-11, AC-12, AC-13, AC-57, AC-58, AC-59, AC-67, AC-84, AC-85] [Phase 3] [Size: M] 执行 M3 端到端冒烟与 AC 回归记录（Playwright）：导入→异步打标→结果与分析指标→标签新增/合并/校正→数据质量管理→统计口径（标注完成前后/重复标记/岗位族）核对。 依赖：T110、T112、T114、T116、T118、T120、T122、T124、T126、T128、T130、T132、T134、T136、T138、T140、T142、T144、T146、T148。 文件：`e2e/m3-tagging.spec.ts`、`docs/verify/M3-AC-regression.md`。 验证：Playwright 全绿并产出 AC 回归记录。 完成判据：M3 范围 12 条 AC 逐条有结论，INV-03/04/09 不变量有人工核对记录。 来源：`plan.md#6.1 M3`；`spec.md` 对应 AC。

---

## 4. Phase 4 —— M4 产业数据看板

### 4.1 统计口径与计算核心

- [ ] T150 [INV-06] [AC-15] [AC-16] [AC-17] [Phase 4] [Size: M] 编写同比/环比计算表格驱动测试：同比 =(本期−同期)/同期×100%、环比 =(本期−上期)/上期×100%；同期/上期为 0 或无数据→显式"无法计算"（yoyComputable=false）禁止显示 0%（AC-17/UH-06）；季度对季度、年度对年度口径（AC-15/16）。 依赖：无。 文件：`data-service/src/test/java/<pkg>/stats/StatsCalculatorTest.java`。 验证：表格驱动全绿（plan §5.2 ② 四组用例）。 完成判据：AC-15/16/17 用例名直引，纯函数无 web/db 依赖。 来源：`spec.md#INV-06, AC-15, AC-16, AC-17`；`plan.md#5.2 ②`。
- [ ] T151 [INV-06] [Phase 4] [Size: S] 实现 StatsCalculator（通过 T150）：纯函数计算类。 依赖：T150。 文件：`data-service/src/main/java/<pkg>/stats/StatsCalculator.java`。 验证：T150 全绿。 完成判据：可被全部看板/分析接口复用（口径单点，AD-09）。 来源：同 T150。
- [ ] T152 [INV-01] [INV-02] [INV-09] [AC-20] [AC-85] [Phase 4] [Size: M] 编写 StatsQueryPolicy 统计口径过滤测试（Testcontainers MySQL）：五组合矩阵（duplicate_flag×job_family×tagging_status）验证——重复不进任何统计（AC-20）、无岗位族不计有效职位（INV-02）、打标未完成计总量不计有效职位/AI 协同占比（AC-85/INV-09）。 依赖：无。 文件：`data-service/src/test/java/<pkg>/stats/StatsQueryPolicyTest.java`。 验证：表格驱动+集成测试全绿（plan §5.2 ③）。 完成判据：AC-20/AC-85 用例名直引，过滤条件唯一出口。 来源：`spec.md#INV-01/02/06/09, AC-20, AC-85`；`plan.md#5.2 ③`、`plan.md#2.4 StatsQueryPolicy`。
- [ ] T153 [INV-01] [INV-02] [INV-09] [Phase 4] [Size: M] 实现 StatsQueryPolicy 与统计 SQL（通过 T152）：Mapper XML 附 INV-01/02/09 口径注释；联合索引（时间归属/重复标记/岗位族/打标状态）。 依赖：T152。 文件：`data-service/src/main/java/<pkg>/stats/StatsQueryPolicy.java`、`data-service/src/main/resources/mapper/StatsMapper.xml`、`data-service/src/main/resources/db/index-*.sql`。 验证：T152 全绿。 完成判据：全部统计 SQL 经该出口，跨服务禁 Join（AD-09）。 来源：同 T152。
- [ ] T154 [REQ-12] [AC-14] [AC-34] [Phase 4] [Size: S] 编写统一筛选契约测试：DashboardFilter{periodType:QUARTER|YEAR, periodValue, region:长三角|省|市, industry, jobFamily}（C-04：企业为查询条件非筛选维度）；筛选组合联动刷新（AC-14）、周期切换联动（AC-34）。 依赖：T153。 文件：`data-service/src/test/java/<pkg>/stats/DashboardFilterTest.java`。 验证：测试全绿。 完成判据：全部看板接口统一接收该筛选结构。 来源：`spec.md#REQ-12, AC-14, AC-34`；`plan.md#3.2 DashboardFilter`。
- [ ] T155 [REQ-12] [AC-14] [Phase 4] [Size: S] 实现统一筛选体系（通过 T154）：筛选参数绑定与校验。 依赖：T154。 文件：`data-service/src/main/java/<pkg>/stats/DashboardFilter.java`。 验证：T154 全绿。 完成判据：页面标注数据口径（去重职位数）字段随接口返回。 来源：同 T154。

### 4.2 看板 10 模块 API

- [ ] T156 [REQ-13] [AC-15] [AC-16] [AC-17] [Phase 4] [Size: S] 编写岗位需求总量卡 API 测试：`GET /api/v1/dashboard/job-demand`（去重岗位数/同比/环比/微型趋势折线；无法计算显式标注）。 依赖：T151、T155。 文件：`data-service/src/test/java/<pkg>/controller/JobDemandCardControllerTest.java`。 验证：测试全绿。 完成判据：AC-15/16/17 断言齐备，微趋势与 REQ-29 口径一致。 来源：`spec.md#REQ-13, AC-15~17`。
- [ ] T157 [REQ-13] [Phase 4] [Size: S] 实现岗位需求总量卡 API（通过 T156）。 依赖：T156。 文件：`data-service/src/main/java/<pkg>/controller/DashboardController.java`。 验证：T156 全绿。 完成判据：时间归属=招聘需求发布时间。 来源：同 T156。
- [ ] T158 [REQ-14] [AC-18] [Phase 4] [Size: S] [P] 编写技能标签热度卡 API 测试：去重技能标签数、较上期新增数（**首次出现判定**：季度比上季度/年度比上一年度）、紧缺项数（课程维护标签未出现在筛选范围岗位数据中）；柱状图**按月展示去重技能标签数量**（AC-18，v2.13）。 依赖：T155。 文件：`data-service/src/test/java/<pkg>/controller/SkillHeatCardControllerTest.java`。 验证：测试全绿。 完成判据：AC-18 用例名直引。 来源：`spec.md#REQ-14, AC-18`（v2.13）。
- [ ] T159 [REQ-14] [AC-18] [Phase 4] [Size: S] 实现技能标签热度卡 API（通过 T158）。 依赖：T158。 文件：`data-service/src/main/java/<pkg>/controller/SkillHeatCardController.java`。 验证：T158 全绿。 完成判据：柱图数据=按月去重标签数量（非提及次数）。 来源：同 T158。
- [ ] T160 [REQ-15] [AC-19] [UH-17] [Phase 4] [Size: M] 编写紧缺双向指标测试（表格驱动）：课程技能集（OpenFeign 课程侧快照）与筛选范围岗位技能集差集计算"课程有岗位无"/"岗位要课程无"（AC-19，plan §5.2 ④）；OpenFeign 降级→显式报错不给数（UH-17）。 依赖：T155。 文件：`data-service/src/test/java/<pkg>/stats/ShortageServiceTest.java`。 验证：表格驱动全绿（含降级断言）。 完成判据：AC-19 用例名直引，禁止静默空集合。 来源：`spec.md#REQ-15, AC-19`；`plan.md#5.2 ④`、`plan.md#1.2 UH-17`。
- [ ] T161 [REQ-15] [AC-19] [Phase 4] [Size: M] 实现紧缺双向指标与 curriculum-service `/internal/course/skill-snapshot` 契约双端（通过 T160）。 依赖：T160。 文件：`data-service/src/main/java/<pkg>/stats/ShortageService.java`、`curriculum-service/src/main/java/<pkg>/controller/InternalCourseController.java`、`data-service/src/main/java/<pkg>/client/CourseClient.java`。 验证：T160 全绿。 完成判据：课程侧快照经 OpenFeign 查询+Redis 短 TTL 缓存，跨库无 Join。 来源：同 T160；`plan.md#3.2 curriculum-service 内部接口`。
- [ ] T162 [REQ-29] [AC-35] [UH-13] [Phase 4] [Size: M] 编写岗位需求趋势 API 测试：按岗位族（固定 3 类：软件研发/数据与 AI/测试运维）多折线展示 **月度有效职位数（存量口径）**；月视图近 12 月按月、年视图近 3 年按年（预测年份由 AI 推断并标注，C-07）；AI 推断失败显式标注该年份不可用（AC-35/UH-13）；**悬浮数据=对比上月新增岗位数+环比%**；**底部分类统计=各族当期总量+同比增速**。 依赖：T155。 文件：`data-service/src/test/java/<pkg>/controller/JobTrendControllerTest.java`。 验证：测试全绿。 完成判据：AC-35 用例名直引，失败年份标注与悬浮口径断言齐备。 来源：`spec.md#REQ-29, AC-35`（v2.16）；`plan.md#1.2 UH-13`。
- [ ] T163 [REQ-29] [AC-35] [Phase 4] [Size: M] 实现趋势 API 与 ai-service `/internal/ai/predict-years` 契约双端（通过 T162）：data-service 组装 + ai-service 年份推断（失败显式标注）。 依赖：T162。 文件：`data-service/src/main/java/<pkg>/stats/JobTrendService.java`、`ai-service/src/main/java/<pkg>/controller/InternalAiController.java`。 验证：T162 全绿。 完成判据：有效职位口径=去重∧已打岗位族∧打标完成（INV-02/09）。 来源：同 T162。
- [ ] T164 [REQ-30] [REQ-31] [AC-72] [Phase 4] [Size: M] 编写 AI 协同岗位结构卡与数据覆盖企业卡 API 测试：**AI 协同岗位结构卡**（**占比=协同岗位数÷当期软件相关岗位数**[三个岗位族全部有效职位]；环形图中心=总占比；图例固定 2 子类[**AI 应用开发**（含 AI 算法研发岗）/ **AI 辅助测试·运维**]占全部软件岗位比例（子类取值=ai_collab_type，AI 打标输出），两子类之和=总占比；**同比提升=PP**（本期占比−上年同期占比，百分点差值）；**无进度条与协同岗位绝对数**；按导入期 AI 打标统计，INV-04 不可修正；判定=使用 AI 工具/AI 辅助能力，不限算法研发岗）；**数据覆盖企业卡**（去重企业主体数/本季度新增[首次入库]/覆盖城市数[范围内企业分布城市去重]/目标企业覆盖率=清单中已有数据企业数÷Excel【软数智云】目标清单总数，基准缺失时显式"基准缺失"）；**目标企业清单维护**（GET 清单摘要 / POST Excel·CSV 导入，覆盖率分母随导入更新；**全量替换**，历史版本不留存）。 依赖：T155。 文件：`data-service/src/test/java/<pkg>/controller/AiCollabCardControllerTest.java`、`data-service/src/test/java/<pkg>/controller/EnterpriseCoverageControllerTest.java`、`data-service/src/test/java/<pkg>/controller/TargetCompanyControllerTest.java`。 验证：测试全绿（软件相关岗位分母、子类之和、PP 边界、覆盖率与全量替换断言）。 完成判据：AC-72 用例名直引，两项统计与实际数据一致，清单导入后分母与旧清单完全不重叠。 来源：`spec.md#REQ-30/31, AC-72`（v2.19/v2.22）；`plan.md#1.2 GAP-03`。
- [ ] T165 [REQ-30] [REQ-31] [AC-72] [Phase 4] [Size: S] 实现 AI 协同与企业覆盖卡 API（通过 T164），含**目标企业清单导入/摘要接口**（`/dashboard/enterprise-coverage/target-list`）。 依赖：T164。 文件：`data-service/src/main/java/<pkg>/stats/AiCollabService.java`、`data-service/src/main/java/<pkg>/stats/EnterpriseCoverageService.java`、`data-service/src/main/java/<pkg>/controller/TargetCompanyController.java`。 验证：T164 全绿。 完成判据：目标企业基准表=Excel 导入维护（GAP-03 裁决 2026-11-26），清单导入后覆盖率分母即时更新；**导入采用全量替换（先清空再写入，单事务；历史版本不留存，v2.22 裁决）**。 来源：同 T164。
- [ ] T166 [REQ-32] [AC-96] [Phase 4] [Size: S] [P] 编写技能热度排行 API 测试：综合得分（**需求频次[当期关键词出现次数]+薪资溢价[技能岗位均值÷全岗位均值−1]+需求增速[同比]** 三项加权，权重 Nacos 可配）0-100（进度条满分 100）、Top N 与"查看全部"参数、增长率展示、**"新晋"标签（本期首次出现且入榜）**断言。 依赖：T155。 文件：`data-service/src/test/java/<pkg>/stats/SkillRankingServiceTest.java`。 验证：表格驱动全绿（新晋边界断言）。 完成判据：权重可配置（变更即时生效），得分公式注释引用 C-09/REQ-32。 来源：`spec.md#REQ-32, AC-96`（v2.17）。
- [ ] T167 [REQ-32] [AC-96] [Phase 4] [Size: S] 实现技能热度排行 API（通过 T166）。 依赖：T166。 文件：`data-service/src/main/java/<pkg>/stats/SkillRankingService.java`。 验证：T166 全绿。 完成判据：与看板技能热度同口径（ASM-10）。 来源：同 T166。
- [ ] T168 [REQ-33] [AC-97] [Phase 4] [Size: S] [P] 编写区域岗位需求对比 API 测试：**岗位数 TOP6 城市动态取前**横向条形（单位：个，周期内有效职位数）；当前账号所在城市在榜时返回**【本地】标记**（高亮），不在榜不返回；**极差倍数=最大值÷最小值**、**需求核心城市=岗位数最多城市**（动态生成，不写死）；城市下钻参数支持（AC-97）。 依赖：T155。 文件：`data-service/src/test/java/<pkg>/controller/RegionCompareControllerTest.java`。 验证：测试全绿（TOP6 截取、【本地】命中/未命中、极差倍数边界断言）。 完成判据：AC-97 用例名直引，城市排序与数值口径（去重）正确。 来源：`spec.md#REQ-33, AC-97`（v2.18）。
- [ ] T169 [REQ-33] [AC-97] [Phase 4] [Size: S] 实现区域对比 API（通过 T168）。 依赖：T168。 文件：`data-service/src/main/java/<pkg>/controller/RegionCompareController.java`。 验证：T168 全绿。 完成判据：TOP6 动态取前/【本地】标记/极差倍数（最大值÷最小值）与 plan §3.2 接口一致。 来源：同 T168。
- [ ] T170 [REQ-35] [AC-37] [Phase 4] [Size: M] 编写岗位需求明细 API 测试：**筛选范围内全部岗位（按岗位名称聚合）按需求量降序分页**；列=岗位名称/需求量/同比（**该岗位需求量同比增速**，无法计算显式标注）/核心技能要求；**断言响应与页面均无紧缺指数、状态字段与列（不保留占位）**；"进入岗位分析"跳转参数。 依赖：T151。 文件：`data-service/src/test/java/<pkg>/stats/JobDetailServiceTest.java`。 验证：表格驱动全绿（排序、同比边界、字段缺失断言）。 完成判据：AC-37 用例名直引，响应 DTO 无紧缺指数/状态字段。 来源：`spec.md#REQ-35, AC-37`（v2.20）。
- [ ] T171 [REQ-35] [AC-37] [Phase 4] [Size: S] 实现岗位需求明细 API（通过 T170）：`GET /api/v1/dashboard/job-details`（JobDetailVO）。 依赖：T170。 文件：`data-service/src/main/java/<pkg>/stats/JobDetailService.java`。 验证：T170 全绿。 完成判据：可进入岗位分析下钻（T223 联动）；DTO 无紧缺指数/状态字段。 来源：同 T170。
- [ ] T172 [REQ-36] [AC-38] [Phase 4] [Size: S] [P] 编写数据覆盖与质量卡 API 测试：**数据来源仅有导入**（副标题=数据来源：导入+最近导入时间）；覆盖区=**导入渠道统计**（人工导入/采集任务/手工补录的批次与记录数；**手工补录按日聚合为批次**：同一天计 1 个批次、记录数按条数），**断言不返回外部数据源分类与覆盖率百分比**；数据质量评分=**软件相关岗位数÷正式表 job_record 全部记录数×100%，1 位小数百分比**（分母不含校验失败/废弃，含重复标记与未打标；GAP-04 单指标）；最近导入任务及状态；数据质量报告下载入口。 依赖：T142。 文件：`data-service/src/test/java/<pkg>/controller/DataQualityCardControllerTest.java`。 验证：测试全绿（渠道统计、手工补录按日聚合、百分比精度、字段缺失断言）。 完成判据：AC-38 断言齐备，DTO 无外部数据源分类/覆盖率字段。 来源：`spec.md#REQ-36, AC-38`（v2.22）；`plan.md#1.2 GAP-04`。
- [ ] T173 [REQ-36] [AC-38] [Phase 4] [Size: S] 实现数据覆盖与质量卡 API（通过 T172）。 依赖：T172。 文件：`data-service/src/main/java/<pkg>/controller/DataQualityCardController.java`。 验证：T172 全绿。 完成判据：与 plan §3.2 接口一致；质量评分分母=正式表全部记录（排除校验失败/废弃），百分比保留 1 位小数。 来源：同 T172。
- [ ] T174 [NFR-03] [ARCH-AD-06] [Phase 4] [Size: S] 编写看板热点缓存测试：筛选组合维度热点缓存（Redis 短 TTL+变更失效——导入/判重/标签变更触发失效）。 依赖：T153。 文件：`data-service/src/test/java/<pkg>/stats/DashboardCacheTest.java`。 验证：测试全绿（失效触发断言）。 完成判据：缓存不改变统计口径（命中与直查结果一致）。（性能守护类，无直接 AC） 来源：`plan.md#4.3 性能保障`、`plan.md#1.4 AD-06`。
- [ ] T175 [NFR-03] [Phase 4] [Size: S] 实现看板热点缓存（通过 T174）。 依赖：T174。 文件：`data-service/src/main/java/<pkg>/stats/DashboardCache.java`。 验证：T174 全绿。 完成判据：失效链路与写路径挂钩。 来源：同 T174。
- [ ] T176 [NFR-02] [NFR-03] [Phase 4] [Size: M] 执行 50 并发 ≤3s 压测验收（M4 门禁）：看板筛选与图表加载压测，不达标则索引调优/缓存加厚/汇总表兜底（plan §6.2 处置）。 依赖：T157、T159、T161、T163、T165、T167、T169、T171、T173、T175。 文件：`perf/dashboard-load-test.*`、`docs/verify/M4-performance.md`。 验证：压测报告输出 P95 响应时间。 完成判据：50 并发下筛选与图表加载 ≤3s（NFR-03）；不达标的处置与结论记录在案。 来源：`spec.md#NFR-02, NFR-03`；`plan.md#6.1 M4`、`plan.md#6.2`。

### 4.3 前端看板

- [ ] T177 [REQ-12] [AC-14] [ARCH-3.3] [Phase 4] [Size: M] 编写前端 dashboard api/store 测试：筛选状态（周期/区域/行业/岗位族）联动、四态加载、"无法计算"文案映射（yoyComputable=false）、错误+traceId 提示。 依赖：T020、T157。 文件：`frontend/src/api/dashboard.test.ts`、`frontend/src/stores/dashboard.test.ts`。 验证：Vitest 全绿。 完成判据：AC-14 前端联动断言齐备。 来源：`spec.md#REQ-12, AC-14`；`plan.md#3.3/3.4`。
- [ ] T178 [REQ-12] [AC-14] [Phase 4] [Size: S] 实现前端 dashboard api/store（通过 T177）。 依赖：T177。 文件：`frontend/src/api/dashboard.ts`、`frontend/src/stores/dashboard.ts`。 验证：T177 全绿。 完成判据：跨页共享筛选态入 Pinia。 来源：同 T177。
- [ ] T179 [REQ-12] [REQ-38] [AC-14] [AC-34] [Phase 4] [Size: M] 编写看板页框架测试：分析周期（季度/年度）选择、三维筛选（区域/行业/岗位族）与"应用筛选"、页面数据口径标注（去重职位数）与当前视图、登录后默认落地看板、周期切换联动刷新。 依赖：T178。 文件：`frontend/src/views/DashboardView.test.ts`。 验证：Vitest 全绿。 完成判据：AC-14/AC-34 用例名直引。 来源：`spec.md#REQ-12, REQ-38, AC-14, AC-34`。
- [ ] T180 [REQ-12] [AC-34] [Phase 4] [Size: M] 实现看板页框架（通过 T179）。 依赖：T179。 文件：`frontend/src/views/DashboardView.vue`、`frontend/src/composables/usePeriodFilter.ts`。 验证：T179 全绿。 完成判据：默认路由 `/dashboard`。 来源：同 T179。
- [ ] T181 [REQ-13] [REQ-14] [REQ-30] [REQ-31] [AC-15] [AC-16] [AC-17] [AC-18] [AC-72] [Phase 4] [Size: M] 编写看板指标卡组件测试：岗位需求总量（同比/环比/微趋势/无法计算）、技能标签热度（去重数/首次出现新增/紧缺项/**月去重数量柱图**）、**AI 协同岗位结构卡（环形图中心总占比+固定 2 子类图例+同比提升 PP，无进度条/绝对数）**、数据覆盖企业卡。 依赖：T157、T159、T165。 文件：`frontend/src/components/dashboard/StatCards.test.ts`。 验证：Vitest 全绿。 完成判据：AC-15~18/72 组件断言齐备。 来源：`spec.md#REQ-13/14/30/31, AC-15~18, AC-72`。
- [ ] T182 [REQ-13] [REQ-14] [Phase 4] [Size: S] 实现指标卡组件（通过 T181）。 依赖：T181。 文件：`frontend/src/components/dashboard/StatCards.vue`。 验证：T181 全绿。 完成判据：ECharts 图表渲染正确。 来源：同 T181。
- [ ] T183 [REQ-29] [REQ-15] [AC-35] [AC-19] [Phase 4] [Size: M] 编写趋势图与紧缺双向组件测试：3 岗位族多折线（月 12/年 3 切换、预测年份标注、AI 失败年份标注不可用）、紧缺双向两数量展示。 依赖：T161、T163。 文件：`frontend/src/components/dashboard/TrendShortage.test.ts`。 验证：Vitest 全绿。 完成判据：AC-35/AC-19 组件断言齐备。 来源：`spec.md#REQ-29/15, AC-35, AC-19`。
- [ ] T184 [REQ-29] [REQ-15] [Phase 4] [Size: S] 实现趋势图与紧缺双向组件（通过 T183）。 依赖：T183。 文件：`frontend/src/components/dashboard/TrendShortage.vue`。 验证：T183 全绿。 完成判据：与后端 VO 一致。 来源：同 T183。
- [ ] T185 [REQ-32] [REQ-33] [REQ-35] [REQ-36] [AC-37] [AC-38] [AC-40] [AC-96] [AC-97] [Phase 4] [Size: M] 编写排行/区域/岗位需求明细/质量卡组件与下钻跳转测试：技能热度排行"查看全部"→分析页、**区域对比 TOP6 横向条形与【本地】标签高亮**、城市下钻→分析页、**岗位需求明细（需求量降序、无紧缺指数/状态列）"进入岗位分析"**、数据覆盖与质量卡（导入渠道统计/质量百分比）下载数据质量报告。 依赖：T167、T169、T171、T173。 文件：`frontend/src/components/dashboard/BottomCards.test.ts`。 验证：Vitest 全绿。 完成判据：AC-37/AC-38/AC-40/AC-96/AC-97 跳转与标签断言齐备。 来源：`spec.md#REQ-32/33/35/36, AC-37, AC-38, AC-40, AC-96, AC-97`。
- [ ] T186 [REQ-32] [REQ-33] [REQ-35] [REQ-36] [Phase 4] [Size: S] 实现底部模块组件（通过 T185）。 依赖：T185。 文件：`frontend/src/components/dashboard/BottomCards.vue`。 验证：T185 全绿。 完成判据：跳转携带筛选参数。 来源：同 T185。
- [ ] T187 [AC-14, AC-15, AC-16, AC-17, AC-18, AC-19, AC-20, AC-33, AC-34, AC-35, AC-37, AC-38, AC-40, AC-72, AC-96, AC-97] [Phase 4] [Size: M] 执行 M4 端到端冒烟与 AC 回归记录（Playwright）：筛选联动/周期切换/同比环比与无法计算/紧缺双向/趋势预测年份标注/**技能热度排行查看全部**/**岗位需求明细按需求量排序与同比（无紧缺指数/状态列）**/区域对比 TOP6 与【本地】标签/下钻跳转/品牌指标与看板一致性。 依赖：T157、T159、T161、T163、T165、T167、T169、T171、T173、T175、T178、T180、T182、T184、T186。 文件：`e2e/m4-dashboard.spec.ts`、`docs/verify/M4-AC-regression.md`。 验证：Playwright 全绿并产出 AC 回归记录。 完成判据：M4 范围 16 条 AC 逐条有结论，性能验收（T176）记录随附。 来源：`plan.md#6.1 M4`；`spec.md` 对应 AC。

---

## 5. Phase 5 —— M5 课程服务（curriculum-service）

- [ ] T188 [REQ-16] [AC-21] [Phase 5] [Size: M] 编写课程体系维护测试（Service 表格驱动 + API 契约）：新建课程体系（专业名称必填、学年、版本号必填、备注）生成空目录并产生版本变更记录；按学年+版本筛选切换查看（AC-21）；维护体系信息留痕。 依赖：T004。 文件：`curriculum-service/src/test/java/<pkg>/service/CourseSystemServiceTest.java`、`curriculum-service/src/test/java/<pkg>/controller/CourseSystemControllerTest.java`。 验证：测试全绿。 完成判据：AC-21 用例名直引，必填校验与变更记录断言齐备。 来源：`spec.md#REQ-16, AC-21`。
- [ ] T189 [REQ-16] [AC-21] [Phase 5] [Size: S] 实现课程体系维护（通过 T188）：`/api/v1/course-systems` 系列。 依赖：T188。 文件：`curriculum-service/src/main/java/<pkg>/service/CourseSystemService.java`、`curriculum-service/src/main/java/<pkg>/controller/CourseSystemController.java`。 验证：T188 全绿。 完成判据：接口与 plan §3.2 curriculum-service 表一致。 来源：同 T188。
- [ ] T190 [REQ-17] [AC-22] [Phase 5] [Size: M] 编写课程维护测试：新建/维护课程（课程名称必填、课程编码必填、课程目录、开课学期、学分、学时、课程负责人、课程定位与能力目标、课程大纲按教学单元组织）；直接维护生效无审批（AC-22）；生成课程变更记录；按目录分组展示（专业核心课/专业方向课/实践课程）；按名称/编码搜索。 依赖：T189。 文件：`curriculum-service/src/test/java/<pkg>/service/CourseServiceTest.java`、`curriculum-service/src/test/java/<pkg>/controller/CourseControllerTest.java`。 验证：测试全绿。 完成判据：AC-22 用例名直引，变更记录断言齐备。 来源：`spec.md#REQ-17, AC-22`。
- [ ] T191 [REQ-17] [AC-22] [Phase 5] [Size: S] 实现课程维护（通过 T190）：`/api/v1/course-systems/{id}/courses`、`/api/v1/courses/{cid}`。 依赖：T190。 文件：`curriculum-service/src/main/java/<pkg>/service/CourseService.java`。 验证：T190 全绿。 完成判据：直接生效（本期无审核发布流程）。 来源：同 T190。
- [ ] T192 [REQ-18] [AC-23] [Phase 5] [Size: M] 编写知识技能点维护测试：新建技能点（技能点编码必填、所属模块、名称必填、能力目标按布鲁姆认知层级校准、关联多个技能标签）；保存后自动参与知识图谱对齐（C-12），可被专业调整建议分析引用（AC-23）。 依赖：T191。 文件：`curriculum-service/src/test/java/<pkg>/service/SkillPointServiceTest.java`、`curriculum-service/src/test/java/<pkg>/controller/SkillPointControllerTest.java`。 验证：测试全绿。 完成判据：AC-23 用例名直引，关联标签与知识图谱写入断言齐备。 来源：`spec.md#REQ-18, AC-23`。
- [ ] T193 [REQ-18] [AC-23] [Phase 5] [Size: S] 实现知识技能点维护（通过 T192）：`/api/v1/courses/{cid}/skill-points`、`/api/v1/skill-points/{sid}`。 依赖：T192。 文件：`curriculum-service/src/main/java/<pkg>/service/SkillPointService.java`。 验证：T192 全绿。 完成判据：技能点关联即刻入知识图谱。 来源：同 T192。
- [ ] T194 [REQ-52] [AC-52] [Phase 5] [Size: M] 编写课程体系概览测试：体系信息（专业/学年/版本/覆盖方向/最近更新）、指标卡（课程总数及分类、知识技能点数及本版新增、待更新技能点数及高优先级数）、知识图谱覆盖率=被课程技能点关联的产业技能标签数 ÷ 产业技能标签总数（单向，OPEN-07 口径）与实际一致（AC-52）。 依赖：T193。 文件：`curriculum-service/src/test/java/<pkg>/service/CourseOverviewServiceTest.java`。 验证：表格驱动全绿（覆盖率分子/分母边界）。 完成判据：AC-52 用例名直引，覆盖率口径注释引用 OPEN-07。 来源：`spec.md#REQ-52, AC-52`。
- [ ] T195 [REQ-52] [AC-52] [Phase 5] [Size: S] 实现课程体系概览（通过 T194）：`/api/v1/course-systems/{id}/overview`。 依赖：T194。 文件：`curriculum-service/src/main/java/<pkg>/service/CourseOverviewService.java`。 验证：T194 全绿。 完成判据：分母（产业技能标签总数）经 data-service 同步副本计算，跨库无 Join。 来源：同 T194。
- [ ] T196 [REQ-53] [AC-53] [ASM-03] [Phase 5] [Size: M] 编写标签同步与 tag-changed 消费测试："同步产业技能标签"（REQ-53）将岗位侧技能要求标签同步至课程侧；消费 tag-changed 事件更新课程侧关联引用（ASM-03，合并后映射指向保留标签）；消费幂等（UH-18）。 依赖：T135。 文件：`curriculum-service/src/test/java/<pkg>/service/IndustryTagSyncServiceTest.java`、`curriculum-service/src/test/java/<pkg>/mq/TagChangedConsumerTest.java`。 验证：测试全绿（同步/事件/幂等断言）。 完成判据：AC-53 用例名直引，双端事件契约（T135）一致。 来源：`spec.md#REQ-53, AC-53, ASM-03`；`plan.md#4.4 ④`。
- [ ] T197 [REQ-53] [AC-53] [Phase 5] [Size: S] 实现标签同步与事件消费（通过 T196）：`/api/v1/course-updates` 域同步服务 + TagChangedConsumer。 依赖：T196。 文件：`curriculum-service/src/main/java/<pkg>/service/IndustryTagSyncService.java`、`curriculum-service/src/main/java/<pkg>/mq/TagChangedConsumer.java`。 验证：T196 全绿。 完成判据：同步生效且课程大纲预览/导出不实现（spec §4.2）。 来源：同 T196。
- [ ] T198 [REQ-54] [AC-54] [ASM-16] [Phase 5] [Size: M] 编写技能变化对课程更新提示测试：基于技能热度自动生成更新提醒列表（技能增长率、课程覆盖情况、具体更新建议）；标记待更新技能点含高优先级标记（ASM-16）；不含"生成更新建议/发起更新"动作（AC-54/spec §4.2）。 依赖：T193。 文件：`curriculum-service/src/test/java/<pkg>/service/UpdatePromptServiceTest.java`。 验证：测试全绿。 完成判据：AC-54 用例名直引，高优先级标记断言齐备。 来源：`spec.md#REQ-54, AC-54`。
- [ ] T199 [REQ-54] [AC-54] [Phase 5] [Size: S] 实现更新提示（通过 T198）：`GET /api/v1/course-updates/prompts`。 依赖：T198。 文件：`curriculum-service/src/main/java/<pkg>/service/UpdatePromptService.java`。 验证：T198 全绿。 完成判据：仅展示提示，无生成更新任务入口。 来源：同 T198。
- [ ] T200 [REQ-27] [AC-32] [GAP-07] [Phase 5] [Size: S] 编写课程附件下载测试：教师下载课程资源=课程大纲/教学单元文档等课程附件（本地磁盘文件存储）；项目资源及其下载不实现（GAP-07 已确认边界，2026-11-26）。 依赖：T017。 文件：`curriculum-service/src/test/java/<pkg>/controller/CourseAttachmentControllerTest.java`。 验证：测试全绿（附件可下载、项目资源路径不存在断言）。 完成判据：AC-32 课程资源下载部分断言齐备。 来源：`spec.md#REQ-27, AC-32`；`plan.md#1.2 GAP-07`。
- [ ] T201 [REQ-27] [AC-32] [Phase 5] [Size: S] 实现课程附件下载（通过 T200）：附件上传/下载（本地磁盘文件存储）。 依赖：T200。 文件：`curriculum-service/src/main/java/<pkg>/controller/CourseAttachmentController.java`。 验证：T200 全绿。 完成判据：与 plan §3.2 一致。 来源：同 T200。
- [ ] T202 [REQ-16] [REQ-17] [REQ-18] [REQ-52] [REQ-54] [AC-21] [AC-22] [AC-23] [AC-52] [AC-54] [Phase 5] [Size: M] 编写前端课程维护页测试（/courses）：课程体系切换（学年+版本）、课程目录分组维护、课程表单（必填校验/大纲教学单元）、技能点与技能标签关联、体系概览指标卡与知识图谱覆盖率、更新提示列表（待更新/高优先级标记）。 依赖：T189、T191、T193、T195、T199。 文件：`frontend/src/views/CoursesView.test.ts`。 验证：Vitest 全绿。 完成判据：AC-21/22/23/52/54 交互断言齐备。 来源：`spec.md#REQ-16~18/52/54`；`plan.md#3.1 /courses`。
- [ ] T203 [REQ-16] [AC-21] [Phase 5] [Size: M] 实现前端课程维护页（通过 T202）。 依赖：T202。 文件：`frontend/src/views/CoursesView.vue`、`frontend/src/api/courses.ts`、`frontend/src/stores/course.ts`。 验证：T202 全绿。 完成判据：教师/数据分析员可见（AC-74 角色隔离联动）。 来源：同 T202。
- [ ] T204 [AC-21, AC-22, AC-23, AC-52, AC-53, AC-54] [Phase 5] [Size: M] 执行 M5 端到端冒烟与 AC 回归记录（Playwright）：新建课程体系→维护课程→技能点关联→概览覆盖率核对→同步产业技能标签→更新提示展示。 依赖：T189、T191、T193、T195、T197、T199、T201、T203。 文件：`e2e/m5-courses.spec.ts`、`docs/verify/M5-AC-regression.md`。 验证：Playwright 全绿并产出 AC 回归记录。 完成判据：M5 范围 6 条 AC 逐条有结论。 来源：`plan.md#6.1 M5`；`spec.md` 对应 AC。

---

## 6. Phase 6 —— M6 专业调整建议 + 岗位与技能分析

### 6.1 专业调整建议

- [ ] T205 [REQ-19] [AC-24] [Phase 6] [Size: S] 编写建议生成预估测试：`POST /api/v1/suggestions/pre-check`（OpenFeign→ai-service `/internal/ai/estimate`）返回预计建议条数、耗时与样本量，生成前提示（AC-24）；ai-service 不可用显式 503。 依赖：T079。 文件：`data-service/src/test/java/<pkg>/controller/SuggestionPreCheckTest.java`、`ai-service/src/test/java/<pkg>/controller/InternalAiEstimateTest.java`。 验证：契约测试全绿。 完成判据：AC-24 预估提示断言齐备。 来源：`spec.md#REQ-19, AC-24`；`plan.md#3.2`。
- [ ] T206 [REQ-19] [AC-24] [Phase 6] [Size: S] 实现预估链路（通过 T205）。 依赖：T205。 文件：`data-service/src/main/java/<pkg>/client/AiClient.java`、`ai-service/src/main/java/<pkg>/controller/InternalAiController.java`。 验证：T205 全绿。 完成判据：双端契约一致。 来源：同 T205。
- [ ] T207 [REQ-19] [REQ-20] [AC-24] [AC-25] [UH-12] [Phase 6] [Size: M] 编写建议生成任务链测试：手动触发参数（目标课程体系学年+版本、分析依据周期、建议范围复选：专业方向/课程内容/知识技能点、补充说明）；发 suggestion-task MQ 异步执行、进度查询（完成/失败显式，NFR-04）；所选范围无岗位数据显式提示"无可分析数据"（UH-12）。 依赖：T206。 文件：`data-service/src/test/java/<pkg>/service/SuggestionGenerateServiceTest.java`。 验证：测试全绿（含无数据路径断言）。 完成判据：AC-24/AC-25 链路断言齐备，任务进度显式。 来源：`spec.md#REQ-19/20, AC-24, AC-25`；`plan.md#1.2 UH-12`、`spec.md#NFR-04`。
- [ ] T208 [REQ-19] [AC-24] [Phase 6] [Size: S] 实现建议生成任务发起与进度接口（通过 T207）：`POST /suggestions/generate`、`GET /suggestions/tasks/{taskId}`。 依赖：T207。 文件：`data-service/src/main/java/<pkg>/service/SuggestionGenerateService.java`、`data-service/src/main/java/<pkg>/controller/SuggestionController.java`。 验证：T207 全绿。 完成判据：异步执行+进度展示模式（NFR-04）。 来源：同 T207。
- [ ] T209 [REQ-20] [AC-25] [Phase 6] [Size: M] 编写 AI 建议生成测试（ai-service 消费者）：多因子加权分析按**标准建议提示词**输出 `{recommendedType:专业方向|课程内容|知识技能点, level:高优先级|中优先级|低优先级, title, desc, accordingTo[]}`；适配后每条建议 type/priority/description/依据说明齐备（AC-25）；建议编号 ADJ-*、状态、dataTags、confidence 由系统生成或可选补充；依据可追溯到课程技能标签与岗位范围；消费幂等。 依赖：T207、T110。 文件：`ai-service/src/test/java/<pkg>/mq/SuggestionConsumerTest.java`、`ai-service/src/test/java/<pkg>/service/SuggestionAnalysisServiceTest.java`。 验证：测试全绿。 完成判据：AC-25 用例名直引，ADJ 编号生成规则与适配断言齐备。 来源：`spec.md#REQ-20, AC-25`（v2.24）；`plan.md#3.2 SuggestionPromptVO`。
- [ ] T210 [REQ-20] [AC-25] [Phase 6] [Size: S] 实现 AI 建议生成（通过 T209）：SuggestionConsumer + 多因子加权分析 + 标准输出适配与结果回发。 依赖：T209。 文件：`ai-service/src/main/java/<pkg>/mq/SuggestionConsumer.java`。 验证：T209 全绿。 完成判据：AI 结论仅为建议不改课程数据（INV-05 上游保障）；标准输出字段不丢失。 来源：同 T209。
- [ ] T211 [REQ-21] [AC-26] [INV-05] [UH-15] [Phase 6] [Size: M] 编写采纳建议测试：逐条"采纳建议"仅改变该条状态为已确认（INV-05，采纳前后课程体系/课程/技能点数据快照比对不变）（AC-26）；未采纳的建议仅归档参考不流转；并发采纳同一建议→后到者 409（乐观锁，UH-15）。 依赖：T208。 文件：`data-service/src/test/java/<pkg>/service/SuggestionAdoptServiceTest.java`。 验证：测试全绿（快照比对+409 断言）。 完成判据：AC-26 用例名直引，INV-05 有显式快照比对证据。 来源：`spec.md#REQ-21, AC-26, INV-05`；`plan.md#1.2 UH-15`。
- [ ] T212 [REQ-21] [AC-26] [Phase 6] [Size: S] 实现采纳建议（通过 T211）：`POST /suggestions/{id}/adopt`（乐观锁 409）。 依赖：T211。 文件：`data-service/src/main/java/<pkg>/service/SuggestionAdoptService.java`。 验证：T211 全绿。 完成判据：单库本地事务，无课程数据写路径。 来源：同 T211。
- [ ] T213 [REQ-47] [AC-47] [Phase 6] [Size: M] 编写建议清单测试：统计卡（建议总数、三类分布）、按类型 Tab 筛选（全部/专业方向/课程内容/知识技能点）仅展示对应类型（AC-47）；每条含状态（待确认/已确认）、责任单位、目标年级、影响范围；支持加载全部。 依赖：T208。 文件：`data-service/src/test/java/<pkg>/controller/SuggestionListControllerTest.java`。 验证：测试全绿。 完成判据：AC-47 用例名直引。 来源：`spec.md#REQ-47, AC-47`。
- [ ] T214 [REQ-47] [AC-47] [Phase 6] [Size: S] 实现建议清单接口（通过 T213）：`GET /api/v1/suggestions`。 依赖：T213。 文件：`data-service/src/main/java/<pkg>/controller/SuggestionListController.java`。 验证：T213 全绿。 完成判据：SuggestionItemVO 字段与 plan §3.2 骨架一致。 来源：同 T213。
- [ ] T215 [REQ-48] [AC-48] [Phase 6] [Size: S] [P] 编写查看依据测试：`GET /suggestions/{id}/evidence` 展示依据说明（accordingTo）与数据依据；数据标签/模型置信度如系统补充则展示（AC-48）；"完整分析依据"跳转参数支持（岗位与技能分析）。 依赖：T214。 文件：`data-service/src/test/java/<pkg>/controller/SuggestionEvidenceTest.java`。 验证：测试全绿。 完成判据：AC-48 用例名直引。 来源：`spec.md#REQ-48, AC-48`（v2.24）。
- [ ] T216 [REQ-48] [AC-48] [Phase 6] [Size: S] 实现查看依据接口（通过 T215）。 依赖：T215。 文件：`data-service/src/main/java/<pkg>/controller/SuggestionEvidenceController.java`。 验证：T215 全绿。 完成判据：与 plan §3.2 一致。 来源：同 T215。
- [ ] T217 [REQ-49] [AC-49] [Phase 6] [Size: S] [P] 编写调整窗口与评审进度测试：调整窗口（起止时间与当前评审阶段）、多级评审进度（系初审→课程委员会→院学术委员会→校长办公会各级状态/时间/负责人/待确认数）；进度为展示性信息（C-11）；窗口与评审阶段由系统管理员/数据分析员配置（T-18）。 依赖：T214。 文件：`data-service/src/test/java/<pkg>/controller/AdjustmentWindowTest.java`。 验证：测试全绿。 完成判据：AC-49 用例名直引。 来源：`spec.md#REQ-49, AC-49`。
- [ ] T218 [REQ-49] [AC-49] [Phase 6] [Size: S] 实现调整窗口与评审进度接口（通过 T217）：`GET /suggestions/adjustment-window`。 依赖：T217。 文件：`data-service/src/main/java/<pkg>/controller/AdjustmentWindowController.java`。 验证：T217 全绿。 完成判据：展示性信息无审批动作。 来源：同 T217。
- [ ] T219 [REQ-50] [AC-50] [Phase 6] [Size: S] [P] 编写调整影响评估测试：影响课程数/覆盖学生数/参与教师数/新增资源数与对齐度进度（课程目标与技能点对齐、教学资源完备度）（AC-50）；"查看汇总"跳转参数支持（报告与成果）。 依赖：T214。 文件：`data-service/src/test/java/<pkg>/controller/SuggestionImpactTest.java`。 验证：测试全绿。 完成判据：AC-50 用例名直引。 来源：`spec.md#REQ-50, AC-50`。
- [ ] T220 [REQ-50] [AC-50] [Phase 6] [Size: S] 实现影响评估接口（通过 T219）：`GET /suggestions/impact`。 依赖：T219。 文件：`data-service/src/main/java/<pkg>/controller/SuggestionImpactController.java`。 验证：T219 全绿。 完成判据：与 plan §3.2 一致。 来源：同 T219。

### 6.2 岗位与技能分析（/analysis）

- [ ] T221 [REQ-39] [AC-41] [GAP-01] [GAP-02] [Phase 6] [Size: M] 编写分析页框架 API 测试：筛选（城市/岗位族/周期，应用/重置联动 AC-41）与 **4 张 KPI（v2.12 裁决）**：①岗位需求总量（所有去重后岗位数量 + 同比/环比 + 微型趋势折线，折线粒度=季度按天/年度按月）②技能标签热度（热度均值/升降标签数）③AI 协同岗位结构（AI 协同岗位占全部软件岗位比例 + 固定 2 子类环形图 + 同比提升 PP）④数据覆盖企业（去重企业数/覆盖率）+ 有效样本数口径说明；同比/环比公式沿用 INV-06，无法计算显式标注；"人才供需比"卡取消（GAP-01）；导入不含"完成招聘日期"（GAP-02），平均招聘周期移出 KPI 区。 依赖：T153。 文件：`data-service/src/test/java/<pkg>/controller/AnalysisKpiControllerTest.java`。 验证：测试全绿（缺失数据显式断言）。 完成判据：AC-41 用例名直引，四卡口径与看板一致、不伪造数值。 来源：`spec.md#REQ-39, AC-41`（v2.12）；`plan.md#1.2 GAP-01/GAP-02`（已关闭）。
- [ ] T222 [REQ-39] [AC-41] [Phase 6] [Size: S] 实现分析 KPI 与筛选（通过 T221）：`GET /api/v1/analysis/kpi`。 依赖：T221。 文件：`data-service/src/main/java/<pkg>/controller/AnalysisController.java`。 验证：T221 全绿。 完成判据：分析口径与看板一致（去重、时间归属）。 来源：同 T221。
- [ ] T223 [REQ-41] [AC-42] [Phase 6] [Size: M] 编写岗位需求结构测试：季度有效岗位数（柱）与平均月薪（折线）组合图数据、城市需求分布 TOP6（占比、最高需求/最高增速城市）、岗位需求明细表（岗位名称/需求量/同比/平均月薪/核心技能/紧缺指数及状态标签）（AC-42）；紧缺指数阈值筛选、岗位与技能双向下钻参数。 依赖：T222。 文件：`data-service/src/test/java/<pkg>/controller/JobStructureControllerTest.java`。 验证：测试全绿。 完成判据：AC-42 用例名直引。 来源：`spec.md#REQ-41, AC-42`。
- [ ] T224 [REQ-41] [AC-42] [Phase 6] [Size: S] 实现岗位需求结构接口（通过 T223）：`GET /api/v1/analysis/job-structure`。 依赖：T223。 文件：`data-service/src/main/java/<pkg>/controller/JobStructureController.java`。 验证：T223 全绿。 完成判据：与 plan §3.2 一致。 来源：同 T223。
- [ ] T225 [REQ-42] [AC-43] [Phase 6] [Size: M] 编写技能热度趋势测试：核心技能热度变化折线（技能热度指数 0-100，C-09 口径）、热度均值与上升/下降技能数统计、技能热度明细与课程覆盖表（热度指数/同比变化/关联岗位数/主要岗位/课程覆盖率/更新提示）（AC-43）；课程覆盖率经 OpenFeign 课程侧（降级显式 UH-17）。 依赖：T222。 文件：`data-service/src/test/java/<pkg>/stats/SkillTrendServiceTest.java`。 验证：测试全绿。 完成判据：AC-43 用例名直引。 来源：`spec.md#REQ-42, AC-43`。
- [ ] T226 [REQ-42] [AC-43] [Phase 6] [Size: S] 实现技能热度趋势接口（通过 T225）：`GET /api/v1/analysis/skill-trend`。 依赖：T225。 文件：`data-service/src/main/java/<pkg>/stats/SkillTrendService.java`。 验证：T225 全绿。 完成判据：与看板技能热度同口径（ASM-10）。 来源：同 T225。
- [ ] T227 [REQ-43] [AC-44] [Phase 6] [Size: M] 编写技能-岗位关联测试：技能-岗位关联矩阵（0-1 系数，关联强度与岗位覆盖度综合）、关联洞察（最强技能组合/课程联动优化/弱关联提示）、高关联能力组合清单（能力组合/主要岗位/关联度/岗位覆盖/能力层级：核心能力/进阶能力/专业能力/建设建议）（AC-44）；无数据维度显式"无数据"。 依赖：T222。 文件：`data-service/src/test/java/<pkg>/stats/SkillJobMatrixServiceTest.java`。 验证：表格驱动全绿。 完成判据：AC-44 用例名直引。 来源：`spec.md#REQ-43, AC-44`。
- [ ] T228 [REQ-43] [AC-44] [Phase 6] [Size: S] 实现关联矩阵与洞察（通过 T227）：`GET /api/v1/analysis/skill-job-matrix`。 依赖：T227。 文件：`data-service/src/main/java/<pkg>/stats/SkillJobMatrixService.java`。 验证：T227 全绿。 完成判据：与 plan §3.2 一致。 来源：同 T227。
- [ ] T229 [REQ-44] [AC-45] [Phase 6] [Size: S] [P] 编写关联洞察转建议测试：`POST /analysis/insights/{id}/to-suggestion` 生成一条知识技能点类专业调整建议并进入建议清单与评审流（AC-45，T-16）。 依赖：T214、T228。 文件：`data-service/src/test/java/<pkg>/controller/InsightToSuggestionTest.java`。 验证：测试全绿。 完成判据：AC-45 用例名直引，生成条目类型=知识技能点。 来源：`spec.md#REQ-44, AC-45`。
- [ ] T230 [REQ-44] [AC-45] [Phase 6] [Size: S] 实现洞察转建议（通过 T229）。 依赖：T229。 文件：`data-service/src/main/java/<pkg>/controller/InsightController.java`。 验证：T229 全绿。 完成判据：与建议清单共用 SuggestionItemVO。 来源：同 T229。
- [ ] T231 [REQ-45] [REQ-46] [AC-46] [Phase 6] [Size: M] 编写分析导出与分析引擎信息测试：导出分析集/岗位明细/技能清单（`POST /analysis/exports`）对应文件可获得（AC-46）；`GET /analysis/engine-info` 展示模型版本、本次分析样本数、运行状态（T-17）。 依赖：T222。 文件：`data-service/src/test/java/<pkg>/controller/AnalysisExportControllerTest.java`。 验证：测试全绿。 完成判据：AC-46 用例名直引。 来源：`spec.md#REQ-45, REQ-46, AC-46`。
- [ ] T232 [REQ-45] [REQ-46] [AC-46] [Phase 6] [Size: S] 实现分析导出与引擎信息（通过 T231）：EasyExcel 导出 + 引擎信息接口。 依赖：T231。 文件：`data-service/src/main/java/<pkg>/controller/AnalysisExportController.java`。 验证：T231 全绿。 完成判据：导出文件字段与分析视图一致。 来源：同 T231。

### 6.2.1 人才需求预测（REQ-72，AD-18，v2.22 裁决 2026-11-26）

- [ ] T304 [REQ-72] [AC-98] [ARCH-AD-18] [Phase 6] [Size: M] 编写统计时序预测模型测试（ai-service）：移动平均/指数平滑/线性回归三种方法（可组合）在固定样本上的预测值断言；输出下一季度与下一年预测值、趋势方向（升/降/平）、增长提示；回测 MAPE≤20%（超阈值显式标注"未达标"，不隐藏）；样本不足显式"数据不足"不伪造；结果含模型版本与生成时间；`/internal/ai/predict-demand` 契约（入参 series/horizon/filters，返回 PredictionVO）。 依赖：T001。 文件：`ai-service/src/test/java/<pkg>/prediction/PredictionModelServiceTest.java`、`ai-service/src/test/java/<pkg>/controller/InternalAiPredictionTest.java`。 验证：表格驱动全绿（方法×样本组合、MAPE 边界、样本不足断言）。 完成判据：AC-98 用例名直引，纯计算无外部模型依赖。 来源：`spec.md#REQ-72, AC-98`（v2.22）；`plan.md#1.4 AD-18`。
- [ ] T305 [REQ-72] [AC-98] [ARCH-AD-18] [Phase 6] [Size: M] 实现统计时序预测模型（通过 T304）：`PredictionModelService`（移动平均/指数平滑/线性回归 + 组合策略 + 回测 MAPE）+ `/internal/ai/predict-demand` 内部接口。 依赖：T304。 文件：`ai-service/src/main/java/<pkg>/prediction/PredictionModelService.java`、`ai-service/src/main/java/<pkg>/controller/InternalAiController.java`。 验证：T304 全绿。 完成判据：输出 PredictionVO（预测值/趋势方向/增长提示/MAPE/模型版本/生成时间）；数据不足显式失败（INV-07）。 来源：同 T304；`plan.md#3.2 ai-service 内部接口`。
- [ ] T306 [REQ-72] [AC-98] [Phase 6] [Size: S] 编写需求预测 API 测试（data-service）：`GET /api/v1/analysis/prediction`（horizon=QUARTER|YEAR，区域/行业/岗位族筛选）；历史月度岗位需求取有效职位口径；经 OpenFeign 调 ai-service 内部接口；结果与模型版本/生成时间落 data_db `prediction_result` 并可查询；ai-service 不可用 503 显式、数据不足显式"数据不足"。 依赖：T305、T153。 文件：`data-service/src/test/java/<pkg>/prediction/PredictionServiceTest.java`、`data-service/src/test/java/<pkg>/controller/PredictionControllerTest.java`。 验证：测试全绿（Mock Feign + 落库/查询断言）。 完成判据：AC-98 用例名直引，不伪造预测数值。 来源：`spec.md#REQ-72, AC-98`（v2.22）；`plan.md#3.2 /analysis/prediction`。
- [ ] T307 [REQ-72] [AC-98] [Phase 6] [Size: M] 实现需求预测 API 与结果存储（通过 T306）：`PredictionService` + `PredictionResult` 实体/Mapper（data_db）+ `GET /api/v1/analysis/prediction`，调用 ai-service 并将预测值/趋势方向/增长提示/MAPE/模型版本/生成时间落库。 依赖：T306。 文件：`data-service/src/main/java/<pkg>/prediction/PredictionService.java`、`data-service/src/main/java/<pkg>/model/PredictionResult.java`、`data-service/src/main/java/<pkg>/controller/PredictionController.java`。 验证：T306 全绿。 完成判据：预测结果可追溯（模型版本+生成时间）；跨库无 Join。 来源：同 T306；`db-design.md`（prediction_result）。
- [ ] T308 [REQ-72] [AC-98] [Phase 6] [Size: S] 编写前端需求预测 Tab 测试：分析页第 4 Tab（下一季度/下一年预测值、趋势方向、增长提示、回测 MAPE、模型版本与生成时间）、筛选（区域/行业/岗位族）联动、数据不足与 ai 不可用显式提示。 依赖：T307、T238。 文件：`frontend/src/components/analysis/PredictionPanel.test.ts`。 验证：Vitest 全绿。 完成判据：AC-98 交互断言齐备，不展示伪造数值。 来源：`spec.md#REQ-72, AC-98`；`plan.md#3.1 /analysis`。
- [ ] T309 [REQ-72] [AC-98] [Phase 6] [Size: M] 实现前端需求预测 Tab（通过 T308）：`frontend/src/components/analysis/PredictionPanel.vue`、`frontend/src/api/prediction.ts`，并挂入分析页第 4 Tab。 依赖：T308、T240。 文件：`frontend/src/components/analysis/PredictionPanel.vue`、`frontend/src/api/prediction.ts`。 验证：T308 全绿。 完成判据：预测 Tab 与既有筛选态共享，切换不丢失条件。 来源：同 T308。

### 6.3 前端建议与分析页

- [ ] T233 [REQ-19] [REQ-21] [AC-24] [AC-26] [ARCH-3.3] [Phase 6] [Size: S] [P] 编写前端 suggestions api/store 测试：生成发起/进度轮询（完成/失败显式）、采纳状态流转与 409 冲突提示、错误映射。 依赖：T020、T208、T212。 文件：`frontend/src/api/suggestions.test.ts`、`frontend/src/stores/suggestion.test.ts`。 验证：Vitest 全绿。 完成判据：AC-24/AC-26 前端行为断言齐备。 来源：`spec.md#REQ-19/21, AC-24, AC-26`；`plan.md#3.3`。
- [ ] T234 [REQ-19] [Phase 6] [Size: S] 实现前端 suggestions api/store（通过 T233）。 依赖：T233。 文件：`frontend/src/api/suggestions.ts`、`frontend/src/stores/suggestion.ts`。 验证：T233 全绿。 完成判据：异步任务进度可见（NFR-04）。 来源：同 T233。
- [ ] T235 [REQ-19] [REQ-47] [REQ-48] [REQ-49] [REQ-50] [AC-24] [AC-26] [AC-47] [AC-48] [AC-49] [AC-50] [Phase 6] [Size: M] 编写建议页测试（/suggestions）：生成向导（目标体系/依据周期/范围复选/补充说明/生成前预估提示）、清单统计卡与类型 Tab、查看依据面板（依据说明/数据依据；置信度如补充则展示/跳转）、逐条采纳交互（仅改状态）、调整窗口与评审进度展示、调整影响评估与"查看汇总"跳转。 依赖：T234、T214、T216、T218、T220。 文件：`frontend/src/views/SuggestionsView.test.ts`。 验证：Vitest 全绿。 完成判据：AC-24/26/47/48/49/50 交互断言齐备。 来源：`spec.md#REQ-19/47~50`；`plan.md#3.1 /suggestions`。
- [ ] T236 [REQ-47] [AC-47] [Phase 6] [Size: M] 实现建议页（通过 T235）。 依赖：T235。 文件：`frontend/src/views/SuggestionsView.vue`。 验证：T235 全绿。 完成判据：专业负责人可采纳、其他角色只读。 来源：同 T235。
- [ ] T237 [REQ-39] [AC-41] [ARCH-3.3] [Phase 6] [Size: S] [P] 编写前端 analysis api/store 测试：筛选（城市/岗位族/周期）应用/重置、KPI 与 Tab 数据联动、"无数据/数据缺失/不可计算"显式态。 依赖：T020、T222。 文件：`frontend/src/api/analysis.test.ts`、`frontend/src/stores/analysis.test.ts`。 验证：Vitest 全绿。 完成判据：AC-41 前端断言齐备。 来源：`spec.md#REQ-39, AC-41`。
- [ ] T238 [REQ-39] [AC-41] [Phase 6] [Size: S] 实现前端 analysis api/store（通过 T237）。 依赖：T237。 文件：`frontend/src/api/analysis.ts`、`frontend/src/stores/analysis.ts`。 验证：T237 全绿。 完成判据：跳转带参（看板下钻入口）解析正确。 来源：同 T237。
- [ ] T239 [REQ-41] [REQ-42] [REQ-43] [REQ-44] [REQ-45] [REQ-72] [AC-40] [AC-42] [AC-43] [AC-44] [AC-45] [AC-46] [Phase 6] [Size: M] 编写分析页测试（/analysis）：**4 张 KPI 卡（岗位需求总量[去重岗位数+同比/环比+微型折线，季度按天/年度按月]/技能标签热度/AI 协同岗位结构/数据覆盖企业，含有效样本数口径）**、**四个分析 Tab（岗位需求结构/技能热度趋势/技能-岗位关联/需求预测；需求预测 Tab 细节由 T308 覆盖，本任务断言 Tab 框架与切换）**、紧缺指数阈值筛选与双向下钻、"将关联洞察转为课程建议"、导出三类文件、分析引擎信息展示、看板三入口跳转落位（AC-40）。 依赖：T238、T224、T226、T228、T230、T232。 文件：`frontend/src/views/AnalysisView.test.ts`。 验证：Vitest 全绿。 完成判据：AC-40/42~46 交互断言齐备，页面标注"分析结果仅供专业建设参考"。 来源：`spec.md#REQ-39/41~46/72, AC-40~46`；`plan.md#3.1 /analysis`（4 Tab，v2.22）。
- [ ] T240 [REQ-41] [REQ-72] [AC-42] [Phase 6] [Size: M] 实现分析页（通过 T239）。 依赖：T239。 文件：`frontend/src/views/AnalysisView.vue`、`frontend/src/components/analysis/*.vue`。 验证：T239 全绿。 完成判据：四个 Tab（含需求预测入口）与 KPI 联动刷新、重置恢复默认。 来源：同 T239。
- [ ] T241 [AC-24, AC-25, AC-26, AC-40, AC-41, AC-42, AC-43, AC-44, AC-45, AC-46, AC-47, AC-48, AC-49, AC-50, AC-98] [Phase 6] [Size: M] 执行 M6 端到端冒烟与 AC 回归记录（Playwright）：生成建议（预估→异步→清单）→查看依据→采纳仅改状态→洞察转建议→分析页四 Tab 与下钻→**需求预测 Tab（预测值/趋势方向/增长提示/MAPE/模型版本/生成时间）**→导出与引擎信息→窗口与影响评估。 依赖：T206、T208、T210、T212、T214、T216、T218、T220、T222、T224、T226、T228、T230、T232、T234、T236、T238、T240、T307、T309。 文件：`e2e/m6-suggestion-analysis.spec.ts`、`docs/verify/M6-AC-regression.md`。 验证：Playwright 全绿并产出 AC 回归记录。 完成判据：M6 范围 15 条 AC 逐条有结论，INV-05 快照比对与 UH-12/15 异常路径有证据，预测数据不足/ai 不可用路径有证据。 来源：`plan.md#6.1 M6`；`spec.md` 对应 AC。

---

## 7. Phase 7 —— M7 报告成果 + 模型配置 + 收尾

### 7.1 报告与成果（data-service）

- [ ] T242 [REQ-26] [AC-32] [GAP-06] [Phase 7] [Size: M] 编写报告编制测试：数据分析员编制分析报告（年度汇总报告/季度岗位与技能热度分析报告/专业动态调整建议报告/课程体系与技能图谱白皮书等），含编号 RPT-\*、版本（汇报版 V1.0/脱敏公开版）、简介、覆盖周期、页数、生成时间（AC-32 报告编制部分）；形态=系统模板生成（基于统计数据）+ 上传附件归档（GAP-06 已确认，2026-11-26）；报告导出见 T266~T273（PDF/Excel 已实现，Word 不实现）。 依赖：T153。 文件：`data-service/src/test/java/<pkg>/service/ReportServiceTest.java`、`data-service/src/test/java/<pkg>/controller/ReportControllerTest.java`。 验证：测试全绿。 完成判据：AC-32 断言齐备，RPT 编号生成规则断言。 来源：`spec.md#REQ-26, AC-32`；`plan.md#1.2 GAP-06`。
- [ ] T243 [REQ-26] [AC-32] [Phase 7] [Size: S] 实现报告编制（通过 T242）：`GET/POST /api/v1/reports`。 依赖：T242。 文件：`data-service/src/main/java/<pkg>/service/ReportService.java`、`data-service/src/main/java/<pkg>/controller/ReportController.java`。 验证：T242 全绿。 完成判据：报告文件存本地磁盘文件存储。 来源：同 T242。
- [ ] T244 [REQ-58] [AC-60] [Phase 7] [Size: M] 编写报告库与在线阅读测试：报告列表（名称/格式/大小/状态：已发布/评审中/已存档）、在线阅读（封面/目录/摘要/指标卡）、报告存档中心（AC-60）；PDF/Excel 导出见 T266~T273。 依赖：T243。 文件：`data-service/src/test/java/<pkg>/controller/ReportLibraryControllerTest.java`。 验证：测试全绿。 完成判据：AC-60 用例名直引。 来源：`spec.md#REQ-58, AC-60`。
- [ ] T245 [REQ-58] [AC-60] [Phase 7] [Size: S] 实现报告库与在线阅读（通过 T244）：`GET /api/v1/reports/{id}`、`PUT /api/v1/reports/{id}/status`。 依赖：T244。 文件：`data-service/src/main/java/<pkg>/controller/ReportLibraryController.java`。 验证：T244 全绿。 完成判据：状态流转合法校验。 来源：同 T244。
- [ ] T246 [REQ-59] [AC-61] [Phase 7] [Size: M] 编写年度统计卡与关键结论测试：调整建议执行率（已确认建议数/建议总数）、课程更新完成率（完成更新课程数/更新目标课程数）、实践项目成果（含新增数与资源完备度）与实际一致；年度关键结论列表含支撑指标与"查看分析依据"跳转（AC-61）。 依赖：T214、T195。 文件：`data-service/src/test/java/<pkg>/service/AnnualStatsServiceTest.java`。 验证：表格驱动全绿（执行率分子分母边界）。 完成判据：AC-61 用例名直引。 来源：`spec.md#REQ-59, AC-61`。
- [ ] T247 [REQ-59] [AC-61] [Phase 7] [Size: S] 实现年度统计与关键结论（通过 T246）：`GET /api/v1/reports/annual-stats`。 依赖：T246。 文件：`data-service/src/main/java/<pkg>/service/AnnualStatsService.java`。 验证：T246 全绿。 完成判据：与 plan §3.2 一致。 来源：同 T246。
- [ ] T248 [REQ-60] [AC-62] [Phase 7] [Size: S] [P] 编写近三年建设成效测试：分组柱状图（更新课程数/新增实践项目数/技能点覆盖率）与累计汇总与实际一致（AC-62）。 依赖：T246。 文件：`data-service/src/test/java/<pkg>/controller/ThreeYearControllerTest.java`。 验证：测试全绿。 完成判据：AC-62 用例名直引。 来源：`spec.md#REQ-60, AC-62`。
- [ ] T249 [REQ-60] [AC-62] [Phase 7] [Size: S] 实现近三年成效接口（通过 T248）：`GET /api/v1/reports/three-year`。 依赖：T248。 文件：`data-service/src/main/java/<pkg>/controller/ThreeYearController.java`。 验证：T248 全绿。 完成判据：与 plan §3.2 一致。 来源：同 T248。
- [ ] T250 [REQ-61] [AC-63] [Phase 7] [Size: M] 编写评审与发布进度测试：四级流程（院内汇总与初审→校领导审阅→外部专家评审→成果发布与归档）状态/时间/评审人可见（AC-63）；外部专家人数、意见截止、评审人查看情况；外部专家只读查看报告并录入评审意见（T-21），无其他修改入口。 依赖：T245。 文件：`data-service/src/test/java/<pkg>/service/ReportReviewServiceTest.java`。 验证：测试全绿（只读+意见录入断言）。 完成判据：AC-63 用例名直引，流程进度为展示性信息。 来源：`spec.md#REQ-61, AC-63`。
- [ ] T251 [REQ-61] [AC-63] [Phase 7] [Size: S] 实现评审进度与专家意见（通过 T250）：`GET /reports/{id}/review-progress`、`GET/POST /reports/{id}/review-opinions`。 依赖：T250。 文件：`data-service/src/main/java/<pkg>/service/ReportReviewService.java`。 验证：T250 全绿。 完成判据：与 plan §3.2 一致。 来源：同 T250。
- [ ] T252 [REQ-62] [AC-64] [Phase 7] [Size: M] 编写年度建设成果台账测试：五类成果（专业方向/课程内容/知识技能点/实践项目/教学资源）的成果名称、本年完成、累计成果、责任单位、完成进度、状态，系统自动汇总与实际一致（AC-64）；导出台账不实现。 依赖：T246。 文件：`data-service/src/test/java/<pkg>/service/AchievementServiceTest.java`。 验证：表格驱动全绿。 完成判据：AC-64 用例名直引。 来源：`spec.md#REQ-62, AC-64`。
- [ ] T253 [REQ-62] [AC-64] [Phase 7] [Size: S] 实现成果台账（通过 T252）：`GET /api/v1/achievements`。 依赖：T252。 文件：`data-service/src/main/java/<pkg>/service/AchievementService.java`。 验证：T252 全绿。 完成判据：与 plan §3.2 一致。 来源：同 T252。
- [ ] T254 [REQ-26] [REQ-58] [REQ-59] [REQ-61] [REQ-62] [AC-32] [AC-60] [AC-61] [AC-63] [AC-64] [Phase 7] [Size: M] 编写前端报告与成果页测试（/reports）：报告库列表与状态、在线阅读（封面/目录/摘要/指标卡）、编制入口（模板生成+上传归档）、年度统计卡与关键结论跳转、近三年成效图、评审与发布进度（含专家意见录入）、成果台账。 依赖：T243、T245、T247、T249、T251、T253。 文件：`frontend/src/views/ReportsView.test.ts`。 验证：Vitest 全绿。 完成判据：AC-32/60/61/63/64 交互断言齐备，审计员/只读角色无修改入口。 来源：`spec.md#REQ-26/58~62`；`plan.md#3.1 /reports`。
- [ ] T255 [REQ-58] [AC-60] [Phase 7] [Size: M] 实现前端报告与成果页（通过 T254）。 依赖：T254。 文件：`frontend/src/views/ReportsView.vue`、`frontend/src/api/reports.ts`、`frontend/src/stores/report.ts`。 验证：T254 全绿。 完成判据：数据分析员可编制、审计员只读。 来源：同 T254。

### 7.1.1 报告导出（REQ-73，v3.0 裁决 2026-11-26）

- [ ] T266 [REQ-73] [AC-89] [Phase 7] [Size: S] [P] 编写年度/分析报告导出测试：报告库指定报告导出 PDF 与 Excel 两种格式、文件可打开、内容与报告正文/指标一致、Word 格式请求显式拒绝、只读角色可导出。 依赖：T243。 文件：`data-service/src/test/java/<pkg>/service/ReportExportServiceTest.java`、`data-service/src/test/java/<pkg>/controller/ReportExportControllerTest.java`。 验证：测试全绿（PDF/Excel 各格式断言）。 完成判据：AC-89 用例名直引。 来源：`spec.md#REQ-73, AC-89`；`plan.md#1.4 AD-12`、`plan.md#3.2 /reports/{id}/export`。
- [ ] T267 [REQ-73] [AC-89] [Phase 7] [Size: M] 实现年度/分析报告导出（通过 T266）：`GET /api/v1/reports/{id}/export`（format=pdf|excel），PDF 用 OpenPDF 内嵌中文字体、Excel 用 EasyExcel，服务端生成文件流。 依赖：T266。 文件：`data-service/src/main/java/<pkg>/service/ReportExportService.java`、`data-service/src/main/java/<pkg>/controller/ReportExportController.java`。 验证：T266 全绿。 完成判据：生成失败显式报错（INV-07），导出记审计。 来源：同 T266。
- [ ] T268 [REQ-73] [AC-90] [Phase 7] [Size: S] [P] 编写看板统计报告导出测试：按当前筛选条件（城市/岗位族/周期）导出 PDF/Excel，数值与看板口径一致（INV-06 去重与时间归属），无数据时显式提示不生成空文件。 依赖：T222。 文件：`data-service/src/test/java/<pkg>/service/DashboardExportServiceTest.java`。 验证：测试全绿（口径一致断言）。 完成判据：AC-90 用例名直引。 来源：`spec.md#REQ-73, AC-90`；`plan.md#3.2 /dashboard/export`。
- [ ] T269 [REQ-73] [AC-90] [Phase 7] [Size: M] 实现看板统计报告导出（通过 T268）：`POST /api/v1/dashboard/export`，复用 StatsQueryPolicy 统计口径，输出 PDF/Excel 文件流。 依赖：T268。 文件：`data-service/src/main/java/<pkg>/service/DashboardExportService.java`、`data-service/src/main/java/<pkg>/controller/DashboardExportController.java`。 验证：T268 全绿。 完成判据：导出口径与看板同源（不重复实现统计 SQL）。 来源：同 T268。
- [ ] T270 [REQ-73] [AC-91] [Phase 7] [Size: S] [P] 编写专业调整建议报告导出测试：建议清单与依据导出 PDF/Excel、内容与页面一致、只读角色可导出但无修改入口（403 断言）、无建议时显式提示。 依赖：T251。 文件：`data-service/src/test/java/<pkg>/service/SuggestionExportServiceTest.java`。 验证：测试全绿。 完成判据：AC-91 用例名直引。 来源：`spec.md#REQ-73, AC-91`；`plan.md#3.2 /suggestions/export`。
- [ ] T271 [REQ-73] [AC-91] [Phase 7] [Size: M] 实现专业调整建议报告导出（通过 T270）：`POST /api/v1/suggestions/export`，导出建议清单（类型/状态/责任单位/影响范围）与依据。 依赖：T270。 文件：`data-service/src/main/java/<pkg>/service/SuggestionExportService.java`、`data-service/src/main/java/<pkg>/controller/SuggestionExportController.java`。 验证：T270 全绿。 完成判据：与 page 一致的字段齐备。 来源：同 T270。
- [ ] T272 [REQ-73] [AC-89] [AC-90] [AC-91] [Phase 7] [Size: S] 编写前端导出入口测试：报告库/看板/建议页的导出按钮（PDF/Excel 二选）、下载触发、失败显式提示、只读角色按钮不隐藏但无修改入口。 依赖：T255。 文件：`frontend/src/components/ExportActions.test.ts`。 验证：Vitest 全绿。 完成判据：AC-89/90/91 交互断言齐备。 来源：`spec.md#REQ-73`。
- [ ] T273 [REQ-73] [AC-89] [AC-90] [AC-91] [Phase 7] [Size: S] 实现前端导出入口（通过 T272）：`frontend/src/api/exports.ts` 与报告库/看板/建议页导出按钮，文件流下载（blob）。 依赖：T272、T267、T269、T271。 文件：`frontend/src/api/exports.ts`、`frontend/src/components/ExportActions.vue`。 验证：T272 全绿。 完成判据：三类导出在对应页面可用。 来源：同 T272。

### 7.2 模型配置完整版与收尾

- [ ] T256 [REQ-67] [REQ-68] [REQ-69] [AC-69] [AC-70] [AC-71] [Phase 7] [Size: M] 编写前端模型配置完整版测试（/admin 模型配置功能区）：模型列表/新增（Key 密文不可回显）/启停/灰度流量验证、路由策略（主备/触发条件/启停）、配额与预警（阈值/通知方式）、调用量统计（AC-69/70/71）。 依赖：T110、T112、T114。 文件：`frontend/src/views/admin/ModelConfigPanel.test.ts`。 验证：Vitest 全绿。 完成判据：AC-69/70/71 交互断言齐备，Key 保存后界面不可再查看。 来源：`spec.md#REQ-67~69, AC-69~71`；`plan.md#3.1 /admin`。
- [ ] T257 [REQ-67] [AC-69] [Phase 7] [Size: S] 实现前端模型配置完整版（通过 T256）。 依赖：T256。 文件：`frontend/src/views/admin/ModelConfigPanel.vue`、`frontend/src/api/models.ts`。 验证：T256 全绿。 完成判据：系统管理员专属功能区。 来源：同 T256。
- [ ] T258 [REQ-69] [AC-71] [INV-10] [Phase 7] [Size: S] [P] 编写配额月度统计与预警 Job 测试：月度 token 用量统计、达 80% 按通知方式提醒管理员、达 100% 暂停非核心任务调用并显式提示（AC-71）。 依赖：T114、T015。 文件：`ai-service/src/test/java/<pkg>/job/QuotaMonthlyJobTest.java`。 验证：测试全绿（80%/100% 边界断言）。 完成判据：AC-71 用例名直引。 来源：`spec.md#REQ-69, AC-71, INV-10`；`plan.md#4.4 定时任务 ④`。
- [ ] T259 [REQ-69] [AC-71] [Phase 7] [Size: S] 实现配额月度统计与预警 Job（通过 T258）。 依赖：T258。 文件：`ai-service/src/main/java/<pkg>/job/QuotaMonthlyJob.java`。 验证：T258 全绿。 完成判据：与 QuotaGuard（T114）联动。 来源：同 T258。
- [ ] T260 [ARCH-AD-08] [Phase 7] [Size: S] [P] 编写最终一致对账 Job 测试：任务表状态与 MQ 消息对账（打标/建议任务）、发现悬挂任务显式告警、死信扫描重触发。 依赖：T015。 文件：`data-service/src/test/java/<pkg>/job/ReconcileJobTest.java`。 验证：测试全绿。（架构守护类，无直接 AC） 来源：`plan.md#1.4 AD-08`、`plan.md#4.1 MQ 可靠性`。
- [ ] T261 [ARCH-AD-08] [Phase 7] [Size: S] 实现对账 Job（通过 T260）。 依赖：T260。 文件：`data-service/src/main/java/<pkg>/job/ReconcileJob.java`。 验证：T260 全绿。 完成判据：补偿机制记录可审计。 来源：同 T260。
- [ ] T262 [ARCH-AD-02] [ARCH-AD-13] [INV-07] [Phase 7] [Size: M] 完成可观测性收尾验证：traceId 全链路日志（Logback JSON 携带 traceId/taskCode/userId）、Actuator `/health` 注册 Nacos、网关聚合下游健康、**Kibana 日志可按 traceId/taskCode 检索、Grafana 面板（服务健康/接口时延/MQ 堆积/DB 连接/调度失败）可用**、MQ 队列堆积/死信数/调度任务失败告警项核对（输出检查单与人工核对记录）。 依赖：T006、T014、T015、T277、T279。 文件：`docs/verify/observability-checklist.md`、各服务 `logback-spring.xml`。 验证：检查单逐项核对通过，AI 调用与 MQ 消息可按 traceId/taskCode 关联。 完成判据：INV-07 诊断信息可从日志追溯到具体任务。（基础设施守护类，无直接 AC） 来源：`plan.md#4.3 可观测性`、`plan.md#1.4 AD-13`。
- [ ] T263 [ARCH-3.3] [Phase 7] [Size: S] [P] 执行契约同步检查：Knife4j 接口文档与前端 `frontend/src/types/` 差异校验（openapi-typescript 生成，v2.7 裁决 2026-11-26），发现差异即"前后端同改"修正并记录。 依赖：T214、T222、T232、T253。 文件：`scripts/check-contract.*`、`docs/verify/contract-sync.md`。 验证：差异清单为空或全部修正。 完成判据：前端 TS 类型与后端 VO 完全一致（plan §3.3）。 来源：`plan.md#3.3 类型契约/契约同步`。
- [ ] T264 [Phase 7] [Size: M] 执行全量 AC 回归：spec §7 全部 91 条 AC 按附录 A 核对表逐条复验（自动化回归 + 人工核对项），产出最终回归记录。 依赖：T059、T108、T149、T187、T204、T241、T255、T257、T273、T287、T299。 文件：`docs/verify/full-AC-regression.md`。 验证：回归记录中每条 AC 有结论与证据链接。 完成判据：91/91 有结论；失败项有缺陷单与熔断记录，不带病发布。 来源：`plan.md#5.3 验收场景映射`、`plan.md#5.4 追踪闭环`。
- [ ] T265 [NFR-06] [Phase 7] [Size: S] 执行发布前检查：Chrome/Edge 最新两个大版本兼容核对（NFR-06）、Nacos 配置核对（3GB/500MB/30min/锁/权重/**参数配置项已发布生效**）、API Key 密文与环境变量密钥核对、审计只读核对、4 库连通与容量核对、**每日全量备份配置（保留 7 天）与恢复演练记录核对、管理面/业务面端口暴露核对（对外仅 Nginx 80/443）**，输出发布检查单。 依赖：T264。 文件：`docs/verify/release-checklist.md`。 验证：检查单全部勾选（含恢复演练记录）。 完成判据：NFR-06~11 逐项有证据；GAP/待确认项处置结论记录在案；部署与网络约束（plan §4.2.1）核对通过。 来源：`spec.md#NFR-06~11`；`plan.md#4.2`、`plan.md#4.2.1`。

---

## 8. 待确认任务裁决记录（已全部关闭，2026-11-26）

> 以下条目已经用户交互裁决全部关闭（最近同步至 spec.md v2.22 / plan.md v3.19）；任务上的 `[GAP-*]` 标记保留作追溯。

| 项 | 内容 | 裁决结果（2026-11-26 用户裁决） | 影响任务 |
| --- | --- | --- | --- |
| GAP-01 | 人才供需比"供给"数据来源 | "人才供需比"KPI 卡取消（T-15 口径废止），改岗位需求量同比/环比增速（公式沿用 INV-06）、岗位明细按岗位数量排序，趋势折线粒度=季度按天/年度按月 | T221 |
| GAP-02 | "完成招聘日期"可选导入字段 | 不含该字段，"平均招聘周期"不可计算时显式标注 | T221、T072 |
| GAP-03 | 目标企业全量基准数据来源 | Excel 导入维护 | T165 |
| GAP-04 | 数据质量评分权重 | 评分=软件相关岗位数 ÷ 正式表全部记录数×100%（1 位小数百分比，单指标，无权重；分母不含校验失败/废弃，含重复标记与未打标）；数据来源仅导入（v2.22） | T172、T173 |
| GAP-05 | 18 项权限点枚举清单 | 已确认；v2.10 调整为 17 项（裁剪课程更新审批/建议提交，新增关键操作复核）；**任务调度、备份恢复保留为占位权限点、本期无功能入口（v2.22）** | T030、T031、T051 |
| GAP-06 | 报告"编制"形态 | 确认"模板生成+上传归档" | T242 |
| GAP-07 | REQ-27 课程资源下载边界 | 确认仅课程资源，不含项目资源 | T200 |
| GAP-08 | 报告/建议归 data-service | 确认归 data-service | 全局 |
| OPEN-03 | 可用性/容错、数据保留 NFR | **已冻结关闭（v2.22）**：可用性/容错按 INV-07 显式失败与重试、不承诺 SLA；审计/登录审计/AI 调用诊断 90 天清理，其余数据不自动清理 | 运维方案、T265 |
| 技术选型 | 对象存储、链路追踪、RocketMQ 版本 | 文件=本地磁盘文件存储（AD-07 v2.7）；链路追踪暂不上（仅 traceId）；RocketMQ 本机容器 5.x | T016/T017、T262 |
| 判重归一规则 | 公司+岗位名称匹配前归一细则 | trim+空白折叠+小写 | T080 |
| 契约校验方式 | 自动生成 vs 评审检查单 | openapi-typescript | T263 |
| CF-05 | 账号登录凭证与密码字段 | 已同步 spec v2.5 REQ-02：登录=account_name+密码，密码创建入口=系统设置→新建系统账户，工号/统一认证账号仅展示 | T027、T049 |
| 文档小修 | spec 头部过时描述；plan"验收标准条数"口径 | 已修订（2026-11-26，现 91 条 AC） | — |
| 总体架构与报告导出 | 总体架构五层确认；报告导出本期实现（年度/分析报告、看板统计、建议报告，PDF+Excel） | 已同步 spec v2.7（REQ-73/AC-89~91）、plan v3.0（AD-12）；新增任务 T266~T273；Word/简报 PPT 仍裁剪 | T266~T273、T264 |
| 部署与网络 | 本地化部署、集中式本期/主备预留、**采集本期部分实现（v3.7 修订）**、每日全量备份+恢复演练、管理面/业务面分区 | 已同步 spec v2.8（§1 部署约束）、plan v3.2（§4.2.1）；备份与端口核对并入 T265，采集实现见 T288~T299 | T265、T288~T299 |
| 检索/日志/监控/调度 | 引入 ES/ELK/Prometheus+Grafana/GitLab CI；恢复 XXL-Job；**采集部分实现（v3.7 补第 6 类调度）**；浏览器仍 Chrome/Edge | 已同步 plan v3.3（AD-13/AD-05 恢复）；新增 T274~T281；T015/T038/T069/T262 改回 XXL-Job 与 ELK/Grafana 核对 | T015、T274~T281、T262 |
| 微服务架构图 | 课程服务改名 curriculum-service/curriculum_db；data 职责含采集（v3.7 部分实现）/预测（结果）；课程项目案例为系统外交付物；文件存储维持本地磁盘 | 已同步 plan v3.4（全局改名）与 v3.18（成果台账归 data-service）；任务路径与 db-design 同步替换；不新增任务 | 全局 |
| 技术选型表 | 向量检索=Redis Stack（不引独立向量库）；模型网关=应用内路由；图表=仅 ECharts；选型原则入档 | 已同步 plan v3.5；新增 T282/T283（Redis 向量检索）；tasks §0.2 Redis 改 Redis Stack | T282/T283 |
| 用户角色与权限 | 六类角色（新增安全管理员）统一命名；三员分立+关键操作双人复核；权限点 17 项；表 7-7 四项裁剪维持 | 已同步 spec v2.10（REQ-63/74、AC-92）、plan v3.6（AD-14、复核接口）；新增 T284~T287；T029~T032/T051 改 17×6 | T284~T287、T029~T032 |
| 指标2 产业数据库 | 采集部分拉回（任务配置/定时增量/文件拉取/手工补录，无爬虫）；原始链接溯源；清洗规则可配置+处理台账；企业作查询条件 | 已同步 spec v2.11（REQ-75/76、AC-93~95）、plan v3.7（AD-15/16、8 个新接口）；新增 T288~T299 | T288~T299 |
| 收尾口径冻结（v2.22） | 质量评分分母；参数配置；模型 Key 生效；REQ-72 预测；OPEN-03；目标企业清单 | 已同步 spec v2.22 / plan v3.19 / db-design v1.11：①质量分母=正式表全部记录（T172/T173）；②参数配置=Nacos OpenAPI、不建 sys_param（新增 T300~T303，AD-17）；③模型 Key 复核通过后由 user-service 编排 ai-service 生效、失败日志+告警不回退（T109/T110/T284/T285）；④REQ-72 统计时序预测+AC-98（新增 T304~T309，AD-18）；⑤OPEN-03 冻结关闭；⑥目标企业清单全量替换（T164/T165） | T109/T110、T164/T165、T172/T173、T284/T285、T300~T303、T304~T309 |
| 补录批次口径（OPEN-DB-05） | 手工补录"批次数"（单条录入无批次实体） | **按日聚合**：同一天的手工补录计 1 个批次，记录数按条数；不需新增表/字段；AC-38/REQ-36 已同步 | T172 |
| 占位权限点 | 任务调度、备份恢复（v2.22 预留/运维执行） | **保留 17 项权限点不变**，矩阵展示但本期无功能入口 | T030、T031、T051 |
| 宣传数据量（NFR-01） | 124 万条/2.8 万家/27 城 | **保留待复核、不阻塞**：以实时库为准，仅作性能基线，不作为验收门禁 | — |
| 导入字段口径（v1.2.1，**已被 v1.4.0 标准提示词口径取代**） | 来源文件字段与必填/行业/技能规则 | 原口径：必填=公司名称/岗位名称/工作城市/发布日期；行业不由文件提供；技能可选。**现行口径见下一行** | T062/T063、T074~T077、T082/T083、T092/T093、T100/T101、T108 |
| 标准提示词对齐（v1.4.0） | 导入映射与建议输出 | 必填=**公司名称/岗位名称/工作地点/发布日期**；**行业文件优先、缺省企业匹配/确认**；**技能不映射、AI 从 JD 抽取**；导入采用标准 15 字段字典与标准输出适配；建议类型统一“课程内容”，AI 输出 recommendedType/level/title/desc/accordingTo | T062/T063、T074~T077、T082/T083、T092/T093、T100/T101、T108、T207/T209/T210、T213、T215/T216 |
| 冗余表清理（v1.3.0） | field_mapping_cache、account_data_scope、permission、role_permission | **四张全部删除**：映射缓存统一 Redis；数据范围收敛为全校/只读/全系统；权限点与角色授权改代码常量、矩阵只读；db-design 表 46→42、实体 42→40、关系 24→22 | T029、T030、T031 |

> **REQ-72 已细化并落地任务（2026-11-26，spec v2.22 / plan v3.19 AD-18）**：人才需求预测=统计时序模型（移动平均/指数平滑/线性回归，可组合），输入历史月度岗位需求（有效职位），支持区域/行业/岗位族筛选，输出下一季度+下一年预测值、趋势方向与增长提示，回测 MAPE≤20%；数据不足显式；结果落 data_db `prediction_result`；任务见 T304~T309，验收 AC-98。预警口径=现有趋势增长提示+配额预警（新兴技能预警仍不实现）；项目案例/实践资源为系统外交付物（项目资源下载仍不实现）。

**明确不展开（spec §4.2 裁剪，后续迭代）**：JSON 导入与导入模板下载、看板"一键生成简报（PPT）"、新兴技能预警、课程大纲预览导出、生成更新建议/发起更新、课程更新审批流与教师建议提交、标签审核、账号审批与外部同步、项目资源下载、报告导出 Word 格式、成果台账导出、数据源配置、单点登录/第三方登录、AI 提示词界面化维护。**（报告导出 PDF/Excel 已于 v3.0 拉回本期，见 T266~T273。）**

---

## 附录 A：AC 全覆盖核对表（spec §7 共 91 条）

| AC | 测试任务 | 实现/验证任务 | AC | 测试任务 | 实现/验证任务 |
| --- | --- | --- | --- | --- | --- |
| AC-01 | T023, T025 | T024, T026, T059 | AC-45 | T229 | T230, T241 |
| AC-02 | T023, T025 | T024, T026, T059 | AC-46 | T231 | T232, T241 |
| AC-03 | T029 | T030, T059 | AC-47 | T213 | T214, T235, T241 |
| AC-04 | T062, T072 | T063, T073, T108 | AC-48 | T215 | T216, T235, T241 |
| AC-05 | T076 | T077, T100, T108 | AC-49 | T217 | T218, T235, T241 |
| AC-06 | T080 | T081, T108 | AC-50 | T219 | T220, T235, T241 |
| AC-07 | T064, T076 | T065, T077, T108 | AC-52 | T194 | T195, T204 |
| AC-08 | T125 | T126, T143, T149 | AC-53 | T196 | T197, T204 |
| AC-09 | T131 | T132, T149 | AC-54 | T198 | T199, T202, T204 |
| AC-10 | T131 | T132, T145, T149 | AC-55 | T064, T088 | T065, T089, T102, T108 |
| AC-11 | T117 | T118, T149 | AC-56 | T082 | T083, T108 |
| AC-12 | T119 | T120, T149 | AC-57 | T115, T123 | T116, T124, T149 |
| AC-13 | T121, T139 | T122, T140, T147, T149 | AC-58 | T090 | T091, T149 |
| AC-14 | T154 | T155, T177, T179, T187 | AC-59 | T137 | T138, T147, T149 |
| AC-15 | T150, T156 | T151, T157, T181, T187 | AC-60 | T244 | T245, T254 |
| AC-16 | T150, T156 | T151, T157, T181, T187 | AC-61 | T246 | T247, T254 |
| AC-17 | T150, T156 | T151, T157, T181, T187 | AC-62 | T248 | T249 |
| AC-18 | T158 | T159, T181, T187 | AC-63 | T250 | T251, T254 |
| AC-19 | T160 | T161, T183, T187 | AC-64 | T252 | T253, T254 |
| AC-20 | T152 | T153, T187 | AC-65 | T007, T027 | T008, T028, T049, T059 |
| AC-21 | T188 | T189, T202, T204 | AC-66 | T011, T035 | T012, T036, T053, T059 |
| AC-22 | T190 | T191, T202, T204 | AC-67 | T127 | T128, T143, T149 |
| AC-23 | T192 | T193, T202, T204 | AC-69 | T109 | T110, T256 |
| AC-24 | T205, T207 | T206, T208, T233, T241 | AC-70 | T111 | T112, T256 |
| AC-25 | T207, T209 | T208, T210, T241 | AC-71 | T113, T258 | T114, T259, T256 |
| AC-26 | T211 | T212, T233, T241 | AC-72 | T164 | T165, T181, T187 |
| AC-30 | T039 | T040, T053, T059 | AC-73 | T027 | T028, T049, T059 |
| AC-31 | —（人工核对） | T059 | AC-74 | T029 | T030, T047, T059 |
| AC-32 | T200, T242 | T201, T243, T254 | AC-75 | T031 | T032, T051, T059 |
| AC-33 | T041, T045 | T042, T046, T059, T187 | AC-76 | T033 | T034, T051, T059 |
| AC-34 | T154 | T179, T187 | AC-77 | T072, T076 | T073, T077, T100 |
| AC-35 | T162 | T163, T183, T187 | AC-78 | T074 | T075, T100 |
| AC-37 | T170 | T171, T185, T187 | AC-79 | T064 | T065, T102, T108 |
| AC-38 | T141, T172 | T142, T173, T185, T187 | AC-80 | T086 | T087, T104, T108 |
| AC-40 | T185 | T239, T241 | AC-81 | T092 | T093, T106 |
| AC-41 | T221 | T222, T237, T241 | AC-82 | T094 | T095, T106 |
| AC-42 | T223 | T224, T239, T241 | AC-83 | T090 | T091, T106, T108 |
| AC-43 | T225 | T226, T239, T241 | AC-84 | T115 | T116, T149 |
| AC-44 | T227 | T228, T239, T241 | AC-85 | T123, T152 | T124, T153 |
|  |  |  | AC-86 | T060 | T061, T098 |
|  |  |  | AC-87 | T070 | T071, T098 |
|  |  |  | AC-88 | T066 | T067 |
|  |  |  | AC-89 | T266, T272 | T267, T273 |
|  |  |  | AC-90 | T268, T272 | T269, T273 |
|  |  |  | AC-91 | T270, T272 | T271, T273 |
| AC-92 | T284, T286, T300, T302 | T285, T287, T301, T303, T059 |
| AC-93 | T288, T290, T292, T298 | T289, T291, T293, T299 |
| AC-94 | T296, T298 | T297, T299 |
| AC-95 | T292, T294 | T293, T295 |
| AC-96 | T166, T185 | T167, T187 |
| AC-97 | T168 | T169, T185, T187 |
| AC-98 | T304, T306, T308 | T305, T307, T309, T241 |

> 编号空洞（AC-27/28/29/36/39/51/68）为 spec 历史裁决遗留，无对应条目，不列入。

---

## 附录 B：任务统计

| 阶段 | 任务数 | 测试 : 实现 : 其他 | 覆盖 AC（含回归） |
| --- | --- | --- | --- |
| Phase 1（M1） | 77（T001~T059、T274~T287、T300~T303） | 37 : 36 : 4 | AC-01/02/03/30/31/33/65/66/73~76/92 |
| Phase 2（M2） | 61（T060~T108、T288~T299） | 31 : 29 : 1 | AC-04~07/55/56/58/77~83/86~88/93~95 |
| Phase 3（M3） | 41（T109~T149） | 21 : 19 : 1 | AC-08~13/57~59/67/84/85 |
| Phase 4（M4） | 38（T150~T187） | 19 : 18 : 1 | AC-14~20/33~38/40/72/96/97 |
| Phase 5（M5） | 17（T188~T204） | 8 : 8 : 1 | AC-21~23/52~54 |
| Phase 6（M6） | 43（T205~T241、T304~T309） | 22 : 20 : 1 | AC-24~26/40~50/98 |
| Phase 7（M7） | 32（T242~T273） | 16 : 15 : 1 | AC-32/60~64/69~71/89~91 + 全量回归 |
| **合计** | **309** | **154 : 145 : 10** | **91/91** |

> "其他"= 基础设施骨架（T001/T015/T022）、性能验收（T176）、E2E 冒烟与回归记录（T059/T108/T149/T187/T204/T241/T264）、检索/日志/监控/流水线基础设施（T276/T280/T281）等非纯测试/实现任务。
> 所有测试任务（单元/表格驱动/契约/集成/E2E）均携带 AC 标签；纯基础设施守护类测试（T002/T003/T009/T011 部分/T016/T174/T260/T262/T276/T278）显式标注"无直接 AC"并追溯 AD/INV/NFR。
