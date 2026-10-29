# Neon Proxy 认证：原生 PostgreSQL、HTTP 密码与 JWT

Neon Proxy 在原生 PostgreSQL Direct 连接中分别扮演服务端和客户端：先验证前端用户，再以该用户角色登录后端。两段 SCRAM 认证独立进行，但可以复用密钥，无需获取用户明文密码。

本文依据 Neon 开源提交 [`fa504217`](https://github.com/neondatabase/neon/tree/fa504217c61bbcaf5c512d75830564541f917f8f)，整理于 2026-10-29。先说明原生 PG 双段 SCRAM，再补充 HTTP 密码和 JWT 路径；不涵盖 console redirect。

## 角色与凭据

| 连接 | Proxy 的角色 | 认证方式 |
|---|---|---|
| 客户端 → Proxy | PostgreSQL 服务端 | SCRAM-SHA-256，或支持 channel binding 的 SCRAM-SHA-256-PLUS |
| Proxy → PostgreSQL | PostgreSQL 客户端 | 使用前端认证得到的密钥，完成新的 SCRAM 认证 |
| Proxy → 控制面 | 内部 API 客户端 | 独立的服务 Bearer Token，与数据库用户密码无关 |

## 原生 PG：核心机制

控制面向 Proxy 提供访问策略和角色的 SCRAM verifier，结构为：

```text
SCRAM-SHA-256$iterations:salt$StoredKey:ServerKey
```

1. **前端认证**：Proxy 识别 endpoint、role 和 database，获取策略与 verifier，检查访问来源并与客户端完成 SCRAM 交互。
2. **得到密钥**：Proxy 从客户端的 ClientProof 恢复并验证 ClientKey，再结合 verifier 中的 ServerKey，形成后端认证凭据。
3. **后端认证**：Proxy 获取或唤醒 Compute，以用户角色发送 PG 启动消息；使用 ClientKey 和 ServerKey，针对后端的新挑战完成 SCRAM。

这不是转发前端认证消息，也不是重放 ClientProof。Neon 修改的 PostgreSQL 客户端库支持直接使用密钥进行认证。

## 认证时序

```mermaid
sequenceDiagram
    participant C as PG客户端
    participant P as Proxy
    participant CP as 控制面
    participant DB as PostgreSQL

    C->>P: TLS握手与StartupMessage
    P->>P: 识别endpoint、role和database
    P->>CP: 获取认证材料和访问策略
    CP-->>P: SCRAM verifier及策略
    P->>P: 访问检查与限流

    P-->>C: AuthenticationSASL
    C->>P: client-first
    P-->>C: server-first，salt及iterations
    C->>P: client-final，ClientProof
    P->>P: 验证证明，得到ClientKey和ServerKey
    P-->>C: server-final及AuthenticationOk

    P->>CP: wake_compute
    CP-->>P: 后端地址及TLS信息
    P->>DB: 建连，按配置协商TLS
    P->>DB: StartupMessage
    DB-->>P: 新的SCRAM挑战
    P->>DB: 使用密钥完成新的SCRAM交互
    DB-->>P: 服务端证明、认证成功及启动响应
    P-->>C: 启动参数、代理取消标识、ReadyForQuery
```

控制面请求可以命中 Proxy 缓存；目标 Compute 休眠时，由控制面协调启动。客户端收到 AuthenticationOk 仅表示前端认证通过，收到 ReadyForQuery 才表示后端会话也已就绪。

## HTTP：密码认证

SQL over HTTP 使用 `POST /sql`，通过 `Neon-Connection-String` 请求头提供 endpoint、role、database 和密码，例如：

```http
POST /sql
Neon-Connection-String: postgresql://app_user:YOUR_PASSWORD@ep-example.example.com/appdb
Content-Type: application/json

{"query":"SELECT $1::int AS value","params":[42]}
```

请求应通过 HTTPS 发送。这里不进行客户端与 Proxy 之间的 SCRAM 消息往返：Proxy 会接收到密码，使用控制面提供的 SCRAM verifier 在本地验证并派生密钥。随后复用连接池中的后端连接，或使用密钥新建后端 SCRAM 连接。HTTP 请求会进行认证，复用后端连接不等于跳过前端认证。

## HTTP：JWT 认证与后端身份传递

JWT 模式使用 `Authorization: Bearer <JWT>`；连接串仍指定 endpoint、role、database，但不携带数据库密码：

```http
POST /sql
Authorization: Bearer YOUR_JWT
Neon-Connection-String: postgresql://authenticated@ep-example.example.com/appdb
Content-Type: application/json

{"query":"SELECT current_user","params":[]}
```

该版本由 `accept_jwts` 配置选择 HTTP 凭据模式，不应假设同一入口会自动兼容密码和 JWT。

典型的 auth broker → local_proxy 路径如下：

```mermaid
sequenceDiagram
    participant C as HTTP客户端
    participant A as 独立Proxy / auth broker
    participant J as 控制面与JWKS服务
    participant L as Compute内local_proxy
    participant DB as 本机PostgreSQL

    C->>A: HTTPS POST /sql，Bearer JWT及连接信息
    A->>J: 按需获取角色规则与JWKS公钥
    J-->>A: 规则及公钥，可缓存
    A->>A: 验证JWT
    A->>A: 发现或唤醒目标Compute
    A->>L: 转发原Bearer JWT、连接信息及SQL
    L->>L: 按本地规则和JWKS再次验证JWT
    L->>DB: 复用或新建本机trust连接
    L->>L: 使用连接专属私钥重签已验证的claims
    L->>DB: auth.jwt_session_init，设置本次身份
    L->>DB: 执行SQL
    DB-->>C: 经local_proxy及独立Proxy返回JSON
```

- **前端校验**：按 endpoint 和请求 role 获取允许的 JWKS 规则，使用 `kid` 选择密钥、验证签名；配置了 audience 时检查 `aud`，存在 `exp`、`nbf` 时检查时间，容忍 30 秒时钟偏差。此实现会记录 `iss`，但不做独立的 issuer 值匹配校验。
- **角色选择**：数据库角色来自连接信息，JWKS 规则限制该角色可使用哪些密钥；不应理解为直接把 JWT 的 `sub` 当作数据库角色。
- **后端身份**：JWT 不会转换为 SCRAM 密码。本机 PostgreSQL 使用受保护的 localhost trust 路径；`local_proxy` 通过 `pg_session_jwt` 将已验证的 claims 绑定到当前请求，供数据库函数和 RLS 策略使用。
- **连接初始化**：`local_proxy` 按需调用 `compute_ctl` 安装扩展、授予 schema 使用权限；每个新后端连接配置专属公钥并执行 `auth.init()`。每次请求再重签 claims、更新 `jti` 并调用 `auth.jwt_session_init()`，避免复用连接时沿用上一请求身份。

图示依赖 auth broker、local_proxy 和数据库扩展的配套配置，不代表所有 HTTP 请求都经过 local_proxy，也不等同于 Databricks Lakebase OAuth 的完整实现。

## 三种路径对照

| 路径 | 前端凭据 | Proxy 是否接收用户明文密码 | 后端身份机制 |
|---|---|---|---|
| 原生 PG Direct | SCRAM 证明 | 否，限本文的 SCRAM 路径 | 派生密钥完成新 SCRAM |
| HTTP 密码 | HTTPS 内的数据库密码 | 是 | 校验派生密钥，复用或新建 PG 连接 |
| HTTP JWT 配套路径 | Bearer JWT | 不需要数据库密码 | JWT 验证、本机 trust 连接及 claims 会话绑定 |

## 自建服务的关键约束

- **verifier 一致**：控制面与后端角色必须使用匹配的 SCRAM verifier。仅分别设置相同明文密码不够，重新设置密码可能生成不同的 salt。
- **密码更新协调**：同步数据库角色、控制面认证材料与 Proxy 缓存失效，避免新旧状态不一致。
- **后端认证匹配**：此密钥复用路径针对 SCRAM，不能将密钥当作通用明文密码用于其他认证方式。
- **保护认证边界**：Proxy 持有敏感 verifier 和认证密钥；前后端 TLS 分别协商，前端 channel binding 不会原样延伸到后端。
- **权限由数据库执行**：后端以用户角色登录，表、schema 等 SQL 权限仍由 PostgreSQL 检查。前端认证成功不能替代后端认证和授权。
- **JWT 配套边界**：保护本机 trust 入口；配置允许的角色、密钥和 audience，并明确 token 时效与 issuer 策略。自建数据库仍需配置 SQL 权限和 RLS，JWT 签名有效本身不代表拥有所有数据权限。

## 源码参考

- [前端认证流程](https://github.com/neondatabase/neon/blob/fa504217c61bbcaf5c512d75830564541f917f8f/proxy/src/auth/flow.rs)
- [SCRAM 证明验证与 ClientKey 恢复](https://github.com/neondatabase/neon/blob/fa504217c61bbcaf5c512d75830564541f917f8f/proxy/src/scram/exchange.rs)
- [后端密钥封装](https://github.com/neondatabase/neon/blob/fa504217c61bbcaf5c512d75830564541f917f8f/proxy/src/auth/backend/classic.rs)
- [后端认证配置](https://github.com/neondatabase/neon/blob/fa504217c61bbcaf5c512d75830564541f917f8f/proxy/src/compute/mod.rs)
- [支持密钥认证的客户端实现](https://github.com/neondatabase/neon/blob/fa504217c61bbcaf5c512d75830564541f917f8f/libs/proxy/postgres-protocol2/src/authentication/sasl.rs)
- [HTTP 凭据解析与模式选择](https://github.com/neondatabase/neon/blob/fa504217c61bbcaf5c512d75830564541f917f8f/proxy/src/serverless/http_util.rs)
- [JWT/JWKS 校验](https://github.com/neondatabase/neon/blob/fa504217c61bbcaf5c512d75830564541f917f8f/proxy/src/auth/backend/jwt.rs)
- [HTTP 认证、连接与扩展初始化](https://github.com/neondatabase/neon/blob/fa504217c61bbcaf5c512d75830564541f917f8f/proxy/src/serverless/backend.rs)
- [auth broker 转发与 SQL 请求处理](https://github.com/neondatabase/neon/blob/fa504217c61bbcaf5c512d75830564541f917f8f/proxy/src/serverless/sql_over_http.rs)
- [JWT 重签与请求身份绑定](https://github.com/neondatabase/neon/blob/fa504217c61bbcaf5c512d75830564541f917f8f/proxy/src/serverless/local_conn_pool.rs)
