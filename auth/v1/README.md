# 玩家认证 RPC v1

自助改密的请求、共享失败次数、响应与踢出约束见 [玩家自助改密 RPC v1](CHANGE_PASSWORD.md)。该扩展单独审查，不改变下述注册登录字段。

本契约为待审查定义，尚未接入中心 Handler 或 Mod。协议源为 [auth.proto](auth.proto)，包名 `realmmesh.auth.v1`，Go 包为 `gen/auth/v1`（`authv1`）。Java 沿用协议命名空间及默认生成规则。

## 接口与身份

| 完整方法 | 请求 | 成功响应 |
| --- | --- | --- |
| /realmmesh.auth.v1.AuthService/Register | player_name、password | user_id、player_name |
| /realmmesh.auth.v1.AuthService/Login | player_name、password | LoginResponse |

两者均为 Unary RPC。每次请求通过 metadata 携带单个 `x-realmmesh-server-id` 和 `authorization: Bearer <server_secret>`，由中心验证当前服务器状态。消息不重复传服务器身份，不携带 Minecraft UUID、JWT、玩家会话票据或管理员权限。

游戏服必须从实际玩家连接取得名字，不能允许客户端提交另一个操作主体。中心信任已登记游戏服报告的名字，并负责密码校验。请求和密码不得进入日志；当前明文 gRPC 不提供链路加密。

## 注册

名字限定 3–16 位 ASCII 字母、数字或下划线，保存原始大小写，忽略大小写唯一；逻辑删除后的名字仍永久占用。密码为 6–12 位英文字母、数字或 `$%&@#^*!`，大小写敏感，不要求混合字符种类，不自动去除空白。

注册只创建普通账户。响应的 user_id 是中心数据库正数雪花主键；成功响应表示账户已持久化。游戏服将发起请求的本次连接标记为已登录，不需要再发 Login，不建立中心在线会话。密码确认由游戏命令入口处理，不重复传给中心。

注册失败不计入登录失败次数。并发同名注册仅能有一次成功，不覆盖已有账户密码。

## 登录结果与字段约束

| status | user_id / player_name | remaining_attempts | cooldown_until | retry_after_seconds | kick |
| --- | --- | --- | --- | --- | --- |
| SUCCEEDED | 当前账户身份 | 清零后的失败阈值 | 不设置 | 0 | false |
| UNREGISTERED | 0 / 空字符串 | 0 | 不设置 | 0 | false |
| WRONG_PASSWORD | 当前账户身份 | 大于 0 | 不设置 | 0 | false |
| COOLING_DOWN | 当前账户身份 | 0 | 原截止时刻 | 大于 0 | true |

表中枚举省略 `LOGIN_STATUS_` 前缀。中心不发送 UNSPECIFIED；客户端收到零值、未知枚举或不符合约束的响应时，不能标记登录成功。身份字段本身不证明认证成功。

失败阈值 N 表示第 N 次失败立即进入休息期，不是第 N+1 次。默认阈值为 3，休息 300 秒，具体值由中心数据库配置决定。所有游戏服及同名账户的不同大小写共用计数。

休息期判断先于密码校验。休息期间即使密码正确也返回 COOLING_DOWN；每次请求都返回 kick=true，且不延长截止时间。允许玩家重新进入游戏服，但重新提交登录仍应被踢出。不存在账户不计数；已有非休息账户提交格式错误密码也计数。到期从零计数，成功登录清零。

`cooldown_until` 使用标准 Timestamp；非休息状态必须不设置，而不是设置为零时刻。`retry_after_seconds` 是中心作出本次判断时向上取整的剩余秒数，用于提示。游戏服不使用本机时间提前放行，后续登录仍以中心结果为准。

## gRPC 错误边界

| 情况 | gRPC 状态 |
| --- | --- |
| 非法玩家名字；注册密码不符合规则 | InvalidArgument |
| 注册名字已占用，包括已逻辑删除的账户 | AlreadyExists |
| 服务器凭据缺失、无效、停用、删除或已轮换 | Unauthenticated |
| 中心依赖暂时不可用 | Unavailable |
| 内部处理失败、配置异常或存储的密码哈希损坏 | Internal |
| 请求取消或超时 | Canceled / DeadlineExceeded |

Login 的未注册、密码错误及休息结果使用 gRPC OK 和 LoginResponse 表达，不使用 Unauthenticated，以免与服务器身份失败混淆。登录密码格式错误不能在业务计数前映射成 InvalidArgument。

错误文案必须为安全摘要，不返回原始密码、密钥、哈希或数据库驱动详情。认证失败的游戏服不能进入注册登录业务，不影响玩家失败计数。

## 连接、重试与版本

- 注册或登录的异步响应只能作用于发起请求的同一次玩家连接；玩家退出后重连，旧响应不能使新连接登录或被踢出。
- 不自动重放注册或登录请求。超时或断线可能发生在中心事务提交之后，不能解释为操作确定未执行；重复 Login 可能额外计数。
- 字段编号和枚举数字属于兼容性契约，删除后应保留为 reserved，不重新分配给其他含义。
- 本步仅定义契约。中心实现、Java 调用、实际自动登录与 kick 均需后续分别验证；生成成功不等于运行时互通。
