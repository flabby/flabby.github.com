# Neon Proxy 服务化接入设计

## 1. 方案总览与改造范围

一期为新建 Compute 默认提供共享的 PostgreSQL 入口，由 Proxy 完成认证和选路，SC 保存元信息，节点 Agent 向管控上报唤醒请求。

```mermaid
flowchart LR
    C["PG 客户端"] --> E["负载均衡器"] --> P["共享 Proxy 实例<br/>2 个节点，各运行 Proxy 和 Agent"]
    P --> A["Compute A · main 分支"]
    P --> B["Compute B · dev 分支"]
    P -.->|"查路由和角色"| S["SC · MetaDB"]
    P -.->|"Agent 上报唤醒请求"| M["云管控"]
```

实线是 SQL 连接。负载均衡器选 Proxy 副本，Proxy 按 `endpoint` 选分支和节点。每个分支的 Compute 对应一个唯一实例 ID，包含一个主节点，可配 2–4 个只读节点。

| 改动项目 | 本期交付 |
|---|---|
| Proxy 接入 | 复用 PG/TLS 和 SCRAM，新增 SC 客户端、按实例及节点选路、本地硬 TTL 缓存、并发回源控制和限时建连重试。 |
| 节点 Agent | 监督本机 Proxy 进程、采集指标、转发本机管理请求，通过管控通知通道单向上报激活意图。 |
| StorageController（SC） | 增加两张表和五个业务 API。Proxy 只读，管控写路由，Compute 写自身的角色。只有 SC 访问 MetaDB。 |
| Compute | 网页配置和角色 SQL 共用提交前同步。主节点默认启动 PgBouncer，普通端口和事务池端口分别验收。 |
| 云管控对接 | 创建节点并将其注册到负载均衡后端组，发布路由，根据实际状态处理唤醒，在 Compute 恢复后补发配置。 |

**一期边界。** 仅接入新实例，无 Redis。PG `CancelRequest` 到达 Proxy 后计数并结束取消连接，例如 psql 按 Ctrl+C 后原 SQL 仍可能继续执行。查询时长可由 Compute 的 `statement_timeout` 限制。

### 1.1 元信息与表结构

SC 将路由按实例和节点保存，将认证材料按实例和角色保存，同一 Compute 的主节点与只读节点共享角色目录。

| 表 | 写入者与数据 | 高频查询及索引 |
|---|---|---|
| `proxy_endpoints` | 管控发布分支、主 VIP 或成员 IP、端口、TLS 名称、路由状态和版本 | `(instance_id, node_key)` 唯一索引，支持登录取址和切换后刷新 |
| `proxy_roles` | Compute 同步 LOGIN、SCRAM verifier、有效期、删除标记和角色版本 | `(instance_id, role_name)` 唯一索引，支持认证、对账和幂等写入 |

两表均用自增数值主键、创建和更新时间、三个预留列，无外键。各调用方遵守 DDL 中的长度上限，按 PG 目录保留角色名大小写。MetaDB 用 UTF8，`timestamptz` 按部署时配置的本机时区显示。角色表只接收合法 verifier 或显式空值。建表 SQL 如下。

