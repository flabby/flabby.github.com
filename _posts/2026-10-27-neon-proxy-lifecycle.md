# Neon Proxy：启动、Compute 唤醒与运行期协作

本文用三张阶段时序图说明 Neon Proxy 在整个服务端中的生命周期：平台启动、客户端连接与 Compute 唤醒、正常查询及运行期协作；另补充 SQL 角色变更同步到控制面的交互场景。

本文依据 Neon 开源代码版本 [`fa504217c61bbcaf5c512d75830564541f917f8f`](https://github.com/neondatabase/neon/tree/fa504217c61bbcaf5c512d75830564541f917f8f)，整理于 2026-10-27。平台编排与 ELB 就绪检查属于自建服务的部署设计，不能等同于 Neon 或 Databricks Lakebase 生产环境的完整编排。

更新于 2026-10-29：补充 SQL 角色变更上报、认证缓存联动与事务一致性边界。

## 组件与职责

| 组件 | 定位与职责 |
|---|---|
| PostgreSQL 客户端 | 使用 psql、JDBC、pgx、psycopg 等原生驱动连接 |
| ELB / TCP 负载均衡器 | 提供稳定入口，将每条 TCP 连接转发到一个就绪的 Proxy 副本 |
| Proxy 副本池 | TLS、身份认证、端点发现、后端建连、会话转发与查询取消 |
| Redis | 共享取消映射，并通过 Pub/Sub 通知 Proxy 清除认证和访问策略缓存 |
| 控制面 | 维护端点、身份与访问策略，协调 Compute 启动及恢复 |
| 管理元数据库 | 保存控制面的权威元数据，与业务 Compute 的生命周期独立 |
| Compute / compute_ctl | 准备配置与基础数据，启动 Postgres，配置角色和数据库 |
| PostgreSQL | 解析和执行 SQL，管理事务、锁、MVCC 与会话 |
| Compute 内的 Neon 扩展 | 捕获角色、数据库 DDL，并在事务提交前向控制面上报变更 |
| 存储层 | Safekeepers 持久化 WAL，Pageserver 提供数据页，对象存储保存长期数据 |

## 阶段一：平台与 Proxy 启动

先启动管理元数据库、Redis 和存储基础服务，再准备控制面及 Proxy 接入层。业务 Compute 可以保持休眠。

```mermaid
sequenceDiagram
    participant LB as ELB
    participant P as Proxy副本池
    participant R as Redis
    participant CP as 控制面
    participant M as 管理元数据库

    CP->>M: 加载端点、角色与策略
    CP->>CP: 启动控制接口
    P->>P: 加载配置、证书并启动监听
    P->>R: 连接、初始化取消映射与通知订阅
    R-->>P: 连接成功
    LB->>P: 接入就绪检查
    P-->>LB: 就绪，允许接流量

    Note over LB,M: 业务Compute可保持休眠；存储基础服务独立运行
```

需要区分“进程启动”“端口可连接”和“接入就绪”：

- 当前版本先启动监听任务，再完成 Redis 初始化；不能只用端口打开作为完整就绪判断。
- Redis 连接成功后，缓存失效订阅以异步任务运行，不能把服务 running 状态等同于订阅已成功。
- 图中的 ELB 接入就绪检查需要在部署侧补齐。源码中的 `/v1/status` 固定返回 HTTP 200，不检查 Redis、控制面或所有业务数据库。
- 就绪检查不应要求所有业务 Compute 都已运行，否则会与休眠设计冲突。
- ELB 使用 TCP 转发、TLS 在 Proxy 终止时，客户端的 TLS 与 SNI 可以到达 Proxy。若启用 PROXY protocol v2，负载均衡器与 Proxy 的配置必须一致。

## 阶段二：客户端连接并唤醒 Compute

Proxy 从控制面获取认证材料及后端地址。目标 Compute 休眠时，由控制面协调启动；Proxy 等待后端可连接，再建立数据库会话。

```mermaid
sequenceDiagram
    participant C as PG客户端
    participant LB as ELB
    participant P as Proxy
    participant R as Redis
    participant CP as 控制面
    participant CT as Compute / compute_ctl + Postgres
    participant S as 存储层 / Safekeeper + Pageserver

    C->>LB: 数据库连接
    LB->>P: TCP转发
    P->>P: TLS握手、解析SNI与端点
    P->>CP: 获取角色认证与访问策略
    CP-->>P: 返回认证材料和策略
    P-->>C: 经ELB完成SCRAM认证交互
    P->>CP: 发现或唤醒目标Compute
    CP->>CT: 启动并下发配置
    CT->>S: 同步状态、获取数据
    CT-->>CP: Postgres及角色配置就绪
    CP-->>P: 返回后端地址和TLS信息
    P->>CT: 建立后端连接并认证
    P->>R: 后台登记、续期取消映射
    P-->>C: 经ELB返回连接就绪

    Note over C,CT: 后续SQL沿既有连接转发，不逐条访问Redis或控制面
```

- 图中控制面调用是缓存未命中时的逻辑路径；命中缓存时可以省略相应远程请求。
- Compute 已运行时，控制面返回其地址，不需要再次启动。
- `compute_ctl` 的准备过程包括存储状态同步、基础数据获取、Postgres 启动和角色配置。进程出现不等同于业务数据库就绪。
- 取消映射的登记和续期是后台工作，图中相邻箭头不表示每次连接都同步等待一次 Redis 写入。
- 客户端响应仍经过 ELB，图中部分返回箭头将中转步骤合并展示。
- 图示为 Direct 路径。Pooled 路径在 Proxy 与 Postgres 之间增加 PgBouncer。

## 阶段三：正常查询与运行期协作

普通 SQL 使用既有 TCP 连接转发；取消查询通过独立连接发送；密码或策略变更通过 Redis 失效通知传播到各 Proxy 副本。

```mermaid
sequenceDiagram
    participant C as PG客户端
    participant LB as ELB
    participant P as Proxy副本池
    participant R as Redis
    participant CP as 控制面
    participant DB as PostgreSQL
    participant S as 存储层

    Note over C,S: 正常查询：使用已建立的连接
    C->>LB: SQL协议消息
    LB->>P: 转发到原Proxy副本
    P->>DB: 转发SQL协议消息
    DB->>S: 按需读页、写入WAL
    DB-->>P: 查询结果
    P-->>C: 经ELB返回结果

    opt 客户端取消查询
        C->>LB: 新连接发送CancelRequest
        LB->>P: 可能进入另一个Proxy副本
        P->>R: 查询真实后端取消信息
        R-->>P: 地址、PID、取消密钥与身份
        P->>CP: 获取访问策略，可命中本地缓存
        CP-->>P: 返回策略
        P->>P: 检查访问权限
        P->>DB: 发送取消请求
    end

    opt 密码或访问策略变更
        CP->>CP: 更新权威元数据
        CP->>R: 发布缓存失效通知
        R-->>P: 通知各Proxy副本
        P->>P: 清除对应本地缓存
    end
```

运行期需要注意：

- ELB 为每条连接选择目标；普通查询始终沿原有连接到达同一个 Proxy。取消请求另建连接，因此可能进入不同副本。
- Proxy 向客户端提供代理生成的取消标识，Redis 将其映射到真实后端地址、PID 和取消密钥。在此版本中，同一 Proxy 上的原生取消路径也依赖该映射。
- 发送取消请求不等于查询已经结束，客户端还需观察原查询连接上的取消结果。
- 认证与访问策略缓存主要位于 Proxy 内存；Redis 传递失效事件，权威状态仍在控制面。
- Pub/Sub 不是可靠事件日志，需要结合缓存 TTL、重连恢复或版本校验约束变更生效时间。密码缓存失效也不等于自动终止已有会话。
- 存储访问是按需发生的；命中本地缓存的读请求不需要每次访问 Pageserver，事务提交则涉及 WAL 持久化。

## 运行期补充：SQL 创建角色如何同步到控制面

用户通过 Proxy 执行 `CREATE ROLE app_user LOGIN PASSWORD '…'` 或 `ALTER ROLE app_user PASSWORD '…'` 时，Proxy 转发 PostgreSQL 协议消息。**捕获并上报角色变更的是 Compute 内的 Neon 扩展**：它通过 `ProcessUtility_hook` 收集变更，再由事务回调向控制面发送 HTTP PATCH。无需让 Proxy 解析 SQL。

下面展示成功路径，省略 ELB。控制面的接收处理需要自建；开源代码可以确认 Compute 的发送逻辑，但不能据此确认商业控制面的内部持久化实现。

```mermaid
sequenceDiagram
    participant C as PG客户端
    participant P as Proxy
    participant DB as Compute / Postgres + Neon扩展
    participant CP as 控制面DDL接收接口

    C->>P: CREATE ROLE / ALTER ROLE PASSWORD
    P->>DB: 转发SQL协议消息
    DB->>DB: 执行DDL，收集事务内变更
    Note over DB,CP: PRE_COMMIT阶段上报，本地事务尚未提交
    DB->>CP: HTTP PATCH，携带角色变更与认证材料
    CP->>CP: 接收并处理角色元数据（需自建）
    CP-->>DB: HTTP 200
    DB->>DB: 继续提交事务，仍可能失败
    DB-->>P: SQL或COMMIT执行结果
    P-->>C: 返回结果
```

显式事务中，上报发生在 `COMMIT` 的提交前阶段；自动提交模式中，发生在该语句所属事务的提交前阶段。子事务的变更会在子事务提交时合并，回滚时丢弃。

### 上报接口与数据

| 项目 | 开源实现中的行为 |
|---|---|
| 接收地址 | `neon.console_url` 指定的完整 URL；未配置则跳过上报 |
| 开关 | `neon.forward_ddl`，默认开启；关闭时跳过上报 |
| 请求 | HTTP PATCH，JSON；配置 `NEON_CONTROL_PLANE_TOKEN` 时使用 Bearer 认证 |
| 角色变更 | 创建、删除、重命名及密码修改；不能理解为同步所有角色属性和权限变更 |
| 角色字段 | `roles` 数组内包含 `op`、`name`；按变更内容附带 `password`、`encrypted_password`、`old_name` |
| 操作 | `op: set` 表示设置角色状态；`op: del` 表示删除 |
| 认证材料 | `encrypted_password` 来自 PostgreSQL 实际保存的密码 verifier；`password` 可能携带 SQL 中提供的明文密码或预计算哈希 |
| 成功判定 | 接收端返回 HTTP 200；其他状态码触发错误 |

控制面应保存与 Compute 一致的 SCRAM verifier，包含相同的 salt、迭代次数和密钥材料，供 Proxy 前端认证使用。仅使用相同明文密码重新生成一份不同 salt 的 verifier，不能满足前后端 SCRAM 密钥复用的要求。上报报文应按凭据处理，使用受保护的传输和接口认证，并避免记录密码字段。

### 与 Redis、Proxy 缓存的联动

控制面接收角色变更后，需要将角色认证元数据更新与阶段三的缓存失效链路接起来：**控制面 → Redis Pub/Sub → 各 Proxy 清除对应角色缓存 → 后续认证重新获取数据**。其中 Proxy 的失效通知消费代码已开源；接收 DDL、更新元数据和发布通知的控制面服务需要自行实现。

这条缓存通知链路不承担 DDL 上报，也不保证与 PostgreSQL 提交原子一致。通知发布时机需要由控制面设计；缓存失效不会自动终止已认证的连接。

### 故障与一致性边界

- **上报失败会影响 DDL 提交。** 单次 HTTP 请求超时为 3 秒；curl 传输错误最多尝试 5 次，失败尝试之间等待约 1 秒。HTTP 非 200 会触发错误，不属于该传输重试循环。启用上报时，最终失败会使正常事务提交失败。
- **上报成功不保证数据库最终提交。** 上报发生在 `PRE_COMMIT`；控制面可能已经收到变更，而 PostgreSQL 随后提交失败。源码明确说明未处理这一窗口，因此不能把它描述成可靠的“提交后事件”或分布式事务。
- **控制面需实现幂等与对账。** 网络超时可能发生在接收端已处理请求之后；重试需避免重复副作用。可利用 `compute_ctl` 的 `GET /dbs_and_roles` 获取当前角色、密码 verifier 和数据库信息进行校验修复；该接口是主动查询能力，不会自行完成同步或修复。
- **避免下发配置形成回环。** `compute_ctl` 在应用控制面配置的相关会话中设置 `neon.forward_ddl = false`，避免由控制面下发的变更再次上报。

对于“Neon Compute 与普通 PostgreSQL 统一接入”的自建场景：Neon Compute 可复用上述扩展发送端，并补齐控制面接收接口；普通 PostgreSQL 不会因接入 Neon Proxy 就自动产生这些上报，需要另外实现角色变更捕获、同步和对账，或统一由管理接口执行角色变更。仅部署 Proxy 不能完成这条控制面闭环。

## Compute 与 Proxy 的生命周期

| 状态 | 独立接入层 Proxy | Compute 内的 Postgres、local_proxy、PgBouncer |
|---|---|---|
| Compute 活跃 | 保持接入与转发 | 按相应访问路径提供服务 |
| 整个 Compute 休眠 | 保持运行，接住新连接并协调唤醒 | 随 Compute 停止 |
| Compute 恢复 | 重新发现目标并建立后端连接 | 初始化后重新提供服务 |

这里的 `local_proxy` 是 Compute 内的 HTTP/serverless 配套组件，不是图中的独立原生 PostgreSQL 接入 Proxy。原生 Direct 或 PgBouncer 路径不要求经过 `local_proxy`。

Compute 休眠或故障后，原有连接、事务与会话状态不能由 Proxy 自动恢复。应用需要重连，并正确处理事务结果不确定性。

## 源码参考

- [Proxy 进程启动与任务监督](https://github.com/neondatabase/neon/blob/fa504217c61bbcaf5c512d75830564541f917f8f/proxy/src/binary/proxy.rs)
- [健康与指标接口](https://github.com/neondatabase/neon/blob/fa504217c61bbcaf5c512d75830564541f917f8f/proxy/src/http/health_server.rs)
- [客户端认证与后端连接主流程](https://github.com/neondatabase/neon/blob/fa504217c61bbcaf5c512d75830564541f917f8f/proxy/src/proxy/mod.rs)
- [控制面地址发现与访问控制客户端](https://github.com/neondatabase/neon/blob/fa504217c61bbcaf5c512d75830564541f917f8f/proxy/src/control_plane/client/cplane_proxy_v1.rs)
- [查询取消映射](https://github.com/neondatabase/neon/blob/fa504217c61bbcaf5c512d75830564541f917f8f/proxy/src/cancellation.rs)
- [Redis 缓存失效通知](https://github.com/neondatabase/neon/blob/fa504217c61bbcaf5c512d75830564541f917f8f/proxy/src/redis/notifications.rs)
- [Compute 启动入口](https://github.com/neondatabase/neon/blob/fa504217c61bbcaf5c512d75830564541f917f8f/compute_tools/src/bin/compute_ctl.rs)
- [Neon 扩展：DDL 捕获、变更报文与 PRE_COMMIT 上报](https://github.com/neondatabase/neon/blob/fa504217c61bbcaf5c512d75830564541f917f8f/pgxn/neon/neon_ddl_handler.c)
- [DDL forwarding 回归测试](https://github.com/neondatabase/neon/blob/fa504217c61bbcaf5c512d75830564541f917f8f/test_runner/regress/test_ddl_forwarding.py)
- [Compute 配置应用与禁用 DDL 回传](https://github.com/neondatabase/neon/blob/fa504217c61bbcaf5c512d75830564541f917f8f/compute_tools/src/compute.rs)
- [compute_ctl HTTP 路由](https://github.com/neondatabase/neon/blob/fa504217c61bbcaf5c512d75830564541f917f8f/compute_tools/src/http/server.rs)
- [角色和数据库目录查询](https://github.com/neondatabase/neon/blob/fa504217c61bbcaf5c512d75830564541f917f8f/compute_tools/src/catalog.rs)
