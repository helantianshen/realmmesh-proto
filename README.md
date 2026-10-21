# RealmMesh 协议与项目文档

`realmmesh-proto` 是 RealmMesh RPC 协议的唯一源仓库。目前包含需求、开发规划和最小 Echo 协议。第一阶段的 Go/NeoForge Echo 通信交付已于 2026-10-19 按用户确认关闭；详细状态见开发计划。

## 最小通信协议

账户变更通知草案见 [账户变更通知 RPC v1](auth/v1/ACCOUNT_EVENTS.md)，使用认证服务的服务端流广播改密和账户删除事件。本次新增协议待审查，尚未接入消费者。

游戏服配置拉取协议草案见 [config/v1/config.proto](config/v1/config.proto)，字段、错误及启动和轮询约定见 [游戏服配置拉取 RPC v1](config/v1/README.md)。本协议待审查，尚未接入中心或 Mod。

玩家认证协议草案见 [auth/v1/auth.proto](auth/v1/auth.proto)，字段语义、结果组合和错误边界见 [玩家认证 RPC v1](auth/v1/README.md)。当前待审查，尚未接入中心或 Mod，消费者协议引用未更新。

`probe/v1/probe.proto` 定义 Unary RPC `realmmesh.probe.v1.ProbeService/Echo`，请求和响应各包含编号为 1 的字符串字段 `message`，约定原样返回，包括空字符串。该接口仅用于通信验证，不处理账户、认证或资产业务。

Go 生成包为 `github.com/helantianshen/realmmesh-server/gen/probe/v1`，包名为 `probev1`。生成代码保存在消费者仓库并纳入 Git，不放入本仓库。Go 生成工具固定为 `protoc 36.1`、`protoc-gen-go v1.36.11` 和 `protoc-gen-go-grpc v1.6.2`；生成命令由服务仓库维护，输出采用 `paths=source_relative`。

协议编译与生成代码构建不代表运行时互通；Go handler、客户端调用及 Minecraft 内的 Java 调用仍需分别验证。

| 文档 | 用途 | 状态 |
| --- | --- | --- |
| [需求文档](REQUIREMENTS.md) | 系统目标、角色、通信、资产与信任边界、待定事项 | 需求草案；不代表全部技术与业务细节已冻结 |
| [仓库组织与版本管理](REPOSITORIES.md) | 四仓库职责、协议共享与手动版本流程 | 用户已确认的组织方案 |
| [认证流程与协议职责](AUTHENTICATION.md) | 统一玩家 ID、中心认证与休息规则、服务器管理、审计和 RPC 边界 | 已确认方向；实现细节与验收待完成 |
| [分阶段开发计划](DEVELOPMENT_PLAN.md) | 阶段范围、依赖、验收与审查事项 | 第一阶段已关闭；后续阶段待确认实施 |

文档整理日期：2026-10-18。文档中的未来步骤不代表执行授权；具体协议、模型、目录及框架在对应阶段确认后再创建。
