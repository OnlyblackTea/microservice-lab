# microservice-lab

**「microservice-bagu」Track 的骨架仓** —— 与发卷平台 [bagu-trainer](https://github.com/OnlyblackTea/bagu-trainer)
对接：平台按 `track.yaml` 渲染看板与体检环境，按 `stages/*/ticket.yaml` 发卷，
判据在**学员自己的机器上**跑。

> **当前状态：设计完成，骨架尚未落地。**
> 本仓目前只有文档（`docs/`）。骨架代码、canonical 测试、`track.yaml`、`stages/*/ticket.yaml`
> 与阶段分支链会随 Stage 1 一起落进来——顺序与判据形状见
> [`docs/设计规格.md`](docs/设计规格.md)，落地清单见
> [`docs/骨架落地清单.md`](docs/骨架落地清单.md)。


## 这是什么

一句话定位：给「Spring Cloud 全家桶背得很熟、但没亲手让一个熔断器真的跳闸过」的 Java 后端——

| 项 | 值 |
|---|---|
| track.id | `microservice-bagu` |
| 语言栈 | Java 17 + Maven（多模块）+ Spring Boot 3.2.5 + JUnit 5 / Surefire `-Dgroups=stageNN` |
| 基础设施 | **默认档：零外部容器**（只跑 JVM，一个进程内三个 Spring Boot 上下文：gateway + order + stock）。常用档：`mysql:8.0@13306` + `redis:7@16379`（复用 `seckill-lab` 的端口约定）。演示档：`nacos/nacos-server:v2.x`（选修，不进判据） |
| stage 数 | **5 个**（Stage 1–4 为判据主干，Stage 5 为**选修**；另有 1 个 board-only 的 `stage-00-onboarding` 只上看板） |
| 每 stage 规模 | 按 [`_SCALE.md`](./_SCALE.md) 的下限复核后（本轮「**v1.1 规模复核**」，见 §1）：5 个 stage **全部「同 stage 扩面」、零合并**，改动面合计 **20 → 29 个文件位置**（v1 的 20 处 + **9 处新增**：`InstanceHealthGuard`、`RetryBudget`、`CallEnvelope`、`IdentityQuotaRegistry`、`LimiterMetricsGuard`、`TraceLogContext`、`SnowflakeLayout`、`IdCollisionRetry`、`IdempotencyRegistry`）。canonical **17 → 28 条**（红 **10** · 单线程/正常路径基线 13 · 边界/拒绝 5），被破坏的不变量 **每 stage 各 2 个**（= 每 stage 的红用例数，合计 10 个）；考点族每 stage ≥ 2 |
| 依赖前置 | `java: "17"` / `build: maven` / `docker: "24"`（仅常用档与演示档需要）/ `memory_gb: "4"`（见 §6.4） |
| 与 database-bagu 的关系 | 无重叠。`seckill-lab` 覆盖的是「单个进程 + 一个数据库」的深水区（索引/事务/锁/MVCC/缓存/死锁）；本 Track 的对象是**进程之间**（实例列表、超时、熔断、限流、traceId），两边唯一的共同词汇是「幂等」 |
| 预计落地工作量 | 骨架 9 人日 / canonical 测试 8.5 人日 / 题面与判据 3 人日 / 分支与 pin 1.5 人日 ≈ **22 人日**（旧口径 14 人日是「每 stage 一个方法 / 两三条用例」的估算，已按 §7 的 v1.1 重新估算块上修；**内存预算没变，规模靠加文件**） |

## stage 一览

| stage | 分支名 | 依赖 | canonical 测试路径 |
|---|---|---|---|
| 0（board-only，不出 ticket） | `—` | — | src/test/java/com/devteam/micro/SmokeTest.java |
| 1 | `stage-01-discovery-and-loadbalance` | `[]`（**DAG 根**） | src/test/java/com/devteam/micro/stage01/LoadBalancerRoundRobinAcceptanceTest.java`、`…/stage01/LoadBalancerFailoverAcceptanceTest.java`、`…/stage01/InstanceListBoundaryAcceptanceTest.java |
| 2 | `stage-02-circuit-breaker-and-timeout` | `[micro-stage-01]` | src/test/java/com/devteam/micro/stage02/CircuitBreakerAcceptanceTest.java`、`…/stage02/DegradationAcceptanceTest.java`、`…/stage02/RetryBoundAcceptanceTest.java |
| 3 | `stage-03-rate-limit-and-overload` | `[micro-stage-01]` | src/test/java/com/devteam/micro/stage03/RateLimitAcceptanceTest.java`、`…/stage03/QuotaRecoveryAcceptanceTest.java`、`…/stage03/OverloadIsolationAcceptanceTest.java |
| 4 | `stage-04-trace-and-log-correlation` | `[micro-stage-01]` | src/test/java/com/devteam/micro/stage04/TraceIdPropagationAcceptanceTest.java`、`…/stage04/LogCorrelationAcceptanceTest.java |
| 5（选修） | `stage-05-distributed-id` | `[micro-stage-01]` | src/test/java/com/devteam/micro/stage05/DistributedIdAcceptanceTest.java`、`…/stage05/IdCollisionAndIdempotencyAcceptanceTest.java |

> 表里的**分支名就是 git 分支名**：平台的 Forge 在 provision 时用
> `base_ref = origin/<分支名>`（起始代码）与 `head_ref = origin/<分支名>-sol`（参考解），
> 两者缺一都会报 `workspace_ref_missing`。分支是**一条链**（每个阶段分支 = 上一阶段 + 本阶段测试），
> 不是兄弟分支——理由见骨架落地后的 `stages/README.md`。

## 目录

```
microservice-lab/
├── README.md              ← 本文件
├── LICENSE                ← MIT（内容仓许可）
├── .gitattributes         ← eol=lf：digest_pin 要求工作树字节跨平台稳定
├── .gitignore
└── docs/
    ├── 设计规格.md          ← 本 Track 的设计规格快照（见下方提醒）
    └── 骨架落地清单.md       ← 落地前要逐条核对的清单
```

骨架落地后还会出现（形状见设计规格 §2）：

```
├── track.yaml             ← 与平台对接的清单（manifest 惰性声明 / runtime 可执行入口 + digest_pin）
├── pom.xml / go.mod / pyproject.toml
├── src/ 或 internal/      ← 业务实现（学员改动面）
├── src/test/... 或 tests/ ← canonical 验收判据【判定面】
├── stages/<stage-目录>/ticket.yaml   ← 权威工单契约（目录名 = 分支名）
├── stages/README.md       ← 分支链契约
├── scripts/               ← build / digest-pin / check-red-green【判定面】
└── TASKS/                 ← 学习者叙述视图与交付证据
```

## 与规格的关系（重要）

`docs/设计规格.md` 是**创建本仓时的规格快照**，权威副本在题库侧的
`bagu-track-specs/microservice-lab.md`（设计仍在演进的阶段，改动先落那边，再同步到这里）。

两份文档的分工别搞混：

- **平台契约**（字段、准入门、分支链、锚点算法）由平台仓单方面定义，本仓只遵守、不复制规范正文；
- **题面怎么写**见平台仓 `docs/工单写作规范.md`；
- **每个 stage 该有多大**见题库侧 `_SCALE.md`（改动面 ≥3 文件、canonical ≥5 条、
  红 ≥2 条不同不变量、含边界路径）——本规格已按它复核过。

## 许可

MIT，见 [LICENSE](LICENSE)。