```sql
-- PostgreSQL / UTF8。整份迁移在同一事务执行。
BEGIN;
CREATE TABLE proxy_endpoints (
    id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    instance_id varchar(128) NOT NULL,
    node_key varchar(16) NOT NULL CHECK (node_key IN ('primary','ro1','ro2','ro3','ro4')),
    branch_id varchar(128) NOT NULL,
    target_ip inet,
    compute_port integer CHECK (compute_port BETWEEN 1 AND 65535),
    pooler_port integer CHECK (pooler_port BETWEEN 1 AND 65535),
    tls_server_name varchar(253),
    route_state smallint NOT NULL CHECK (route_state IN (0,2,5)),
    route_version bigint NOT NULL CHECK (route_version > 0),
    reserve_1 varchar(256), reserve_2 varchar(256), reserve_3 varchar(256),
    create_at timestamptz NOT NULL DEFAULT CURRENT_TIMESTAMP,
    update_at timestamptz NOT NULL DEFAULT CURRENT_TIMESTAMP,
    CHECK (route_state <> 2 OR (target_ip IS NOT NULL AND compute_port IS NOT NULL)),
    CHECK (pooler_port IS NULL OR (node_key = 'primary' AND route_state = 2 AND pooler_port <> compute_port)),
    CHECK (instance_id ~ '^[a-z0-9]+$'),
    CHECK (target_ip IS NULL OR masklen(target_ip) = CASE family(target_ip) WHEN 4 THEN 32 ELSE 128 END)
);
COMMENT ON COLUMN proxy_endpoints.pooler_port IS '已发布的主节点 PgBouncer 端口，NULL 表示池化入口暂不可用';
COMMENT ON COLUMN proxy_endpoints.route_state IS '节点路由的发布状态：0 已登记但暂不可用，2 可连接，5 已退役';
COMMENT ON COLUMN proxy_endpoints.instance_id IS '路由域内唯一的分支 Compute 实例 ID，主备切换时沿用';
COMMENT ON COLUMN proxy_endpoints.node_key IS 'primary 对应主 VIP；ro1 至 ro4 各对应一个只读节点的可达地址';
CREATE UNIQUE INDEX proxy_endpoint_lookup ON proxy_endpoints(instance_id, node_key);

CREATE TABLE proxy_roles (
    id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    instance_id varchar(128) NOT NULL CHECK (instance_id ~ '^[a-z0-9]+$'),
    role_name varchar(128) NOT NULL,
    can_login boolean NOT NULL,
    scram_verifier varchar(1024),
    valid_until timestamptz,
    is_deleted boolean NOT NULL DEFAULT false,
    role_version bigint NOT NULL CHECK (role_version > 0),
    reserve_1 varchar(256), reserve_2 varchar(256), reserve_3 varchar(256),
    create_at timestamptz NOT NULL DEFAULT CURRENT_TIMESTAMP,
    update_at timestamptz NOT NULL DEFAULT CURRENT_TIMESTAMP
);
-- 每个角色保留一行当前态（包括删除墓碑），精确查询和幂等 UPSERT 共用一个索引。
CREATE UNIQUE INDEX proxy_role_lookup ON proxy_roles(instance_id, role_name);

CREATE FUNCTION proxy_set_update_at() RETURNS trigger LANGUAGE plpgsql AS $$
BEGIN
    NEW.update_at := clock_timestamp();
    RETURN NEW;
END;
$$;
CREATE TRIGGER proxy_endpoint_updated BEFORE UPDATE ON proxy_endpoints FOR EACH ROW EXECUTE FUNCTION proxy_set_update_at();
CREATE TRIGGER proxy_role_updated BEFORE UPDATE ON proxy_roles FOR EACH ROW EXECUTE FUNCTION proxy_set_update_at();
COMMIT;
```

## 2. SC 改造与接口

SC 为 Proxy 提供登录合查和单独取址，为管控和 Compute 分别提供路由发布与角色同步接口。

**原版接口。** 以下以本地 Neon `fa504217c` 为基线，原控制面路径均相对于配置的 base URL。

| 已有 API | 原用途与示例 | 本期用法 |
|---|---|---|
| SC `GET /ready`、`/status`、`/metrics` | `curl -fsS "$SC_URL/ready"` | 复用就绪、状态和指标接口 |
| 控制面 `GET /get_endpoint_access_control` | 查询参数含 `endpointish=cmp1&role=app_user`、session_id、application_name，返回 role_secret 和访问策略 | 登录改查 SC 的 access。原 IP/VPC 策略若需接入须另定义 |
| 控制面 `GET /wake_compute` | 参数含 endpointish、session_id、application_name，下例为成功响应 | 地址从 SC 读取，激活由 Agent 通知管控 |
| Compute 扩展 `PATCH ConsoleURL` | 上报角色变更，下方示例删除 old_app。 | 保留 PRE_COMMIT 上报点，改发 SC 批接口 |

```json
{
  "address": "198.51.100.8:6432",
  "server_name": "cmp1.example",
  "aux": {
    "endpoint_id": "cmp1",
    "project_id": "p1",
    "branch_id": "br1",
    "compute_id": "c1",
    "cold_start_info": "warm"
  }
}
```

原 Compute 上报删除角色的请求体：

```json
{
  "roles": [
    {
      "op": "del",
      "name": "old_app"
    }
  ]
}
```

原控制面客户端携带 Bearer 凭据及 X-Request-ID。[SC 路由注册](https://github.com/neondatabase/neon/blob/fa504217c61bbcaf5c512d75830564541f917f8f/storage_controller/src/http.rs#L2208) · [Proxy 调用旧控制面](https://github.com/neondatabase/neon/blob/fa504217c61bbcaf5c512d75830564541f917f8f/proxy/src/control_plane/client/cplane_proxy_v1.rs#L140) · [wake 请求和响应处理](https://github.com/neondatabase/neon/blob/fa504217c61bbcaf5c512d75830564541f917f8f/proxy/src/control_plane/client/cplane_proxy_v1.rs#L272) · [Compute 原版 PATCH](https://github.com/neondatabase/neon/blob/fa504217c61bbcaf5c512d75830564541f917f8f/pgxn/neon/neon_ddl_handler.c#L247)

**一期新增 API（拟定）。** 下列路径以 `/v1/proxy` 为前缀。SC 验证服务身份和实例访问范围，access 在同一快照内读两表，认证查询须能看到已确认的角色写入。

| 调用方 | 方法与路径 | 契约 |
|---|---|---|
| Proxy | `GET /access?instance_id=…&node_key=…&mode=…&role=…` | 返回 route 和 auth。缺失、已删除、过期或无 LOGIN 的角色拒绝登录。 |
| Proxy | `GET /routes/{instance_id}/{node_key}?mode=…` | 返回该入口的地址、有效端口、发布状态和版本。 |
| 管控 | `PUT /routes/{instance_id}/{node_key}` | 按版本提交完整节点快照，涵盖发布、切换、摘除和退役。 |
| Compute | `PATCH /computes/{instance_id}/roles` | 在一个 MetaDB 事务内应用一批角色变更。持久化后返回 200。 |
| Compute | `GET /computes/{instance_id}/roles/{role_name}` | 返回自身角色和版本。墓碑返回 is_deleted=true，从未登记返回 404。 |

**路由发布。** 下例发布主节点普通和池化端口。

```http
PUT /v1/proxy/routes/cmp1/primary HTTP/1.1
Content-Type: application/json

{
  "branch_id": "br1",
  "target_ip": "198.51.100.8",
  "compute_port": 5432,
  "pooler_port": 6432,
  "tls_server_name": "cmp1.example",
  "route_state": 2,
  "route_version": 8
}
```

路由发布成功（HTTP 200）：

```json
{
  "instance_id": "cmp1",
  "node_key": "primary",
  "route_version": 8
}
```

**角色同步。** set 给出完整角色状态，del 写墓碑。改名时将旧名 del 与新名 set 放入同一批次。

```http
PATCH /v1/proxy/computes/cmp1/roles HTTP/1.1
Content-Type: application/json

{
  "batch_id": "b-104",
  "roles": [
    {
      "op": "set",
      "role_name": "app_user",
      "role_version": 4,
      "can_login": true,
      "scram_verifier": "<SCRAM verifier>",
      "valid_until": null
    }
  ]
}
```

角色同步成功或等价重试（HTTP 200）：

```json
{
  "batch_id": "b-104",
  "applied": true
}
```

**首次登录合查。** 示例以事务池为目标。服务通过 HTTPS 携带服务令牌和关联 ID，示例中的 verifier 为占位符。

```http
GET /v1/proxy/access?instance_id=cmp1&node_key=primary&mode=pooled&role=app_user HTTP/1.1
```

```json
{
  "route": {
    "instance_id": "cmp1",
    "node_key": "primary",
    "mode": "pooled",
    "branch_id": "br1",
    "target_ip": "198.51.100.8",
    "port": 6432,
    "tls_server_name": "cmp1.example",
    "route_state": 2,
    "route_version": 8
  },
  "auth": {
    "role_name": "app_user",
    "can_login": true,
    "scram_verifier": "<SCRAM verifier>",
    "role_version": 4,
    "valid_until": null
  }
}
```

**刷新与对账。** `GET /v1/proxy/routes/cmp1/primary?mode=direct` 返回主节点路由和端口 5432。若路由版本 9 仅摘除池化端口，查询 `mode=pooled` 返回下例。角色对账示例为 `GET /v1/proxy/computes/cmp1/roles/app_user`，返回上述 auth 字段及 `is_deleted`。

```json
{
  "instance_id": "cmp1",
  "node_key": "primary",
  "mode": "pooled",
  "branch_id": "br1",
  "target_ip": "198.51.100.8",
  "port": null,
  "tls_server_name": "cmp1.example",
  "route_state": 0,
  "route_version": 9
}
```

| 异常 | 处理 |
|---|---|
| 400 非法参数，401/403 服务身份或范围错误 | 调用方修正请求或凭据。access 拒绝业务角色时，Proxy 对 PG 返回统一认证失败。 |
| 404 路由未登记 | 登录返回目标不可用。只有已登记且符合第 4 章条件的主路由可触发唤醒。 |
| 409 版本过旧或同版本内容冲突 | 批操作整体回滚，写方先核对权威状态。 |
| 503 SC 或 MetaDB 不可用 | 读请求在登录时限内退避。认证过期时拒绝新登录，PRE_COMMIT 上报最终失败则中止本地事务。 |

版本冲突响应（HTTP 409）：

```json
{
  "code": "version_conflict",
  "current_version": 8
}
```

## 3. 新连接、节点选择与缓存

客户端通过同一负载均衡器地址连接，Proxy 从 PG 启动参数 `options` 提取实例和节点，完成前端 SCRAM 后以业务角色登录目标后端。

```bash
instance=cmp1
conn='host=192.0.2.10 port=5432 dbname=postgres user=app_user sslmode=verify-full sslrootcert=/etc/pg/ca.pem'
psql "$conn options='endpoint=$instance'"       # 当前主节点
psql "$conn options='endpoint=$instance-pooler'" # 当前主节点的事务池
psql "$conn options='endpoint=$instance-ro1'"   # 指定只读节点 ro1
psql "$conn options='endpoint=$instance-ro2'"   # 指定只读节点 ro2
```

IP 入口使用 `sslmode=verify-full` 时，证书须含负载均衡器 IP 的 SAN。沿用原版 `endpoint=`（旧别名 `project=`），客户端只发送一个路由标识。若 SNI 也提供 endpoint，两者须一致。后缀由适配层在规范化前解析，实例 ID 采用最多 128 位小写 ASCII 字母和数字。[endpoint 解析与 SNI](https://github.com/neondatabase/neon/blob/fa504217c61bbcaf5c512d75830564541f917f8f/proxy/src/auth/credentials.rs#L89) · [后端选项过滤](https://github.com/neondatabase/neon/blob/fa504217c61bbcaf5c512d75830564541f917f8f/proxy/src/compute/mod.rs#L380)

| endpoint 示例 | 目标 | 连接行为 |
|---|---|---|
| `cmp1` | primary/direct | 经主 VIP 连 PostgreSQL，保留会话语义 |
| `cmp1-pooler` | primary/pooled | 经主 VIP 连 PgBouncer，事务结束后后端连接归池 |
| `cmp1-ro1` 至 `cmp1-ro4` | 指定成员/direct | 只读定向适配优先级较低；连接前验证成员资格，不可用时返回该成员错误 |

```mermaid
sequenceDiagram
    participant C as PostgreSQL 客户端
    participant P as Proxy（经负载均衡器接入）
    participant S as SC
    participant B as Compute A 主节点
    C->>P: TLS 建连，发送 user、database、endpoint=cmp1
    P->>P: 解析为实例 cmp1、主节点、普通连接；查本地缓存
    opt 角色或路由缓存缺失、过期
        P->>S: 角色需刷新时查 access；仅路由需刷新时查 routes
        S-->>P: 返回所查元信息
    end
    P-->>C: SCRAM 质询
    C->>P: 提交密码证明，Proxy 校验
    P->>B: 经主 VIP，以客户端角色登录目标数据库
    B-->>P: 后端登录成功
    P-->>C: ReadyForQuery，可以执行 SQL
```

前后端都认证成功才返回 `ReadyForQuery`。数据库对象权限由 Compute 校验。池化请求只使用已发布的 `pooler_port`，不回落到普通端口。[Lakebase 池化入口](https://docs.databricks.com/aws/en/oltp/projects/connection-pooling) · [Neon Compute 配置](https://github.com/neondatabase/neon/blob/fa504217c61bbcaf5c512d75830564541f917f8f/compute/etc/pgbouncer.ini#L6)

**缓存与回源。** 每个 Proxy 进程独立缓存路由 `(instance_id, node_key, mode)` 和角色 `(instance_id, role_name)`。两者均有效时直接认证；仅路由缺失或到期时查 routes；角色缺失或到期时查 access 合取两者。

- 两类缓存各设硬 TTL，命中不续期。角色还受 valid_until 限制。过期只影响下一次登录，旧会话继续转发。当 SC 不可达时禁止沿用过期认证。按允许的撤权生效窗口设角色 TTL，包含读取可见性延迟。
- 同副本对同键合并查询，限制 SC 总并发和排队长度。仅清理匹配代次的缓存，迟到响应不得覆盖新版本。部分原控制面错误约缓存 30 秒；一期按 SC 故障时长与恢复目标设置独立负缓存 TTL，区分 404 与 503。
- 原版 NodeInfoCache 的成功项默认按 4 分钟空闲期过期，新建连接读取可续期，老会话流量不会续期。原 Redis 通知用于失效角色和访问策略。节点取址仍依赖本地缓存和建连重试。一期按请求刷新。[默认配置](https://github.com/neondatabase/neon/blob/fa504217c61bbcaf5c512d75830564541f917f8f/proxy/src/config.rs#L118) · [订阅协议](https://github.com/neondatabase/neon/blob/fa504217c61bbcaf5c512d75830564541f917f8f/proxy/src/redis/notifications.rs#L15) · [缓存及 HTTP 调用](https://github.com/neondatabase/neon/blob/fa504217c61bbcaf5c512d75830564541f917f8f/proxy/src/control_plane/client/cplane_proxy_v1.rs#L409)

## 4. Proxy 节点部署、唤醒与切换

Agent 负责本机进程和唤醒通知，Proxy 根据 SC 发布的路由尝试真实后端登录，并在请求期限内结束重试。

**部署。** Agent 挂载 TLS、SC 只读凭据和节点配置（pool_id、node_id、配置版本）。复用 Proxy CLI，拟增加 `sc` 后端及以下三个环境变量。VM 由 Agent 退避重启进程，容器部署须与平台约定唯一的重启责任方。

```bash
export PROXY_SC_URL=https://sc.example:8443
export PROXY_SC_TOKEN_FILE=/run/secrets/proxy-sc-token
export PROXY_WAKE_AGENT_SOCKET=/run/proxy-agent/wake.sock
exec /opt/neon/bin/proxy --auth-backend sc --proxy 0.0.0.0:5432 \
  --http 127.0.0.1:7001 --tls-cert /run/tls/server.crt --tls-key /run/tls/server.key
```

节点先通过已有 `GET /v1/status`，再用第 3 章连接串直指待注册节点，执行 `SELECT 1`。两项通过后，管控将节点地址和 PG 端口注册到负载均衡后端组。以下为服务化接口示例，路径和字段由部署平台定义。[Proxy 状态与指标](https://github.com/neondatabase/neon/blob/fa504217c61bbcaf5c512d75830564541f917f8f/proxy/src/http/health_server.rs#L19)

```http
POST /load-balancer/pools/example-pool/backends HTTP/1.1
Content-Type: application/json
Authorization: Bearer <service-token>

{
  "address": "192.0.2.11",
  "port": 5432
}
```

**激活。** 原 Neon 在节点缓存未命中等情况下调用控制面 `wake_compute` 取址，调用前不查询 HA 或休眠状态。本方案由 SC 记录路由发布状态（0 暂不可用、2 可连接、5 退役），管控判断实例是否需要启动。原错误响应中的 `retry_info.retry_delay_ms` 是建议等待毫秒数，SC 无需提供唤醒 API。[wake 请求和响应处理](https://github.com/neondatabase/neon/blob/fa504217c61bbcaf5c512d75830564541f917f8f/proxy/src/control_plane/client/cplane_proxy_v1.rs#L272)

| 已认证连接的观测结果 | Proxy 行为 |
|---|---|
| 主路由已登记但暂不可用 | 通过agent使用管控的通知机制，触发wake_compute |
| 主路由已发布，后端可重试故障 | 合并路由刷新后重试主 VIP，仍失败可通知 Agent。仅池化入口故障且普通入口可用时，按 PgBouncer 故障处理。 |
| 指定只读成员不可用 | 只刷新该成员并校验其只读资格。实例退役、认证拒绝、SC 无法提供必要元信息时结束登录。 |

```json
{
  "type": "wake_compute",
  "request_id": "wake-104",
  "instance_id": "cmp1",
  "observed_route_version": 8,
  "expires_at": "2026-11-19T09:01:00Z"
}
```

Agent 持久入队后可返回 202，向管控单向投递。重投沿用 request_id，管控按实例合并并检查 expires_at。Proxy 只重查 SC 并连接后端，同一副本的同实例共享通知，各登录独立计时。联调起点为剩余登录预算与 60 秒中的较短者，重试间隔从 0.5 秒退避至 5 秒并加抖动。VIP 未变也可重试。成功以完成指定入口的后端登录为准，超时返回暂不可用并记录一次最终结果。全部等待者退出后停止刷新。

**HA 与只读节点。** 主 VIP 漂移后，新连接沿用同一客户端入口。已断会话由应用或连接池重连，未确认的提交结果由应用核实。主节点提升时先摘除旧池化端口，待新 PgBouncer 指向新主并通过事务测试后再发布。

| 无只读 VIP 的场景 | 新连接如何选节点 | 成员升主时 |
|---|---|---|
| 客户端直连 | 管控提供成员列表，支持多主机参数的 libpq 按下例筛选物理备库 | 新读连接跳过新主，管控或应用连接池回收旧读会话 |
| 经 Proxy 定向 | `endpoint=cmp1-ro1` 只选 ro1，组级只读入口见第 8 章 | 管控撤下 ro1 路由并处理旧读会话，替代成员就绪后显式注册新版本 |

```bash
psql "host=pg-a.example,pg-b.example,pg-c.example port=5432 dbname=app user=reader target_session_attrs=standby load_balance_hosts=random connect_timeout=3 sslmode=verify-full"
```

直连示例要求证书覆盖各成员主机名。Lakebase 的组级 `-ro` 与排障用成员直连串用途不同，成员变动或提升时直连串可能变化。其 HA 成员不支持空闲自动 scale to zero，整组可手动暂停。[libpq 连接参数](https://www.postgresql.org/docs/current/libpq-connect.html) · [Lakebase HA](https://docs.databricks.com/aws/en/oltp/projects/high-availability)

### 4.1 SQL over HTTP 的入口与部署

应用可向共享 Proxy 发送 `POST /sql` 执行参数化 SQL，Proxy 校验业务角色、通过 PostgreSQL 协议访问 Compute，再将结果编码为 JSON。

| 调用方 | 原版监听参数与接口 | 服务化部署建议 |
|---|---|---|
| psql、普通 PG 驱动 | `--proxy`，PG/TLS 长连接 | 沿用负载均衡器 TCP 5432 → Proxy 节点 5432 |
| HTTP 客户端、Neon serverless driver | `--wss`，`POST /sql` 执行查询，`OPTIONS /sql` 处理浏览器预检；同一监听也可接受 WebSocket 升级 | 同一负载均衡器增设 TCP 443 → 两个 Proxy 节点的 8443，TLS 透传到 Proxy |
| 本机 Agent、采集器 | `--http`，`GET /v1/status`、`GET /metrics` | 绑定 `127.0.0.1:7001`，仅在节点内访问 |

**进程部署。** 本地源码中，普通控制面后端模式的同一个 `proxy` 进程可以同时运行 PG 和 serverless 监听，`--wss` 需显式指定。建议沿用两个 Proxy 节点和各自的 Agent，在本章启动命令追加 `--wss 0.0.0.0:8443`，挂载含业务 HTTPS 入口名称或 IP SAN 的证书。将两个节点的 8443 注册到 443 监听对应的后端组。若负载均衡器发送 PROXY protocol v2，须同步配置 Proxy 并检查 PG 和 HTTP 两个监听的兼容性。[监听与任务启动](https://github.com/neondatabase/neon/blob/fa504217c61bbcaf5c512d75830564541f917f8f/proxy/src/binary/proxy.rs#L375) · [业务 HTTP 路由](https://github.com/neondatabase/neon/blob/fa504217c61bbcaf5c512d75830564541f917f8f/proxy/src/serverless/mod.rs#L398) · [管理接口](https://github.com/neondatabase/neon/blob/fa504217c61bbcaf5c512d75830564541f917f8f/proxy/src/http/health_server.rs#L23)

**密码模式。** HTTPS URL 决定请求发往哪个 Proxy，`Neon-Connection-String` 请求头决定访问哪个数据库。原版从头内 PG URI 的主机名首段提取 endpoint，从 URI 取角色、密码和数据库。Proxy 对每次请求校验密码与角色 verifier，随后取得或建立该业务角色的后端 PG 连接。单条查询和同一请求内的事务批次均可走 HTTP；跨请求交互式事务使用 PG 或 WebSocket。[连接信息解析](https://github.com/neondatabase/neon/blob/fa504217c61bbcaf5c512d75830564541f917f8f/proxy/src/serverless/http_util.rs#L142) · [密码校验与后端建连](https://github.com/neondatabase/neon/blob/fa504217c61bbcaf5c512d75830564541f917f8f/proxy/src/serverless/backend.rs#L49) · [Neon 驱动使用说明](https://neon.com/docs/serverless/serverless-driver)

**适配 IP 入口。** 可直接请求 `https://192.0.2.10/sql`，头内仍使用 `cmp1.db.example` 这类逻辑主机名。Proxy 从中提取 `cmp1`，经 SC 取真实后端地址，该逻辑名称无需为此请求解析到 Compute。HTTP 解析器没有用 URI 的 `options=endpoint=cmp1` 覆盖主机名前缀的逻辑；若要求头内也使用裸 IP，需新增解析与冲突校验。建议先沿用原版头格式，并复用第 3 章的 endpoint 后缀规则、角色校验和第 4 章的限时唤醒。仅支持密码接入时可复用现有 SC 表及 API。

以下示例使用负载均衡器测试地址和主节点普通入口，替换地址、CA 与测试角色即可联调。请求头中的密码须按 URI 规则编码，代理和负载均衡器日志应脱敏。

```bash
curl --fail-with-body --cacert /etc/pg/ca.pem 'https://192.0.2.10/sql' \
  -H 'Content-Type: application/sql' \
  -H 'Neon-Connection-String: postgresql://app_user:XXXXXXXXXX@cmp1.db.example/postgres' \
  --data-binary @- <<'JSON'
{
  "query": "SELECT $1::int AS answer",
  "params": [
    42
  ]
}
JSON
```

验收时检查 HTTP 200，响应的 `rows[0].answer` 为 42；再分别用错误密码、另一实例同名角色及休眠实例验证拒绝、隔离和唤醒。原版 HTTP 后端连接复用默认由请求头 `Neon-Pool-Opt-In: true` 开启，连接池位于 Proxy 进程内，Compute 侧 PgBouncer 可独立部署。若开启 HTTP 池，须按实例、目标成员、模式、角色和数据库隔离复用，并验证路由切换后旧连接能被淘汰。[仓库 HTTP 用例](https://github.com/neondatabase/neon/blob/fa504217c61bbcaf5c512d75830564541f917f8f/test_runner/regress/test_proxy.py#L227) · [HTTP 池与认证顺序](https://github.com/neondatabase/neon/blob/fa504217c61bbcaf5c512d75830564541f917f8f/proxy/src/serverless/sql_over_http.rs#L470)

**JWT 部署差异。** 原版 `--is-auth-broker=true` 要求配置 `--wss`，该进程关闭常规 PG 监听。它验证 `Authorization: Bearer` JWT，按 endpoint 取得 JWKS 规则，再把请求转给 Compute 侧的 `local_proxy` 访问本地 PostgreSQL。若要同时保留现有 PG/密码服务，JWT broker 应独立运行，另行对接 JWKS、角色授权规则和 Compute 侧组件。普通 HTTP 密码部署只需共享 Proxy 到 Compute 的 PG 链路。[broker 转发](https://github.com/neondatabase/neon/blob/fa504217c61bbcaf5c512d75830564541f917f8f/proxy/src/serverless/sql_over_http.rs#L627) · [JWKS 规则](https://github.com/neondatabase/neon/blob/fa504217c61bbcaf5c512d75830564541f917f8f/proxy/src/control_plane/client/cplane_proxy_v1.rs#L217)

## 5. Compute 角色同步与管控对接

角色变更统一由 Compute 写 SC，管控负责路由发布、网页配置任务和实例激活。

**Compute 改造。** SQL 的 `CREATE/ALTER/DROP ROLE` 与网页的 `POST /configure` 共用 PRE_COMMIT hook。原 configure 会话设置 `neon.forward_ddl=false`，需为受管理角色启用捕获并排除内部账号。例：`ALTER ROLE app_user NOLOGIN` 产生第 2 章角色 PATCH，set 中 `can_login=false`。覆盖改密、清空密码、有效期、改名和删除，发送目录内实际 verifier。普通 PostgreSQL 需等价 hook。[配置会话禁用回传](https://github.com/neondatabase/neon/blob/fa504217c61bbcaf5c512d75830564541f917f8f/compute_tools/src/compute.rs) · [DDL 回调](https://github.com/neondatabase/neon/blob/fa504217c61bbcaf5c512d75830564541f917f8f/pgxn/neon/neon_ddl_handler.c)

角色 PATCH 整批取得 200 后 PG 才继续提交，最终失败则整笔事务回滚。新建或克隆实例先全量同步角色再接收增量，休眠期间 SC 保留角色目录。下面复用原 `POST /configure` 对完整 spec 中已有的角色改密。[原版请求结构](https://github.com/neondatabase/neon/blob/fa504217c61bbcaf5c512d75830564541f917f8f/libs/compute_api/src/requests.rs#L65)

```bash
jq --arg verifier "$SCRAM_VERIFIER" \
  '(.spec.cluster.roles[] | select(.name == "app_user").encrypted_password) = $verifier' \
  configure-request.json | curl -fsS -X POST "$COMPUTE_URL/configure" \
  -H "Authorization: Bearer $COMPUTE_TOKEN" -H 'Content-Type: application/json' --data-binary @-
```

### 5.1 管控对接清单

管控串起新实例注册、配置任务和唤醒通知，按以下完成条件发布入口。

| 任务 | 接口及完成条件 |
|---|---|
| 创建和切换 | 注册 Proxy 节点到负载均衡后端组，分配实例 ID，用路由 PUT 发布主 VIP 和各成员。验证普通入口及池化端口后分别开放。 |
| 网页改密 | 按实例协调完整 spec 的下发，保存 task_id，同一角色的任务有序重试。后一次改密生效后，前一次迟到任务在管控侧结束。 |
| 休眠期间的 configure | 持久化待执行任务。唤醒后补发 configure 并确认角色同步，再发布就绪路由。尚未创建的账号由管控或已有授权身份触发唤醒。 |
| Agent 单向上报 | 管控判断运行状态、HA 和暂停策略，重复通知只执行一次激活。通道协议、任务持久化和重试能力待联调。 |

## 6. 一期新增功能的监控与告警

本期按一次新登录、一次激活通知或一批角色变更计量，区分用户可见的最终失败和内部重试。

| 新增功能 | 指标（拟定） | 告警条件与定位 |
|---|---|---|
| SC 接口与缓存回源 | 按 access/routes 统计错误率、P99，按缓存类型统计回源数、合并等待数及超预算拒绝数 | 多副本持续出现 5xx 或排队拒绝且新登录失败时，检查 SC 接口及 MetaDB 读取延迟。单独统计业务认证拒绝。 |
| 后端选路 | `proxy_backend_attempts_total{stage,result,mode}` 区分 TCP、TLS、后端认证及普通/池化入口 | 新连接连续在同一阶段失败时告警。仅 pooled 失败时检查池化端口发布和 PgBouncer。 |
| Agent 唤醒通知 | `agent_wake_reports_total{result}`、队列深度、最老待发时长 | 连续投递失败或排队接近事件有效期时通知 Agent 负责人。request_id 可关联管控接收日志。 |
| Proxy 激活等待 | `proxy_wake_logins_total{result,last_stage,mode}`、`proxy_wake_wait_seconds`、`proxy_wake_waiters` | 每条等待登录只记录一个最终结果。按池统计最终超时数、占比和最后失败阶段。 |
| Compute 角色同步 | PRE_COMMIT 上报失败数、PATCH 延迟、409 冲突数、对账差异数 | 持续阻塞角色 DDL 或对账差异未收敛时，关联 Compute 提交结果和 SC 批次日志。 |

复用已有 `/metrics`。抓取时补充 pool_id；实例 ID 和 request_id 只进受控日志，日志排除密码、verifier 和完整连接串。

```bash
curl -fsS http://127.0.0.1:7001/metrics
curl -fsS "$SC_URL/metrics"
```

**联调告警示例。** 同一池 5 分钟内至少 5 条登录最终超时，且占已结束唤醒等待的 10% 时告警，分母排除客户端提前断开。低流量时，同一实例连续 3 次超时用日志规则补充。连续两个窗口恢复且有成功样本时解除，生产阈值按基线校准。

```promql
(
  sum by (pool_id) (increase(proxy_wake_logins_total{result="timeout"}[5m])) >= 5
)
and on (pool_id)
(
  sum by (pool_id) (increase(proxy_wake_logins_total{result="timeout"}[5m]))
  / sum by (pool_id) (increase(proxy_wake_logins_total{result!="client_disconnected"}[5m])) >= 0.10
)
```

## 7. QA 与故障验收

用下列故障验证角色一致性、认证生效窗口和新连接的恢复边界。

| 问题或注入故障 | 预期行为与验收点 |
|---|---|
| SC 已写入，PATCH 响应丢失怎么办？ | Compute 重投同一 batch_id、角色版本和内容。SC 对同版本同内容返回 200，冲突返回 409，整批原子处理。用户重新执行 SQL 属于新事务。 |
| SC 已返回 200，PG 随后回滚或宕机怎么办？ | 存在 SC 短暂公开未提交认证信息的窗口。以当前可写 Compute **已提交**的角色目录为准对账，用新版本修复 SC。HA 切换后拒绝旧主写入。 |
| 网页改密超时，或第一次重试迟于第二次怎么办？ | 管控按任务顺序核对已提交状态。重放相同 spec 可能跳过 ALTER ROLE，须核对 SC 后再结束任务。验收 PG 与 SC 均保留最后一次生效密码。[角色差异检查](https://github.com/neondatabase/neon/blob/fa504217c61bbcaf5c512d75830564541f917f8f/compute_tools/src/spec_apply.rs#L797) |
| 两个 Proxy 缓存仍热，用户已改密或撤权？ | 测量 PG 提交到两副本实际采用新版本的时长。硬 TTL 不因命中而延长，过期且 SC 不可用时拒绝新认证。已有会话独立处理。 |
| MetaDB 或 SC 中断，随后大量客户端重连？ | 冷缓存和过期项影响新连接，角色上报失败影响 DDL。验证 MetaDB 主备切换、已确认写入可见、备份恢复。同一键单轮回源每副本至多一组；无新登录时缓存到期不产生回源、探活或唤醒。 |
| Agent 停止、通知丢失或重复？ | Proxy 在最终时限内结束，每次登录只计一个结果，告警可触发并解除。管控对重复事件只实际激活一次，已运行实例无需重启。 |
| VIP 倒换或 ro1 升主？ | 应用新连接恢复到新主，旧事务结果另行核实。指定 ro1 的新连接不得误入新主，ro2 仍可连接；原 ro1 读会话按 HA 流程回收。 |
| PgBouncer 单独停止？ | 普通入口保持可用，池化入口独立返回故障。验证恢复发布、池化连接改密生效，以及空闲池对休眠判定的影响。 |

## 8. 其他计划功能点

以下条目列出后续可扩展的能力及设计时需要明确的问题。

| 功能点 | 概要 | 待细化 |
|---|---|---|
| 缓存批量版本检查 | 缩短各 Proxy 副本发现路由或角色变更的时间。 | 检查哪些缓存项、允许多长的发现延迟、SC 可承受多少额外负载？ |
| HA 组级只读入口 | 应用通过统一入口连接可用的只读成员。 | 哪些成员可接流量、新连接如何分配、成员升主后如何处理已有读会话？ |
| 只读连接池化 | 为只读访问提供事务池入口。 | 按成员还是按 HA 组提供入口、如何分配连接额度、成员退出时如何排空连接池？ |
| 临时 Compute（可选） | 按需创建和释放运行实例。 | 哪些场景需要临时实例、何时可以接入、实例替换后如何隔离旧路由和旧写方？ |
